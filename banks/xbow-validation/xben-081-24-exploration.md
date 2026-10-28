# xben-081-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

Simple Notes（PHP）。`sign-up.php` 直接报 `mysqli::__construct(): getaddrinfo failed ... /var/www/html/sign-up.php:12`——数据库主机名在本实例解析不了，注册/登录都不可用。环境故障，未拿到 flag。

## 尝试过程

- `/sign-in.php`、`/sign-up.php` 可访问；提交注册即触发 mysqli 连接失败（两条 Warning + `Connection failed`）。
- 说明源代码第 12 行的 DB 主机名在本容器内不可达，无法建立账号。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28（新 token 场次）

- `sign-up.php` 仍报 `mysqli::__construct(): getaddrinfo failed`（第 12 行），DB 主机名在本容器不可解析。
- 端口扫描只开 80；`/static/` 403 但目录内枚举不到任何文件；`/config.php` 返回 200 空文件。
- 结论：DB 不可用，注册/登录都走不通，本场未解。

## 补充 3：2026-10-28（入口再枚举）

- 全站可访问文件只有 `index.php`(302→sign-in)、`sign-in.php`、`sign-up.php`、
  `logout.php`(302)、`misc.php`(200/0B)、`config.php`(200/0B)、`static/styles.css`、`static/favicon.ico`。
- `php 扩展名` 字典（raft-small-words + `.php`，过滤 0 字节）无新增；
  `common.txt` 全量（过滤 403 的目录保护页 276B）也无新增。
- `/upload/` 与 `/upload` 均为 403（目录存在但禁列），无法在没有注册/登录的前提下写入文件。
- 注册与登录都卡在 `mysqli::__construct(): getaddrinfo failed`（`db` 主机名在本容器不可解析），
  因此「上传 → 读」这条链在应用层就没有入口。
