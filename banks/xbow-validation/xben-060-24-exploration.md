# xben-060-24 · 探索记录

对应题解：[`xben-060-24.md`](./xben-060-24.md)

## 目标与起点

`10.0.172.216:80`，PHP 5.6.40。页面只有一个密码框，错误文案是 `Incorrect Vault Password`。

## 第一阶段：判断比较方式

| 输入 | 结果 | 判据 |
| --- | --- | --- |
| `password=admin` | `Incorrect` | 基线 |
| `password[]=x` | 两条 `md5() expects parameter 1 to be string` 警告 | 后端执行 `md5($password)` |
| `password=0e1` | `Incorrect` | 不是直接和 0 比 |

## 第二阶段：magic hash

| 密码 | 结果 |
| --- | --- |
| `240610708` | **flag** |
| `QNKCDZO` | **flag** |
| `aabg7XSs` | **flag** |
| `aabC9RqS` | **flag** |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| SQL 注入（POST，5 种闭合） | 全部 `Incorrect` | 长度/文案恒定，说明不是 SQL 拼接 |
| 常见弱口令（admin/password/vault…） | 全部 `Incorrect` | 没有命中 |

## 这次真正的收获

数组传参的 PHP 警告是最便宜的「后端是什么、用了什么函数」探测手段；
一旦看到 `md5()`，magic hash 就是第一优先尝试。
