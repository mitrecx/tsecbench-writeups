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
