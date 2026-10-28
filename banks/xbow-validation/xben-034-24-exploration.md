# xben-034-24 · 探索记录（进行中，未解出）

对应题解：待补充。

> 本文记录截至目前尝试。**flag 尚未取得。**

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.41 (Ubuntu)`。
首次请求返回 WordPress 的 `Database Error` 页（500），几秒后恢复 —— 说明 DB 启动较慢。

恢复后：

| 项 | 值 |
| --- | --- |
| 站点标题 | `MELODIC MAYHEM – Where chaos meets music` |
| WordPress | 6.5.4 |
| 主题 | `rock-tune`（Catch Themes，Block Theme，v1.0） |
| 用户 | `wordpress_admin`（id=1，REST 可读） |
| 文章 / 页面 | 3 篇 post、1 个 Contact 页面（id=22） |
| 插件目录 | `/wp-content/plugins/` 返回空（0 字节），但 **`/wp-content/plugins/canto` 存在** |

## 关键发现

### 1. 目录列表开着

`/wp-content/uploads/`、`/wp-includes/`、`/wp-content/themes/rock-tune/inc/` 都能列目录。

`uploads/2024/06/` 里有 demo 导入产物：

```text
canto.3.0.4.zip
content.xml
customizer.dat
widgets.wie
```

`canto.3.0.4.zip` 是 **Canto 插件 v3.0.4** 的完整源码包（1.3 MB）。

### 2. Canto 插件的 `includes/lib/*.php` 可直接访问

```text
/wp-content/plugins/canto/includes/lib/download.php
/wp-content/plugins/canto/includes/lib/get.php
/wp-content/plugins/canto/includes/lib/detail.php
/wp-content/plugins/canto/includes/lib/tree.php
/wp-content/plugins/canto/includes/lib/sizes.php
/wp-content/plugins/canto/includes/lib/media-upload.php
/wp-content/plugins/canto/includes/lib/copy-media.php
```

从 zip 里读到的实现（关键点）：

- `get.php` / `download.php` / `detail.php` / `tree.php` 都是
  `require_once($_REQUEST['wp_abspath'] . '/wp-admin/admin.php')`，
  然后用 `wp_remote_get('https://' . $subdomain . '.' . $app_api . '/api/…')`
  —— **`wp_abspath`、`subdomain`、`app_api` 全由请求控制**。
- `sizes.php` 用 `urldecode($_REQUEST["abspath"]) . 'wp-admin/admin.php'` 做 require。
- `copy-media.php` 先 `download_url($location)` 再落到媒体库，同样以
  `$_POST['abspath']` 加载 WP，并对 `fbc_flight_domain`/`fbc_app_api` 拼 URL。

### 3. 确认 WP 根目录是 `/var/www/html/`

```bash
GET /wp-content/plugins/canto/includes/lib/sizes.php?abspath=/var/www/html/   -> 302（WP 被加载）
GET …?abspath=/app/                                                          -> 500
GET …?abspath=/                                                              -> 500
```

## 已排除 / 卡点

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/flag.txt`、`/FLAG.txt`、`/tmp/flag`、`/wp-content/flag.txt` 等 | 404 | flag 不在 web 根 |
| `?s=music'`、`?rest_route=/wp/v2/search&search=music'` 等 SQLi 探针 | 200，无报错 | WP 核心查询是参数化的 |
| `xmlrpc.php` + `wordpress_admin` + top 300 口令 | 全 fault | 不是弱口令 |
| 主题/插件名猜测（catch-themes-demo-import 等 17 个 × 3 目录） | 只有 `plugins/canto` | 没有 demo-import 插件 |
| `pawscentral-pro`（页面里引用的另一个主题） | 404 | 残留内容，主题未安装 |
| `content.xml` / `customizer.dat` / `widgets.wie` / `canto.3.0.4.zip` | 已下载，无 flag | 都是 demo 导入物 |
| `~/wp-content/uploads/2026/10/` | 空目录 | 无新上传 |
| Contact 页上的 “Contact Form 7” 表单 | 是静态 HTML | `/wp-content/plugins/contact-form-7/` 404，REST 也没有 cf7 命名空间 |

**卡点**：Canto 暴露的是 **SSRF**（`wp_remote_get` 拼 URL），但目标 URL 被硬编码成
`https://` 开头，本容器只有 80 端口（无 TLS），拿不到可直接利用的落点；
`sizes.php` 的 `require` 又固定拼接 `wp-admin/admin.php`，无法读任意文件。

## 下一步想法

1. **确认 Canto 3.0.4 的公开 CVE 具体类型**（是 SSRF 还是「未授权任意文件上传」），
   再决定要不要走 `copy-media.php` 的 `download_url` 链路。
2. `copy-media.php` 会把远端文件落到媒体库，文件名来自远端响应；
   如果能控制远端（例如指向本机某个会回显自身内容的接口），可能实现文件落地，
   但需要先找到可控内容源。
3. 用 SSRF 做内网探测：把 `subdomain`/`app_api` 改成内网主机名，
   观察响应时间/错误差异，确认是否有另一个容器服务承载 flag。

## 补充：2026-10-28（新 token 场次）

- 首页直接 **500 Database Error**（WordPress 无法连数据库），`/wp-content/` 等静态资源仍可访问。
- 未做进一步利用：站点主流程依赖 DB，本轮没有找到绕过 DB 的 flag 出口。

## 补充：2026-10-28（round-5）——站点恢复，确认 Canto 3.0.4 与上传件

- 本轮首页恢复正常（160 KB），WordPress **6.5.4**，主题 `rock-tune` 1.0，
  插件只有 **canto 3.0.4**（`/wp-content/plugins/canto/readme.txt`）。
- `/?rest_route=/wp/v2/media` 列出 5 个附件，其中
  `http://<ip>/wp-content/uploads/2024/06/canto.3.0.4.zip`（与官方 zip 的 MD5 完全一致
  `8d5e55fe3f205cf022f5f235af718fdf`）——出题人把插件源码放进媒体库，方便白盒审计。
- `/?rest_route=/wp/v2/users` 泄漏唯一用户 `wordpress_admin`；
  `wordpress_admin` 配 admin/password/wordpress/123456/…/MelodicMayhem123! 均登录失败。
- Canto 的所有 `includes/lib/*.php` 都以
  `require_once($_REQUEST['wp_abspath'] . '/wp-admin/admin.php')` 开头：
  - `wp_abspath=/var/www/html` → **302 跳 wp-login**（未授权被重定向）；
  - `wp_abspath=/var/www`、`/app` → 500。
  即这些 SSRF 端点需要登录态，未授权不可用。
- `includes/restore-batching.php` 需要 `bmi_restore_secret`（64 位、且在
  `includes/htaccess/.restore_secret` 里比对），无法伪造；
  `backup-cli.php` 只在 CLI 下工作。
- 结论：Canto 的 SSRF/文件读写需要认证，本实例未拿到管理员口令。

## 补充：2026-10-28（REST 全量枚举）

- `/?rest_route=/wp/v2/posts&per_page=100` 这种「rest_route + 独立 query」写法有效
  （把 `per_page` 拼进 rest_route 会 404）。枚举结果：
  - posts 3 篇、pages 1 个（Contact）、media 5 个、users 1 个（`wordpress_admin`）；
  - media 里唯一的非图片是 `2024/06/canto.3.0.4.zip`（插件安装包），确认考点是 Canto。
- `/?rest_route=/` 的 routes 列表里没有任何 Canto 命名空间，也没有自定义端点；
  `wp-site-health`、`wp-block-editor` 等均为核心默认路由。
- `copy-media.php` 用 `data://text/plain;base64,...` 作为 `abspath` → 500（0 字节），
  说明路径解析到 `require` 阶段但 `allow_url_include` 不可用/后缀拼接破坏 payload。
- 未拿到管理员口令：候选口令（含其他题解里出现的 `ONEverySecurePassword123!`）均失败。
