---
layout: post
title: VIWI学习记录
description: "大众提出的infortainment后台交互协议"
category: develop
tags: [viwi, api]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---



# VIWI协议学习

### RESTful概念

REST架构风格包含六点原则：
* 统一的接口样式(基于资源的，自描述的)
* 无状态的(请求操作的状态包含在请求内,如PUT,GET,POST,DELETE等)
* 可缓存的(客户端可以缓存返回结果　304 not modified)
* C/S架构(接口分割了客户端和服务端)
* 分层系统(客户端不清除它是直接还是间接连接服务的)
* 代码按需调节(服务端有能力通过向客户端传递逻辑的方式，临时扩展或者变更客户端的功能)

REST风格的好处：让分布式系统的高性能，大规模，简洁，可扩展，可移植，可视化，稳定等特点获得提升

自描述的含义是指：消息体内包含了如何来处理消息的逻辑

HEAD：　获取对象状态，如是否存在、有效期等

DELETE: 删除整个对象或者通过fileds查询参数删除对象的部分属性

POST：　创建或修改一个对象

GET: 获取对象属性

AND查询：　GET /api/v1/service/resource/?name=foo&type=bar

OR查询：　GET　/api/v1/service/resource/?name=foo,bar

CORS: Cross-origin resource sharing
跨域的ajax请求(POST, PUT, DELETE等)由于会导致各类xss,csrf等跨站攻击，因此默认是被禁止的
但是这个策略可以被配置来让浏览器允许或禁止跨域请求

常见的允许任何域名的跨域请求的http header如下:

{% highlight http %}

Access­Control­Allow­Origin *
Access­Control­Expose­Headers 'location'
Access­Control­Allow­Credentials

{% end highlight %}

认证

HTTP协议中使用了Authorization头部来存储accessToken
WebSockets协议中　subscribe消息包含一个可选的Authorization字段，一旦这个字段的accessToken过期，对应的订阅服务者应该发送一个Error消息到客户端，客户端应该进行重新鉴权并提交新的subscribe消息

Token类型

JWT 　RefreshToken+accessToken

Token字段解读

jti token的唯一标示符
aud token的授权对象
iss token的授权颁发者

这两个属性在多服务共享的token认证中非常重要

# OAuth2.0学习

OAuth2.0基本流程图
![oauth2.0](http://7xwdx7.com1.z0.glb.clouddn.com/oauth2.0.png)

在Authorization Server完成对Resource Owner的认证后，可以返回给Client一个Authorization Code来继续换取accessToken，也可以直接返回给Client AccessToken（Implicit模式）。

OAuth2.0　AccessToken&RefreshToken逻辑流程图
![oauth2.0_refreshtoken](http://7xwdx7.com1.z0.glb.clouddn.com/OAuth2.0-RefreshToken&AccessToken.png)


Client端身份认证

Client端一般通过client_id（ID编号）,client_secret（认证私钥）两个字段来进行身份认证。这些字段可以在Client端注册时被分配，在Client端注销时被收回。

通常来说建议Client端通过POST请求在body中提交这些字段

{% highlight shell %}

POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&client_id=s6BhdRkqt3&client_secret=7Fjfp0ZBr1KtDRbnfVdmIw

{% end highlight %}

协议接入点

认证过程需要两个认证服务接入点：　　

认证接入点-　客户端用来将resource owner重定向到该接入点，完成认证。

认证接入点必须采用TLS协议，支持GET请求，并可以同时支持POST请求。对于返回结果，Client端可以指明code或token来确定使用Authorization Code模式或Implicit模式。认证接入点完成认证后，可以将Resource Owner重定向到Client端注册时留下的接入点地址。缺乏注册时重定向URL的验证，可能会导致攻击者利用这个Open Rediect进行攻击。

客户端的动态重定向URL配置:Client端请求认证接入点时，可以加入一个redirect_uri字段来指明认证后的重定向地址。

Token接入点- Client端获取Authorization code后，用来换取AccessToken&RefreshToken的接入点。这个接入点一般既需要验证Authorization code,又需要验证Client端的id和secret。

Client端必须使用HTTP POST方法来获取AccessToken。

Client端认证主要有以下几个目的，将Authorization code或RefreshToken与Client端进行绑定；可以通过禁用或重置Client端id和key来恢复受攻击的Client端；遵从认证管理的最佳实践，需要周期性地进行密钥更新。

AccessToken的Scope: 当请求AccessToken时,Client端可以加入scope参数，这样Authorization　Server在返回AccessToken时，可以将授权的scope进行返回。

OAuth2.0的四种认证方式：
1、Authorization code
2、Implicit
3、resource owner password crendentials
4、client crendentials

Authorization code模式的流程图
![oauth2.0-Authorization-code](http://7xwdx7.com1.z0.glb.clouddn.com/OAuth2.0-Authorization-code.png)

认证请求的字段: response_type(必须，在Authorization code模式下肯定为code), client_id(必须), redirect_uri(可选), scope(可选), state(推荐，便于Client端防御CSRF攻击)

认证请求示例:

{% highlight shell %}

GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz
&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
Host: server.example.com

{% end highlight %}

认证返回的字段: code(必须，返回Authorization code，一般该授信码最长有效时间应该为10分钟), state(与请求中的state值保持一致，便于Client端防御CSRF攻击)

认证返回示例:

{%  highlight shell %}

HTTP/1.1 302 Found
Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA
&state=xyz

{% end highlight %}

失败返回示例:

{% highlight shell %}

HTTP/1.1 302 Found
Location: https://client.example.com/cb?error=access_denied&state=xyz

{% end highlight %}

AccessToken请求的字段：grant_type(必须，在Authorization code模式下肯定为authorization_code),code(必须，包含从Authorization Server获取的Authorization code)，redirect_uri(必须，与之前的redirect_uri保持一致),client_id(必须，客户端唯一编号)

AccessToken请求示例：

{% highlight shell %}

POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded
grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb

{% end highlight %}

Authorization Server的认证安全要求：
1、对Client端的私钥进行验证
2、确保这个Authorization code确实是授予这个Client对应的client_id的
3、验证Authorization　code是真实有效的
4、确认redirect_uri是与初始化注册时一致的

AccessToken响应示例:

{% highlight shell %}

HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache
{
"access_token":"2YotnFZFEjr1zCsicMWpAA",
"token_type":"example",
"expires_in":3600,
"refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
"example_parameter":"example_value"
}

{% end highlight %}

其他三种模式的认证过程这里不再详细描述。

AccessToken的颁发

一般来说AccessToken请求的返回包含如下字段: access_token(必须，token具体内容)，token_type(必须，token类型)，expires_in(推荐，token失效时间),refresh_token(可选，用来更新access_token的refresh_token),scope(可选，如果请求中传递了scope字段，response中必须返回)

Authorization server必须在返回的HTTP头部中包含一个"Cache-Control"头部(也可以是"Pragma"头部)，写明"no-cache"策略。任何返回token, crendentials或其他敏感信息的返回消息体都应该包含这个头部。

AccessToken的更新

更新请求的消息体一般包含以下字段: grant_type(必须，更新情况下肯定为refresh_token)，refresh_token(必须，包含具体refresh_token的值), scope(可选，授权的范围)

AccessToken更新请求示例:

{% highlight shell %}

POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded
grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA

{% end highlight %}

Authorization Server的认证安全要求：
1、对Client端的私钥进行验证
2、对refresh_token的有效性进行确认

带有AccessToken的请求示例:

bearer token

{% highlight shell %}

GET /resource/1 HTTP/1.1
Host: example.com
Authorization: Bearer mF_9.B5f-4.1JqM

{% end highlight %}

http-MAC

{% highlight shell %}

GET /resource/1 HTTP/1.1
Host: example.com
Authorization: MAC id="h480djs93hd8",
nonce="274312:dj83hs9s",
mac="kDZvddkndxvhGRXZhvuDjEWhGeE="

{% end highlight %}

* 之前OAuth2.0漏洞的一些启示

RFC文档中非常强调Client端身份的识别和client_id的鉴权，但是忽略了Resource Owner也可能存在平行越权的风险。例如利用weibo的OAuth2.0接口进行登录，利用AccessToken在客户端获取Resource Owner身份信息。这里攻击者可以通过篡改返回的JSON数据，将Resource Owner篡改为其他微博用户。而Client端验证AccessToken时，只能验证到这是一个合法的token，并不能获取这个AccessToken与Resource Owner的绑定关系。因此可以导致攻击者冒充其他Resource Owner的情况出现。

在做开放平台OAuth2.0认证时，应该注意不仅仅要将AccessToken与client_id绑定，也要维护AccessToken与Resource Owner id的绑定关系，并通过AccessToken提供这样的查询接口。这样可以避免这样的平行越权问题。

# JWT学习

一个JWT示例结构如下：

eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJIUzI1NiJ9
.
eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFt
cGxlLmNvbS9pc19yb290Ijp0cnVlfQ
.
dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk

它被.分割为三部分，第一部分为JOSE Header，描述了HMAC算法；第二部分为JWT Claims Set，也就是JWT的内容主体；第三部分为JWS Signature,也就是用Header中的算法对Claims Set内容进行HMAC运算的结果。这些内容通过UTF-8编码+base64转换形成示例中的数据。

* 标准JWT Claim字段

"iss"字段：可选字段，声明Issuer身份，是一个大小写敏感的string值
"sub"字段: 可选字段，声明授权token的scope，是一个大小写敏感的string值
"aud"字段: 可选字段，声明这个token的接收者身份,可以是一个string或string数组
"exp"字段: 可选字段，声明token的过期时间，是一个数值格式
"nbf"字段: 可选字段，声明token的开始生效时间(Not Before)，可以支持在更新时预分发token，是一个数值格式
"iat"字段: 可选字段，声明token的生成时间，是一个数值格式
"jti"字段: 可选字段，JWT ID，一个JWT token的唯一ID编号

JOSE Header字段

"typ"： 可选字段，说明JWT的类型
"cty":　可选字段，说明Content Type
"alg": 必须字段，说明token的签名算法

JWS: JSON Web Signature
JWE：JSON　Web Encrypt
