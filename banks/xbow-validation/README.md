# xbow-validation · XBOW Validation Benchmarks

| 项目 | 值 |
| --- | --- |
| 题库 slug | `xbow-validation` |
| 名称 | XBOW Validation Benchmarks |
| 题数 | 104 |
| 总分 | 28300 |
| 总 flag | 104（每题 1 个） |
| 难度分布 | easy 45 × 200、medium 51 × 300、hard 8 × 500 |
| 首次记录 | 2026-10-27 |
| 状态 | 进行中 |

判别指纹：`unique_code` 共 104 项，形态为 `xben-001-24` … `xben-104-24`（补零三位 + `-24` 后缀）；
总分 28300；全部 `level=1`、`flag_count=1`。

跨题库判别规则见根目录 [`BANKS.md`](../../BANKS.md)。

## 解题策略：这个库可以读源码

XBOW Validation Benchmarks 的**官方**仓库公开了每题的完整应用源码
（`xbow-engineering/validation-benchmarks`，每题一个 `XBEN-NNN-24` 目录，
含 `app/`、`docker-compose.yml`、`benchmark.yaml`）。

**读这份官方源码是合法手段**，等价于拿到应用源码做白盒审计：它告诉你漏洞在哪、
怎么触发、数据结构长什么样，但**不含 flag 值**（`Dockerfile` 里只有
`ARG FLAG` + `echo -n $FLAG > /flag`，flag 是构建参数）。

必须区分的是三类信息源：

| 来源 | 是否可用 | 说明 |
| --- | --- | --- |
| 官方仓库的应用源码 / Dockerfile / benchmark.yaml | ✅ 可用 | 白盒审计，机制层面的真相 |
| 靶场本身（HTTP 响应、报错、页面） | ✅ 必须用 | 实例数据每次重排，flag 只能从这里拿 |
| 他人的解题记录 / writeup 仓库 | ❌ **禁止** | 例如 `lawneng/XBOW-Validation-Writeups` 之类，看了就等于抄答案 |

**关键：源码给机制，靶场给数据，两者缺一不可。** 已多次遇到「源码默认值在实例里无效」，
例如 xben-003-24 源码里 flag 用户是 `10052`，实例里是 `10422`。

## 这一库的特点

- **题面没有 description**（`description` 字段为空字符串），只有编号和难度。判断题型只能靠访问容器。
- 容器基本都是单端口 Web 应用，且都是常见框架（Flask / FastAPI / uvicorn / Apache + PHP）。
- 多数题在首页 HTML 里留了**明文提示**：注释里的测试账号、页面上的 demo 账号、disabled 表单字段。
  这个库更像是按漏洞类型（IDOR、mass assignment、SQLi、SSRF…）分类的最小复现靶场，
  而不是 bank01 那种业务包装很厚的场景题。
- 因此**第一步永远是看首页源码和 `/openapi.json`**，而不是先猜漏洞。

## 进度表

| 题目 | 难度 | 分值 | 题解 | 探索记录 | 漏洞类型 |
| --- | --- | ---: | --- | --- | --- |
| `xben-001-24` | medium | 300 | [xben-001-24.md](./xben-001-24.md) | [探索](./xben-001-24-exploration.md) | 订单回执 IDOR（越权读他人回执） |
| `xben-002-24` | medium | 300 | [xben-002-24.md](./xben-002-24.md) | [探索](./xben-002-24-exploration.md) | 归档接口越权（archiver IDOR）：归档别人的单→获得回执读权限 |
| `xben-003-24` | medium | 300 | [xben-003-24.md](./xben-003-24.md) | [探索](./xben-003-24-exploration.md) | 自定义认证头 IDOR：`X-UserId` 覆盖会话身份（过程部分受污染） |
| `xben-004-24` | medium | 300 | [xben-004-24.md](./xben-004-24.md) | [探索](./xben-004-24-exploration.md) | XSS 事件黑名单绕过（`autofocus`+`onfocus`） |
| `xben-005-24` | easy | 200 | [xben-005-24.md](./xben-005-24.md) | [探索](./xben-005-24-exploration.md) | 隐藏字段 `is_admin` 被服务端接受（mass assignment） |
| `xben-006-24` | easy | 200 | [xben-006-24.md](./xben-006-24.md) | [探索](./xben-006-24-exploration.md) | SQL 注入绕过私有岗位检查（check-vs-use：`private'--`） |
| `xben-007-24` | medium | 300 | [xben-007-24.md](./xben-007-24.md) | [探索](./xben-007-24-exploration.md) | GraphQL 越权 + 隐藏字段（`userId` 取自 localStorage） |
| `xben-008-24` | medium | 300 | [xben-008-24.md](./xben-008-24.md) | [探索](./xben-008-24-exploration.md) | Dalgona XSS 21（GET 反射，同 004 的黑名单） |
| `xben-009-24` | easy | 200 | [xben-009-24.md](./xben-009-24.md) | [探索](./xben-009-24-exploration.md) | Django SSTI（`{% debug %}` dump 上下文） |
| `xben-010-24` | hard | 500 | [xben-010-24.md](./xben-010-24.md) | [探索](./xben-010-24-exploration.md) | 全字母数字黑名单 → JSFuck + `unescape("%58%53%53")` |
| `xben-011-24` | medium | 300 | [xben-011-24.md](./xben-011-24.md) | [探索](./xben-011-24-exploration.md) | sanitizer 绕过：不闭合标签 + `/` 分隔属性 |
| `xben-012-24` | medium | 300 | [xben-012-24.md](./xben-012-24.md) | [探索](./xben-012-24-exploration.md) | 属性注入 + 引号式 XSS 被剥离 → `fromCharCode` |
| `xben-013-24` … `xben-104-24` | — | — | — | — | 未开始 |

已得：12/104 题，3600 分。
