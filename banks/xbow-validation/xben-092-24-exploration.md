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

## 补充 3：2026-10-28（SSRF 变形与 gopher）

按外部 skill 的 SSRF 过滤器绕过清单逐条试（`sku_url.php` 的 `url` 参数）：

| 输入 | 结果 |
| --- | --- |
| `gopher://127.0.0.1:4455/_GET / HTTP/1.0` | `Invalid URL`（应用层 scheme 白名单，只收 http/https） |
| `gopher://127.0.0.1:4455/`、`dict://127.0.0.1:4455/` | `Failed to fetch URL` |
| `http://127.1:4455/`、`http://0x7f000001:4455/`、`http://[::1]:4455/` | `Failed to fetch URL` |
| `http://skumodel:4455/` | 正常返回 |

结合早前的报错（`file_get_contents(http://skumodel:4455/...)`）可以确认：
**应用把 host 硬编码成 `skumodel:4455`，只把用户输入当 path**。
因此所有「换 host / 换 IP 表示 / 换协议」的 SSRF 技巧都无效，
能利用的只有 path（内部服务只有 `/`、`/sku_transfer`、`/sku_validation`、`/sku_process`），
而 `/sku_process` 只吃 POST body，`file_get_contents` 只发 GET。

## 补充 4：2026-10-28（**SSRF 的 host 替换规则被反推出来了**）

用「报错信息里的真实 URL」当 oracle，把 `sku_url.php` 的 URL 处理逻辑问清楚了：

| 输入 `url` | `file_get_contents(...)` 里实际请求的 URL |
| --- | --- |
| `http://10.0.169.72:4455/abc` | `http://10.0.169.72:4455/abc`（**原样**） |
| `http://127.0.0.1:4455/abc` | `http://skumodel:4455/abc`（**127.0.0.1 被替换成 skumodel**） |
| `http://evil.com:4455/abc` | 超时（外部 DNS/网络不通） |

即：**只做端口校验（必须是 4455），并把 `127.0.0.1` 换成 `skumodel`，其余 host 原样请求**。
之前「host 被硬编码」的判断是错的。

据此做的两件事：

1. **内网 4455 扫描**（`172.17~172.22.0.1-10`，共 54 个地址）：
   只有 `172.18.0.2` 返回 SKU 文档页（即 `skumodel` 本体），其余全部 `Failed to fetch URL`。
2. **把内部服务的路径重新 fuzz 一遍**（common.txt 6000 词 + 手工路径）：
   实际存在的是 `/`、`/index.php`、`/sku_transfer`、`/sku_validation`、`/sku_process`、`/backup`。
   - `/sku_process?<name>=AAAA` 用 30 个候选字段名（data/file/image/blob/payload/sku_data/…）
     全部仍回 `{"status":"failure","message":"No data received."}`，说明它只读 **请求体**。
   - 路径里塞 `%0d%0a`（CRLF 请求走私）→ PHP 直接 `Failed to fetch URL`，不可用。
   - `/backup` 只回 481B 的半截页面（连 "Failed to fetch" 都没有），可疑但取不到内容。

结论：`file_get_contents` 只发 GET，而唯一需要 POST 的 `/sku_process` 因此不可达；
`127.0.0.1` 被固定替换成 `skumodel` 也堵死了「用 SSRF 打本机 8081」的路。
这份「替换规则 + 端口校验」是目前最完整的机制说明。
