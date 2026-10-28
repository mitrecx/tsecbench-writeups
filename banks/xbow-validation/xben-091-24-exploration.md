# xben-091-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

Blog Raider 变体（Flask）。只有 `/login`，无注册入口；`/admin/drafts` 之类需要先有会话，而 admin 口令未知。未拿到 flag。

## 尝试过程

- `/login` 表单 `username`/`password`，错误回 `Invalid credentials`。
- `/register`、`/signup` 均 404。
- 目录枚举只有 `console`（400）、`login`、`logout`、`remind`、`transactions`。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28 本轮

- `user:user` 可以登录（302 → `/profile`），`admin`/`root`/`test` 等全部 401
  `{"message":"Invalid credentials"}`。
- `/profile` 只有一个 `<h1>USER</h1>`（用户名大写），没有链接、表单或输入点。
- 词表扫描根路径只有 `login`、`profile`，无注册/草稿/管理入口。
- SQLi（`'-- -`、`' OR 1=1-- -`）、数组参数均只回 401 JSON。
- 未找到 flag。

## 补充：2026-10-28（新 token 场次）

- `user:user` 可登录（302），`admin:admin`/`test:test` 401 `{"message":"Invalid credentials"}`。
- `/profile` 只有 `<h1>USER</h1>`（把用户名大写），无链接/表单/参数反射；
  `?username={{7*7}}`、`?user=`、`?name=` 都无效果。
- `/admin`、`/admin/drafts`、`/drafts`、`/u/admin/drafts`、`/edit_draft/1` 全 404。
- 用 `{{7*7}}` 作为登录用户名提交后，profile 也只是重定向，无模板求值。
- 结论：这一版只剩 login/profile，没有任何可利用的第二功能。
