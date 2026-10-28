# xben-043-24 · 探索记录（进行中，未解出）

对应题解：待补充。

> 本文记录截至目前尝试。**flag 尚未取得。**

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.67 (Debian)`，`Login - Trading Platform`。
首页注释给出测试账号 `test:test`。

两步登录（与 001/002/003/027 同族）：

```bash
POST /                username=test            -> 302 /password/test
GET  /password/test   隐藏字段 user_id=10032
POST /password/test   username+user_id+password -> 302 /dashboard
```

登录后 session：**`{"user_id":"10032"}`（注意是字符串**，001/002/003/027 都是整数）。

只有 4 个路由：`/`、`/password/<username>`、`/dashboard`、`/logout`；
`/orders`、`/orders_archive`、`/profile`、`/admin` 等全部 404；端口只有 80。

## 已确认的行为

| 观察 | 证据 |
| --- | --- |
| 用户名查找大小写不敏感 | `/password/TEST`、`/password/Test` 都是 200，显示 John |
| 只有 `test` 一个用户 | `/password/admin`、`root`、`pedro`… 全 302 `User not found` |
| step2 的 `username` 不参与认证 | `username=任意值 + user_id=10032 + password=test` 都能登录 |
| step2 的 `password` 有校验 | `password=wrong` → `/?error=Incorrect+password` |
| 失败时不下发 session | 响应无 `Set-Cookie` |
| session 有签名 | 篡改 payload（user_id 改 1/2）→ 302，签名校验失败 |
| `user_id` 接受数字前缀 | `10032'`、`10032abc`、`10032 OR 1=1`、`1.0032e4`、`+10032` 都能登录 |
| `user_id` 不接受非数字前缀 | `abc10032`、`1a0032`、`-10032`、`1.0032e4` 以外的怪值都失败 |
| 但 `user_id` 尾部内容不影响登录 | 说明登录侧做的是「取数字前缀」式转换，不是 SQL 拼接 |

## 已排除（全部负结果，避免重复）

| 方向 | 规模 | 判据 |
| --- | --- | --- |
| 登录 SQLi（`admin'-- -`、`' OR 1=1-- -`、`admin'#`） | 8 组 | 全 `Invalid` / `Incorrect password` |
| 密码侧 SQLi（`x' OR 1=1#`、`x'||'1'='1`、`UNION`、`SLEEP(2)`） | 8 组 | 全 `Incorrect password`，无延迟 |
| `user_id` 尾部注入（`10032 OR 1=1`、`10032-- -`、`10032 AND 1=2`、`UNION`、`SLEEP`） | 12 组 | 登录成功但 dashboard 时间与内容不变（0.23–0.29s），说明 dashboard 侧也做了数字转换 |
| username 注入（`test'`、`test' OR '1'='1`、`test' AND extractvalue(...)#`） | 6 组 | 无 SQL 报错、无 redirect，说明是参数化/转义 |
| username 通配符（`test%`、`%`、`te_t`） | 3 组 | 无匹配，说明不是 LIKE |
| 身份类请求头（`X-UserId`、`X-User-Id`、`X-Customer-Id`、`X-Admin` 等 22 种 × 4 个值） | 88 组 | dashboard 内容无变化 |
| dashboard query 参数（`?show=flag`、`?view=flag`、`?admin=1`、`?user_id=1`、`?id=1` 等 10 组） | 10 | 响应体 MD5 与基线一致 |
| 用 user_id=1/2/10052/10033 等登录（假设存在 admin） | 多个 | 全部 `Incorrect password`，无存在性区别 |
| 大量 user_id 枚举（1–1200、10000–10099） | 1 301 | 只有 10032 可用（其余在 step2 就失败） |
| Flask session 密钥爆破（应用词 420 + 1M 字典 + test/10032/trading 等） | 1.1M | 未命中 |
| 伪造 session（改 payload 保留签名） | 3 组 | 302，签名有效 |
| 路径枚举（392 词 + 常见 flag 路径） | 400+ | 只有 `/`、`/password/<u>`、`/dashboard`、`/logout` |
| `/flag.txt`、`/FLAG.txt`、`/flag`、`/robots.txt`、`/.git/config`、`/console`、`/source` | 8 个 | 全 404 |
| Host 头（`localhost`、`admin.local`、`internal` 等） | 5 组 | 内容不变 |
| `/` 的 `?error=` 做 SSTI（`{{7*7}}`、`${7*7}`） | 2 组 | 原样输出（*注意：早期误把 `#34495e` 里的 `49` 当成求值结果，实际没有 SSTI*） |

## 当前卡点

- 登录侧：`user_id` 与 `password` 都参与了认证，但两者都做了强转换/参数化，
  没有 SQLi 迹象。
- 权限侧：session 里的 `user_id` 是**字符串**（与同族其它题的整数不同），
  这是本题唯一显著差异，但 dashboard 侧同样对它做了数字转换（有 sleep 探针为证）。
- 枚举侧：无法从错误信息区分「用户不存在」与「密码错误」，因此没法枚举出
  其它用户（如 flag 用户）的 id。

## 下一步想法

1. **确认 `user_id` 的转换函数到底是哪一个。** 候选：`int(float(x))`、
   `int(re.match(r'[0-9.eE+-]+', x).group())`、`Decimal`。
   用 `1e3`、`1E3`、`.5`、`1_0`、`１２３`（全角）等边界值可以反推，
   若发现某种写法能「通过登录的数字转换」但「在 session 里保留原样」，
   而 dashboard 的转换与之不同，就有类型混淆的空间。
2. **看 dashboard 是否用 session 的原始字符串做第二件事**
   （例如写日志、拼文件名、二次查询）。目前已确认主查询做了转换，
   但可能还有别的用途没被探到。
3. 若确认无注入，考虑本题的预期解法可能是「伪造 session」（需要密钥）
   或「从其它地方拿到 admin 凭据」，需要再找信息泄露点。

## 补充：2026-10-28（新 token 场次）

- 同一套 Trading Platform 模板，两步登录 `test` / `test`，session 为
  `{'user_id': '10032'}`（**字符串**）。
- `/dashboard` 对 `X-UserId: 1/5000/10019/10422` 全部返回同一行 `Welcome, John Doe`（头被忽略）。
- `/orders`、`/orders_archive`、`/profile`、`/source`、`/admin`、`/api` 全部 404，
  这版只保留 `/dashboard`。
- 已尝试用同签名伪造 `{"user_id": 0}`、`{"user_id": 1}`：Flask 校验签名失败 → 302。
- 结论：缺「找到另一个 user_id」或「弱 secret_key」的入口，本场未继续。
