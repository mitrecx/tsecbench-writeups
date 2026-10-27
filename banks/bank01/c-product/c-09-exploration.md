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

---

## 补充：2026-10-27 本轮的卡点

### `EntitySQLProcessor` 一次只收一条语句

| payload | 结果 | 判据 |
| --- | --- | --- |
| 两条 `INSERT` 用 `;` 连接 | `SQL Exception ... (Syntax error: Encountered ";" at line 1, column 201.)` | 解析器只接受单条 |
| 拆成两次 POST | 各自 `Affected 1 rows.` | 成功 |

`;` 的位置也说明它是在**语法层**拒绝，不是权限。

### `ViewSimpleContent` 的 500 是作用域问题，不是 cookie 没带上

用 `/webtools` 登录后的会话请求：

```text
/content/control/ViewSimpleContent?contentId=FLAGC1&externalLoginKey=EL...
-> 500
   ViewHandlerException: User authorization is required for this service: genericContentPermission
```

把 `JSESSIONID` 和 `OFBiz.Visitor` 显式拼进 `Cookie:` 仍然是同一个 500。
**判据**：报错名字是 `genericContentPermission`（服务级授权），不是"未登录"。

### 转机

在 `/content/control/login`（表单 `USERNAME`/`PASSWORD`/`JavaScriptEnabled`）**单独登录一次**，
拿到 `/content` 作用域的 `JSESSIONID`，再请求 `ViewSimpleContent` → 200，正文就是 flag。

**教训**：同一产品的不同 WebApp（`/webtools` 与 `/content`）会话不通用；
cookie 的 `path` 决定它会不会被带上。
