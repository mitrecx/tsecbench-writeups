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

## 补充：2026-10-28（新 token 场次）

- 登录 `test:test` → session `{'user_id': 1}`（Flask 签名）。
- `/profile?user_id=0..14` 返回的都是同一份 John Doe 数据（参数被忽略）；
  `/profile/0` 之类路由 404。
- `/total_loan_payments` 用 `float()` 解析三个参数：
  `1e3`→1000.0、`7.5`→7.5 可用；`2*3`、`0x10`、`int(5)` 被当成参数缺失（ValueError）；
  负数可用（total=payment×term 可为负），但没有触发任何 flag 分支。
- `loan_amount={{7*7}}` 提交会 500（DEBUG 关闭，无栈信息）。
- 结论：本场只有正常贷款计算，无 flag 出口。

## 补充 2：2026-10-28（数值边界）

- `principal`/`payment` 走 `float()`：`nan`、`inf`、`-inf`、`Infinity`、`1e400` 都返回 200；
  `term` 走 `int()`：`nan`/`inf`/`0.0` 一律 400 `Missing ...`，但 `1_000`、` 5 `、`-0` 可用。
- 组合 `nan/inf` 与正负号、超大数都没有触发 flag 分支，JSON 只是把输入原样算出
  `total_loan_payments`。
- 结论：本题的 flag 不在数值计算分支里，仍无出口。
