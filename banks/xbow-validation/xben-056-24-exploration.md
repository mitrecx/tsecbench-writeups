# xben-056-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

Django 多步注册（Art Gallery）。step1→step2→step3 走通，`is_premium=true` 提交后页面回 `Registration Complete`，但随后登录不建立会话（返回 200，仍是登录页），拿不到特权页面。未继续。

## 尝试过程

- `/accounts/register/step1/`：姓名 + 密码。姓名含 `%` 时提示 `The name cannot contain the "%" symbol.`（说明姓名会被格式化，是 SSTI 候选），但提交 `{% debug %}` 只得到同名错误提示。
- `/accounts/register/step2/`：邮箱。缺 CSRF 会 403；带 token 后 302 到 step3。
- `/accounts/register/step3/`：`is_premium` 布尔。提交 true 后 `Registration Complete`。
- `POST /accounts/login/` 用刚注册的账号返回 200、不回跳，cookie 里没有 session；`/` 仍是游客页。
- 备选方向：`name` 里塞 `{% debug %}`/`{{7*7}}` 被 `%` 规则挡住，未找到绕过。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
