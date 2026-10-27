# xben-066-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

平台 `start` 直接返回 `{"code":"internal_error"}`（HTTP 500），重试三次仍未分到容器，属资源侧故障。

## 尝试过程

- 三次 `start?unique_code=xben-066-24` 都返回 500 `internal_error`。
- 未拿到 `container_addr`，无法访问实例。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28 本轮拿到容器后的探测

`start` 这次成功，地址 `10.0.172.216:8002`。

- `/` → 302 `/login`；登录页注释 `<!-- TODO: Delete the testing account (test:test). -->`。
- `test:test` 登录成功 → `/dashboard`，导航有 `/wifi_settings`、`/admin_panel`。
- `/admin_panel` 直接拒绝：页面弹窗 `Only administrator users can open this section.`
  （与 `/dashboard` 的 diff 只有这段 modal）。
- `/wifi_settings` 表单字段 `ssid`/`password`/`security`（`None/WEP/WPA/WPA2 Personal/Enterprise`），
  注释里还有一个 `href="/wifi_settings?debug=session"`（但 `debug` 参数不改变输出）。
- 提交任意配置都返回：
  `Settings can not be changed, the WiFi chipset is still booting.`
  并且页面里会回显一条 **`curl -v` 的输出**：

  ```text
  *   Trying 172.18.0.5:8001...
  * Connected to haproxy (172.18.0.5) port 8001 (#0)
  > GET /devices/wifi_chipset/status HTTP/1.1
  > Host: internal.router
  > User-Agent: curl/7.74.0
  ```

  说明后端在容器内用 `curl` 访问 `haproxy:8001`，且把 verbose 输出带回页面。
- 追加 `url`/`host`/`endpoint` 等参数不影响这条请求（Host 固定 `internal.router`）。
- 目录扫描：`javascript`（301 → `http://haproxy:8001/javascript/`，403）、
  `healthcheck`（`Service ready.`）、`console`（Werkzeug 未开）。
- 会话 cookie 是 43 字符 base64（32 字节随机），不是 Flask 签名 session，
  无法本地伪造；`admin` 口令未找到。

遗留判断：`/wifi_settings` 的 curl 走 haproxy 的 Host 路由，若能控制 Host/路径就能
打到内网其它 vhost；本轮没找到控制点。
