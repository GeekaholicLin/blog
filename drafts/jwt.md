### 概述

[JSON Web Token](https://tools.ietf.org/html/rfc7519) 是一个非常轻巧的规范，允许我们使用 JWT 在客户端和服务器之间传递安全可靠的消息。

JWT 实际上是字符串，由三部分组成--Header 头部、Payload 负载以及 Signature 签名，格式为`Header.Payload.Signature`。

头部用于描述关于该 JWT 最基本的信息，比如类型以及签名所用的算法，比如`{ "typ": "JWT", "alg": "HS256" }`，表示类型 Token 的类型为`JWT`且加密算法为`HS256`。对此 JSON 对象字符串进行 Base64 编码，得到头部字符串`eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9`。

负载是 Token 的具体内容，可以填写部分标准字段的同时掺合其他字段。也是 JSON 字符串，然后进行 Base64 编码。标准字段为：

- iss: Issuer，发行者
- sub：Subject，主题
- aud: Audience，观众或接受者
- exp：Expiration time，过期时间
- nbf: Not before，在此时间之前 JWT 不接受处理
- iat: Issued at，发行时间
- jti: JWT ID

JWT 的最后一部分是签名。先使用经过 Base64 编码的`header`和经过 Base64 编码的`payload`使用`.`进行连接成新的字符串，并使用 header 中指定的加密算法以及特定的密钥`secret`进行加密。处理过程用伪代码表示如下：

```js
const encodedString = base64UrlEncode(header) + "." + base64UrlEncode(payload);
HMACSHA256(encodedString, 'secret');
```

签发与验证 JWT 的功能包，可以使用`jsonwebtoken`。

因为 header 以及 payload 是 Base64 编码，所以不要存放敏感数据。而最后一步的签名过程使用了加密算法以及特定的 secret，即使 header 或 payload 被篡改，不知道 secret 准确的值，服务器端将篡改的 JWT 进行加密对比也会不相同。

JWT 适合用于向 Web 应用传递一些非敏感信息，经常用于用户认证（Authentication）以及实现 Web 应用的单点登录。

### JWT VS Session

Session 方式可能需要保存许多状态，存储占用大量服务器内存，而且需要借助缓存机制以及同步机制。而 JWT 是将用户状态分散到客户端，减轻服务端的内存压力。

### SSO 单点登录

美团的 SSO 单点登录是基于 CAS 协议进行简化，采用标准的 oauth2.0 流程。Nodejs 的核心流程如下：

![SSO 单点登录](http://image.geekaholic.cn/20191022153611.png@0.8)

特别说明：

1. 业务方种下的 ssoid 有效期只能固定 3 天 , 业务方每次请求都要用 ssoid（来自 HTTP 头部`access-token`或`${clientId}_ssoid`的 cookie） 到 sso 验证其有效性（这个 SDK 已经做了）
2. 在上图步骤 5 中，扫码/账号密码 登录后，sso 域名下会种上叫 TGC 的 cookie，TGC 的有效期较长
3. 当 ssoid 失效，TGC 未失效时，在步骤 4 时，sso 会用 TGC 让用户自动重新登入，直接到步骤 5

开放平台、IAM 以及 SSO 的架构：

![开放平台-IAM-SSO](http://image.geekaholic.cn/20191022155825.png@0.8)