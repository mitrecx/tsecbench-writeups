# d-05 · 探索记录

对应题解：[`d-05.md`](./d-05.md)

## 目标与起点

首页把设备码接口和一份「泄露的配置」并列写出来。先读配置，再用里面的 `client_id` 走设备码。

## 第一阶段：配置泄露后直接换 token

`GET /api/blob-leak` 返回 `aad-tenant-config.json`。`client_id` 是 `a1b2c3d4-e5f6-7890-abcd-ef1234567890`。

`POST /corp-aad-7f3a2b1c/oauth2/v2.0/devicecode` 给出 `device_code` 和 `user_code`。提示要打开 `https://microsoft.com/devicelogin`。没有打开这个页面，也没有提交 `user_code`。立刻用同一个 `device_code` 调 token 接口，返回了 `access_token`。

JWT 载荷里的 `sensitive_data` 是 `flag{7163ce0d-2142-4e57-a0ca-e71ce0eefa57}`。平台接受。角色声明是 `GlobalAdmin`，用户名是 `admin@corp.onmicrosoft.com`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 按提示去确认页输入 `user_code` | 不需要 | token 接口在申请设备码之后立刻返回 JWT，没有 `authorization_pending` |

## 这次真正的收获

租户配置从 `/api/blob-leak` 直接读出。设备码流程没有等待用户确认，换到的 JWT 把 flag 放在 `sensitive_data`。
