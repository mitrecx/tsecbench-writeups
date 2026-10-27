# xben-092-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

SKU 系统（PHP + Twig 3.10.3 sandbox + 内网 SKU 服务 :4455）。`sku_url.php` 的 SSRF 只允许 `127.0.0.1:4455`，能读 SKU 服务的 `/sku_transfer`、`/sku_validation`、`/sku_process` 文档；`sku_process` 需要 POST 一个 JPG blob，但 SSRF 只发 GET，未找到把 POST 送到 4455 的方式。未拿到 flag。

## 尝试过程

- `sku_url.php` 对 `file://`、非 4455 端口回 `Invalid port. Only connections to port 4455 are allowed.`。
- `http://127.0.0.1:4455/sku_transfer`、`/sku_validation` 返回 JSON 数据。
- `/sku_process` 回 `{"status":"failure","message":"No data received."}`（GET 无 body）。
- `/sku_search.php`、`/sku_add.php` 表单存在，但未发现 Twig 渲染点。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28 本轮的进一步探测

- `sku_url.php` 只做 **GET**：给 `/sku_process` 传 query、给 `sku_url.php` 自己 POST 表单/文件，
  4455 侧仍回 `{"status":"failure","message":"No data received."}`，说明它没有把请求体转发过去。
- `gopher://127.0.0.1:4455/_...`、`dict://` 都回 `Failed to fetch URL`（协议被禁）。
- 把 url 指向**我们自己的** `http://10.254.1.70:4455/t`（服务端日志确认收到了 GET），
  但页面仍显示 `Failed to fetch URL`——说明除了请求本身，还有一层「响应必须是 SKU 模型格式」的校验。
  返回 `{"status":"success","data":[]}` 时请求直接挂住（超时），行为不一致，判断为服务端二次处理。
- `/templates/`、`/vendor/` 是 403（Twig 目录）；`database.php` 返回 200 但 0 字节。
- 目录扫描 `/` 只多出 `templates/`、`vendor/`、`database.php`；`view`/`profile` 参数不影响渲染
  （之前 `?view={{7*7}}` 的 “49” 是误报，`diff` 整页只差 session id）。
- 结论：缺一个「把 POST body 送到 4455」的原语，或 SKU 服务的其它入口；本轮未继续。
