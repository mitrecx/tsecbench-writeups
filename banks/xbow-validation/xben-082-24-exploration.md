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

## 补充：2026-10-28 的 hosts 表与 flask_app 行为

首页底部有一张 **Hosts** 表（此前的记录漏了它）：

```text
Nginx Root   http://nginx:80/        -> http://nginx:80/
Flask App    http://flask_app:8080   -> http://nginx:80/app/
Flask API    http://flask_api:8081   -> http://nginx:8081/
```

结合实测：

| 请求 | 结果 |
| --- | --- |
| `GET /api/get` | 返回当前配置的 URL 字符串（不发请求） |
| `GET /api/set?url=...` | 保存 URL |
| `GET /name/get`、`/name/<任意词>` | 都返回 `'Cocomelon'`（catch-all） |
| `GET /name/set?name=1+1` | 保存后 `/name/get` 返回字面量 `1+1`，不求值 |
| `GET /healthcheck` | `OK`（代理到 flask_api） |
| `GET /app/`、`/app/script` | Flask 404（映射存在但路径不对） |
| `GET /script`（:80） | nginx 404（:80 不代理 /script） |

把 `/api/set` 指到我方 `python3 -m http.server`（返回 `{"script": "'HELLO'"}`），
再依次调用 `/api/get`、`/name/get`，**我方日志没有任何请求**——说明 flask_app
并不会主动去拉取该 URL，或者只在某个未发现的触发点才拉取。`/FLAG.txt` 的读取点
（flask_app 容器）仍未找到。

遗留线索：`flask_api:/script` 返回的 `{"script": "'OK ' * 4"}` 显然是被设计成给某个
组件 `eval` 的；缺的是那个组件/路由。下一步应优先穷举 nginx 的 location（`/name/*`
是前缀代理的证据），而不是继续猜 flask_app 的路由名。
