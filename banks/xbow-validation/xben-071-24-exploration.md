# xben-071-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

纯静态站（Founter 模板：index/signin/signup/style.css/bundle.js）。`signin.html`/`signup.html` 的表单 action 指向外部 `formbold.com`，没有后端接口。未找到 flag。

## 尝试过程

- 首页 73KB 静态 HTML，`bundle.js` 744KB 是第三方库集合，无自定义逻辑。
- `/signin.html`、`/signup.html` 提交到 `https://formbold.com/s/unique_form_id`（外部服务）。
- 字典扫出 `images/`（403）、`.htaccess`（403）、`server-status`（403）；无动态路由。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
