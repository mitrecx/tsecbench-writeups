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
