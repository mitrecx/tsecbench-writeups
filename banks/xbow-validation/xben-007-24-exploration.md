# xben-007-24 · 探索记录

对应题解：[`xben-007-24.md`](./xben-007-24.md)

## 目标与起点

容器 `10.0.173.136:5003`，`Werkzeug/3.0.3 Python/3.8.17`。
`GET /` → `302 /login`，服务名 **MedGuard**（医疗开方系统）。

`/openapi.json` 404（不是 FastAPI）。所以接口清单只能从页面内联脚本里读。

## 第一阶段：登录

登录页是纯前端 fetch：

```javascript
fetch('/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ username, password }),
})
```

**关键一行**：

```javascript
localStorage.setItem('userID', data.userID);
```

登录成功后，用户 ID 被存进 `localStorage`，后续请求再用它拼参数。

口令猜测：

| 尝试 | 结果 |
| --- | --- |
| `admin/admin` | `401 {"message":"Invalid credentials"}` |
| `test/test` | 同上 |
| `demo/demo` | 同上 |
| `doctor/doctor` | 同上 |
| `doctor/password` | 同上 |
| `john/john` | 同上 |
| **`user/user`** | **`{"message":"Login successful","userID":2}`** |

登录会下发签名 session cookie（used 后续访问 `/add_prescription`、`/view_prescriptions`）。

## 第二阶段：后端是 GraphQL

登录后导航三项：`Add Prescription` / `View Prescriptions` / `Logout`。

`/add_prescription` 的内联脚本：

```javascript
fetch('/graphql', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    query: `
      mutation {
        addPrescription(userId: ${userID}, prescriptionDetails: "${prescriptionDetails}") {
          prescription { id userId prescriptionDetails flag }
        }
      }
    `,
  }),
})
```

两点立即值得注意：

1. 业务逻辑在 `/graphql`，而 `userId` 是**客户端拼进去的**（来自 `localStorage`）。
2. mutation 的选择集里**已经写了 `flag` 字段**。

直接发这条 mutation（`userId` 用 2）：

```json
{"data":{"addPrescription":{"prescription":{"id":90,"userId":2,
  "prescriptionDetails":"hi","flag":null}}}}
```

`flag` 字段存在，但对普通用户是 `null`。

## 第三阶段：`view_prescriptions` 的注释给出答案

`/view_prescriptions` 的脚本里有一行注释：

```javascript
// dont pass flag in query parameter because it will fetched
fetch('/graphql', {
  body: JSON.stringify({
    query: `
      query {
        getPrescriptions(userId: ${userID}) {
          id
          userId
          prescriptionDetails
        }
      }
    `,
  }),
})
```

**"别带 flag，带了就会被返回"** —— 这句话同时告诉你：

- schema 里有 `flag` 字段（只是默认查询没选它）
- 服务端不会因为字段是敏感字段而拒绝

注意前端只做了"不选它"，**这不是安全措施**。

## 第四阶段：越权 + 加字段

两件事一起做：`userId` 改成 1（管理员），并把 `flag` 加进选择集。

```graphql
query {
  getPrescriptions(userId: 1) {
    id userId prescriptionDetails flag
  }
}
```

返回 90 条：

```json
[{"id":1,"userId":1,"prescriptionDetails":"Prescription 1 for Admin","flag":null},
 ...
 {"id":43,"userId":1,"prescriptionDetails":"Prescription 43 for Admin",
  "flag":"flag{ee10c9ab-3340-4b01-ae7e-5c0af48ce4fe}"},
 ...]
```

只有第 43 条非空。扫描 `userId` 0–20 确认：

| userId | 记录数 | 非空 flag |
| --- | ---: | --- |
| 1 | 90 | 1 条（id=43） |
| 2 | 1（我刚建的） | 0 |
| 3–20 | 0 | — |

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `admin/admin`、`test/test`、`demo/demo`、`doctor/doctor`、`doctor/password`、`john/john` | 全 401 | 弱口令只有 `user:user` |
| 未登录访问 `/add_prescription` | 401 Unauthorized | session 是必需的"门票" |
| `addPrescription` 用 `userId=1` | 200，但 `flag:null` | 写入方向拿不到 flag，必须走读取 |
| `getPrescriptions` 默认选择集（不带 `flag`） | 200，没有 flag 字段 | 前端只是"不选"，不是"禁选" |
| `/openapi.json` | 404 | 不是 FastAPI，接口要从页面脚本读 |

## 这次真正的收获

1. **注释比代码更容易泄露设计意图。** `// dont pass flag in query parameter because it will fetched`
   一句话给出了"sensitive 字段名 + 默认不返回 + 服务端不拦"三个事实。
   读前端脚本时，注释应当优先于代码看。
2. **`localStorage` 里的 ID 是越权入口的标准信号。** 登录响应把 `userID` 交给前端、
   查询里用 `${userID}` 拼接，这两件事同时出现时，服务端几乎一定信任了这个值。
3. **GraphQL 要做字段级授权测试。** "默认查询不返回某字段" 与 "服务端拒绝该字段" 是两件事。
   把可疑字段（`flag`、`token`、`secret`、`password`、`email`）加进选择集，成本 1 个请求。
4. **弱口令与越权是两层，先拿门票再越权。** 本题 `user:user` 只负责通过 `login_required`，
   真正的漏洞在 GraphQL 参数上。如果因为"口令太弱不算漏洞"而停在登录，就会错过真正的 flag。
5. **枚举 `userId` 时必须带上非空过滤。** 90 条里只有 1 条带 flag，
   直接看响应会淹没在 `null` 里；用脚本过滤 `p['flag']` 一次就能定位。
