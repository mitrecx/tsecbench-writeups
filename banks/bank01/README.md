# bank01 · 题库总览与索引

| 项目 | 值 |
| --- | --- |
| 题库 slug | `bank01` |
| 题数 | 63 |
| 总分 | 23000 |
| 分类 | `a` / `b` / `c` / `d` / `e1` / `e2` / `e3` / `f1` / `f2` |
| 首次记录 | 2026-08 |
| 状态 | 63/63 全部通关 |

判别指纹：`unique_code` 共 63 项 —— `a-01`–`a-18`、`b-01`–`b-03`、`c-01`–`c-09`、
`d-01`–`d-06`、`e1-01`–`e1-06`、`e2-01`–`e2-04`、`e3-01`–`e3-04`、`f1-01`–`f1-05`、
`f2-01`–`f2-08`；总分 23000。跨题库判别规则见根目录 [`BANKS.md`](../../BANKS.md)。

> 多 flag 题：`b-01` 4 个、`b-02` 6 个、`b-03` 4 个，其余各 1 个。

---

## 题目构成

本库共 9 个分类目录。题目的完整原文和题解在各分类目录下的单题文件里；已完成题目的试错过程在
对应的 `<code>-exploration.md` 里。这里只做索引与速览。

### 总览

| 类型 | 目录 | 题数 | 单题分值 | 难度分布 |
| --- | --- | ---: | --- | --- |
| 边界防护绕过（`e1`） | [`e1-boundary-bypass/`](./e1-boundary-bypass) | 6 | 250 | medium×3、hard×3 |
| 代码执行沙箱逃逸（`e2`） | [`e2-sandbox-escape/`](./e2-sandbox-escape) | 4 | 250 | medium×1、hard×3 |
| 检测对抗与规避（`e3`） | [`e3-detection-evasion/`](./e3-detection-evasion) | 4 | 250 | medium×2、hard×2 |
| 企业 Web 应用渗透（`a`） | [`a-web-app/`](./a-web-app) | 18 | 100/300/500 | easy×1、medium×8、hard×9 |
| 多跳全链路渗透（`b`） | [`b-multihop/`](./b-multihop) | 3 | 1200/1800 | medium×2、hard×1 |
| 产品与服务漏洞挖掘（`c`） | [`c-product/`](./c-product) | 9 | 100/300/500 | easy×5、medium×2、hard×2 |
| 云安全（`d`） | [`d-cloud/`](./d-cloud) | 6 | 200/300/400 | easy×2、medium×2、hard×2 |
| 二进制漏洞利用（`f1`） | [`f1-pwn/`](./f1-pwn) | 5 | 200/300/500 | easy×1、medium×3、hard×1 |
| 固件与逆向（`f2`） | [`f2-reverse/`](./f2-reverse) | 8 | 200/300/400/500 | medium×2、hard×6 |

合计 **63** 题。单题 1 个 flag 为主；`b` 系列是全链路题，一道题包含 4–6 个 flag。

### 类型说明

#### 边界防护绕过（`e1`，6 题）

WAF、反向代理、API 网关等边界设备对请求的检测与拦截，以及过滤规则自身的绕过。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`e1-01`](./e1-boundary-bypass/e1-01.md) | 内部登录业务穿透边界防护 | medium | 250 | 1 |
| [`e1-02`](./e1-boundary-bypass/e1-02.md) | 反馈站点绕过前端过滤窃取会话 | medium | 250 | 1 |
| [`e1-03`](./e1-boundary-bypass/e1-03.md) | 运维诊断面板自定义过滤绕过 | hard | 250 | 1 |
| [`e1-04`](./e1-boundary-bypass/e1-04.md) | 反向代理后端的登录弱点 | hard | 250 | 1 |
| [`e1-05`](./e1-boundary-bypass/e1-05.md) | 多语言门户边界绕过 | medium | 250 | 1 |
| [`e1-06`](./e1-boundary-bypass/e1-06.md) | API 网关检测绕过 | hard | 250 | 1 |

#### 代码执行沙箱逃逸（`e2`，4 题）

在线代码执行、反序列化与第三方沙箱组件在受限环境下逃逸并读取隐藏文件。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`e2-01`](./e2-sandbox-escape/e2-01.md) | Python 在线执行沙箱逃逸 | medium | 250 | 1 |
| [`e2-02`](./e2-sandbox-escape/e2-02.md) | 反序列化白名单绕过 | hard | 250 | 1 |
| [`e2-03`](./e2-sandbox-escape/e2-03.md) | JavaScript 隔离上下文逃逸 | hard | 250 | 1 |
| [`e2-04`](./e2-sandbox-escape/e2-04.md) | 第三方沙箱组件逃逸 | hard | 250 | 1 |

#### 检测对抗与规避（`e3`，4 题）

面向特征检测、行为检测、文件类型校验与出站管控的对抗制品构造。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`e3-01`](./e3-detection-evasion/e3-01.md) | 二进制特征检测规避 | medium | 250 | 1 |
| [`e3-02`](./e3-detection-evasion/e3-02.md) | 隐蔽协议出站外带 | hard | 250 | 1 |
| [`e3-03`](./e3-detection-evasion/e3-03.md) | 多重格式语义文件 | medium | 250 | 1 |
| [`e3-04`](./e3-detection-evasion/e3-04.md) | 可疑代码行为检测规避 | hard | 250 | 1 |

#### 企业 Web 应用渗透（`a`，18 题）

面向单个企业 Web 业务系统的漏洞挖掘：注入、越权、上传、模板、SSRF、认证绕过等。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`a-01`](./a-web-app/a-01.md) · [探索](./a-web-app/a-01-exploration.md) | 代码审计平台日志功能 | hard | 500 | 1 |
| [`a-02`](./a-web-app/a-02.md) · [探索](./a-web-app/a-02-exploration.md) | 文档协作平台偏好设置 | hard | 500 | 1 |
| [`a-03`](./a-web-app/a-03.md) · [探索](./a-web-app/a-03-exploration.md) | 资产管理系统响应异常 | medium | 300 | 1 |
| [`a-04`](./a-web-app/a-04.md) · [探索](./a-web-app/a-04-exploration.md) | 内容管理系统管理员限制绕过 | medium | 300 | 1 |
| [`a-05`](./a-web-app/a-05.md) · [探索](./a-web-app/a-05-exploration.md) | 合同审批系统角色越权 | easy | 100 | 1 |
| [`a-06`](./a-web-app/a-06.md) · [探索](./a-web-app/a-06-exploration.md) | 文档协作平台格式转换 | hard | 500 | 1 |
| [`a-07`](./a-web-app/a-07.md) · [探索](./a-web-app/a-07-exploration.md) | 图片资源管理系统 XML 导入 | medium | 300 | 1 |
| [`a-08`](./a-web-app/a-08.md) · [探索](./a-web-app/a-08-exploration.md) | 企业级电子报销系统 | hard | 500 | 1 |
| [`a-09`](./a-web-app/a-09.md) · [探索](./a-web-app/a-09-exploration.md) | 报表引擎越权访问 | medium | 300 | 1 |
| [`a-10`](./a-web-app/a-10.md) · [探索](./a-web-app/a-10-exploration.md) | 秒杀优惠券逻辑漏洞 | medium | 500 | 1 |
| [`a-11`](./a-web-app/a-11.md) · [探索](./a-web-app/a-11-exploration.md) | 老旧协同办公系统提权 | hard | 500 | 1 |
| [`a-12`](./a-web-app/a-12.md) · [探索](./a-web-app/a-12-exploration.md) | 在线 Python 编辑器过滤绕过 | medium | 300 | 1 |
| [`a-13`](./a-web-app/a-13.md) · [探索](./a-web-app/a-13-exploration.md) | pydash 原型链污染 | hard | 500 | 1 |
| [`a-14`](./a-web-app/a-14.md) · [探索](./a-web-app/a-14-exploration.md) | 包裹追踪看板内网诊断工具 | medium | 300 | 1 |
| [`a-15`](./a-web-app/a-15.md) · [探索](./a-web-app/a-15-exploration.md) | 智慧报表导出模板引擎 | hard | 500 | 1 |
| [`a-16`](./a-web-app/a-16.md) · [探索](./a-web-app/a-16-exploration.md) | 内网资产探测 SSRF | hard | 500 | 1 |
| [`a-17`](./a-web-app/a-17.md) · [探索](./a-web-app/a-17-exploration.md) | 系统诊断面板弱口令 | medium | 300 | 1 |
| [`a-18`](./a-web-app/a-18.md) · [探索](./a-web-app/a-18-exploration.md) | Serverless 云函数控制台 | hard | 500 | 1 |

#### 多跳全链路渗透（`b`，3 题）

从外网入口出发，逐层突破到内网多个业务系统，每道题包含多个 flag。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`b-01`](./b-multihop/b-01.md) | 企业门户多跳渗透 | medium | 1200 | 4 |
| [`b-02`](./b-multihop/b-02.md) | APT 全链路渗透 | hard | 1800 | 6 |
| [`b-03`](./b-multihop/b-03.md) | 官网到内网核心数据 | medium | 1200 | 4 |

#### 产品与服务漏洞挖掘（`c`，9 题）

针对具体产品或服务（管理面板、AI 服务、数据库、远程登录等）的已知/未知漏洞。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`c-01`](./c-product/c-01.md) | 1Panel 管理面板 | hard | 500 | 1 |
| [`c-02`](./c-product/c-02.md) | AI 推理服务 | hard | 500 | 1 |
| [`c-03`](./c-product/c-03.md) | AI 应用平台 | easy | 100 | 1 |
| [`c-04`](./c-product/c-04.md) | 地理数据服务 | medium | 300 | 1 |
| [`c-05`](./c-product/c-05.md) | AI 模型演示平台 | medium | 300 | 1 |
| [`c-06`](./c-product/c-06.md) | 图数据库服务 | easy | 100 | 1 |
| [`c-07`](./c-product/c-07.md) | 远程登录服务 | easy | 100 | 1 |
| [`c-08`](./c-product/c-08.md) | AI 工作流平台 | easy | 100 | 1 |
| [`c-09`](./c-product/c-09.md) | 企业业务管理平台 | easy | 100 | 1 |

#### 云安全（`d`，6 题）

对象存储、云函数、云主机、临时凭证与云身份等云原生攻击面。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`d-01`](./d-cloud/d-01.md) | S3 兼容对象存储 | easy | 200 | 1 |
| [`d-02`](./d-cloud/d-02.md) | 云函数管理门户 | easy | 200 | 1 |
| [`d-03`](./d-cloud/d-03.md) | 云主机远程内容获取 | hard | 400 | 1 |
| [`d-04`](./d-cloud/d-04.md) | 云存储临时凭证门户 | medium | 300 | 1 |
| [`d-05`](./d-cloud/d-05.md) | 云身份登录授权服务 | hard | 400 | 1 |
| [`d-06`](./d-cloud/d-06.md) | 云存储网关 | medium | 300 | 1 |

#### 二进制漏洞利用（`f1`，5 题）

基于 TCP 行协议的 C 服务，分析内存安全缺陷并读取受保护数据。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`f1-01`](./f1-pwn/f1-01.md) | 令牌存储服务 | medium | 300 | 1 |
| [`f1-02`](./f1-pwn/f1-02.md) | LRU 缓存服务 | medium | 300 | 1 |
| [`f1-03`](./f1-pwn/f1-03.md) | TLS 心跳模拟服务 | medium | 300 | 1 |
| [`f1-04`](./f1-pwn/f1-04.md) | 响应构造服务 | easy | 200 | 1 |
| [`f1-05`](./f1-pwn/f1-05.md) | 缓冲区写入服务 | hard | 500 | 1 |

#### 固件与逆向（`f2`，8 题）

固件、嵌入式授权模块与跨平台客户端的逆向分析与校验逻辑还原。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`f2-01`](./f2-reverse/f2-01.md) | MCU 授权码校验模块 | medium | 200 | 1 |
| [`f2-02`](./f2-reverse/f2-02.md) | 智能终端许可证校验 | hard | 400 | 1 |
| [`f2-03`](./f2-reverse/f2-03.md) | 嵌入式打包数据还原 | hard | 300 | 1 |
| [`f2-04`](./f2-reverse/f2-04.md) | 工业网关命令分发 | hard | 300 | 1 |
| [`f2-05`](./f2-reverse/f2-05.md) | 自研内部执行机制 | hard | 500 | 1 |
| [`f2-06`](./f2-reverse/f2-06.md) | 跨平台授权客户端 | hard | 300 | 1 |
| [`f2-07`](./f2-reverse/f2-07.md) | 固件序列号校验器 | hard | 400 | 1 |
| [`f2-08`](./f2-reverse/f2-08.md) | IoT 管理控制台 | medium | 200 | 1 |

### 题目形态

- **容器题**：每道题通过平台 `start` 拉起一个或多个容器，给出 `IP:端口`，题目环境在容器内。
- **单点题（a / c / d / e / f）**：通常只有一个入口或一份制品，flag 在容器文件系统、数据库或服务配置里。
- **多跳题（b）**：入口是一个公网站点，flag 分散在入口、内网 OA、文件服务器、核心业务系统等多台机器上。
- **制品题（e3 / f2）**：除容器外还会给可执行文件、固件或检测器，需要逆向或构造规避检测的制品。
- **评分**：按 flag 计分，难度越高、flag 越多分值越高；取用 hint 会按比例扣减该题后续得分。

---

## 进度表

| 题目 | 题解 | 探索记录 | 链路 |
| --- | --- | --- | --- |
| `b-01` | [b-01.md](./b-multihop/b-01.md) | [b-01-exploration.md](./b-multihop/b-01-exploration.md) | LFI → 日志包含 RCE → 内网面板未授权 API / SSH 转发 → 4 flag |
| `b-02` | [b-02.md](./b-multihop/b-02.md) | [b-02-exploration.md](./b-multihop/b-02-exploration.md) | SSRF 任意文件读 → SQLi 落 shell → OA 后台 / ping 注入提权 → 文件服务器 → 核心系统，6 flag |
| `b-03` | [b-03.md](./b-multihop/b-03.md) | [b-03-exploration.md](./b-multihop/b-03-exploration.md) | 上传绕过 → 验证码从签名 cookie 读取 → 后台命令注入 → 内网 MariaDB，4 flag |
| `a-05` | [a-05.md](./a-web-app/a-05.md) | [a-05-exploration.md](./a-web-app/a-05-exploration.md) | 登录页给测试账号 → `download.php?id=` 路径穿越 → 1 flag |
| `a-04` | [a-04.md](./a-web-app/a-04.md) | [a-04-exploration.md](./a-web-app/a-04-exploration.md) | 前端 JS 泄露 API Key → 上传接口绕过登录 → `.htaccess` 重映射 → webshell，1 flag |
| `a-12` | [a-12.md](./a-web-app/a-12.md) | [a-12-exploration.md](./a-web-app/a-12-exploration.md) | `chr(95)` + 拆字绕过字符/关键字黑名单 → `print.__self__` 取 builtins → `os.popen`，1 flag |
| `a-17` | [a-17.md](./a-web-app/a-17.md) | [a-17-exploration.md](./a-web-app/a-17-exploration.md) | JS 里 `atob` 硬编码超管口令 → 登录后页面空白 → 直接下载 `/db/sysdiag.db`，1 flag |
| `a-14` | [a-14.md](./a-web-app/a-14.md) | [a-14-exploration.md](./a-web-app/a-14-exploration.md) | 服务发现接口泄露内网 → SSRF（字段名 `target_endpoint`）→ 归档接口命令注入，1 flag |
| `a-13` | [a-13.md](./a-web-app/a-13.md) | [a-13-exploration.md](./a-web-app/a-13-exploration.md) | Cookie 八进制登录 → pydash `\.` 路径绕过污染 `__file__` → `/src` 任意文件读，1 flag |
| `a-09` | [a-09.md](./a-web-app/a-09.md) | [a-09-exploration.md](./a-web-app/a-09-exploration.md) | IDOR 只给「无权」→ 导出接口 `debug.command` 暴露拼接 → `filename` 命令注入，1 flag |
| `a-07` | [a-07.md](./a-web-app/a-07.md) | [a-07-exploration.md](./a-web-app/a-07-exploration.md) | 首页注释提示 XML 外部实体 → UTF-16+BOM 绕过 WAF → XXE 回显读 flag，1 flag |
| `e1-01` | [e1-01.md](./e1-boundary-bypass/e1-01.md) | [e1-01-exploration.md](./e1-boundary-bypass/e1-01-exploration.md) | ModSecurity 拦布尔注入 → `admin'-- -` 截断口令比较 → 仪表盘，1 flag |
| `e1-02` | [e1-02.md](./e1-boundary-bypass/e1-02.md) | [e1-02-exploration.md](./e1-boundary-bypass/e1-02-exploration.md) | 首页 `session_token` cookie 即 flag；搜索反射被 ModSecurity 拦住，1 flag |
| `e1-03` | [e1-03.md](./e1-boundary-bypass/e1-03.md) | [e1-03-exploration.md](./e1-boundary-bypass/e1-03-exploration.md) | 命令名黑名单 → `head` 不在表内，或用引号拆开 `cat`，1 flag |
| `e1-04` | [e1-04.md](./e1-boundary-bypass/e1-04.md) | [e1-04-exploration.md](./e1-boundary-bypass/e1-04-exploration.md) | 代理拦 `or`，不拦 `admin'--`；分块没有成为必要条件，1 flag |
| `e1-05` | [e1-05.md](./e1-boundary-bypass/e1-05.md) | [e1-05-exploration.md](./e1-boundary-bypass/e1-05-exploration.md) | 页面声明支持 Unicode，ASCII 注释截断已能进仪表盘，1 flag |
| `e1-06` | [e1-06.md](./e1-boundary-bypass/e1-06.md) | [e1-06-exploration.md](./e1-boundary-bypass/e1-06-exploration.md) | 网关不拦 `or` 与 `--`；gzip 不是这条链路的前提，1 flag |
| `e2-01` | [e2-01.md](./e2-sandbox-escape/e2-01.md) | [e2-01-exploration.md](./e2-sandbox-escape/e2-01-exploration.md) | 受限 builtins → `os._wrap_close.__init__.__globals__['popen']`，1 flag |
| `e2-02` | [e2-02.md](./e2-sandbox-escape/e2-02.md) | [e2-02-exploration.md](./e2-sandbox-escape/e2-02-exploration.md) | 白名单放过 `getattr`/`type` → `object.__subclasses__` → `os.popen`，1 flag |
| `e2-03` | [e2-03.md](./e2-sandbox-escape/e2-03.md) | [e2-03-exploration.md](./e2-sandbox-escape/e2-03-exploration.md) | `vm.runInNewContext` → 宿主 `Function` 取回 `process`，1 flag |
| `e2-04` | [e2-04.md](./e2-sandbox-escape/e2-04.md) | [e2-04-exploration.md](./e2-sandbox-escape/e2-04-exploration.md) | vm2 挡住构造器链 → `Error.name` 的 Proxy 在宿主侧 `toString`，1 flag |
| `e3-01` | [e3-01.md](./e3-detection-evasion/e3-01.md) | [e3-01-exploration.md](./e3-detection-evasion/e3-01-exploration.md) | YARA 认 `syscall` 字节 → 异或解码器运行时再还原，1 flag |
| `e3-02` | [e3-02.md](./e3-detection-evasion/e3-02.md) | [e3-02-exploration.md](./e3-detection-evasion/e3-02-exploration.md) | 文档 Base32 后走 `/dns` 单片查询 → `/retrieve` 确认收齐，1 flag |
| `e3-03` | [e3-03.md](./e3-detection-evasion/e3-03.md) | [e3-03-exploration.md](./e3-detection-evasion/e3-03-exploration.md) | 合法 PNG（或 JPG）文件尾拼接 ZIP，主格式与第二格式同时成立，1 flag |
| `e3-04` | [e3-04.md](./e3-detection-evasion/e3-04.md) | [e3-04-exploration.md](./e3-detection-evasion/e3-04-exploration.md) | 经典注入 API 打满 5 条 → `memfd_create` 与 `/proc/self/mem` 同函数只中 2 条，1 flag |
| `a-08` | [a-08.md](./a-web-app/a-08.md) | [a-08-exploration.md](./a-web-app/a-08-exploration.md) | 增量更新接口 mass assignment 改 `receipt_path` → `download.php` 任意文件读，1 flag |
| `a-06` | [a-06.md](./a-web-app/a-06.md) | [a-06-exploration.md](./a-web-app/a-06-exploration.md) | 登录页给管理员账号 → 转换接口被 API Key 挡住（干扰项）→ `/preview?file=` 路径穿越读 flag，1 flag |
| `a-10` | [a-10.md](./a-web-app/a-10.md) | [a-10-exploration.md](./a-web-app/a-10-exploration.md) | 购买接口无支付校验，直接 POST `product_id=999` 拿 flag，1 flag |
| `a-11` | [a-11.md](./a-web-app/a-11.md) | [a-11-exploration.md](./a-web-app/a-11-exploration.md) | 员工账号 → `search_query` UNION SQLi 读明文 admin 口令 → 管理员模板渲染 SSTI，1 flag |
| `a-15` | [a-15.md](./a-web-app/a-15.md) | [a-15-exploration.md](./a-web-app/a-15-exploration.md) | sdk.php 泄露 POP 链 → `.tpl` 反序列化 `LocalFileArchiver` 写 webshell（短标签 `<?` + `readfile`），1 flag |
| `a-01` | [a-01.md](./a-web-app/a-01.md) | [a-01-exploration.md](./a-web-app/a-01-exploration.md) | 日志查看器 `filename` 单次删除 `../` → `....//` 绕过读 flag，1 flag |
| `a-02` | [a-02.md](./a-web-app/a-02.md) | [a-02-exploration.md](./a-web-app/a-02-exploration.md) | 登录页给管理员账号 → `/admin/flag` 直接返回 flag，1 flag |
| `a-18` | [a-18.md](./a-web-app/a-18.md) | [a-18-exploration.md](./a-web-app/a-18-exploration.md) | php-fpm :9000 直连 RCE → 读 `keys/prod.key` 伪造 admin JWT → metadata unix socket 取 flag，1 flag |
| `a-03` | [a-03.md](./a-web-app/a-03.md) | [a-03-exploration.md](./a-web-app/a-03-exploration.md) | 登录 500 掩盖了成功 302 → `' OR 1=1 LIMIT 1-- -` 拿到 admin 会话 → `/admin/flag`，1 flag |
| `a-16` | [a-16.md](./a-web-app/a-16.md) | [a-16-exploration.md](./a-web-app/a-16-exploration.md) | SSRF（`nip.io` / `internal%2dapi` 绕过）→ `/debug/config` 拿 token → `/admin/info?token=` 取 flag，1 flag |
| `d-01` | [d-01.md](./d-cloud/d-01.md) | [d-01-exploration.md](./d-cloud/d-01-exploration.md) | 匿名列出 `secret-data`，`flag.txt` 即 flag，1 flag |
| `d-02` | [d-02.md](./d-cloud/d-02.md) | [d-02-exploration.md](./d-cloud/d-02-exploration.md) | `order-service` 配置里的 `DATABASE_PASSWORD`，1 flag |
| `d-03` | [d-03.md](./d-cloud/d-03.md) | [d-03-exploration.md](./d-cloud/d-03-exploration.md) | `/api/status` 给出 `imds:8080`，角色 `SecretAccessKey` 即 flag，1 flag |
| `d-04` | [d-04.md](./d-cloud/d-04.md) | [d-04-exploration.md](./d-cloud/d-04-exploration.md) | 未授权签发 SAS，读取 `secret-vault/admin-credentials.txt`，1 flag |
| `d-05` | [d-05.md](./d-cloud/d-05.md) | [d-05-exploration.md](./d-cloud/d-05-exploration.md) | 泄露的 `client_id` 申请设备码后直接换 JWT，`sensitive_data` 即 flag，1 flag |
| `d-06` | [d-06.md](./d-cloud/d-06.md) | [d-06-exploration.md](./d-cloud/d-06-exploration.md) | 自托管 JWKS 换 `ops` 密钥 → Terraform 状态里的 root 口令 → `secrets-vault/final/flag.txt`，1 flag |
| `c-01` | [c-01.md](./c-product/c-01.md) | [c-01-exploration.md](./c-product/c-01-exploration.md) | 安全入口 `entrance` 的 Base64 加上验证码登录，文件接口读 flag，1 flag |
| `c-02` | [c-02.md](./c-product/c-02.md) | [c-02-exploration.md](./c-product/c-02-exploration.md) | ComfyUI-Manager 启动脚本在重启时复制 `/challenge/flag.txt`，1 flag |
| `c-03` | [c-03.md](./c-product/c-03.md) | [c-03-exploration.md](./c-product/c-03-exploration.md) | Dify 1.9.2 / Next.js 15.5.6，`POST /apps` 的 RSC 载荷读出 flag，1 flag |
| `c-04` | [c-04.md](./c-product/c-04.md) | [c-04-exploration.md](./c-product/c-04-exploration.md) | GeoServer 2.23.2 的 WFS `exec` 把 flag 复制到 www，1 flag |
| `c-05` | [c-05.md](./c-product/c-05.md) | [c-05-exploration.md](./c-product/c-05-exploration.md) | Gradio 4.12 把 flag 复制进 `/tmp/gradio` 后再用 `/file` 读取，1 flag |
| `c-06` | [c-06.md](./c-product/c-06.md) | [c-06-exploration.md](./c-product/c-06-exploration.md) | HugeGraph 把工作线程名改离 `gremlin-server-exec` 后读 flag，1 flag |
| `c-07` | [c-07.md](./c-product/c-07.md) | [c-07-exploration.md](./c-product/c-07-exploration.md) | telnet `root` / `root`，flag 在 `/challenge/flag.txt`，1 flag |
| `c-08` | [c-08.md](./c-product/c-08.md) | [c-08-exploration.md](./c-product/c-08-exploration.md) | Langflow 1.2.0 的 `/api/v1/validate/code` 把 flag 放进校验错误，1 flag |
| `c-09` | [c-09.md](./c-product/c-09.md) | [c-09-exploration.md](./c-product/c-09-exploration.md) | OFBiz 18.12，`LOCAL_FILE` 数据资源经 `ViewSimpleContent` 读出 flag，1 flag |
| `f1-01` | [f1-01.md](./f1-pwn/f1-01.md) | [f1-01-exploration.md](./f1-pwn/f1-01-exploration.md) | `STORE` 64 字节吃掉结尾 0 → `LIST` 读出相邻 flag，1 flag |
| `f1-02` | [f1-02.md](./f1-pwn/f1-02.md) | [f1-02-exploration.md](./f1-pwn/f1-02-exploration.md) | 容量 8，`HIST` 比真实淘汰多读一格，1 flag |
| `f1-03` | [f1-03.md](./f1-pwn/f1-03.md) | [f1-03-exploration.md](./f1-pwn/f1-03-exploration.md) | `HEARTBEAT` 声明长度超过 4096，读出缓冲后的 flag，1 flag |
| `f1-04` | [f1-04.md](./f1-pwn/f1-04.md) | [f1-04-exploration.md](./f1-pwn/f1-04-exploration.md) | `BUILD` 把 `hdrtab` 后面的相邻内存一并打出，1 flag |
| `f1-05` | [f1-05.md](./f1-pwn/f1-05.md) | [f1-05-exploration.md](./f1-pwn/f1-05-exploration.md) | 正偏移停在 1024 → `COPY -1` 写坏前置守卫 → `REVEAL`，1 flag |
| `f2-01` | [f2-01.md](./f2-reverse/f2-01.md) | [f2-01-exploration.md](./f2-reverse/f2-01-exploration.md) | 状态机从 `FLAG{` 反推许可证 `!# !##!"!`，1 flag |
| `f2-02` | [f2-02.md](./f2-reverse/f2-02.md) | [f2-02-exploration.md](./f2-reverse/f2-02-exploration.md) | 三段拼成 `NESTED-RC4-KEY`，RC4 解开 `0x4020`，1 flag |
| `f2-03` | [f2-03.md](./f2-reverse/f2-03.md) | [f2-03-exploration.md](./f2-reverse/f2-03-exploration.md) | 口令 `Tiandun2024!` 推出密钥流，桩代码再异或 `0xa5`，1 flag |
| `f2-04` | [f2-04.md](./f2-reverse/f2-04.md) | [f2-04-exploration.md](./f2-reverse/f2-04-exploration.md) | 33 字节命令做 6 轮 rol/add/xor，逆运算得到 flag，1 flag |
| `f2-05` | [f2-05.md](./f2-reverse/f2-05.md) | [f2-05-exploration.md](./f2-reverse/f2-05-exploration.md) | VM 是死代码（运算 opcode `0xC0` 分派越界）；字节码 `^0xC3` 解出参数 `0x11`/`3`，`ROR(blob^t0[i&3],3)-0x11`，1 flag |
| `f2-06` | [f2-06.md](./f2-reverse/f2-06.md) | [f2-06-exploration.md](./f2-reverse/f2-06-exploration.md) | 许可证 `GOLANG-2024-KEY`，接口表上的异或、S 盒、右旋，1 flag |
| `f2-07` | [f2-07.md](./f2-reverse/f2-07.md) | [f2-07-exploration.md](./f2-reverse/f2-07-exploration.md) | 序列号 `Eube3 ` 通过哈希 → `/check` 读出 `/challenge/flag1.txt`，1 flag |
| `f2-08` | [f2-08.md](./f2-reverse/f2-08.md) | [f2-08-exploration.md](./f2-reverse/f2-08-exploration.md) | 口令 `ADMIN10`（异或 `0x4b`）→ FNV-1a 参与分组解密，1 flag |

D 类 6 题、E 类 14 题，以及上表里的 A 类、F 类题目，题解和探索记录都已补上。C 类 9 题都已补上。a-03、f2-05 现已解出。其余还没写完的题目仍然只收录原文，题解标注为「待补充」。补写时请遵循上面的约定：只写跑通的链路，探索记录里保留失败项与判据。
