- authentication: 认证。HTTP 中相关的响应头部可能是`WWW-Authenticate`，表示用于获取资源的认证方法。比如`WWW-Authenticate: Basic`。一般与响应状态码`401 Unauthorized`一起发送。

- authorization: 授权。HTTP 中的响应状态码`401 Unauthorized`，以及相关的 HTTP 请求头部`Authorization`

![BA 认证](http://image.geekaholic.cn/20191025110755.png@0.8)

图为前端认证方式中的 BA 认证，一般用于内部安全性要求不高的系统中。BA 认证的大致流程如下：

- 未认证的情况下访问受限资源
- 服务器端返回 `401 Unauthorized` 状态码以及`WWW-Authenticate: Basic realm="Access to the staging site"`，指示浏览器使用 Basic 的方式进行验证，保护区域为`realm`指定部分。
- 浏览器弹出验证弹窗，让用户输入账号密码，点击确认。且`realm`可能会被显示成`网站名称：Access to the staging site`
- 浏览器重新发起请求，`Authorization: Basic <credentials>`作为验证头部，其中`Basic`为验证方法，`<credentials>`为账号密码以`username:password`的方式拼接且进行 Base64 编码后的字符串。
- 根据用户请求资源的地址，确认对应的`realm`。解析出`Authorization`中的账号密码，判断是否匹配，且判断当前用户是否拥有此`realm`权限。如果账号密码不匹配或者资源不匹配，则返回`403 Forbidden`状态码

下边为 Nginx 的例子，其中`.htpasswd`文件是包含了加密用户凭证的文件，文件中的每行以`username:MD5(password)`的形式存储

```nginx
location /status {
    auth_basic "Access to the staging site";
    auth_basic_user_file /usr/local/nginx/conf/vhost/nginx_passwd;
}
```

为了信息安全，应该使用 HTTPS 进行加密传输。因为 Base64 编码可逆，如果以明文形式在网络中进行传输，会被提取账号密码。但是目前而言，外网中很少使用 BA 认证。在上古时代我们可能还会见过形如`https://username:password@www.example.com/`的 URL，这是使用编码的 URL 进行 BA 认证的同时避免弹出登录框，但是因为安全问题已经废弃。

更多内容详见 [一文读懂 HTTP Basic 身份认证](https://juejin.im/entry/5ac175baf265da239e4e3999) 以及 [HTTP 身份验证](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Authentication)

HTTP 认证方式中除了 BA 认证，常见的还有：

- 基于 Session 的认证
- 基于 Token 的认证（JWT）
- OAuth: 不是认证方式，是属于授权方式的一种。但是授权前需要验证，结果类似于"认证+授权"。目前常用 OAuth 2.0，像 Google、Github、QQ、微博授权第三方网站。
- OpenID：允许你使用已经存在的账号登录多个网站而不需要创建新的帐号密码。与 OAuth 的区别在于，OpenID 侧重于认证，除了 OpenID 中提供的身份信息，无法获取用户在授权方的其他数据与权限。

![OAuth 与 OpenID](http://image.geekaholic.cn/20191025123224.png@0.8)

### 拓展

深入阅读 [浅谈 SAML, OAuth, OpenID 和 SSO, JWT 和 Session](https://juejin.im/post/5b3eac6df265da0f8815e906) 以及公司内部的 SSO 核心原理（anki#JWT）
