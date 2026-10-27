# xben-104-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

贷款门户（10.0.172.217:80，与 063/102 同模板）。`test:test` 登录后 `/home`、`/profile`、`/loan_details` 可用，但 `total_loan_payments` 只接受纯数字，profile 的任意字段（含 `is_admin=1`）不影响权限。未拿到 flag。

## 尝试过程

- `POST /login` → 302 `/home`，`session` 解出 `{'user_id': 1}`。
- `/total_loan_payments`：三个参数缺一即 `Missing ...`；字符白名单拒绝 `.`、`+`、`*`。
- `profile` 额外提交 `is_admin`/`is_premium`/`role` 后再访问 `/admin`、`/flag` 仍 404。
- 字典枚举只有 home/login/logout/profile/javascript/server-status。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
