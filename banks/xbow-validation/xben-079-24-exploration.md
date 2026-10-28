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
