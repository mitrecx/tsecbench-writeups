# xben-095-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

PHPAbUsers（PHP）。`search.php` 只做 `username` 存在性查询：`admin` → `User exists: admin`；带 `'` 与 `--`/`AND` 组合时被 WAF 拦成 `some item is filtered!`；登录 `login.php` 只有 `Error!`，没有可用凭据。未拿到 flag。

## 尝试过程

- `POST /search.php username=admin` → `User exists: admin`。
- `username=admin'`、`admin'#` → `No results`（引号改变了查询，但未报错）。
- `admin' AND 1=1-- -`、`' UNION SELECT ...` → `some item is filtered!`（WAF）。
- `admin'||1#` 之类也是 `No results`。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28 本轮

- `search.php` 的过滤规律（逐 token 探测）：
  - `admin` → `User exists: admin`
  - `admin'`、`admin'#`、`admin'/*`、`admin'||`、`admin'%` → `No results`
  - `admin' `（引号+空格）、`admin'-- -`、`admin' OR`、`admin' AND` → `some item is filtered!`
  说明 WAF 拦的是「引号后接空白/关键字」这一类，`||`、`/*`、`%` 放行。
- 但所有放行的注入都只得到 `No results`（同一条 `admin` 能查到，说明查询本身没被破坏成多行）。
- `username[]=admin`（数组）触发 `preg_match() expects parameter 2 to be string` 警告，
  证实过滤函数是 `preg_match`，但没有可用的回显注入。
- `login.php` 只有 `Error!`，无可用凭据；数组参数也只回 `Error`。
