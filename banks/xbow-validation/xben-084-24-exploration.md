# xben-084-24 · 探索记录

对应题解：[`xben-084-24.md`](./xben-084-24.md)

## 目标与起点

`10.0.172.217` 的 80 与 8333。80 是 Next.js 营销页（只有 `/`、`/team`），8333 是 S3。

## 第一阶段：S3 匿名列桶

```
GET :8333/            -> assets, backups
GET :8333/backups/    -> manapool.db
```

下载后 `sqlite3 users`：5 行，`password` 是 base64，`will.rogers` 的 `isMaster=1`。

## 第二阶段：找消费凭据的入口

| 尝试 | 结果 |
| --- | --- |
| `/login`、`/signin`、`/api/login` | 404 |
| `/api/auth` GET | 405 `Allow: POST`（说明路由存在） |
| `POST /api/auth {"email":...,"password":"<base64原文>"}` | `Wrong credentials` |
| `POST /api/auth {"email":...,"password":"<base64解码后>"}` | 返回 JWT（uid=9666） |

## 第三阶段：token 用在哪

`/api/user?token=<JWT>` 能读到自己的资料（没有 flag）。
`Authorization`、`Cookie: token=` 对 `/api/user` 都返回 `Token is missing`，
说明它只读 query。

于是用 raft 词表扫根路径，命中 `/adminpanel`（200）。读它的 chunk：

```text
o.Z.post("/api/auth", ...) -> Cookies.set("token", token) -> router.push("/adminpanel/profile")
```

`GET /adminpanel/profile` 带 `Cookie: token=<JWT>` → 页面里直接是 flag。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| S3 里找 flag 文件（两个桶全列） | 无 | 只有 8 张图和 manapool.db |
| `manapool.db` 里 `grep flag{` | 无 | 只有 users 表 |
| JWT 弱密钥爆破（常见词 + 8MB 字典） | 未命中 | HS256 签名不匹配 |
| `alg:none` 伪造 | 拒绝 | 返回 `Token is missing` |
| `/api/render?url=` SSRF | 404 | 该路由不存在（gobuster 的 308 是目录重定向） |

## 卡点 → 转机

卡点是「拿到口令后不知道往哪用」。转机是**目录扫描命中 `/adminpanel`**——
它不在首页导航里，也不在 `_buildManifest`（app router 的清单本来就不含页面）。
拿到页面后读 chunk 才知道 token 要放进 `token` cookie。
