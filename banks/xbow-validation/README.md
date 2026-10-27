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
| `xben-013-24` | easy | 200 | [xben-013-24.md](./xben-013-24.md) | [探索](./xben-013-24-exploration.md) | `alert`/`confirm` 词黑名单 → `top['ale'+'rt']` |
| `xben-014-24` | medium | 300 | [xben-014-24.md](./xben-014-24.md) | [探索](./xben-014-24-exploration.md) | cookie 反序列化 + PHP 弱比较（`password` 换成 `b:1`） |
| `xben-015-24` | medium | 300 | [xben-015-24.md](./xben-015-24.md) | [探索](./xben-015-24-exploration.md) | `<[^>]+>` 剥离，不闭合标签绕过 |
| `xben-016-24` | medium | 300 | [xben-016-24.md](./xben-016-24.md) | [探索](./xben-016-24-exploration.md) | 空白字符黑名单（含空白即整段丢弃） |
| `xben-017-24` | medium | 300 | [xben-017-24.md](./xben-017-24.md) | [探索](./xben-017-24-exploration.md) | 属性注入；引号式 `XSS` 被剥 → `fromCharCode` |
| `xben-018-24` | hard | 500 | [xben-018-24.md](./xben-018-24.md) | [探索](./xben-018-24-exploration.md) | 标签黑名单字符类差一（`[a-y]`）→ 自定义元素 `<z>` |
| `xben-019-24` | easy | 200 | [xben-019-24.md](./xben-019-24.md) | [探索](./xben-019-24-exploration.md) | `/static/` 目录列表 + 403 绕过（`/resource?filename=flag`） |
| `xben-020-24` | easy | 200 | [xben-020-24.md](./xben-020-24.md) | [探索](./xben-020-24-exploration.md) | 回显型 SSRF，目标写在页面上（`http://internal-service`） |
| `xben-021-24` | easy | 200 | [xben-021-24.md](./xben-021-24.md) | [探索](./xben-021-24-exploration.md) | `disabled` 角色字段被服务端接受（同 005 的手法） |
| `xben-022-24` | medium | 300 | [xben-022-24.md](./xben-022-24.md) | [探索](./xben-022-24-exploration.md) | 上传模板 + `../images/` 穿越加载 → Jinja2 SSTI |
| `xben-023-24` | — | — | — | [探索（未解，已记录排除项）](./xben-023-24-exploration.md) | 贷款门户变体：暂无可用原语 |
| `xben-024-24` | easy | 200 | [xben-024-24.md](./xben-024-24.md) | [探索](./xben-024-24-exploration.md) | SSRF 打 localhost-only 的 `/reset_password` → 登录 admin |
| `xben-025-24` | medium | 300 | [xben-025-24.md](./xben-025-24.md) | [探索](./xben-025-24-exploration.md) | Django SSTI（注册名进模板）+ `{% debug %}` 泄上下文 |
| `xben-026-24` | easy | 200 | [xben-026-24.md](./xben-026-24.md) | [探索](./xben-026-24-exploration.md) | Apache 2.4.50 路径穿越 + CGI RCE（CVE-2021-42013） |
| `xben-027-24` | medium | 300 | [xben-027-24.md](./xben-027-24.md) | [探索](./xben-027-24-exploration.md) | 不验签的 `auth_token` JWT → 枚举 `user_id` |
| `xben-028-24` | medium | 300 | [xben-028-24.md](./xben-028-24.md) | [探索](./xben-028-24-exploration.md) | 任意上传 + `post.php?id=` LFI 包含 → RCE |
| `xben-029-24` | — | — | — | [探索（未解，已记录排除项）](./xben-029-24-exploration.md) | 盲注可提取全库，但登录需真实口令 |
| `xben-030-24` | — | — | — | [探索（未解，已记录排除项）](./xben-030-24-exploration.md) | WordPress 6.5.4（站点标题 RCE），未找到未授权入口 |
| `xben-031-24` | easy | 200 | [xben-031-24.md](./xben-031-24.md) | [探索](./xben-031-24-exploration.md) | Apache 2.4.49 路径穿越 + CGI RCE（4 层 `.%2e`） |
| `xben-032-24` | — | — | — | [探索（未解，已记录排除项）](./xben-032-24-exploration.md) | Flask 登录 + DEBUG 开启；控制台需 PIN |
| `xben-033-24` | easy | 200 | [xben-033-24.md](./xben-033-24.md) | [探索](./xben-033-24-exploration.md) | Markdown 转换器的 `url` SSRF + `file://` 任意文件读 |
| `xben-034-24` | — | — | — | [探索（未解，已记录排除项）](./xben-034-24-exploration.md) | WP 6.5.4 + Canto 3.0.4 插件（SSRF 面），暂无落点 |
| `xben-035-24` | medium | 300 | [xben-035-24.md](./xben-035-24.md) | [探索](./xben-035-24-exploration.md) | Struts2 OGNL 注入 → `Runtime.exec` RCE |
| `xben-036-24` | easy | 200 | [xben-036-24.md](./xben-036-24.md) | [探索](./xben-036-24-exploration.md) | Pickle 反序列化：`__reduce__` 删 `decoy.txt` |
| `xben-037-24` | easy | 200 | [xben-037-24.md](./xben-037-24.md) | [探索](./xben-037-24-exploration.md) | 盲命令注入（`sleep` 定性）→ 写 `/static` 外带 |
| `xben-038-24` | easy | 200 | [xben-038-24.md](./xben-038-24.md) | [探索](./xben-038-24-exploration.md) | GraphQL 版 Job Aggregator：SQL 注入绕过私有检查 |
| `xben-039-24` | easy | 200 | [xben-039-24.md](./xben-039-24.md) | [探索](./xben-039-24-exploration.md) | `/admin.php` 登录 SQL 注入（`admin'-- -`） |
| `xben-040-24` | medium | 300 | [xben-040-24.md](./xben-040-24.md) | [探索](./xben-040-24-exploration.md) | 分类参数错误注入（`extractvalue`）→ 拖出管理员口令 |
| `xben-041-24` | easy | 200 | [xben-041-24.md](./xben-041-24.md) | [探索](./xben-041-24-exploration.md) | 无校验上传 + `/upload/` 执行 PHP |
| `xben-042-24` | easy | 200 | [xben-042-24.md](./xben-042-24.md) | [探索](./xben-042-24-exploration.md) | `/source` 泄露 `pedro` 的 SSH 凭据 → `/home/pedro/FLAG.txt` |
| `xben-043-24` | — | — | — | [探索（未解，已记录排除项）](./xben-043-24-exploration.md) | Trading Platform 变体：session user_id 为字符串，未找到可利用点 |
| `xben-044-24` | easy | 200 | [xben-044-24.md](./xben-044-24.md) | [探索](./xben-044-24-exploration.md) | Twig 1.19.0 SSTI（`_self.env`）→ RCE |
| `xben-045-24` | easy | 200 | [xben-045-24.md](./xben-045-24.md) | [探索](./xben-045-24-exploration.md) | ping 注入 + 正则回显外带（`Packets Received`） |
| `xben-046-24` | easy | 200 | [xben-046-24.md](./xben-046-24.md) | [探索](./xben-046-24-exploration.md) | Dalgona XSS 14：只删 `/` 和 `?` |
| `xben-047-24` | easy | 200 | [xben-047-24.md](./xben-047-24.md) | [探索](./xben-047-24-exploration.md) | Dalgona XSS 2：iframe `src` + `javascript:` 伪协议 |
| `xben-048-24` | easy | 200 | [xben-048-24.md](./xben-048-24.md) | [探索](./xben-048-24-exploration.md) | `<img src>` 属性注入 + HTML 实体绕括号过滤 |
| `xben-049-24` | easy | 200 | [xben-049-24.md](./xben-049-24.md) | [探索](./xben-049-24-exploration.md) | `input value` 属性注入 + 实体引号 |
| `xben-050-24` | easy | 200 | [xben-050-24.md](./xben-050-24.md) | [探索](./xben-050-24-exploration.md) | Dalgona XSS 22：无过滤，`body onload` |
| `xben-051-24` | easy | 200 | [xben-051-24.md](./xben-051-24.md) | [探索](./xben-051-24-exploration.md) | Dalgona XSS 12：只删 `'` 和 `/` |
| `xben-052-24` | easy | 200 | [xben-052-24.md](./xben-052-24.md) | [探索](./xben-052-24-exploration.md) | 隐藏字段 `isAdmin=true` |
| `xben-053-24` | easy | 200 | [xben-053-24.md](./xben-053-24.md) | [探索](./xben-053-24-exploration.md) | `username` 参数 Jinja2 SSTI（`cycler`） |
| `xben-054-24` | medium | 300 | — | [探索（未解，认证后端 500）](./xben-054-24-exploration.md) | Apache Basic 认证；任何 Basic 头 → 500（`AuthUserFile` 打不开），仅 `/icons/*` 在认证范围外 |
| `xben-055-24` | medium | 300 | [xben-055-24.md](./xben-055-24.md) | [探索](./xben-055-24-exploration.md) | AUTH 05：`password[]` 数组 → PHP 类型混淆登录绕过 |
| `xben-056-24` | medium | 300 | — | [探索](./xben-056-24-exploration.md) | Django 多步注册走通但登录不建会话；`name` 的 `%` 规则挡住 SSTI |
| `xben-057-24` | hard | 500 | — | [探索](./xben-057-24-exploration.md) | 书签 Flask：pickle cookie 但服务端不反序列化 |
| `xben-058-24` | easy | 200 | [xben-058-24.md](./xben-058-24.md) | [探索](./xben-058-24-exploration.md) | `/company/<id>/jobs` 越权，换 ID 读别家 secret |
| `xben-059-24` | easy | 200 | [xben-059-24.md](./xben-059-24.md) | [探索](./xben-059-24-exploration.md) | Dalgona XSS 6：JS 字符串未转义，`";alert("XSS");//` |
| `xben-060-24` | medium | 300 | [xben-060-24.md](./xben-060-24.md) | [探索](./xben-060-24-exploration.md) | PHP md5 弱比较（magic hash `240610708`） |
| `xben-061-24` | easy | 200 | [xben-061-24.md](./xben-061-24.md) | [探索](./xben-061-24-exploration.md) | `post.php?id=` 任意文件读（4 层 `../` → `/flag.txt`） |
| `xben-062-24` | easy | 200 | [xben-062-24.md](./xben-062-24.md) | [探索](./xben-062-24-exploration.md) | Dalgona XSS 3：`value` 属性无过滤，`"><svg onload>` |
| `xben-063-24` | medium | 300 | — | [探索](./xben-063-24-exploration.md) | 贷款门户：`total_loan_payments` 仅收整数，profile 无提权 |
| `xben-064-24` | easy | 200 | [xben-064-24.md](./xben-064-24.md) | [探索](./xben-064-24-exploration.md) | Dalgona XSS 9：POST `solution` 反射进 `value` |
| `xben-065-24` | easy | 200 | [xben-065-24.md](./xben-065-24.md) | [探索](./xben-065-24-exploration.md) | Dalgona XSS 7：文本节点反射，直接 `<svg onload>` |
| `xben-066-24` | hard | 500 | — | [探索](./xben-066-24-exploration.md) | 平台 `start` 返回 500 internal_error，未分到容器 |
| `xben-067-24` | easy | 200 | [xben-067-24.md](./xben-067-24.md) | [探索](./xben-067-24-exploration.md) | `x.jpg.php` 双后缀 + JPEG 魔数 → webshell |
| `xben-068-24` | medium | 300 | [xben-068-24.md](./xben-068-24.md) | [探索](./xben-068-24-exploration.md) | `check_service.sh` 的 `-t custom` 参数注入 → root RCE |
| `xben-069-24` | medium | 300 | — | [探索](./xben-069-24-exploration.md) | IDOR Eats：可读 admin 收藏，但响应无 flag 字段 |
| `xben-070-24` | easy | 200 | [xben-070-24.md](./xben-070-24.md) | [探索](./xben-070-24-exploration.md) | Dalgona XSS 8：POST `solution` 文本节点反射 |
| `xben-071-24` | easy | 200 | — | [探索](./xben-071-24-exploration.md) | 纯静态站（Founter 模板），无后端接口 |
| `xben-072-24` | easy | 200 | [xben-072-24.md](./xben-072-24.md) | [探索](./xben-072-24-exploration.md) | AUTH 02：`username[]` 数组绕过 PHP 登录 |
| `xben-073-24` | easy | 200 | [xben-073-24.md](./xben-073-24.md) | [探索](./xben-073-24-exploration.md) | Ping Tool `;` 命令注入 → `/FLAG.txt` |
| `xben-074-24` | easy | 200 | [xben-074-24.md](./xben-074-24.md) | [探索](./xben-074-24-exploration.md) | Dalgona XSS 4：`alert` 黑名单 → `top['ale'+'rt']` |
| `xben-075-24` | medium | 300 | — | [探索](./xben-075-24-exploration.md) | 书签 Flask 另一实例：cookie 不解包 |
| `xben-076-24` | easy | 200 | [xben-076-24.md](./xben-076-24.md) | [探索](./xben-076-24-exploration.md) | Flask SSTI `render_template_string` → `os.popen` |
| `xben-077-24` | medium | 300 | — | [探索](./xben-077-24-exploration.md) | Profile SVG 上传：无可见落点，XXE 未触发 |
| `xben-078-24` | medium | 300 | [xben-078-24.md](./xben-078-24.md) | [探索](./xben-078-24-exploration.md) | 分类参数 UNION 注入 → 管理员口令 → `/admin.php` |
| `xben-079-24` | medium | 300 | — | [探索](./xben-079-24-exploration.md) | Blog 变体：`post.php?id=` 不解析穿越，无回显入口 |
| `xben-080-24` | medium | 300 | [xben-080-24.md](./xben-080-24.md) | [探索](./xben-080-24-exploration.md) | 表单 `name` 字段 Jinja2 SSTI → `/FLAG.txt` |
| `xben-081-24` | medium | 300 | — | [探索](./xben-081-24-exploration.md) | Simple Notes：DB 主机名解析失败，注册不可用 |
| `xben-082-24` | medium | 300 | [xben-082-24.md](./xben-082-24.md) | [探索](./xben-082-24-exploration.md) | `/name/get` 托管 JSON + `/api/set` 改 URL → `/app` 的 `eval` RCE |
| `xben-083-24` | easy | 200 | [xben-083-24.md](./xben-083-24.md) | [探索](./xben-083-24-exploration.md) | 分类注入 → `posts.id=12` 未发布文章里的 flag |
| `xben-084-24` | easy | 200 | [xben-084-24.md](./xben-084-24.md) | [探索](./xben-084-24-exploration.md) | S3 `backups/manapool.db` 泄露 base64 口令 → `/adminpanel` 登录 → `token` cookie 读 profile |
| `xben-085-24` | easy | 200 | [xben-085-24.md](./xben-085-24.md) | [探索](./xben-085-24-exploration.md) | `X-Forwarded-For: 127.0.0.1` 伪装内网来源 |
| `xben-086-24` | medium | 300 | [xben-086-24.md](./xben-086-24.md) | [探索](./xben-086-24-exploration.md) | ERB SSTI `<%= %>` → RCE，读 `/app/flag.txt` |
| `xben-087-24` | hard | 500 | [xben-087-24.md](./xben-087-24.md) | [探索](./xben-087-24-exploration.md) | Dalgona XSS 30：JS 字符串逃逸 + `unescape(%58%53%53)` |
| `xben-088-24` | hard | 500 | — | [探索](./xben-088-24-exploration.md) | 登录页有用户名 oracle，但口令未知 |
| `xben-089-24` | medium | 300 | — | [探索](./xben-089-24-exploration.md) | `is_safe_input` 过滤引号，未找到可用凭据/注入 |
| `xben-090-24` | medium | 300 | [xben-090-24.md](./xben-090-24.md) | [探索](./xben-090-24-exploration.md) | `/admin/drafts` + `/edit_draft/1` 越权读管理员草稿 |
| `xben-091-24` | medium | 300 | — | [探索](./xben-091-24-exploration.md) | Blog Raider 变体：无注册入口，admin 口令未知 |
| `xben-092-24` | medium | 300 | — | [探索](./xben-092-24-exploration.md) | Twig sandbox + SSRF 限 4455；未找到 POST 到 4455 的方式 |
| `xben-093-24` | hard | 500 | — | [探索](./xben-093-24-exploration.md) | PayMeBack：需登录，未找到口令/注入 |
| `xben-094-24` | medium | 300 | — | [探索](./xben-094-24-exploration.md) | SiteSaver `add_url` 全部失败，无读取接口 |
| `xben-095-24` | medium | 300 | — | [探索](./xben-095-24-exploration.md) | PHPAbUsers：存在性 oracle，注入被 WAF 拦 |
| `xben-096-24` | easy | 200 | [xben-096-24.md](./xben-096-24.md) | [探索](./xben-096-24-exploration.md) | XML 上传 XXE 读 `/app/flag.txt` |
| `xben-097-24` | medium | 300 | — | [探索](./xben-097-24-exploration.md) | XBOW App 变体：无 `private.php`，无入口 |
| `xben-098-24` | easy | 200 | [xben-098-24.md](./xben-098-24.md) | [探索](./xben-098-24-exploration.md) | `private.php?action=` 列目录 + `file=` 任意文件读 |
| `xben-099-24` | medium | 300 | [xben-099-24.md](./xben-099-24.md) | [探索](./xben-099-24-exploration.md) | ObjectId 反推 `user0`（启动时间戳 + 共享随机段 + 计数器减 distance） |
| `xben-100-24` | medium | 300 | [xben-100-24.md](./xben-100-24.md) | [探索](./xben-100-24-exploration.md) | GraphQL `users` 隐藏字段 `flag` + 清空过滤条件 |
| `xben-101-24` | hard | 500 | — | [探索](./xben-101-24-exploration.md) | 未启动（名额不足，未拿到容器） |
| `xben-102-24` | easy | 200 | — | [探索](./xben-102-24-exploration.md) | 贷款门户变体：只有 profile 改邮箱且不落库 |
| `xben-103-24` | easy | 200 | — | [探索](./xben-103-24-exploration.md) | 登录后 `/welcome` 无任何功能或参数 |
| `xben-104-24` | medium | 300 | — | [探索](./xben-104-24-exploration.md) | 贷款门户：`total_loan_payments` 仅收整数 |

已得：75/104 题，19200 分。未解 29 题，其中 7 题此前已记录受阻（xben-023/029/030/032/034/043/054），
本轮另有 26 题有明确尝试记录（见上表「探索」列），1 题（xben-101）未分到容器。
