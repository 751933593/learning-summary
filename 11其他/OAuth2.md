## OAuth2(open authorization)

[文章链接](https://www.cnblogs.com/linianhui/p/oauth2-authorization.html)

### 1.简介

OAuth2是一个开放授权标准，允许用户让第三方应用访问，通过授权机制已经授权的服务器接口

### 2.四个角色

- **Resource Owner**：资源拥有者；
- **Resource Server**：资源服务器；
- **Client**：第三方应用客户端；
- **Authorization Server** ：授权服务器，管理Resource Owner，Client和Resource Server的三角关系的中间层。

OAuth工作流程：

![OAuth工作流程](../img/OAuth2/1560929469(1).jpg)

### 3.OAuth2四种授权许可

- Authorization Code 授权码

- implicit 隐式许可
- Resource Owner Password Credentials 资源拥有者密码凭据
- Client Credentials 客户端凭据

#### 3.1Authorization Code

（1）**Client**使用浏览器（用户代理）访问**Authorization server。**也就是用浏览器访问一个URL，这个URL是**Authorization server**提供的，访问的收Client需要提供（客户端标识，请求范围，本地状态和重定向URL）这些参数。

Authorization Server的请求参数：

- response_type：必选。值固定为“code”。
- client_id：必选。第三方应用的标识ID。
- state：**推荐**。Client提供的一个字符串，服务器会原样返回给Client。
- redirect_uri：必选。授权成功后的重定向地址。
- scope：可选。表示授权范围。

示例：

![](../img/OAuth2/1561022536(1).jpg)

（2）**Authorization server**验证**Client**在（A）中传递的参数信息，如果无误则提供一个页面供**Resource owner**登陆，登陆成功后选择**Client**可以访问**Resource server**的哪些资源以及读写权限。无误后返回一个**授权码（Authorization Code）**给Client。

Authorization Server的响应信息：

- code：授权码。
- state：步骤（A）中客户端提供的state参数原样返回。

示例：

![](../img/OAuth2/1561022693(1).jpg)

（3）**Client**拿着（C）中获得的**授权码（Authorization Code）**和（客户端标识、重定向URL等信息）作为参数，请求**Authorization server**提供的获取访问令牌的URL。

Access Token Request：

- grant_type：必选。固定值“authorization_code”。
- code : 必选。Authorization Response 中响应的code。
- redirect_uri：必选。必须和Authorization Request中提供的redirect_uri相同。
- client_id：必选。必须和Authorization Request中提供的client_id相同。

示例：

![](../img/OAuth2/1561023033(1).jpg)

（4）**Authorization server**返回**访问令牌**和可选的**刷新令牌**以及**令牌有效时间**等信息给**Client**。

Access Token Response：

- access_token：访问令牌。
- refresh_token：刷新令牌。
- expires_in：过期时间。

示例：

![](../img/OAuth2/1561023109(1).jpg)

#### 3.2Implicit

​    这个是Authorization Code的简化版本。其中省略掉了颁发授权码（Authorization Code）给客户端的过程，而是直接返回访问令牌和可选的刷新令牌。其适用于没有Server服务器来接受处理Authorization Code的第三方应用，其流程如下：

![](../img/OAuth2/1561023326(1).jpg)

（1）Authorization Request

- response_type：必选。值固定为“token”。
- client_id：必选。第三方应用的标识ID。
- state：**推荐**。Client提供的一个字符串，服务器会原样返回给Client。
- redirect_uri：可选。授权成功后的重定向地址。
- scope：可选。表示授权范围。

重点区别在于**response_type为“token”**，而不再是“code”，redirect_uri也变为了可选参数。

示例：

![](../img/OAuth2/1561023517(1).jpg)

（2）Access Token Response

- access_token：访问令牌。
- refresh_token：刷新令牌。
- expires_in：过期时间。

示例：

![](../img/OAuth2/1561023581(1).jpg)

#### 3.3Resource Owner Password Credentials

这种模式更简化，和Authorization Code的区别是省略了Authorization Request和Authorization Response。而是Client直接使用Resource Owner的账号密码直接请求并获得access_token。这种模式一般适用于Resource Owner高度信任第三方Client的情况下。

流程如下：

![](../img/OAuth2/1561339625(1).jpg)

（1）Client请求Authorization Server的参数：

- grant_type：必选。值固定为“password”。
- username：必选。用户登陆名。
- passward：必选**。**用户登陆密码。
- scope：可选。表示授权范围。

示例：

![](../img/OAuth2/1561340052(1).jpg)

#### 3.4Client Credentials Grant

这种类型就更简化了，Client直接已自己的名义而不是Resource owner的名义去要求访问Resource server的一些受保护资源。流程如下：

![](../img/OAuth2/1561340246(1).jpg)

（1）Client请求Authorization Server的参数：

- grant_type：必选。值固定为“client_credentials”。
- scope：可选。表示授权范围。

示例：

![](../img/OAuth2/1561340324(1).jpg)

响应信息和3.1 Access Token Response保持一致。

### 4.OAuth2刷新令牌

得到访问令牌（access_token）时，一般会提供一个过期时间和刷新令牌。以便在访问令牌过期失效的时候可以由客户端自动获取新的访问令牌，而不是让用户再次登陆授权。那么问题来了，是否可以把过期时间设置的无限大呢，答案是可以的，笔者记得Pocket的OAuth2拿到的访问令牌就是无限期的，好像豆瓣的也是。如下是刷新令牌的收客户端需要提供给Authorization Server的参数：

- grant_type：必选。固定值“refresh_token”。
- refresh_token：必选。客户端得到access_token的同时拿到的刷新令牌。

示例：

![](../img/OAuth2/1561340845(1).jpg)

响应信息和3.1 Access Token Response保持一致。

### 5.Token的传递方式

在第三方Client拿到access_token后，如何发送给Resouce Server这个问题并没有在RFC6729种定义，而是作为一个单独的RFC6750来独立定义了。这里做以下简单的介绍，主要有三种方式。

#### 5.1URI Query Parameter

在我们请求受保护的资源的Url后面追加一个access_token的参数即可。另外还有一点要求，就是Client需要设置以下Request Header的**Cache-Control:no-store**，用来阻止access_token不会被Web中间件给log下来，属于安全防护方面的一个考虑。

示例：

> *GET /resource?access_token=mF_9.B5f-4.1JqM  HTTP/1.1*
>
> *Host: server.example.com*

#### 5.2Authorization Request Header Field

因为在HTTP应用层协议中，专门有定义一个授权使用的Request Header，所以也可以使用这种方式：

> *GET /resource HTTP/1.1*
> *Host: server.example.com*
> *Authorization: Bearer mF_9.B5f-4.1JqM*

其中"Bearer "是固定的在access_token前面的头部信息。

#### 5.3Form-Encoded Body Parameter

使用Request Body这种方式，有一个额外的要求，就是Request  Header的"Content-Type"必须是固定的“application/x-www-form-urlencoded”，此外还有一个限制就是不可以使用GET访问，这个好理解，毕竟GET请求是不能携带Request  Body的。

> *POST /resource HTTP/1.1*
> *Host: server.example.com*
> *Content-Type: application/x-www-form-urlencoded*
>
> *access_token=mF_9.B5f-4.1JqM*

### 6.安全问题

在OAuth2早期的时候爆发过不少相关的安全方面的漏洞，其实仔细分析后会发现大都都是没有严格遵循OAuth2的安全相关的指导造成的，相关的漏洞事件百度以下就有了。

其实OAuth2在设计之初是已经做了很多安全方面的考虑，并且在RFC6749中加入了一些安全方面的规范指导。比如

- 要求Authorization server进行有效的Client验证；

- client_serect,access_token,refresh_token,code等敏感信息的安全存储（不得泄露给第三方）、传输通道的安全性（TSL的要求）；

- 维持refresh_token和第三方应用的绑定，刷新失效机制；

- 维持Authorization Code和第三方应用的绑定，这也是state参数为什么是推荐的一点，以防止CSRF；
- 保证上述各种令牌信息的不可猜测行，以防止被猜测得到；

安全无小事，这方面是要靠各方面（开放平台，第三方开发者）共同防范的。如QQ互联的OAuth2 API中，state参数是强制必选的参数，授权接口是基于HTTPS的加密通道等；同时作为第三方开发者在使用消费这些服务的时候也应该遵循其相关的安全规范。

### 7.总结&案例

OAuth2是一种**授权**标准框架，用来解决的是第三方服务在无需用户提供账号密度的情况下访问用户的私有资源的一套流程规范。与其配套的还有其他相关的规范，都可以到<https://oauth.net/2/>去延伸阅读和了解。

相关参考：

<https://oauth.net/2/>

<https://www.oauth.com/>

<https://aaronparecki.com/oauth-2-simplified/>

[RFC6749 : The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)

[RFC6749中文版（https://github.com/jeansfish/RFC6749.zh-cn）](https://github.com/jeansfish/RFC6749.zh-cn)

[RFC6750 - The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://tools.ietf.org/html/rfc6750).

[RFC6819 - OAuth 2.0 Threat Model and Security Considerations](https://tools.ietf.org/html/rfc6819).

OAuth2案例：

[豆瓣OAuth2 API（Authorization Code）](https://developers.douban.com/wiki/?title=oauth2)

[QQ OAuth2 API（Authorization Code）](http://wiki.connect.qq.com/使用authorization_code获取access_token)

[豆瓣OAuth2 API（Implicit )](https://developers.douban.com/wiki/?title=browser)

[QQ OAuth2 API（Implicit）](http://wiki.connect.qq.com/使用implicit_grant方式获取access_token)

[微信公众号获取access_token（Client Credentials Grant）](https://mp.weixin.qq.com/wiki?id=mp1421140183&t=0.2731444596120334)。

至于Resource Owner Password Credentials  Grant这种类型的许可方式，由于其适用常见，平时作为第三方开发者的开发工作中，没有遇到此类的案例。其适用场景在于第三方应用和Resoure  server属于同一方这样高度可信的环境下。