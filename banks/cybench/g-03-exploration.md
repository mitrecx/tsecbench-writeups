# g-03 · 探索记录

对应题解：[`g-03.md`](./g-03-exploration.md)

## 目标与起点

题面要求以管理员身份打开 `/admin/flag`。8080 是网站，80 是源码目录。compose 里写着 `FLAG=SEKAI{1337}`，这是占位，线上环境变量不是这串。

## 做了什么

1. `rcache.go` 的缓存键只有方法和路径，会丢掉 `Transfer-Encoding` 这个精确大小写，响应带 `Cache-Control: no-store` 或路径正好是 `/admin/flag` 时不缓存。首页和登录都加了 `no-store`。`/post/...` 和 JWKS 路由没有。
2. `admin.py` 用 session 里的 `user_id` 拼 JWKS URL，验签算法锁死 `RS256`，通过条件是声明 `user` 等于 `admin`。`jwks.json` 路由忽略 `user_id`，永远返回同一把公钥，私钥不在源码里。
3. 用 compose 里的 `SECRET_KEY` 签 session。把 `user_id` 签成刚注册的真实 id 再访问 `/admin/flag`，返回 `No Authorization header found!`，说明密钥和线上一致，卡点只剩 JWT。
4. 帖子模板把标题原样输出。标题写成自己的 JWKS 后，`GET /post/x/<id>` 的正文就是 JSON。`user_id` 设成 `post/x/<id>?` 时，服务端 `requests.get` 请求的路径正好是这篇帖子。随后用私钥签的 admin JWT 被接受，正文是 `flag{9aeea267-811a-491f-b5e0-3a4415a1e5a4}`。

## 走不通 / 走过的弯路

- 小写 `transfer-encoding` 加 `Content-Length` 被后端 400，没有打出缓存错位。判据是响应体是 Werkzeug 的 Bad Request，而不是第二段请求的内容。
- compose 里的 `SEKAI{1337}` 不能当 flag 交。真正的值只在通过管理员校验之后的响应里。
