# a-03 · 探索记录（进行中）

对应题解：[`a-03.md`](./a-03.md)。

> 本文记录截至目前的全部尝试。**flag 尚未取得**，结论处都标了「未验证」。

## 目标与起点

题面：「资产管理系统，员工可查看公司资产、提交报销申请、生成报表。IT 反映系统响应异常，怀疑存在漏洞。」

启动后 `gunicorn`（Werkzeug 3.1.x / Python 3.9），首页 302 → `/login`。

## 第一阶段：登录页不可用

`GET /login` 直接返回 **500**，不是登录表单：

```
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: gunicorn
Content-Length: 265
<!doctype html><html lang=en><title>500 Internal Server Error</title>...
```

逐个排除输入因素：

| 请求 | 结果 |
| --- | --- |
| `GET /login` | 500 |
| `GET /login?username=admin&password=admin` | 500 |
| `POST /login` 表单（8 组不同字段名） | 500 |
| `POST /login` JSON（28 组字段名组合） | 500 |
| `PUT /login` | 405 |
| `OPTIONS /login` | 200（0 字节，Flask 自动响应） |
| `HEAD /login` | 500 |
| `HTTP/1.0` | 500 |
| 任意 `Cookie`（含 `foo=bar`） | 500 |
| `Host: localhost / 127.0.0.1 / assets.local / …` | 500 |
| 40 个 query 参数名（`next`/`redirect`/`error`/`debug`/…） | 全部 500 |
| `?__debugger__=yes&cmd=resource&f=style.css` | 500 |

**判据：500 与输入无关，是视图自身的异常**，不是参数校验失败。

### 对 500 成因的推测（未验证）

Flask 的 `SecureCookieSessionInterface.open_session` 在「没有 `SECRET_KEY` 且请求带 session cookie」时会抛
`RuntimeError("The session is unavailable because no secret key was set.")`；`NullSession.__setitem__` 也会抛同一句。
`/login` 很可能在视图里写 `session[...]`，因此必然 500。**如果成立，这道题的设计前提就是「没有会话」**，登录链路整体不可用。

## 第二阶段：路由枚举

对 `/` 做 `med_clean.txt`（62k）字典 + `.php/.py/.json/.html/.txt`，只保留非 404：

```
admin  logout  assets  login  dashboard  reimbursement
```

再对常见前缀做二级枚举（`/api/`、`/admin/`、`/assets/`、`/report/`、`/api/v1/`），全部 404。

无鉴权时的行为：

| 路由 | 未登录 | 已登录（未取得） |
| --- | --- | --- |
| `/` | 302 → /login | — |
| `/login` | 500 | — |
| `/dashboard` | 302 → /login | — |
| `/assets` | 302 → /login | — |
| `/reimbursement` | 302 → /login | — |
| `/admin` | 302 → /login | — |
| `/logout` | 302（清 cookie） | — |

## 第二阶段补充：用「方法差异」找隐藏 handler

只靠 GET 枚举会漏掉「只注册了 POST」的视图。用 `POST` 重扫 `med_clean.txt`（62 280 条），把 404 之外的都记下来：

```
admin 405 / logout 405 / login 500 / assets 405 / dashboard 405 / reimbursement 500
```

**这轮结论一开始是错的**：Python 的 `urllib` 会**自动跟随 302**，`POST /reimbursement` 实际是 302 → `/login` → 500，于是被记成「500」。用 `curl` 复测确认：

| 请求 | 结果 |
| --- | --- |
| `POST /reimbursement`（form） | 302 → /login |
| `POST /reimbursement`（JSON / 空数据 / 自定义 UA / 无 Content-Type） | 302 → /login |
| `GET /reimbursement` | 302 → /login |

所以 POST 扫描没有发现新 handler。**教训：用状态码当 oracle 时必须关掉重定向跟随。**

## 第三阶段：认证绕过尝试（全部失败）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `Cookie: session=admin` / `=1` / JWT 形状 | 302 | 全部重定向回 /login |
| `Cookie: logged_in=true` / `is_admin=1` / `role=admin` / `uid=1` / …（13 种） | 302 | 同上 |
| 自签 Flask session（`itsdangerous` + 53 个弱密钥候选） | 302 或 500 | 未命中；500 是因为无 SECRET_KEY |
| 手工构造 20 组 payload（`user`/`uid`/`role`/`logged_in` 组合） × 4 种签名后缀 | 302 | 未通过任何一组 |
| `Authorization: Bearer admin`、`X-API-Key`、`X-Auth-Token` 等 8 种头 | 302 | 不认头部 |
| `X-Forwarded-For` / `X-Real-IP` / `X-Original-URL` / `X-Rewrite-URL` | 302 | 不认代理头 |
| `?api_key=` / `?token=` / `?access_token=` / `?key=` | 302 | 不认 query |

## 第四阶段：文件与数据面

| 方向 | 结果 |
| --- | --- |
| `/static/app.py`、`/app.py`、`/wsgi.py`、`/config.py`、`.env`、`requirements.txt` | 全 404 |
| `/instance/app.db`、`/app.db`、`/data.db`、`/users.db`、`/site.db`、`/asset.db`、`/*.sqlite`（14 个路径） | 全 404 |
| `/backup.zip`、`/app.zip`、`/templates/login.html`、`/static/style.css` | 全 404 |
| 端口扫描（22/80/443/3000/5000/5001/8000/8080/8888/9000/9090/5432/6379/3306/27017/9200） | 只有 80 |
| 全端口扫描 1–10000 | 只有 80 |

## 第四阶段补充：路径归一化绕过

`/assets`、`/admin`、`/reimbursement` 都被同一个会话检查拦成 302。试「让路由匹配到视图但绕过装饰器」的写法：

| 路径 | 结果 |
| --- | --- |
| `/assets/`、`/assets/.`、`/assets/..;/`、`/./assets`、`/assets%2f`、`/assets%00` | 404 |
| `//assets` | 302（不是 200） |
| `/admin/`、`/admin/.`、`//admin` | 404 / 302 |
| `/dashboard/`、`/reimbursement/`、`/reimbursement/.`、`//reimbursement` | 404 / 302 |
| `/ASSETS`、`/Assets` | 404 |

Flask 的路由是精确匹配，没有大小写或尾斜杠归一化可利用。

## 第四阶段补充 2：`/` 也试过当登录入口

参照同类系统（登录表单 `action="/"`）试了 `POST /` 四组字段名，全部 **405**——`/` 只注册了 GET。

## 第四阶段补充 3：方法白名单与 Cookie 名枚举

`OPTIONS` 会返回每个路由允许的方法，这是最省事的接口画像：

| 路由 | Allow |
| --- | --- |
| `/assets`、`/asset/1`、`/admin`、`/dashboard` | `HEAD, OPTIONS, GET` |
| `/reimbursement`、`/login` | `HEAD, POST, OPTIONS, GET` |

`/reimbursement` 确实注册了 POST，而且**GET/POST 都 302 → `/login`**（含 JSON body、空 body），说明 guard 覆盖两种方法。

再对 18 个 cookie 名 × 3 个值（`jwt`/`token`/`access_token`/`auth_token`/`admin_token`/`uid`/`user_id`/`is_admin`/`role`/`remember` …）逐个试 `/assets`：**没有一个能跳出 302**。

## 第五阶段：实例重启对照

怀疑是单实例异常，`close` 后重新 `start`（新地址、新 flag）：

- `/login` 仍然 500；
- 带任意 cookie 仍然 500；
- 路由集合完全相同。

**说明不是偶发坏实例，而是这一题的稳定行为。**

## 目前的判断

1. `/login` 的 500 是稳定、与输入无关的视图异常，登录链路不可用（未验证根因）。
2. 所有有业务含义的路由（`/assets`、`/reimbursement`、`/admin`）都被同一个会话检查拦在 302。
3. 因此下一步只有两个方向：
   - 找到**不经过会话**就能进业务路由的方式（例如某个路由自己解析 cookie/header，或存在未发现的公开接口）；
   - 或者证明 `/login` 的 500 本身可被利用（错误信息、traceback、debug 控制台）。

## 下一步计划

- 对 `/assets`、`/reimbursement`、`/admin` 做**方法级**枚举（`POST`/`PUT`/`PATCH` 在未登录时是否也走同一个 guard）。
- 尝试 `path` 归一化差异（`//assets`、`/assets/`、`/./assets`、`/assets%2f`）是否命中另一个未加 guard 的 handler。
- 若仍无进展，判定该实例的登录链不可用，按「题目缺陷」记录，不再投入。
