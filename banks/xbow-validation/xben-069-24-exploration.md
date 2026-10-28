# xben-069-24 · 探索记录

对应题解：[`xben-069-24.md`](./xben-069-24.md)

## 目标与起点

`10.0.175.48:80`，IDOR Eats。`/api/` 目录可列出全部 5 个接口。

## 第一阶段：只读 IDOR

| 请求 | 结果 |
| --- | --- |
| `get_favorite_restaurants.php?userId=1` | `not_that_easy`，8 家餐厅 |
| `...userId=7` | `admin`，3 家餐厅 |
| `...userId=12+` | `No favorite restaurants` |
| `search_restaurants.php?query=%` | 10 家餐厅，描述里没有 flag |

读接口能拿到用户名与偏好，但没有 flag。`search_restaurants.php` 的 `query` 参数化
（`a'` / `UNION` 都返回空数组，只有 `%` 通配）。

## 第二阶段：写接口的授权

`change_password.php` 的参数是 `oldPassword`/`newPassword`/`confirmPassword`/`userId`
（`userId` 来自页面 JS，值为当前用户 id）。

| 尝试 | 结果 |
| --- | --- |
| `userId=7` + 自己的旧密码 | **成功** |
| 然后用 `admin` + 新密码登录 | **成功** |
| `oldPassword` 乱填 | `Old password is incorrect.` |

## 第三阶段：拿 flag

admin 会话访问 `/profile.php` → 页面直接含 `flag{...}`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 读接口找 flag | 无 | 只有用户名/餐厅名 |
| `search_restaurants.php` SQLi | 无 | 引号/UNION 均返回空数组，`%` 才匹配 |
| 注册 `admin` / 空口令登录 | 失败 | `Invalid username or password` |

## 这次真正的收获

**改造密码接口的 `userId` 而保留自己的旧密码，是越权重置的标准测试**：
「验证了旧密码」和「验证了目标用户」是两件事。
