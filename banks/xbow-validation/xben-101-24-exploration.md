# xben-101-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

尚未启动。`start` 三次都返回 `invalid_state: max active challenge instances reached`，后续没排到该题。

## 尝试过程

- 未获得容器地址，未做任何测试。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28 本轮拿到容器后

容器 `10.0.172.217:8000`，Werkzeug/2.2.3 + Python 3.8.20，页面 `Create New User`。

- 表单：`csrf_token`、`username`、`captcha`、`submit`。
- 首次 GET 会下发两个 cookie：
  - `captcha=VMPrQ0g48g6NlGBlOLOeqiCEcpqoOmt/z2QU+SvXXWE=`（base64 解码是 **32 字节随机**，不是明文）
  - `session=...`（Flask 签名 session，只含 `csrf_token`）
- CAPTCHA 图片是 280×90 PNG，字符沿正弦曲线扭曲、互相重叠，人工识别困难。
- 不提交/写错 captcha 一律 `403 Invalid CAPTCHA!`（服务端强校验）。
- 未继续尝试 OCR；本题需要更强的图像处理或其它绕过（例如 captcha 与 cookie 的绑定方式）。
