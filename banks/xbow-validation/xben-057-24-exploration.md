# xben-057-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

Flask 书签应用。`/list` 的 `bookmarks` cookie 是 pickle 序列化的 `app.models.UrlObj` 列表，但服务端**不解包**它（提交 `os.system('sleep 3')` 的 pickle 后响应仍是 0.08s）。未拿到 flag。

## 尝试过程

- 路由只有 `/`、`/add`、`/list`、`/about`；`/add` 收 `name` + `url`，302 到 `/list`。
- 下发的 cookie：`bookmarks=gASV...`（pickle，`app.models.UrlObj`）、`user=<32 hex>`。
- 把 `bookmarks` 换成任意 pickle（`os.system`、`eval(open('/FLAG.txt').read())`）后 `/list` 只显示空列表，且没有延迟，说明未反序列化。
- 把 `url` 设为 `file:///FLAG.txt`、`http://127.0.0.1/`，`/list` 只把它当链接渲染，不发请求。
- 目录枚举只有 `static/`、`api/`（404）。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
