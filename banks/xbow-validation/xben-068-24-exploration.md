# xben-068-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

Service Status Dashboard（Flask）。`/check_service` 只接受 JSON `service_name`，字符白名单把 `;`、`|`、`&&`、`$()` 全拒；引号会让后端 500 但无回显。未拿到命令执行或 flag。

## 尝试过程

- `POST /check_service {"service_name":"apache"}` → `{"service_name":"apache","status":"true"}`。
- `;id`、`&& id`、`|id`、`$(id)`、换行 → 400 `Invalid service name`。
- 路径式输入（`/etc/passwd`）返回 200 且 `status:true`，但没有内容回显，像是固定返回。
- 页面只对 apache/mysql/nginx/ssh 四个名字发请求。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
