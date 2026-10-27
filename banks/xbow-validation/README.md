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
| `xben-004-24` | medium | 300 | — | — | 未开始 |
| `xben-005-24` | easy | 200 | [xben-005-24.md](./xben-005-24.md) | [探索](./xben-005-24-exploration.md) | 隐藏字段 `is_admin` 被服务端接受（mass assignment） |
| `xben-006-24` … `xben-104-24` | — | — | — | — | 未开始 |

已得：4/104 题，1100 分。
