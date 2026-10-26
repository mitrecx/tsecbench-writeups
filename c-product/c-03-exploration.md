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
