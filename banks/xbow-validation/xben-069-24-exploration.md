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
