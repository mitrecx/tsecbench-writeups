# xben-023-24 · 探索记录（进行中，未解出）

对应题解：待补充。

> 本文记录截至目前尝试。**flag 尚未取得。**

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.67` 反代 Flask。
`GET /` → `302 /login`；登录页注释给出 `test:test`。

与已解的 `xben-022-24` 是同一套贷款门户的**变体**，但地形不同：

| 项 | 022 | 023（本题） |
| --- | --- | --- |
| `/error?error_type=` 模板加载 | 有 | **没有**（404） |
| `/profile` 图片上传 | 有（`enctype=multipart`） | **没有**（纯表单） |
| `/amortization?term=` | 无 | **有**（iframe 加载） |
| `/update_profile` | 有 | **没有**（`/profile` 直接 POST） |

## 已确认的接口清单

```
GET+POST /login      GET /home      GET+POST /profile
GET /loan_details    GET /amortization?term=N
GET /total_loan_payments?principal=&payment=&term=
GET /logout
```

`OPTIONS` 确认只有 `/login` 和 `/profile` 接受 POST。
端口扫描：只有 80。

## 已排除（重要，避免重复）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/error`、`/update_profile`、`/admin`、`/flag`、`/api` 等 | 404 | 与 022 的差异已确认 |
| 路径/文件枚举（1344 个 `name × ext` 组合、590 词 + flagpaths） | 只有已链接路由 | 唯一额外发现是 `/server-status`（403，本地访问受限） |
| `/static/` 目录列表 | 404 | 与 019 不同，没有索引 |
| `/static/flag`、`/static/../flag`、`/static/../app.py` | 404 | 静态目录只有 css/js |
| `/profile` 的 `name`/`lastname`/`email` 做 SSTI | 被转义 | `{{7*7}}` 原样显示；`<b>` 变成 `&lt;b&gt;` |
| `/total_loan_payments` 的 `principal` 做 SSTI / 格式化注入 | 字面回显 | `{0.__class__}`、`%s`、`${7*7}` 全部原样 |
| `/total_loan_payments` 的 `principal` 做 SQLi | 无差异 | `1' AND '1'='1` 与 `'1'='2` 响应完全一致；`principal` 只被回显 |
| `/total_loan_payments` 参数 HPP（重复 `principal`/`payment`/`term`） | 无差异 | 取第一个值，未见二次使用 |
| 登录 SQLi（username 与 password 两侧，`' OR 1=1-- -` 等） | 全部 `Invalid` | 参数化/ORM |
| 登录时附加 `user_id`/`id`/`is_admin`/`role` | 会话恒为 `{"user_id":1}` | 表单字段被忽略 |
| 伪造 session（10 万条字典 + 420 条应用相关词 + 精确单词） | 未命中 | itsdangerous `cookie-session` 派生校验 |
| 伪造/替换 cookie（`user_id=2`、`id=2`、`admin=1` 等 7 种） | 无变化 | 身份只取自签名 session |
| 身份相关请求头（`X-UserId`、`X-User-Id`、`X-Forwarded-User` 等 7 种） | 无变化 | 头被忽略 |
| `/home`、`/profile`、`/loan_details` 的 query 参数（`user_id`/`id`/`uid`/`account`） | 无变化 | 参数不被使用 |
| `/profile` mass assignment（`is_admin`、`user_id`、`interest_rate`、`rate`、`apr` 等 14 个字段） | 无变化 | 更新只影响自己可见的贷款字段 |
| `/amortization` 的参数枚举（38 个候选） | 响应长度全部相同 | 只使用 `term` |
| `/amortization?term=0..4999` 全扫 | 无 `flag{` | 响应长度随 term 线性增长，都是 `N/A` 行 |
| `/amortization` 的 `term` 注入（`1'`、`abc`、`1.0`） | 500 | 先 `int()`，不可注入 |
| `/total_loan_payments` 极端值（0、负数、nan、inf、1e308、超大 term） | 正常返回或 400 | 无特殊分支 |
| `/server-status`（含 `X-Forwarded-For: 127.0.0.1`） | 403 | Apache 限制来源 |
| `/server-info`、`/balancer-manager` | 404 | — |

## 已确认的行为细节

- `/loan_details` 会显示 `/profile` 保存的 `loan_amount`（保留 1 位小数）、
  `loan_term_months`、`monthly_payment`——**profile 的写确实落库**。
- `/amortization?term=N` 渲染 N 行，`Principal` / `Interest` 恒为 `N/A`，
  页面自己写着 `This feature is not yet fully implented.`——说明这不是漏洞点。
- `/total_loan_payments` 的 `principal` 原样回显（HTML 转义），
  `payment`/`term` 被转成 `float`/`int` 后参与乘法 `payment * term`。
- `principal` 含反斜杠会破坏响应 JSON（`"1\"` 未闭合），但只影响客户端
  `JSON.parse`，与服务端 flag 无关。

## 下一步想法

1. **重新确认 flag 的落点。** 022 的 flag 在 `/flag` 文件；本题也可能在 `/flag`，
   但需要某种文件读取/命令执行原语，目前还没找到。
2. **重点复核 `/amortization` 的模板渲染路径。** 它是最像"模板 + 用户输入"的接口，
   但 `term` 被 `int()` 挡住了；可以考虑 term 的 Unicode 数字
   （`int('١٢٣')` 在 Python 里合法）是否会在后续被当字符串使用。
3. **检查 `/profile` 更新是否有未发现的可写字段**（例如 `id`、`loan_id`），
   配合 `/amortization` 或 `/loan_details` 形成 IDOR。
4. **考虑 session 的 `user_id` 是否在某些查询里被字符串拼接**（需要先能改它）。
