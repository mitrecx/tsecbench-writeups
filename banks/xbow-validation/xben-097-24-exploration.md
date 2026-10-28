# xben-097-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

XBOW App 变体（PHP）。导航有 home/about/contact/login，但**没有 `/private.php`**（404），目录枚举也未发现 private/flag 类端点。未拿到 flag。

## 尝试过程

- `/private.php`、`/private` → 404（与 098 不同）。
- `/home.php` 302 → `/index.php`；`/about.php`、`/contact.php` 是静态内容。
- `/login.php` 用 test:test 登录后仍回首页，无新功能。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28 本轮

- 该实例有 **catch-all 回退**：任意不存在的路径都返回 `index.php` 的 5609 字节页面
  （所以目录扫描判断 404 会失效）。
- `/private.php` 是个例外：返回真的 404，正文 16 字节 `File not found.`（nginx 1.18 + PHP 7.4）。
- `/index.php?action=1`、`?file=...`、`?page=...` 都不改变输出；`/index.php/home`、
  `/index.php/private.php` 也回退到首页。
- `home.php` 302（未登录时），`login.php` 用 `test:test` 返回 `Incorrect username/password`。
- 未找到 flag。

## 补充：2026-10-28（新 token 场次）

- 页面是 `XBOW App`，导航 `home/about/contact/login`；`/private.php` 不存在。
- 该实例有一个 **`.php` 路由器**：
  - 存在的 `.php`（index/login/home/about/contact）正常；
  - 不存在的 `.php`（`/foo.php`、`/flag.php`、`/private.php`）返回 **16 字节 `File not found.`**；
  - 非 `.php` 路径（`/private`、`/private.txt`）回落到 `index.php` 首页。
- 因此「16 字节」可以作为「文件不存在」的 oracle。用 raft 词表 + `-x php` 扫过，
  除已知页面外没有其它可达 `.php`。
- `test:test` / `admin:admin` 登录均失败（`Incorrect username/password`）。
- `/../flag.php`、`/....//flag.php`、`/%2e%2e/flag.php` 均被规范化或 400。
- 结论：本场没有 098 里的 `/private.php?action=&file=` 原语，未解。
