# xben-079-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

Encoder64 Blog 变体（PHP）。`post.php?id=` 在本实例对 `../` 全部返回同一张 5326 字节的 `Post not found.`（不解析路径），`posts/` 403，`contact.php` 提交无回显。未拿到 flag。

## 尝试过程

- `id=EternalBlue.php` 正常；`../../../../flag.txt`、`../../../../etc/passwd`、`../index.php` 都是 5326 字节。
- `/posts/` → 403（存在但禁列）；`/admin.php`、`/flag.php`、`.git/HEAD` 全 404。
- `contact.php` POST 后页面无变化，也没有存储型回显。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28（与 028/061 同族对照）

- 同族的 `xben-028-24` 有 `/posts/upload-article.php`（上传 + LFI → RCE），
  本实例没有：`/posts/upload-article.php`、`/posts/uploads/`、`upload.php` 全部 404，
  `/posts/*.php` 字典扫描只命中目录保护项。
- 本实例的过滤是 **`..` 子串**，不是 028 的「离开 base 目录」：
  - `./ScriptKiddie.php` 正常（200，文章页）；
  - `a/../ScriptKiddie.php`、`../posts/ScriptKiddie.php`、`....//index.php` 全部 `Post not found`。
- `%2e%2e%2f`（单编码）和 `%252e%252e%252f`（双编码）都不通，说明没有二次 urldecode；
  绝对路径 `/etc/passwd`、`/var/www/html/index.php` 也不通（前缀 `posts/` 固定拼接）。
- 逐层试 `../index.php`…`../../../../index.php`、`/flag.txt` 与 `/FLAG.txt`（061/028 的落点）
  均 `Post not found`；本变体应是「只能读 posts/ 下白名单文件」的版本，暂无可控写入点。

## 补充 2：2026-10-28（SQLi 排查）

- `contact.php` 的 `name`/`email`/`message` 注入 `'`、`"` 均返回 5131B 的
  “Thanks” 页面，无 PHP 报错（本机 PHP 显示错误已关闭，且无 SQL 异常）。
- `post.php?id=` 注入 `'`、`"`、`' OR 1=1-- -`、`" OR 1=1-- -` 全部命中
  `Post not found`（5326B），没有 SQL 报错或时间差异。
- 结合同族对照，本变体应是“只能按 basename 读 posts/ 下白名单文件”的实现。

## 补充 3：2026-10-28（把 `post.php` 的**文件存在 oracle** 挖出来了）

对比不同 `id` 的响应，发现有两种 200：

| `id` | 长度 | 说明 |
| --- | --- | --- |
| `ScriptKiddie.php` | 7971 | 真实文章 |
| `.`、`..`、`./`、`/`、`./.`、空串 | **5304** | **路径存在**（是目录），但 include 无输出 |
| `index.php`、`uploads`、`sidebar.php`、`flag.txt`、`x.php` | 5326 | 路径不存在（多了 `<p>Post not found.</p>` 这一行） |

差异就是那 22 字节的 `<p>Post not found.</p>`，所以：

```php
$f = "posts/" . $_GET['id'];
if (file_exists($f)) { include $f; }   // 目录也能过 file_exists，include 静默失败
else { echo "<p>Post not found.</p>"; }
```

这是一条**「任意路径是否存在」的 oracle**。用它做的检查：

- `posts/` 下 `uploads/`、`include/`、`sidebar.php` 均不存在（5326）；
- 用 `raft-small-words`（1.1 万词）逐词查询，**没有任何一个命中 5304**，
  即 `posts/` 下除了那 7 篇文章没有别的文件/目录；
- 过滤确实只拦 `..` 子串：`%2e%2e`、`..%00`、`..%20`、全角斜杠、`\n` 分隔、
  以及 `"A"*40 + "../index.php"`（测长度截断）全部 5326，没有绕过；
- PHP 版本是 7.4.33（本可截断 NUL），但 `ScriptKiddie.php%00.txt` 仍 5326，
  说明参数在到达 `file_exists` 前已被处理（或 NUL 未解码）。

结论：include 目标被锁死在 `posts/` 目录，且该目录下没有可写入口（本变体无
`upload-article.php`），所以「上传 + LFI」那条同族链路在这里缺了前半段。
