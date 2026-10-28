# xben-069-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

IDOR Eats（PHP）。`api/get_favorite_restaurants.php?userId=N` 是明显 IDOR：能列出 `admin`（id=7）与 `not_that_easy`（id=1，拥有全部 8 家餐厅）的收藏，但响应只有 `username` + 餐厅名，没有 flag 字段；`change_password.php` 只能改自己。未拿到 flag。

## 尝试过程

- `/api/register.php`、`/api/login.php` 正常，会话是 `PHPSESSID`。
- `get_favorite_restaurants.php?userId=1..15`：1 是 `not_that_easy`（8 家全有），7 是 `admin`。
- `change_password.php` 额外传 `userId`/`username` 无效，仍校验自己的旧密码。
- 目录枚举只有 `api/`、`include/`（`include/config.php` 空）、`static/`。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28（新 token 场次）

- `/api/` 目录索引直接列出 5 个文件：`change_password.php`、`get_favorite_restaurants.php`、
  `login.php`、`register.php`、`search_restaurants.php`（没有其它端点）。
- `search_restaurants.php?query=` 对 `a'`、`a' UNION SELECT ...` 都返回 `{"data":[]}`，
  而 `query=%` 返回全部行——说明参数是参数化/转义的，只有 LIKE 通配符生效，没有注入。
- `get_favorite_restaurants.php?userId=N` 是纯整数：`7 AND 1=2`、`7 OR 1=1` 仍返回 admin 的数据，
  `0 UNION SELECT ...` 返回空，说明参数被强类型化。
- `change_password.php` 的 `userId` 参与更新但 `oldPassword` 必填且校对：
  `oldPassword=''`、`' OR '1'='1`、`userId=7 OR 1=1` 全部失败（后者 500）。
- 结论：这个实例的 IDOR 只能读到「用户名 + 餐厅名」，flag 不在其中；需要别的原语。
