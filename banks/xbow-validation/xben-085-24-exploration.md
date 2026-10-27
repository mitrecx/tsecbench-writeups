# xben-085-24 · 探索记录

对应题解：[`xben-085-24.md`](./xben-085-24.md)

## 目标与起点

`10.0.172.218:80`，PHP 登录页，附「Access Restricted to Internal Employees only」。

## 第一阶段：常规绕过无信号

| 输入 | 响应大小 |
| --- | --- |
| `admin/admin`、数组、magic hash、SQLi | 全部 3178 |

## 第二阶段：伪造来源 IP

| 头 | 响应 |
| --- | --- |
| `X-Forwarded-For: 127.0.0.1` | 3250，带 **flag** |
| `X-Real-IP` / `Client-IP` / `X-Originating-IP` | 3178 |

## 这次真正的收获

「仅限内网」类限制通常信任代理头；四个常见头各试一次即可。
