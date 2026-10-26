# e1-04 · 探索记录

对应题解：[`e1-04.md`](./e1-04.md)

## 目标与起点

页面脚注点名两件事：反向代理上的检测，以及分块编码检查。登录表和 e1-01 是同一形状，所以先用同一组用户名看规则是不是同一套，再单独试分块。

## 第一阶段：注释截断已经能过

`admin'--`、`admin'-- -`、`admin'/**/-- -` 都是 302 到 `dashboard.php`。`admin' or '1'='1` 是 403。`admin'#` 到了后端，但是口令错误。和 e1-01 的差别是：这里单独的 `admin'--` 不再 403。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 普通口令 `admin` / `x` | 认证失败 | 200，Invalid username or password |
| 布尔注入 `admin' or '1'='1` | 代理拦截 | 403，nginx Forbidden |
| 同一布尔注入改成 chunked body | 仍然拦截 | 响应码还是 403，没有 Location |
| `admin'#` | 认证失败 | 200，说明 `#` 没有截断口令比较 |
| `admin'--` | 进入仪表盘 | 302 `dashboard.php`，页面上有 flag |

## 这次真正的收获

脚注里的分块检查没有成为这条链路的必要条件。代理拦的是 `or` 这种布尔形式，注释截断用普通的 `application/x-www-form-urlencoded` 就能到后端。flag 在 `dashboard.php`。
