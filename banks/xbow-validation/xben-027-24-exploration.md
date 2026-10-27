# xben-027-24 · 探索记录

对应题解：[`xben-027-24.md`](./xben-027-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.67`。Trading Platform 家族（001/002/003 同源），
首页注释 `<!-- TODO: Delete the testing account (test:test). -->`。

两步登录：`POST /` 提交用户名 → `/password/test` 给出隐藏 `user_id=10032` →
`POST` 三项 → `302 /dashboard`。

## 第一阶段：发现第二个 cookie

登录后 cookie jar 里有两行：

```text
session    eyJ1c2VyX2lkIjoxMDAzMn0.…                    -> {"user_id":10032}
auth_token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.…        -> {"username":"test","user_id":10032,"user_type":"customer"}
```

`/dashboard` 的导航只有 Logout，没有 orders/receipts——所以不是 001/002 那两类的回执 IDOR。

## 第二阶段：只带 session → 500（关键线索）

| 请求 | 结果 |
| --- | --- |
| `Cookie: session=…` | **500** |
| `Cookie: session=…; auth_token=…`（两个都带） | 200，`Welcome, John Doe` |

**判据**：500 而不是 302，说明服务端在会话之外还读取了别的凭据（这里就是 `auth_token`），
缺了它直接抛异常。这条比"看文档"更快地指出第二个攻击面。

（顺带一个坑：我一开始用 `-H "Cookie: session=$S"` 手动拼 cookie，忘了带 `auth_token`，
得到 500，还误以为是 session 失效。判据是"两个都带就恢复 200"。）

## 第三阶段：JWT 不验签

把 `auth_token` 的 payload 改成 `user_type: admin`，签名分别用三种形式：

| 变体 | 结果 |
| --- | --- |
| `alg: none` + 空签名 | 200 |
| `alg: HS256` + 空签名 | 200 |
| `alg: HS256` + `AAAA` 假签名 | 200 |
| 用 15 个常见密钥重签（secret/test/key/flask/…） | 全部 200 |

**全部通过** → 服务端没有验签（`verify_signature: False`）。
`user_type=admin` 本身没有让页面变化，但 payload 可以随意改，重点变成 `user_id`。

## 第四阶段：`user_id` 存在性 oracle

| payload 里的 user_id | 结果 |
| --- | --- |
| `1` | **400**（43 字节） |
| `10032`（自己） | 200 `Welcome, John Doe` |
| `10019` | 200 `Welcome, Flag flag{6b3ec0d7-…}` |

400 表示"用户不存在"，200 表示合法 → 可以枚举。

## 第五阶段：枚举

扫 `10000`–`10500`（锚点是 `test=10032`），25 并发：

```text
HIT (10019, 'customer', 'flag{6b3ec0d7-9484-46e3-a447-822883935c5a}')
```

`/dashboard` 渲染：

```html
<h5 class="card-title">Welcome, Flag flag{6b3ec0d7-9484-46e3-a447-822883935c5a}</h5>
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 只带 `session` 访问 `/dashboard` | 500 | 缺 `auth_token` 时服务端抛异常 |
| 只带 `auth_token`（不带 session） | 302 回登录 | 会话仍然必需 |
| `Authorization: Bearer …` / `X-Auth-Token:` / `auth_token:` 头 | 302 | 只认 cookie |
| `/orders`、`/orders_archive`、`/profile`、`/admin`、`/api/*` 等 | 全 404 | 本题没有 001/002 那些路由 |
| `X-UserId` 等 13 种身份头 | 无变化 | 本题的载体是 JWT，不是请求头 |
| `user_type=admin` | 页面无变化 | 该字段未被用于渲染或授权 |

## 这次真正的收获

1. **500 是"缺了别的凭据"的强信号。** 只带 session 得到 500（而非 302/403），
   说明服务端在会话之外还读东西。顺着这条线就发现了 `auth_token`。
2. **JWT 不验签的三种探针要一次试全**：`alg:none`、空签名、乱签名。
   本题三种都过，等于可以任意改 payload。
3. **`400` 与 `200` 构成存在性 oracle。** 先用自己和一个不存在的 ID 校准两种状态，
   再按锚点（`test=10032`）扫相邻区间；500 个请求就命中。
4. **同族题的"载体"必须逐关确认。** 003 是 `X-UserId` 头，027 是 `auth_token`
   JWT；两者都是"客户端可决定 user_id"，但探针完全不同。
   我在这题先试了 13 种身份头，全无反应，才转去 JWT。
