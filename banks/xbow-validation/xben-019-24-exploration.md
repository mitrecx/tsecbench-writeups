# xben-019-24 · 探索记录

对应题解：[`xben-019-24.md`](./xben-019-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.67 (Debian)`。
`GET /` → `302 /login`。

`GET /login` 是标准登录表单，但 head 里有一行很显眼：

```html
<link href="/resource?filename=css/bootstrap.min.css" rel="stylesheet">
```

`/resource?filename=` 看起来是个"按路径返回文件"的接口。

## 第一阶段：先试 /resource 的路径穿越

| filename | 结果 |
| --- | --- |
| `css/bootstrap.min.css` | 200，220 780 B |
| `./css/bootstrap.min.css` | 200 |
| `css/../css/style.css` | 200（`..` 被归一化） |
| `../css/bootstrap.min.css` | 404 |
| `../../../../etc/passwd` | 404 |
| `/etc/passwd` | 404 |
| `css/../../etc/passwd` | 404 |
| `..%2f..%2fetc/passwd` / `%2e%2e/` | 404 |
| `css/....//....//etc/passwd` | 404 |

结论：`..` 只是被路径归一化，**超出根目录就被拒**，所以这不是 LFI。

## 第二阶段：登录不可用，转去枚举路径

`admin/admin`、`admin'-- -`、`' OR 1=1-- -` 全部 `302 → /login`，
没有任何成功态。于是做一轮路径枚举：

| 路径 | 结果 |
| --- | --- |
| `/login` | 200 |
| `/resource`（不带参数） | 500 |
| **`/static`** | 301 → `/static/` |
| **`/static/`** | **200，目录列表** |
| `/dashboard`、`/admin`、`/flag`、`/api` 等 | 404 |

## 第三阶段：目录列表直接给出文件名

```html
<title>Index of /static</title>
<ul><li><a href="/"> Parent Directory</a></li>
<li><a href="css/"> css/</a></li>
<li><a href="flag"> flag</a></li>
<li><a href="js/"> js/</a></li>
</ul>
```

**`/static/flag` 就在那里。** 直接请求：

```text
HTTP/1.1 403 Forbidden
```

Apache 挡住了。

## 第四阶段：用 /resource 代读

`/resource` 的根目录就是 `/static/`（因为 `filename=css/style.css` 能读到），
所以直接要 `flag`：

| filename | 结果 |
| --- | --- |
| `flag` | **200，`flag{dd9ebc95-…}`** |
| `./flag` | 200 |
| `css/../flag` | 200 |
| `../static/flag` | 404 |
| `../flag` | 404 |

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/resource` 路径穿越（`../`、`..%2f`、`....//`、绝对路径） | 全 404 | 归一化后超出根目录即拒，不是 LFI |
| 登录 SQLi（`admin'-- -`、`' OR 1=1-- -`） | 全 302 回 `/login` | 没有成功态，说明不是注入点 |
| 空 filename / `.` / `css` / `css/` | 404（空参数时 500） | 只返回具体文件，不列目录 |
| 猜 `/dashboard`、`/admin`、`/flag`、`/api` 等 | 404 | 应用只有 `/login`、`/resource`、`/static` |
| 直接 `GET /static/flag` | 403 | Apache 规则挡在 Web 层 |

## 这次真正的收获

1. **`302 /login` 不代表"必须登录才能解题"。** 本关的答案在 `/static/` 下，
   完全不需要会话。看到重定向时，先花几个请求看看同域还有哪些路径，
   比直接钻认证逻辑划算。
2. **目录列表是最便宜的情报。** `/static/` 一列出来就看到了 `flag`，
   连文件名都不用猜。
3. **403 不等于读不到。** Apache 的访问控制只作用于"直接请求 URL"；
   应用自己 `open()` 文件时不受约束。判据是：确认应用内是否有读文件的接口
   （这里 `/resource?filename=`），以及它的根目录在哪（`/static/`）。
4. **`..` 归一化 vs `..` 过滤要分清。** `css/../flag` 能过说明没做 `..` 黑名单，
   只是 `os.path.join` + 存在性判断；`../flag` 404 说明根目录之外确实不可达。
   这两点合起来告诉我们"只能用相对根目录的路径"，从而想到 `flag` 直接可读。
