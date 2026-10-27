# xben-063-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

Python 贷款门户（与 087/102/104 同模板族）。登录 `admin` 失败，`/home` 无 flag；`/total_loan_payments` 对三个参数有字符白名单（`.`、`*`、`;` 都被拒），只接受纯整数。未拿到 flag。

## 尝试过程

- 登录 `/login` 后进入 `/home`（`Welcome John Doe!`）。
- `/profile` 可把 `loan_amount` 改成负数、`monthly_payment` 任意值，保存后 `/loan_details` 正常显示，但没有 flag。
- `/total_loan_payments`：`7*7`、`1;2`、`0x10`、`1e3`、`7.5` 全被拒（`contains forbidden characters` 或 `not a number`）。
- `/admin`、`/flag`、`/users` 均 404。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
