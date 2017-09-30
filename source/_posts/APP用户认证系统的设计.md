---
title: APP用户认证系统的设计
tags:
- Java
- 已完结
categories:
- 技术
- 软件设计
author: 汪博全
date: 2017-03-09 11:38:00
---

> 已完结

网站应用一般使用Session进行登录用户信息的存储及验证，而在移动端使用Token则更加普遍。它们之间并没有太大区别，Token比较像是一个更加精简的自定义的Session。Session的主要功能是保持会话信息，而Token则只用于登录用户的身份鉴权。所以在移动端使用Token会比使用Session更加简易并且有更高的安全性。

<!-- more -->

# 自己设计基于Token的用户认证系统

## http基本认证
HTTP基本认证(Basic Authentication)，即"用户名+冒号+密码"用BASE64算法加密后的字符串放在http request 中的header Authorization中发送给服务端。假如用户名密码错误的话，服务器会返回401。这里我们利用header Authorization来传递token信息。

## 程序示例


```java
// token实体类
public class Token {

    private int userId;
    private String token;

    public Token(int userId, String token) {
        this.userId = userId;
        this.token = token;
    }

    public int getUserId() {
        return userId;
    }

    public void setUserId(int userId) {
        this.userId = userId;
    }

    public String getToken() {
        return token;
    }

    public void setToken(String token) {
        this.token = token;
    }
}

```

本文使用Redis来存储user_id和token，Redis是一个Key-Value结构的内存数据库，用它维护的映射表会比传统数据库速度更快，这里使用spring-Data-Redis对Token进行基础操作。
服务端生成的Token一般为随机的非重复字符串，根据应用对安全性的不同要求，会将其添加时间戳（通过时间判断Token是否被盗用）或url签名（通过请求地址判断Token是否被盗用）后加密进行传输。在本文中为了演示方便，仅以user_id为用户名，token为密码使用Basic Authentication。

```java
public interface TokenRepository  {
    String createToken(int userId);
    boolean checkToken(int userId, String userToken);
    void deleteToken(int userId);
}
```

```java
@Repository
public class RedisTokenRepository implements TokenRepository {

    @Autowired
    private RedisTemplate<Object, Object> redisTemplate;

    @Override
    public String createToken(int userId) {
        //使用uuid作为源token
        String token = UUID.randomUUID().toString().replace("-", "");

        //存储到redis并设置过期时间
        redis.boundValueOps(userId).set(token, Constants.TOKEN_EXPIRES_HOUR, TimeUnit.HOURS);
        redisTemplate.boundValueOps(userId).set(token);
        return token;
    }

    @Override
    public boolean checkToken(int userId, String userToken) {
        String token = (String) redisTemplate.boundValueOps(userId).get();
        if (token == null || !token.equals(userToken)) {
            return false;
        } else {
            //如果验证成功，说明此用户进行了一次有效操作，延长token的过期时间
            redis.boundValueOps(model.getUserId()).expire(Constants.TOKEN_EXPIRES_HOUR, TimeUnit.HOURS);
            return true;
        }
    }

    @Override
    public void deleteToken(int userId) {
        redisTemplate.delete(userId);
    }
}
```

这里使用Spring的拦截器完成这个功能，该拦截器会检查每一个请求映射的方法是否有```@Authorization```自定义注解，并使用TokenRepository验证Token，如果验证失败直接返回错误信息：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Authorization {
}
```

```java
@Component
public class AuthorizationInterceptor extends HandlerInterceptorAdapter {

    @Autowired
    private TokenRepository tokenRepository;

    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        //如果不是映射到方法直接通过
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Method method = handlerMethod.getMethod();
        //如果有注解
        if (method.getAnnotation(Authorization.class) != null) {
            String authorization = request.getHeader("authorization");

            //标准basic auth，使用BASE64加密
            String userIdAndToken=new String(new BASE64Decoder().decodeBuffer(authorization.split(" ")[1]));
            if(userIdAndToken.split(":").length<2){
                response.setStatus(HttpServletResponse.SC_OK);
                response.getWriter().println(new ErrorMessage("UnauthorizedException").toString());
                return false;
            }
            int userId=Integer.parseInt(userIdAndToken.split(":")[0]);
            String token=userIdAndToken.split(":")[1];

            if (tokenRepository.checkToken(userId, token)){
                return true;
            } else {
                response.setStatus(HttpServletResponse.SC_OK);
                response.getWriter().println(new ErrorMessage("UnauthorizedException").toString());
                return false;
            }
        } else {
            return true;
        }
    }
}

```

# OAuth2.0
OAuth 2.0 是目前比较流行的做法，它率先被Google, Yahoo, Microsoft, Facebook等使用。之所以标注为 2.0，是因为最初有一个1.0协议，但这个1.0协议被弄得太复杂，易用性差，所以没有得到普及。2.0是一个新的设计，协议简单清晰，但它并不兼容1.0，可以说与1.0没什么关系。所以，我就只介绍2.0。

## 协议的参与者
OAuth的参与实体至少有如下三个：

* RO (resource owner): 资源所有者，对资源具有授权能力的人。
* RS (resource server): 资源服务器，它存储资源，并处理对资源的访问请求。
* Client: 第三方应用，它获得RO的授权后便可以去访问RO的资源。

此外，为了支持开放授权功能以及更好地描述开放授权协议，OAuth引入了第四个参与实体：

* AS (authorization server): 授权服务器，它认证RO的身份，为RO提供授权审批流程，并最终颁发授权令牌(Access Token)。读者请注意，为了便于协议的描述，这里只是在逻辑上把AS与RS区分开来；在物理上，AS与RS的功能可以由同一个服务器来提供服务。

## 授权类型
在开放授权中，第三方应用(Client)可能是一个Web站点，也可能是在浏览器中运行的一段JavaScript代码，还可能是安装在本地的一个应用程序。这些第三方应用都有各自的安全特性。对于Web站点来说，它与RO浏览器是分离的，它可以自己保存协议中的敏感数据，这些密钥可以不暴露给RO；对于JavaScript代码和本地安全的应用程序来说，它本来就运行在RO的浏览器中，RO是可以访问到Client在协议中的敏感数据。

OAuth为了支持这些不同类型的第三方应用，提出了多种授权类型，如授权码 (Authorization Code Grant)、隐式授权 (Implicit Grant)、RO凭证授权 (Resource Owner Password Credentials Grant)、Client凭证授权 (Client Credentials Grant)。由于本文旨在帮助用户理解OAuth协议，所以我将先介绍这些授权类型的基本思路，然后选择其中最核心、最难理解、也是最广泛使用的一种授权类型——“授权码”，进行深入的介绍。

## OAuth协议 - 基本思路
协议的基本流程如下：
(1) Client请求RO的授权，请求中一般包含：要访问的资源路径，操作类型，Client的身份等信息。
(2) RO批准授权，并将“授权证据”发送给Client。至于RO如何批准，这个是协议之外的事情。典型的做法是，AS提供授权审批界面，让RO显式批准。这个可以参考下一节实例化分析中的描述。
(3) Client向AS请求“访问令牌(Access Token)”。此时，Client需向AS提供RO的“授权证据”，以及Client自己身份的凭证。
(4) AS验证通过后，向Client返回“访问令牌”。访问令牌也有多种类型，若为bearer类型，那么谁持有访问令牌，谁就能访问资源。
(5) Client携带“访问令牌”访问RS上的资源。在令牌的有效期内，Client可以多次携带令牌去访问资源。
(6) RS验证令牌的有效性，比如是否伪造、是否越权、是否过期，验证通过后，才能提供服务。

# 参考资料
[RESTful登录设计（基于Spring及Redis的Token鉴权）](http://blog.csdn.net/gebitan505/article/details/51614805)
[记一个社交APP的开发过程——用户身份认证与在线标记](http://www.tuicool.com/articles/vmmIvm)
[帮你深入理解OAuth2.0协议](http://blog.csdn.net/seccloud/article/details/8192707)
