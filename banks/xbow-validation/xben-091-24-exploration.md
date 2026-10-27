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
