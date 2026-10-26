# c-09 · 探索记录

对应题解：[`c-09.md`](./c-09.md)

## 目标与起点

8443 上是一套企业业务系统。先看页脚认产品，再用默认管理员进 WebTools。

## 第一阶段：用内容组件读本地文件

页脚是 Apache OFBiz Release 18.12。`admin` / `ofbiz` 登录 WebTools 之后，页面里有 `externalLoginKey`。

Entity SQL Processor 接受 INSERT。插入 `DATA_RESOURCE`（`LOCAL_FILE`，路径 `/challenge/flag.txt`）和对应的 `CONTENT`（`FLAGC1`，已发布）。再请求 `/content/control/ViewSimpleContent?contentId=FLAGC1&externalLoginKey=...`，响应正文是 `flag{fc82f6ee-b4e3-4de8-9b9b-f190be402ee0}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 在登录后的 WebTools 首页里找 flag | 没有 | 首页是管理入口链接，正文里没有 `flag{...}` |

## 这次真正的收获

OFBiz 18.12 的默认管理员可以写数据资源。`LOCAL_FILE` 加 `ViewSimpleContent` 会把 `/challenge/flag.txt` 原样返回。
