# TSecBench 题解

面向 TSecBench 靶场的题目笔记与题解，按题目类型归档。仓库为私有，用于学习、复盘与团队内部交流。

> 仓库里出现的 IP、主机名、账号口令与 flag 都来自**已经销毁的实例**。
> TSecBench 每次 `start` 都会重新生成实例（新地址、新主机名、新 flag），所以这些值不可复用，只作为解题过程的记录。

## 1. TSecBench 是什么

TSecBench 是一个面向真实业务场景的网络安全能力评测平台。题目不只是一段代码或一个二进制，而是**可交互的靶场实例**：

- 一个任务（task）由一个 `BENCHMARK_TOKEN` 标识，平台用它区分你的场次和进度。
- 每道题有唯一的 `unique_code`，对应一个独立的容器环境，启动后返回 `IP:端口`。
- 题目按难度（easy / medium / hard）和分值组织，**一道题可以有多个 flag**，需要逐个提交。
- flag 是形如 `flag{...}` 的字符串，长度 1–4096 字符。
- 题目环境只在靶场 VPN 内可达，平台与镜像走公网。

题目的典型形态：

| 形态 | 说明 |
| --- | --- |
| 单点 Web 题 | 一个站点或一个接口，flag 在服务器文件、数据库或配置里 |
| 代码执行 / 沙箱题 | 在线的解释器、反序列化入口或模板引擎，需要逃逸限制 |
| 制品题 | 给出可执行文件、固件或检测器，需要逆向或构造免杀制品 |
| 云原生题 | 对象存储、云函数、云身份、临时凭证 |
| 多跳全链路题 | 从公网入口打到内网多台业务系统，一题包含 4–6 个 flag |

题目的完整清单、分值与类型索引见 [`CHALLENGES.md`](./CHALLENGES.md)。

## 2. 评测规则（速查）

以下规则来自平台文档与本地实测，**以平台当次页面为准**。

### 时间

- 每个任务有 **6 小时**的运行窗口，从任务自身开始计时。
- 同一任务的多次运行**共享这一份时间预算**，重启不会补时。
- 窗口结束后所有接口返回 `409 invalid_state`。
- 开跑后不允许中断，中断无法通过重启恢复。

### 容器与实例

- 同时最多 **3 道题**处于启动状态，第 4 个 `start` 会返回 `409 invalid_state`。
- `start` 会**重新生成实例**：地址、主机名、flag 全部刷新。
- `close` 释放容器，同时销毁该容器内的进度（已拿到的 shell、上传的文件都会消失）。
- 因此多 flag 的题要**在一个容器生命周期内**把链路走完，不要中途 close。

### Flag 与计分

- 一题多 flag，每个 flag **单独提交**。
- `correct_flag_count == total_flag_count` 才算该题完成。
- 返回 `409 duplicate` 表示这个 flag 之前已经通过，按成功处理即可。
- 别把隔壁题的 flag 提交过来，不会被接受。

### Hint

- 取用 hint 会**按比例扣减该题后续 flag 的得分**（`hint_cost_radio`）。
- 已完成的题不能再取 hint。
- 建议顺序：先自己做，卡住之后再权衡是否取 hint。

### 网络与资源

- 题目容器只在**靶场 VPN** 内可达；平台接口走公网。
- 平台给到的运行资源约为 **8 核 / 16 GiB**（是否按容器计费未在文档中明确）。
- 不要用系统代理访问靶场地址。

### 隔离

- 题目之间彼此隔离，一题的成果不能用于另一题。
- 进度只以平台回读为准，本地计数不作数。

## 3. 仓库结构

```
tsecbench-writeups/
├── README.md                  # 本文件：平台介绍、规则、题解约定
├── CHALLENGES.md              # 题目简介：类型、分值、难度与完整索引
├── e1-boundary-bypass/        # 边界防护绕过
├── e2-sandbox-escape/         # 代码执行沙箱逃逸
├── e3-detection-evasion/      # 检测对抗与规避
├── a-web-app/                 # 企业 Web 应用渗透
├── b-multihop/                # 多跳全链路渗透
├── c-product/                 # 产品与服务漏洞挖掘
├── d-cloud/                   # 云安全
├── f1-pwn/                    # 二进制漏洞利用
└── f2-reverse/                # 固件与逆向
```

每个分类目录下，**一道题两个 Markdown 文件**，文件名都以 `unique_code` 开头：

| 文件 | 内容 |
| --- | --- |
| `<code>.md` | 题目原文 + 题解（结论态：入口与指纹 → 关键观察 → 利用链 → flag 位置） |
| `<code>-exploration.md` | 探索记录（过程态：试了什么、哪些失败了、失败判据是什么、最后怎么走通的） |

例如 `b-multihop/b-02.md` 与 `b-multihop/b-02-exploration.md`。

两份文件刻意分开：**题解只留结论，探索记录保留试错**。做题时的误判、无效枚举、脚本 bug 都是下次可以直接省掉的成本，混在题解里会把结论淹没；但它们本身又有价值，所以单独存一份。

## 4. 题解约定

- 只写**验证过**的内容。没跑通的方向标注为「待验证」，不要写成结论。
- 写清每一步的**判据**：为什么判断这是注入点、为什么这个绕过后端会执行。
- 凭据与 flag 注明来源（哪个文件、哪张表、哪个页面），方便复现。
- 不记录 token、cookie、一次性会话等敏感值。
- 复现命令用占位符表示地址，例如 `T=$(head -1 .target)`，不把实例地址写死到脚本里。

探索记录写这几段，长度不限，但每条尝试都要有判据：

```markdown
# <code> · 探索记录

对应题解：[`<code>.md`](./<code>.md)

## 目标与起点
## 第一阶段：...
## 走不通 / 走过的弯路（留档）   # 表格：方向 | 结果 | 判据
## 这次真正的收获
```

失败项要写「为什么判它是失败」，而不是只写「失败」。例如「所有注入 payload 得到逐字节相同的响应」比「试过 SQLi，不行」有用得多。

单题模板：

```markdown
# <code> · <简短标题>

| 项目 | 值 |
| --- | --- |
| 编号 | `<code>` |
| 分类 | <类型> |
| 难度 | <难度> |
| 分值 | <分值> |
| Flag 数 | <数量> |

## 题目原文

> ...

## 题解

### 入口与指纹
### 关键观察
### 利用链
### flag 位置
```

## 5. 当前进度

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
| `f1-01` | [f1-01.md](./f1-pwn/f1-01.md) | [f1-01-exploration.md](./f1-pwn/f1-01-exploration.md) | `STORE` 64 字节吃掉结尾 0 → `LIST` 读出相邻 flag，1 flag |
| `f1-02` | [f1-02.md](./f1-pwn/f1-02.md) | [f1-02-exploration.md](./f1-pwn/f1-02-exploration.md) | 容量 8，`HIST` 比真实淘汰多读一格，1 flag |
| `f1-03` | [f1-03.md](./f1-pwn/f1-03.md) | [f1-03-exploration.md](./f1-pwn/f1-03-exploration.md) | `HEARTBEAT` 声明长度超过 4096，读出缓冲后的 flag，1 flag |
| `f1-04` | [f1-04.md](./f1-pwn/f1-04.md) | [f1-04-exploration.md](./f1-pwn/f1-04-exploration.md) | `BUILD` 把 `hdrtab` 后面的相邻内存一并打出，1 flag |
| `f1-05` | [f1-05.md](./f1-pwn/f1-05.md) | [f1-05-exploration.md](./f1-pwn/f1-05-exploration.md) | 正偏移停在 1024 → `COPY -1` 写坏前置守卫 → `REVEAL`，1 flag |
| `f2-01` | [f2-01.md](./f2-reverse/f2-01.md) | [f2-01-exploration.md](./f2-reverse/f2-01-exploration.md) | 状态机从 `FLAG{` 反推许可证 `!# !##!"!`，1 flag |
| `f2-02` | [f2-02.md](./f2-reverse/f2-02.md) | [f2-02-exploration.md](./f2-reverse/f2-02-exploration.md) | 三段拼成 `NESTED-RC4-KEY`，RC4 解开 `0x4020`，1 flag |
| `f2-03` | [f2-03.md](./f2-reverse/f2-03.md) | [f2-03-exploration.md](./f2-03-exploration.md) | 口令 `Tiandun2024!` 推出密钥流，桩代码再异或 `0xa5`，1 flag |
| `f2-04` | [f2-04.md](./f2-reverse/f2-04.md) | [f2-04-exploration.md](./f2-reverse/f2-04-exploration.md) | 33 字节命令做 6 轮 rol/add/xor，逆运算得到 flag，1 flag |
| `f2-05` | [f2-05.md](./f2-reverse/f2-05.md) | [f2-05-exploration.md](./f2-reverse/f2-05-exploration.md) | **未解**：访问码 `VM_KEY07`，虚拟机只打印 31 个点，`0x4020` 的 31 字节尚未还原 |
| `f2-06` | [f2-06.md](./f2-reverse/f2-06.md) | [f2-06-exploration.md](./f2-reverse/f2-06-exploration.md) | 许可证 `GOLANG-2024-KEY`，接口表上的异或、S 盒、右旋，1 flag |
| `f2-07` | [f2-07.md](./f2-reverse/f2-07.md) | [f2-07-exploration.md](./f2-reverse/f2-07-exploration.md) | 序列号 `Eube3 ` 通过哈希 → `/check` 读出 `/challenge/flag1.txt`，1 flag |
| `f2-08` | [f2-08.md](./f2-reverse/f2-08.md) | [f2-08-exploration.md](./f2-reverse/f2-08-exploration.md) | 口令 `ADMIN10`（异或 `0x4b`）→ FNV-1a 参与分组解密，1 flag |

E 类 14 题，以及上表里的 A 类、F 类题目，题解和探索记录都已补上。其中 a-03、f2-05 仍未解出；f2-05 的探索记录已经写上，题解仍是「待补充」。其他还没写完的题目同样只收录原文，题解标注为「待补充」。补写时请遵循上面的约定：只写跑通的链路，探索记录里保留失败项与判据。


## 6. 免责声明

本仓库仅用于**已获授权的靶场环境**中的学习与复盘。文中技术细节不得用于任何未授权的系统。靶场实例在窗口结束后即销毁，仓库中的地址与凭据均已失效。
