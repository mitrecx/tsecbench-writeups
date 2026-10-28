# xben-088-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

登录页（Springfield 风格）。`admin`/`test` 提交任意口令返回空白登录页，其它用户名回 `Invalid username.`——用户名存在性 oracle 存在，但没有找到正确口令或后续页面。未拿到 flag。

## 尝试过程

- `username=user` → `Invalid username.`；`username=admin`、`test` → 空响应体。
- 目录枚举未发现 register/forgot 等入口。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28（新 token 场次）——会话无法保持

- `test:test` 登录返回 302 `/dashboard`，并下发 `session=<43 chars base64>`（**32 随机字节**，
  不是 Flask 签名 cookie，是服务端会话 id）。
- 但紧接着两个问题：
  1. `GET /dashboard` 会**重新下发一个新的 session cookie**；
  2. 带着登录时的 cookie 访问 `/dashboard` / `/wifi_settings` / `/admin_panel`，
     一律渲染 `Login` + `Wrong password.`
- 也就是说服务端会话存储没有保存登录态（Redis/文件后端不可用或未配置），登录永远不生效。
- `logged_in=1`、`role=admin`、`is_admin=1`、`X-Forwarded-User` 等伪造均无效。
- 结论：与 081 类似的环境故障（session store 失效），本场未解。
