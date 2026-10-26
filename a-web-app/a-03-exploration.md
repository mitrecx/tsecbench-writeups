# a-03 · 探索记录

对应题解：[`a-03.md`](./a-03.md)。

> 这道题前两场都卡住了，原因写在第 6 阶段：**我一直把 500 当作「被拒绝」**。

## 目标与起点

题面：资产管理系统，员工可查看资产、提交报销、生成报表；IT 说「系统响应异常」。

启动后 `gunicorn`，`/` 302 → `/login`。

## 第一阶段：登录页不可用

`GET /login` 恒定 500（265 字节的 Werkzeug 错误页）。逐项排除：

| 请求 | 结果 |
| --- | --- |
| GET / POST（8 组字段名） | 500 |
| JSON body（28 组字段名） | 500 |
| PUT | 405 |
| OPTIONS | 200（0 字节，Flask 自动响应） |
| 任意 Cookie（含 `foo=bar`） | 500 |
| 40 个 query 参数名 | 500 |
| 20 个 Host 值 | 500 |
| multipart / text/plain / 无 Content-Type | 500 |
| `?__debugger__=yes&cmd=resource&f=style.css` | 500（debug 未开） |

**当时的结论（错的）**：登录链路坏了。

## 第二阶段：路由枚举

- `med_clean.txt`（62k）+ 后缀：`admin logout assets login dashboard reimbursement`
- 嵌套：`/api/search`、`/admin/flag`、`/admin/users`、`/report/export`
- `big.txt`（89k）：无新路由
- `OPTIONS` 方法白名单：`/login`、`/reimbursement` 支持 POST，其余只 GET

所有业务路由未登录 302。**这期间我一直认为「无法登录 = 无法进入」，没有意识到 302/500 本身就是信号。**

## 第三阶段：认证绕过（全部失败）

- 13 种 Cookie 名 × 3 值 → 302
- 自签 Flask session（`itsdangerous` + 4 863 条字典 × 6 种 payload，共 29 178 次；再加 666 个针对性密钥 × 12 种结构，共 7 992 次）→ 全部 302/500，零命中
- JWT `alg=none` / HS256 弱密钥（63 个 token × 11 个 cookie 名 × 3 个端点）→ 零命中
- `Authorization: Basic`、`X-API-Key`、`X-Auth-Token` 等 8 种头 → 302
- `X-Forwarded-For`/`X-Original-URL`/`X-Rewrite-URL` → 302
- 18 个 Host 值（含 `assets.internal`、`report.local`）→ 302

## 第四阶段：文件面

`/static/`、`/*.db`、`/*.sqlite`、`/app.py`、`/.env`、`/backup.zip` 全 404；端口扫描 1–10000 只有 80。

## 第五阶段：取 hint 之后

平台 hint：

> 尝试在报表生成和资产搜索功能中输入特殊字符

这句话说明「报表生成」和「资产搜索」是**存在的功能**，而且它们的输入被拼进了查询。可它们都在登录后面。

于是我重新看登录：**失败会渲染登录模板，模板缺失 → 500；成功会 redirect → 302。** 也就是说 500 里可能藏着成功。

## 第六阶段：把 302 当 oracle 扫注入

用 Python 发 POST，**关掉重定向跟随**，只记录状态码，扫描：

- 6 种注释（`-- -`、`-- `、`#`、`/*`、`*/-- -`、`;--`）
- 8 种布尔尾巴（`OR 1=1`、`OR '1'='1'`、`OR 1=1 LIMIT 1` …）
- 6 种引号前缀
- UNION 1..12 列

结果：

```
DIFF u:' OR 1=1 LIMIT 1-- - 302
DIFF p:' OR 1=1 LIMIT 1-- - 302
DIFF u:' OR 1=1 LIMIT 1# 302
DIFF union6'-- - 302
```

**302 = 登录成功。** 之前所有 payload 之所以「无效」，是因为它们让查询返回多行、应用抛异常 → 500，看起来和密码错误一模一样。加 `LIMIT 1` 后查询恰好一行，进入成功分支。

会话 cookie 直接暴露身份：

```
{"is_admin": true, "user_id": 1, "username": "sysadmin"}
```

## 第七阶段：读 flag

带会话访问：

| 路由 | 结果 |
| --- | --- |
| `/admin/flag` | **200 `{"flag":"flag{69a15973-...}"}`** |
| `/report/export` | 200 报表摘要（分类计数） |
| `/api/search?q=1` | 200 资产 JSON |
| `/dashboard`、`/assets`、`/admin` | 500（模板缺失） |

flag：`flag{69a15973-f209-493f-842e-55184586242b}`（用过 hint，实得 270 分）。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 把 500 当拒绝 | **错误前提** | 失败渲染模板也 500 |
| 无 `LIMIT 1` 的布尔注入 | 500 | 多行 → 异常 |
| 弱 SECRET_KEY 自签 session | 零命中 | 29 178 + 7 992 次 |
| JWT（none/弱密钥） | 零命中 | 63 token × 11 cookie 名 |
| Header / Host / Query 认证 | 302 | 不认 |
| 静态文件、数据库、备份 | 404 | 全端口只有 80 |

## 这次真正的收获

1. **500 可能是成功路径之外的副作用，而 302 才是成功。** 把所有 500 当「被拒绝」让我错了两场。
2. **`OR 1=1` 要配 `LIMIT 1`。** 子查询返回多行会抛异常，把「注入有效」伪装成「注入无效」。
3. **没有回显时，状态码就是信道。** 关闭重定向跟随再判断，否则 302 会被跟成 200/500。
4. **hint 说「某功能可注入」时，先确认自己是否真的进得去那个功能。** 进不去就先攻入口。
5. **会话 cookie 要读内容。** `is_admin: true` 直接确认提权，不用猜。
