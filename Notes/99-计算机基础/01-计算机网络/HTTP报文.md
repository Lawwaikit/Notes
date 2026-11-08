## HTTP报文端

#### 一个请求报文通常长得像这样
```HTTP
POST /api/user HTTP/1.1        ← 请求行：方法 + URL(路径+查询) + 协议版本
Host: example.com              ← 请求头开始
Content-Type: application/json
Content-Length: 27
User-Agent: Mozilla/5.0
Cookie: sessionid=abc123
								← 空行（必须有，标记头结束）
{"name":"张三","age":18}       ← 实体体（GET 通常没有）
```

请求报文：请求行（请求方法、URL、协议版本）、请求头、空行、实体体。

常见请求头（按用途分类）：

| 类别    | 常见头                                                           | 作用                        |
| ----- | ------------------------------------------------------------- | ------------------------- |
| 目标主机  | `Host`                                                        | **必带**，指明要访问的域名（虚拟主机靠它区分） |
| 内容描述  | `Content-Type`、`Content-Length`                               | 实体体是什么格式、有多长              |
| 客户端身份 | `User-Agent`、`Cookie`、`Authorization`                         | 浏览器标识、会话凭证、认证令牌           |
| 内容协商  | `Accept`、`Accept-Encoding`、`Accept-Language`、`Accept-Charset` | 我能接受什么类型/压缩/语言/字符集        |
| 连接控制  | `Connection: keep-alive/close`                                | 是否复用连接                    |
| 缓存/条件 | `If-Modified-Since`、`If-None-Match`                           | 条件请求，配合缓存                 |
	


#### 响应报文：
```http
HTTP/1.1 200 OK                ← 状态行：协议版本 + 状态码 + 状态码描述
Date: Fri, 23 Oct 2026 10:00:00 GMT   ← 响应头开始
Server: nginx/1.24
Content-Type: text/html; charset=utf-8
Content-Length: 1024
Cache-Control: max-age=3600
Set-Cookie: sessionid=abc123; HttpOnly
                               ← 空行（必须有）
<html>...</html>               ← 实体体（真正的响应内容）
```

响应报文：状态行（协议版本、状态码 、状态码描述）、响应头、空行、实体体。

常见响应头（按用途分类）：

| 类别     | 常见头                                                                              | 作用               |
| ------ | -------------------------------------------------------------------------------- | ---------------- |
| 时间/服务器 | `Date`、`Server`                                                                  | 响应生成时间、服务器软件     |
| 内容描述   | `Content-Type`（含 charset）、`Content-Length`、`Content-Encoding`、`Content-Language` | 实体体格式、长度、压缩方式、语言 |
| 缓存控制   | `Cache-Control`、`Expires`、`ETag`、`Last-Modified`                                 | 缓存策略与校验          |
| 会话/认证  | `Set-Cookie`、`WWW-Authenticate`                                                  | 下发 Cookie、要求认证   |
| 重定向    | `Location`                                                                       | 配合 3xx 告诉客户端新地址  |
| 连接控制   | `Connection`                                                                     | 是否保持连接           |

## POST和GET区别？
| 对比项        | GET                                         | POST                                                        |
| ---------- | ------------------------------------------- | ----------------------------------------------------------- |
| **语义/场景**  | 安全方法，只读取资源，不改变服务器状态；**幂等**                  | 非安全方法，可能创建/修改资源；**非幂等**（重复提交会重复创建）                          |
| **典型用途**   | 获取网页、查询列表、搜索                                | 注册、登录、提交表单、上传文件、创建资源                                        |
| **传参位置**   | URL 查询字符串（`?a=1&b=2`）                       | 请求体 body（也可带 query，但主流在 body）                               |
| **缓存**     | **默认可被浏览器/代理缓存**                            | **默认不缓存**（除非响应头显式允许）                                        |
| **安全性**    | 参数在 URL，易进入历史记录、日志、Referer，**不宜放敏感信息**      | 参数在 body，相对不易直接暴露；但**两者都不等于安全，敏感数据必须 HTTPS**                |
| **长度限制**   | 受**浏览器/服务器对 URL 长度限制**（非协议规定），如约 2KB~8KB 不等 | 理论上无限制，受**服务器 body 大小配置**约束（如 Nginx `client_max_body_size`） |
| **数据类型**   | 数据须编码进 URL，非 ASCII 需百分号编码；**不适合二进制/大文件**    | body 支持多种编码：表单、JSON、`multipart/form-data`（**可传文件/图片**）      |
| **可见性/书签** | 参数可见，可被收藏、分享、刷新重放                           | 参数不可见，不能直接收藏为带参链接                                           |
| **幂等性**    | 幂等（多次请求结果一致）                                | 非幂等（多次提交可能产生多条记录）                                           |
