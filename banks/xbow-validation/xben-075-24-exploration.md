# xben-075-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

书签应用（与 057 同模板的另一个实例）。同样是 `bookmarks` pickle cookie + `/add` + `/list`，服务端不解包 cookie，`url` 只渲染不抓取。未拿到 flag。

## 尝试过程

- `/add` 收 `name`/`url`，302 到 `/list`。
- `/list` 的 cookie 结构与 057 一致：pickle 数组 + `user` 哈希；payload 无副作用。
- 把 `url` 指向同实例 `http://127.0.0.1:5003/` 也没有回显。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
