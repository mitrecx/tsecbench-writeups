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
