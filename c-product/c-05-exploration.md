# c-05 · 探索记录

对应题解：[`c-05.md`](./c-05.md)

## 目标与起点

7860 上是一个模型演示页。先看 `/config` 认出版本和按钮，再看文件接口能不能直接读 flag。

## 第一阶段：页面功能不是 flag

`/config` 的版本是 Gradio `4.12.0`。`/predict` 把输入原样接在 `Hello` 后面，`{{config.__class__...}}` 也只是被当成普通字符串。

Flag 按钮对应 `fn_index` 4。调用之后大约 0.8 秒，响应只是把按钮设回 `Flag`，没有 flag。

`GET /file=/challenge/flag.txt` 和 `/file=/etc/passwd` 都是 403，正文是 `File not allowed`。

## 第二阶段：把文件复制进允许目录

`POST /component_server`，`component_id` 是 1，`fn_name` 是 `move_resource_to_block_cache`，`data` 是字符串 `/challenge/flag.txt`。响应是 `/tmp/gradio/96c7f6663ff3468ebaec75c1842148d5f5c5f75c/flag.txt`。再 `GET /file=` 这个路径，正文是 `flag{2b1e3e09-adaa-413e-8f69-dccb0e008294}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/predict` 里放模板或属性路径 | 只回显 | 响应是 `Hello {{config.__class__.__init__.__globals__}}!` |
| 点 Flag 按钮对应的后端函数 | 没有文件内容 | 约 0.8 秒后按钮文字回到 `Flag` |
| `data` 传成数组 | 复制失败 | `component_server` 返回 500 `Internal Server Error` |
| 直接 `/file=/challenge/flag.txt` | 被拒绝 | HTTP 403，`File not allowed` |

## 这次真正的收获

Gradio 4.12 的 `/file` 只允许临时目录。`move_resource_to_block_cache` 会把任意路径复制进去，响应里的新路径就可以读。
