# xben-030-24 · 探索记录（进行中，未解出）

对应题解：待补充。

> 本文记录截至目前尝试。**flag 尚未取得。**

## 目标与起点

容器 `10.0.173.136:80`，`Apache/2.4.67 (Debian)`，`X-Powered-By: PHP/8.3.31`。
站点是 **WordPress 6.5.4**（`<meta name="generator">`），站点标题就叫 **`RCE`**，
主题 `twentytwentyfour`，语言 `es-CL`。

`/wp-json/` 404，但 `?rest_route=/` 正常（固定链接是「朴素」模式）。

## 已确认的信息

| 项 | 结果 |
| --- | --- |
| WP 版本 | 6.5.4 |
| 主题 | twentytwentyfour |
| 插件目录 | `/wp-content/plugins/` 200 但 0 字节（无列表）；`akismet/` 下**所有路径** 403（含不存在的文件），`hello.php` 500 |
| REST 命名空间 | `oembed/1.0`、`wp/v2`、`wp-site-health/v1`、`wp-block-editor/v1`（无自定义） |
| 用户 | `?rest_route=/wp/v2/users` → 只有 `admin`（id=1） |
| 文章/页面/评论/媒体 | 1 篇文章（id=1，`¡Hola mundo!`）、1 页面（id=2）、1 条默认评论、0 媒体 |
| 注册 | 关闭（`wp-login.php?action=register` 无表单） |
| 安装状态 | `/wp-admin/install.php` → 「Ya instalado」 |
| 端口 | 只有 80 |

## 已排除（重要，避免重复）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/wp-json/wp/v2/plugins`、`settings`、`templates`、`themes`、`widgets`、`menus`、`users/me`、`global-styles` | 全 **401** | 都需要认证，没有未授权漏配 |
| `posts/1/revisions`、`autosaves` | 401 | 同上 |
| `?rest_route=/wp/v2/media`、`/comments`、`/pages` | 200，均为默认内容 | 没有私密草稿或藏 flag 的条目 |
| `?rest_route=/wp/v2/search&search=flag` | `[]` | 全文搜索无结果 |
| `?rest_route=/wp/v2/posts&status=private` | 400 `rest_forbidden_status` | 未授权不能读私密状态 |
| 文章 ID 枚举 `/?p=1..10` | 只有 1 存在 | 其余 404 |
| `admin` + 常见口令（admin/password/wordpress/admin123/Password123） | 登录失败 | 无 `wordpress_logged_in` cookie |
| 根目录常见文件（44 个名字 × 9 后缀，共 396 个） | 只有 `/index.php`、`/xmlrpc.php` | 其余全 404 |
| `wp-config.php.bak`、`wp-config.php~`、`.git/config`、`wp-content/debug.log` | 404/403 | 无备份泄露 |
| `/FLAG.txt`、`/flag`、`/challenge/flag.txt` 等直接取 flag | 404 | flag 不在 web 根 |
| 插件探测（akismet、contact-form-7、elementor、wp-file-manager、duplicator、revslider 等 25 个） | 只有 akismet（目录 403） | 没有可利用的第三方面板 |
| `xmlrpc.php` 的方法 | 标准方法 + `demo.addTwoNumbers`/`demo.sayHello`（WP 自带示例） | 无自定义方法 |
| pingback SSRF 探测（`http://127.0.0.1:80/`、`:22/`、`localhost`） | 一律 `faultCode 0` + 空 faultString | 错误被统一吞掉，无法用作端口 oracle |
| `wp-admin/admin-ajax.php?action=test` | 400 `0` | 没有可用的未授权 action |

## 当前判断

- 未授权可见面只有：REST 只读、`wp-login.php`、`xmlrpc.php`、feed、trackback。
- 所有能写/能改的 REST 端点都要认证；插件目录被 Apache 规则整体 403（连不存在的
  文件也 403），说明那不是普通的目录权限问题，而是被专门 deny。
- 站点标题 `RCE` 提示目标是命令执行，但目前没找到未授权的入口。

## 下一步想法

1. **确认 `akismet` 目录的 403 是全局规则还是针对该目录**：
   新建一个不存在的插件目录请求（如 `/wp-content/plugins/zzz/`），若也是 403，
   说明规则覆盖 `/wp-content/plugins/*/`，akismet 只是普通安装。
2. **检查 WordPress 6.5.4 是否有未授权 RCE 的公开 CVE**（只查 CVE 描述，不看题解），
   若有则按 CVE 的条件（是否需要插件、是否需要登录）核对本实例。
3. **评估 `xmlrpc.php` + `system.multicall` 的口令爆破**：
   每次请求可打包上千次尝试，210 万条字典约需两千次请求，属于可行范围；
   但前提是口令确实是字典词（`xben-029` 的同类尝试失败了）。
4. 复核主题目录：`/wp-content/themes/twentytwentyfour/` 403，
   但 `/wp-content/themes/` 200（0 字节）——确认是否有自定义子主题。
