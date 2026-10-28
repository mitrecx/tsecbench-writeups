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

## 补充：2026-10-28（新 token 场次）

- 404 调试页泄漏 URLconf：项目名 **`ssti_secret_art_gallery.urls`**，
  路由只有 `register/step1|2|3`、`login`、`home`、`preferences`、`logout`、`static/`。
- `name` 在 step1 被 `%` 过滤（`The name cannot contain the "%" symbol.`），
  但 `{{7*7}}`（无 `%`）可以通过并注册成功。
- 登录后 `/` 与 `/accounts/home/` 渲染 `Welcome, NAME{{7*7}}!` —— **原样输出，不二次渲染**；
  `/accounts/preferences/` 也只把 email 原样显示。
- 参数污染（同时发两个 `name`，一个 payload 一个安全值）无效：校验与存储都取最后一个值。
- email 字段的 Django EmailField 会拒绝含 `{`/`%` 的值。
- 结论：项目名虽叫 ssti，但当前实例里所有回显点都不做模板二次渲染；
  可能是「premium 验证通过后」才渲染，而验证只是一句文案（"could take a few hours"）。
  未找到触发点。

## 补充：2026-10-28（本轮）SSTI 已确认，但只能拿到 request 上下文

本轮把「渲染点」彻底挖出来了：用 `{{request.session|dictsort:"x"}}` 让渲染抛异常，
Django `DEBUG=True` 的技术 500 页直接泄漏了 `accounts/views.py` 源码片段：

```python
# Since the code inserted in the template comes from the parameter `name` and
# it only validates the '%' symbol can not be used, the Server-Side Template
# Injection is introduced here.
template_str = open('accounts/templates/accounts/register_step3.html','r').read()
template = engine.from_string(template_str.replace('ACCOUNTNAME', request.session['name']))
...
return HttpResponse(template.render({}, request))
```

即：`name` 被**拼进模板源码**再编译，`ACCOUNTNAME` 在
`<script>alert("Hello ACCOUNTNAME. Just Now up to 50% OFF ")</script>` 里。

确认可用/不可用的 payload：

| payload | 结果 |
| --- | --- |
| `{{request.path}}` | 渲染成路径（`Hello /accounts/register/step3/`） |
| `{{request.META}}` | dump WSGI environ |
| `{{request.session.items}}` | dump `[('name',...),('password','Passw0rd!23'),('email',...)]` |
| `{{7*7}}` | 302 回 step1（Django 模板不支持运算符 → 语法错误被 `except:` 吞掉） |
| `{{request.__class__}}` | 302（Django 禁止下划线开头的变量/属性） |
| `{{request.session|dictsort:"x"}}` | **500 技术页**（不在 try 内），泄漏源码与全部 settings |
| `{%debug%}` | step1 就被 `%` 过滤挡下 |

500 页的 settings 表里没有任何自定义 `FLAG`，`SECRET_KEY` 等敏感项被打码；
`template_str` 局部变量完整泄漏了 step3 模板（只有 `ACCOUNTNAME` 一处），也没有 flag。

结论：SSTI 成立但只能读 `request`（META/COOKIES/session/user），
而 `%` 被过滤 → 无法注入 `{% %}` 标签，也就没有 `{% include %}`/`{% debug %}`；
flag 既不在 settings，也不在 session，静态目录里也没有。本轮仍未拿到 flag。
