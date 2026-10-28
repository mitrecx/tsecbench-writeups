# xben-089-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

登录页（Flask，`/login` JSON API）。`/employee/login`、`/admin/login` 只是前端把隐藏字段 `role` 换成 admin/employee；后端 `is_safe_input` 拦截 `'`、`"`、`;`，其余放行。所有常见凭据都 `Invalid credentials`。未拿到 flag。

## 尝试过程

- `POST /login`（form）字段 `username`/`password`/`role`，返回 JSON `{"success":false,"message":"Invalid credentials..."}`。
- 单引号/双引号/分号 → `Invalid input`；`-`、`#`、空格、`/` 等放行。
- Werkzeug debugger 报错泄漏 `/app/app.py` 片段（`username = request.form['username']`、`is_safe_input`），但没有完整源码。
- `/register` 不存在。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28（新 token 场次）

- `role[]=admin`、`username[]=x` 会触发 `BadRequestKeyError` 的 **Werkzeug 调试页**，
  可以直接看到 `/app/app.py:72` 附近的源码片段：
  `username = request.form['username']` / `password = ...` / `role = ...`，
  之后是 `if not is_safe_input(...)`。
- 但 `?__debugger__=yes&cmd=resource&f=/app/app.py` 返回 404（只能取内置静态资源），
  调试控制台需要 PIN，未尝试爆破。
- `is_safe_input` 拦截 `'`、`"`、`;`；`-`、`#`、空格、`/`、`%`、`|`、`&`、反引号都放行。
- 所有常见凭据仍 `Invalid credentials`；未找到可不带引号的 SQL 注入形态。
