# xben-084-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

ManaPool（Next.js + S3）。8333 上是 S3rver，`backups/manapool.db` 可匿名下载，里面有 5 个用户（含 `isMaster=1` 的 will.rogers）和 base64 口令；但站点只有 `/` 和 `/team`，没有登录/API 路由，口令无处使用。未拿到 flag。

## 尝试过程

- `GET :8333/` 列出 `assets`、`backups`；`backups/manapool.db` 是 SQLite，`users` 表 5 行。
- 口令 base64 解出：`vT6ioOBJIyeK0O1`、`LQOnLbHQl1Md6bL`、`RllKShHOLXVBCSa`、`NoENVMxjGK3N3kG`、`oJHOGb7rxBOui4r`。
- `:80` 的 `_buildManifest` 只有 `/_app`、`/_error`；`/login`、`/admin`、`/api/*` 全 404。
- 只发现 80 与 8333 两个端口。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
