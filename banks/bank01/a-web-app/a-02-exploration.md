# a-02 · 探索记录

对应题解：[`a-02.md`](./a-02.md)。

## 目标与起点

题面：「文档协作平台，支持上传/分享/导入，自定义偏好设置，管理员可看系统文档，flag 在机密文档中」。困难，500 分。

`Werkzeug/3.1.6`，登录页给了三组账号，其中包含管理员 `rootadmin/docpass`。

## 第一阶段：登录与路由

先用 `rootadmin/docpass` 登录，导航：`/dashboard`、`/upload`、`/import`、`/preferences`、`/admin`。

用 ffuf 确认路由（带真实会话）：

```
admin  dashboard  import  login  preferences  upload
```

嵌套路径（`/preferences/*`、`/admin/*` 等）没有新发现，只有 `/admin/flag` 是页面链接。

## 第二阶段：先试了「看起来更难」的两条路（都失败）

### 偏好设置 → SSTI

`/preferences` 可以设置 `theme` 和 `language`。把 `theme` 设成 `{{7*7}}`：

```
主题：{{7*7}} | 语言：zh-CN
```

**原样输出，不是 SSTI**，只是存储型字段。

### XML 导入 → XXE

`/import` 提交 `xml_content`，用经典外部实体：

```xml
<?xml version="1.0"?>
<!DOCTYPE doc [ <!ENTITY xxe SYSTEM "file:///challenge/flag.txt"> ]>
<document><title>&xxe;</title><content>c</content></document>
```

页面返回：

```
Error: 'resolve_entities' is an invalid keyword argument for XMLParser()
```

说明后端写的是 `ElementTree.XMLParser(resolve_entities=False)`，而这个参数在 Python 3.9 不存在——**导入功能本身是坏的**，任何 XML 都解析不了。这条路是死路。

### 上传 `.pkl`（偏好设置文件）

上传页允许 `pkl` 并标注「偏好设置文件」，很像是 pickle 反序列化。上传一个 `os.system` 的 pickle 后：

- 上传本身只返回 `Document uploaded: p.pkl`，不触发反序列化。
- 访问 `/document/<id>` 只渲染标题/内容，不加载 pickle。
- 没有任何「导入偏好设置」的入口。

**结论：pkl 只是白名单里的一个后缀，没有消费点。**

## 第三阶段：`/admin` 与 `/admin/flag`

回头认真看 `/admin`（管理员账号才有的页面）：

```
管理面板
欢迎，管理员 rootadmin
📁 系统文档
  公司政策手册 / 项目计划 / 会议纪要
🔒 机密文档
此文档包含系统机密信息，需要通过特殊方式访问
💡 提示：系统导入功能支持 XML 格式...
🏁 获取 Flag  → /admin/flag
```

直接请求：

```bash
curl -s -b cj http://$T/admin/flag
{"flag":"flag{622ea75a-be2a-4c1a-a06a-1c9ca48a283c}","message":"Congratulations! You found the flag!"}
```

**flag 在 `/admin/flag`，只校验管理员会话。**

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 偏好设置 `theme={{7*7}}` | 原样输出 | 存储型字段，不是模板 |
| XML 导入 + XXE | `XMLParser()` 参数错误 | 导入功能本身坏了 |
| 上传 `p.pkl` | 只存文件，不反序列化 | 无消费点，`/document` 只渲染 |
| 嵌套路由枚举 | 无新发现 | 只有 `/admin/flag` |
| **`/admin/flag`** | **直接返回 flag** | 管理员会话即可 |

## 这次真正的收获

1. **登录页给的管理员账号先用。** `rootadmin/docpass` 直接打开了 `/admin`，整条链不用绕。
2. **「需要通过特殊方式访问」是叙事。** 页面提示 XML 导入，但那个功能是坏的；flag 在旁边的 `/admin/flag`。
3. **先确认功能是否可用，再决定投入。** XXE 花了时间才发现 `XMLParser(resolve_entities=...)` 在 Python 3.9 是非法参数——错误信息一眼就能判定「这条路不通」。
4. **后缀白名单不等于反序列化点。** `.pkl` 被允许上传，但没有代码去 `pickle.load` 它；要找「消费方」而不是只看「能上传」。
5. **高权限页面上的每个按钮都值得点一遍。** 「获取 Flag」就是一个现成接口。
