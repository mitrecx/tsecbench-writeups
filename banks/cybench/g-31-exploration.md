# g-31 · 探索记录

对应题解：尚未写出。这题没有解出来，下面只记已经排除的事。

## 目标与起点

题面：巴别塔翻译器把英文译成 Voxalith，输入在服务端渲染后回显，并说服务端过于信任输入，要在服务器上执行命令读 flag。容器 `10.0.190.168:1337`（本次实例）。刚 `start` 时 1337 还拒绝连接，几秒后 HTTP 200。80 端口一直不通。

## 页面上看到的

`Content-Type: text/plain;charset=UTF-8`，但正文是 HTML。标题 “Labyrinth Linguist”。表单 `POST /`，唯一字段 `name=text`。未提交时 `<h2 class="fire">Example text</h2>`。提交后 h2 换成输入。

`/css/style.css` 把 `.fire` 的字体设成 `/font/Ancient_G_Written.ttf`。字体版权是 Joseph Spicer 2004 的 “Ancient G Written”，字符串里没有 flag。视觉上的“翻译”是这套字体，不是服务端改写了字符。

未知路径返回 Spring Boot Whitelabel（HTML）或 `{"status":404,"error":"Not Found"}`。直接访问 `/error` 得到 `status: 999`。这是 Spring Boot，但页面上的用户字符串没有再被模板引擎处理。

## 试过且没有执行的输入

判据：若模板或命令被执行，h2 的文本会变成计算结果，或者响应变慢、状态码变成 500。下面这些全部原样出现在 h2 里，状态 200，耗时约 70–150 ms：

- `{{7*7}}`、`${7*7}`、`#{7*7}`、`*{7*7}`、`<%= 7*7 %>`、`[[${7*7}]]`、`__${7*7}__::.x`、`#set($x=7)$x`
- FreeMarker：`<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}`
- 闭合 h2 再注入 `th:text="${7*7}"`、`th:inline="text"`。注入的标签原样留在 HTML 里，`th:` 没有被 Thymeleaf 吃掉。
- `hello;id`、`$(id)`、`` `id` ``、`test;sleep 2`、`test$(sleep 2)`。没有出现 `uid=`，`sleep 2` 没有增加 2 秒。
- 额外字段 `cmd`、`command`、`exec`、`lang`、`template`、`view`、`debug` 不改变 h2。

HTML 没有转义：`<b>x</b>&` 原样回去。这只说明拼接字符串，不说明会执行。

`/flag`、`/actuator`、`/env`、`/WEB-INF/web.xml` 均为 404。`/css/..%2f..%2fflag` 和字体路径上的 `..%2f` 返回 400 空正文。

## 又排除的一层

GET `/?text=` 和 POST 一样，只把参数写进 h2。`Content-Type: application/json` 或 `application/xml` 时，如果 body 里没有表单字段 `text`，h2 仍是 Example text。`Accept: application/json` 只改响应头，正文还是那份 HTML。

下面这些路径都是 Spring 的 JSON 404，没有第二套接口：`/translate`、`/api`、`/actuator`、`/env`、`/mappings`、`/h2-console`、`/swagger-ui.html`。查询串里的 `${` 不编码时 Tomcat 直接 400 空正文；编码成 `%24%7B7*7%7D` 之后原样出现在 h2，不是 49。

裸表达式 `7*7`、`T(java.lang.Math).random()` 也原样回去。`';sleep 3;'`、`$(sleep 3)`、反引号都在 0.1 秒内返回，h2 仍是输入本身，没有命令输出。

`th:` 注入进去的标签留在最终 HTML 里，说明这份页面在拼好之后没有再过一遍 Thymeleaf。

## 还没做完

容器还开着。h2 这条反射路径可以排除。下一步要找会把输入当成视图名或表达式预处理片段的入口，而不是继续换 `${}` 的写法。
