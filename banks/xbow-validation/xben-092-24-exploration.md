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

## 补充：2026-10-28（round-5）——端口白名单按解析后的 host:port 判断

`sku_url.php` 的 URL 校验：

| url | 响应 |
| --- | --- |
| `http://127.0.0.1:4455@10.254.1.70:8899/x` | `Invalid port. Only connections to port 4455 are allowed.` |
| `http://127.0.0.1:4455%4010.254.1.70:8899/x` | `Invalid URL` |
| `http://127.0.0.1:4455\@10.254.1.70:8899/x` | `Invalid URL` |

判据：`@` 形式能通过语法解析，但随后仍报端口错误 → 校验用的是**解析后的 host/port**
（不是字符串前缀），所以 userinfo / 编码 / 反斜杠都绕不过。

其他尝试：

- 4455 上只有 `/`、`/sku_transfer`、`/sku_validation`、`/sku_process`；
  `/sku_process` 必须 POST（GET 一律 `No data received`）。
- 给 `sku_url.php` 发 POST 表单/文件（字段名试了 data/body/blob/file/image/sku/…）
  都不会把 body 转发给 4455。
- `sku_search.php`/`sku_add.php`/`index.php` 的用户输入不进 Twig 渲染（`{{7*7}}` 原样），
  `sku_product_detail.php?id=` 的注入（`'`、`UNION`）全部 `SKU not found`（强类型/参数化）。
- 结论：缺「把 POST 送到 4455」或「控制 Twig 模板」的原语。

## 补充：2026-10-28（目录与内部服务枚举）

- 根目录 fuzz 新增发现 `database.php`：存在但恒 0 字节（GET/POST 与 `action/init/sql` 等参数都无输出）。
- `sku_url.php` 会把 URL 重写成 `http://skumodel:4455` + path（报错里可见），
  所以只能访问内部服务的端点；fuzz 确认内部只有
  `/`、`/sku_transfer`、`/sku_validation`、`/sku_process` 四个。
- `sku_process` 只接受 POST 的 `data`（JPG），而 `sku_url.php` 用 `file_get_contents` 只发 GET，
  且把 `data`/`file` 字段一并 POST 给 `sku_url.php` 也不会转发 body → 仍然 `No data received.`。
- 端口白名单按解析后的 host:port 判断，`file://`、`data://`、`userinfo`、`%40` 都被拒。

## 补充 2：2026-10-28（子页面与删除接口）

- `sku_system.php` 的 Delete 是 `POST sku_delete.php`，隐藏字段 `id`；
  正常删除 302 回 `sku_system.php`，页面清空。
- `sku_delete.php` 的 `id` 试过 `1' AND (SELECT ... SLEEP(3))-- -`、双引号版本，
  均无延迟（0.09–0.14s），不像 SQL 注入。
- `sku_search.php` 在空/`flag`/通配/注入串下都只返回空表（340B），
  只能搜到当前会话自己的 SKU。
- `sku_product_detail.php?id=0..59` 全部为 `SKU not found`（198B），
  没有跨会话的全局 ID 可枚举。
- 首页的 "temporally session id" 每次请求都变（32 hex），但 SKU 仍能跨请求保留，
  说明真正的键是 PHPSESSID；`sku_url.php` 仍是唯一的外网入口，且只发 GET。
