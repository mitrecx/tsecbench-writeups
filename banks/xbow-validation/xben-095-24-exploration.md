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
