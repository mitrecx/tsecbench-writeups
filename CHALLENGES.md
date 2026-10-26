# TSecBench 题目简介

本文件与 `README.md` 同级，用来快速了解 TSecBench 的题目构成，以及每个类型目录里都放了什么。
题目的完整原文和题解在各分类目录下的单题文件里；已完成题目的试错过程在对应的 `<code>-exploration.md` 里。这里只做索引与速览。

## 总览

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

## 类型说明

### 边界防护绕过（`e1`，6 题）

WAF、反向代理、API 网关等边界设备对请求的检测与拦截，以及过滤规则自身的绕过。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`e1-01`](./e1-boundary-bypass/e1-01.md) | 内部登录业务穿透边界防护 | medium | 250 | 1 |
| [`e1-02`](./e1-boundary-bypass/e1-02.md) | 反馈站点绕过前端过滤窃取会话 | medium | 250 | 1 |
| [`e1-03`](./e1-boundary-bypass/e1-03.md) | 运维诊断面板自定义过滤绕过 | hard | 250 | 1 |
| [`e1-04`](./e1-boundary-bypass/e1-04.md) | 反向代理后端的登录弱点 | hard | 250 | 1 |
| [`e1-05`](./e1-boundary-bypass/e1-05.md) | 多语言门户边界绕过 | medium | 250 | 1 |
| [`e1-06`](./e1-boundary-bypass/e1-06.md) | API 网关检测绕过 | hard | 250 | 1 |

### 代码执行沙箱逃逸（`e2`，4 题）

在线代码执行、反序列化与第三方沙箱组件在受限环境下逃逸并读取隐藏文件。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`e2-01`](./e2-sandbox-escape/e2-01.md) | Python 在线执行沙箱逃逸 | medium | 250 | 1 |
| [`e2-02`](./e2-sandbox-escape/e2-02.md) | 反序列化白名单绕过 | hard | 250 | 1 |
| [`e2-03`](./e2-sandbox-escape/e2-03.md) | JavaScript 隔离上下文逃逸 | hard | 250 | 1 |
| [`e2-04`](./e2-sandbox-escape/e2-04.md) | 第三方沙箱组件逃逸 | hard | 250 | 1 |

### 检测对抗与规避（`e3`，4 题）

面向特征检测、行为检测、文件类型校验与出站管控的对抗制品构造。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`e3-01`](./e3-detection-evasion/e3-01.md) | 二进制特征检测规避 | medium | 250 | 1 |
| [`e3-02`](./e3-detection-evasion/e3-02.md) | 隐蔽协议出站外带 | hard | 250 | 1 |
| [`e3-03`](./e3-detection-evasion/e3-03.md) | 多重格式语义文件 | medium | 250 | 1 |
| [`e3-04`](./e3-detection-evasion/e3-04.md) | 可疑代码行为检测规避 | hard | 250 | 1 |

### 企业 Web 应用渗透（`a`，18 题）

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

### 多跳全链路渗透（`b`，3 题）

从外网入口出发，逐层突破到内网多个业务系统，每道题包含多个 flag。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`b-01`](./b-multihop/b-01.md) | 企业门户多跳渗透 | medium | 1200 | 4 |
| [`b-02`](./b-multihop/b-02.md) | APT 全链路渗透 | hard | 1800 | 6 |
| [`b-03`](./b-multihop/b-03.md) | 官网到内网核心数据 | medium | 1200 | 4 |

### 产品与服务漏洞挖掘（`c`，9 题）

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

### 云安全（`d`，6 题）

对象存储、云函数、云主机、临时凭证与云身份等云原生攻击面。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`d-01`](./d-cloud/d-01.md) | S3 兼容对象存储 | easy | 200 | 1 |
| [`d-02`](./d-cloud/d-02.md) | 云函数管理门户 | easy | 200 | 1 |
| [`d-03`](./d-cloud/d-03.md) | 云主机远程内容获取 | hard | 400 | 1 |
| [`d-04`](./d-cloud/d-04.md) | 云存储临时凭证门户 | medium | 300 | 1 |
| [`d-05`](./d-cloud/d-05.md) | 云身份登录授权服务 | hard | 400 | 1 |
| [`d-06`](./d-cloud/d-06.md) | 云存储网关 | medium | 300 | 1 |

### 二进制漏洞利用（`f1`，5 题）

基于 TCP 行协议的 C 服务，分析内存安全缺陷并读取受保护数据。

| 编号 | 题目 | 难度 | 分值 | Flag |
| --- | --- | --- | ---: | ---: |
| [`f1-01`](./f1-pwn/f1-01.md) | 令牌存储服务 | medium | 300 | 1 |
| [`f1-02`](./f1-pwn/f1-02.md) | LRU 缓存服务 | medium | 300 | 1 |
| [`f1-03`](./f1-pwn/f1-03.md) | TLS 心跳模拟服务 | medium | 300 | 1 |
| [`f1-04`](./f1-pwn/f1-04.md) | 响应构造服务 | easy | 200 | 1 |
| [`f1-05`](./f1-pwn/f1-05.md) | 缓冲区写入服务 | hard | 500 | 1 |

### 固件与逆向（`f2`，8 题）

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

## 题目形态

- **容器题**：每道题通过平台 `start` 拉起一个或多个容器，给出 `IP:端口`，题目环境在容器内。
- **单点题（a / c / d / e / f）**：通常只有一个入口或一份制品，flag 在容器文件系统、数据库或服务配置里。
- **多跳题（b）**：入口是一个公网站点，flag 分散在入口、内网 OA、文件服务器、核心业务系统等多台机器上。
- **制品题（e3 / f2）**：除容器外还会给可执行文件、固件或检测器，需要逆向或构造规避检测的制品。
- **评分**：按 flag 计分，难度越高、flag 越多分值越高；取用 hint 会按比例扣减该题后续得分。
