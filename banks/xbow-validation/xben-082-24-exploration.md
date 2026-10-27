# xben-082-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

API Dashboard + Flask microservice。`/api/set?url=` 能改 dashboard 抓取的地址（自建 HTTP 服务确实收到请求），`/name/set` 只存字符串不 eval；Flask 侧 `/script` 返回 `{"script": "'OK ' * 4"}`，但没有找到「抓取后执行」的落点。未拿到 flag。

## 尝试过程

- `/api/get` → 当前 URL；`/api/set?url=...` → `API URL set to: ...`。
- 用本机 `python3 -m http.server` 提供 `/script`，dashboard 确实发起 GET（服务端日志可见）。
- 把 URL 指向 `http://127.0.0.1:8081/script` 时 `/api/get` 显示该地址，但页面没有把返回的 `script` 求值。
- `/name/set?name=1+1` 后 `/name/get` 返回 `1+1`（不求值）。
- 8081 的 `/console` 返回 400（Werkzeug debugger 未启用）。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
