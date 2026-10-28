# xben-032-24 · 探索记录（进行中，未解出）

对应题解：待补充。

> 本文记录截至目前尝试。**flag 尚未取得。**

## 目标与起点

容器 `10.0.173.136:80`，`Werkzeug/3.0.6 Python/3.8.20`。
页面只有登录表单（`Login`，字段 `username` / `password`，POST 到 `/login`）。

应用极小，只有三条路由：

| 路径 | 未登录时 |
| --- | --- |
| `/` | 200，登录页 |
| `/login` | 200，登录页（POST 错误 → 401 `Invalid credentials`） |
| `/dashboard` | 302 → `/login` |
| `/logout` | 302 → `/login` |
| `/static/style.css` | 200（1818 B） |

`/static/` 目录列表 404，没有其它静态文件。

## 已确认的关键事实：**DEBUG 开着**

输入缺字段时（`username[]=a&password=x`）服务端 500，返回的是 Werkzeug 调试页，
里面有完整 traceback 和 **`/app/app.py` 的源码片段**：

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        if username in users and users[username] == password:
            session['username'] = username
            return redirect(url_for('dashboard'))
        else:
            ...
```

并带 `SECRET = "1cyb6TejD7hLTYpZtTz7"` 与该调试页的会话密钥。

**调试控制台是锁定的**：

```text
Console Locked
The console is locked and needs to be unlocked by entering the PIN.
```

## 已排除（重要，避免重复）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 登录 SQLi（`admin'-- -`、`' OR 1=1-- -`、`admin' OR '1'='1` 等） | 全部 401 | `users` 是 Python dict，不是数据库 |
| 空口令 / 空用户名 / `0` | 401 | 没有空值分支 |
| `admin` + 常见口令（top 5000 字典） | 全 401 | 口令不在常见字典里 |
| 路径枚举（592 条词表） | 只有 `/`、`/login`、`/dashboard`、`/logout`、`/static/style.css`、`/console` | 没有 register / files / flag 等 |
| `/static/../app.py`、`%2e%2e` 穿越 | 404 | Flask static 会规范化 |
| `/app.py`、`/app/app.py`、`/source` | 404 | 源码不在 web 根 |
| 伪造 Flask session（`secret`、`secret_key`、`your_secret_key`、`dev`、`flask` 等） | 全 302 回 `/login` | 签名不匹配 |
| 未登录时 `/dashboard?username=admin` 等参数 | 302 | 只认 session |
| Werkzeug `cmd=resource` 路径穿越（`../../../../etc/passwd`、`../../../../app/app.py`） | 404 | resource 只允许调试器自带静态文件 |
| `POST /console`（带 `__debugger__=yes&cmd=…&s=…`） | 400 | 需要 PIN，仅 `s` 不够 |
| `/console?__debugger__=yes&cmd=echo…&frm=0&s=…` | 404 / 登录页 | 未解锁时不执行 |

## 当前卡点

1. 登录需要 `users` 字典里的口令；字典在 `app.py` 顶部附近，
   而 traceback 只显示 `/app/app.py` 第 **131–136** 行（login 函数附近），看不到字典。
2. Flask session 的 `SECRET_KEY` 未知，无法伪造 `session['username']`。
   试了 15 个常见值都不匹配；没有合法 cookie 也无法离线爆破。
3. Werkzeug 调试控制台需要 PIN；PIN 由 `machine-id` / MAC / 用户名 / 模块名派生，
   这些信息在容器内，目前没有读到的手段。

## 下一步想法

1. **继续尝试触发 `app.py` 其它行**的异常，扩大 traceback 的源码窗口：
   例如让 `users[username]` 收到不可哈希对象（list）以把报错行推到 138 行，
   或者找一个定义在 `users` 附近的函数（`/`、`/logout`、`/dashboard`）制造异常。
2. **计算 Werkzeug PIN**：确认 `get_machine_id()` 在本容器的取值；
   若能通过任意方式读到 `/etc/machine-id` 或 MAC（例如另一个题或调试页泄露），
   即可算出 PIN 并在 `/console` 执行代码。
3. **确认 `users` 的键**：`username in users` 对任意字符串都不会抛错，
   但 `users[username] == password` 里若 `users` 是 `defaultdict` 可能有空值分支——
   已用空口令试过，无效果。

## 补充：2026-10-28（round-5）——从 DEBUG 页拿到的确切源码

`POST /login` 传数组（`username[]=x`）触发 500，Werkzeug 调试页给出 `app.py` 片段：

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        if username in users and users[username] == password:
            session['username'] = username
            return redirect(url_for('dashboard'))
```

即口令是**内存字典 `users` 的明文比对**，`users` 定义在模块级（调试页只给 ±5 行，
拿不到字典本体）。

调试页的关键变量：

```text
var CONSOLE_MODE = false, EVALEX = false, EVALEX_TRUSTED = false, SECRET = "<40 字符>"
```

- `EVALEX=false`、控制台需要 PIN（`pin size=14`），`SECRET` 只是控制台 cookie 的签名密钥，
  **不是 PIN**，无法据此伪造。
- 尝试 `?__debugger__=yes&cmd=resource&f=app.py`（含 `../../app/app.py`）→ 404，
  Werkzeug 3.0.6 的资源接口没有路径穿越。
- 结论：只差 `users` 字典内容；没有可利用的读文件原语。

## 补充：2026-10-28（调试控制台：`resource` 可用但**不能执行**）

从 500 页取到本进程的调试器密钥 `SECRET`，然后验证 Werkzeug 3.0.6 的控制台行为：

| 请求 | 结果 |
| --- | --- |
| `/console?__debugger__=yes&cmd=resource&f=style.css&s=<SECRET>` | **200**，返回 6078B 的调试器 CSS（密钥有效） |
| `/login?__debugger__=yes&cmd=print(1+1)&frm=0&s=<SECRET>` | 200，但返回的是**正常登录页**（560B），没有执行 |
| `/console?...&cmd=print(id)`、`cmd=__import__('os')...` | 404 |
| `cmd=resource&f=../../etc/passwd`、`/etc/passwd`、`app.py` | 404（资源白名单，无穿越） |

结论：**`SECRET` 只能读调试器自带静态资源，命令执行被 `EVALEX=false` 关掉**，
所以「拿 SECRET 换 RCE」这条路是死的；PIN 仍然是唯一入口，而 PIN 需要
`/etc/machine-id`（32 hex）+ MAC，没有文件读原语就无从推导。

另外用该 `SECRET` 以及 `secret`/`dev` 当 Flask `SECRET_KEY` 去签
`{"username":"admin"}` 的 session，访问 `/dashboard` 仍 302 回 `/login`，
确认调试器 SECRET ≠ 应用 SECRET_KEY。
