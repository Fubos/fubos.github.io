---
title: "[Tech]当前主流的用户登录认证方案对比"
date: 2025-02-09T20:22:03+08:00
lastmod: 2025-02-09T20:22:03+08:00
author: ["Fubos"]

categories:
- tech

tags:
- 用户认证
- token

keywords:
- 用户认证
- token

description: "在涉及用户登录的项目中，经常需要结合token来获取当前用户的信息，从而实现各种个性化的业务需求。目前，用户token在服务端的实现主流方式有 1）使用SessionID 2）使用JWT 3）中心化存储token。" # 文章描述，与搜索优化相关
summary: "在涉及用户登录的项目中，经常需要结合token来获取当前用户的信息，从而实现各种个性化的业务需求。目前，用户token在服务端的实现主流方式有 1）使用SessionID 2）使用JWT 3）中心化存储token。" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true
showToc: true # 显示目录
TocOpen: true # 自动展开目录
autonumbering: true # 目录自动编号
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
searchHidden: false # 该页面可以被搜索到
showbreadcrumbs: true #顶部显示当前路径
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

在各种经常涉及用户登录以及认证的项目中，会使用token来区分不同的用户，而token的生成方式有多种，不同的生成方式存在不同的特性，需要根据不同的业务场景来使用不同的生成方法。


目前主要的token生成方法有：
1）使用随机数生成算法得到随机字符串作为token，比如UUID，Snowflake等方法来生成一个随机的字符串作为token。由于随机字符串是随机分布，因此具有较高的安全性。

2）使用JWT（JSON Web Token）实现，JWT是一种基于JSON格式的开放标准（RFC 7519），用于在多方之间进行安全传输信息。将用户身份信息和权限等相关信息编码成一个JSON对象，并通过数字签名或加密等方式进行验证和保护。JWT除了用于Token登录外，还可用于API认证，单点登录等场景。

3）使用SessionID作为token，浏览器第一次跟服务器请求的时候，服务器会创建一个session（会话），并生成一个唯一的key（SessionID），把用户的一些信息，状态存在session里，并把SessionID和session当作key，value存起来（可以存在缓存里），然后服务器把SessionID以cookie的形式发送给浏览器，浏览器下次访问服务器时直接携带上cookie中的SessionID，服务器再根据SessionID找到对应的session进行匹配，对应的session就有对应用户的信息。

通常token在服务端的实现方式有：

## 使用SessionID实现token功能

由于HTTP协议是无状态的，也就是当我们向服务端发送HTTP请求后，服务器根据请求返回数据，但服务端不知道客户端的状态，不会记录客户端的相关状态。

为了解决HTTP无状态的问题，出现了Cookie。它是服务端发送给客户端的一段特殊信息，这些信息以文本的形式存储在客户端，客户端在下一次向服务端发送请求时会带上这些特殊信息，服务端从而知道当前客户端的相关信息。

![Cookie和Session](/posts_imgs/cookie_and_session.png)

具体的流程是：

1）前端输入账号密码，提交给后端；

2）后端验证成功后，创建一个Session，用于保存用户会话信息的机制，用于识别多次请求之间的逻辑。

3）后端将Session ID返回给前端， 通过cookie的方式将Session ID保存到浏览器，从而保证用户再次请求时，后端可以直接使用Session ID来识别用户身份。

4）在后续请求中，浏览器将保存的Cookie信息发送给后端验证，如果Session ID有效，则返回对应数据，否则重新登录获取新的Session ID。

5）用户退出时，后端删除对应的Session信息。

缺点：

1）存在跨域问题：cookie只能在同域名下共享，跨域访问无法访问到对应的cookie，此时需要采用其他方法，比如JSONP，CORS。

2）扩展性问题：由于Session存在服务器端，当系统扩展到多台服务器时，需要采取集中存储Session的方案，否则会出现Session不一致的情况。

因此可以看出，服务器给的SessionID相当于一个token，只不过前端是无感知的设置进cookie的。由于cookie的限制，该token最好由前端主动保存到localStorage，登陆时主动从请求头携带。另外由于现在都是集群部署，token的保存最好是集中化管理，或者无状态管理。


## 使用JWT

![JWT实现原理](/posts_imgs/jwt.png)

简单来说，JWT是通过可逆加密算法，生成一串包含用户、过期时间等关键信息的Token，前端每次请求服务器时用该token进行解密就能得到用户信息，从而判断用户状态。

优点：服务器端不保存任何会话状态，无论是从哪个服务器解析出来的token信息都是一样的，同时不需要任何查询操作，从而省掉了数据库/Redis的查询开销，实现了去中心化存储、管理、查询、对比。

缺点：由于JWT在使用期间无法取消token或者更改token的权限，一旦JWT签发，则在有效期内将会一直有效，从而无法实现主动更新token的有效性。JWT token的解析也需要消耗服务器cpu。

### 双token

为了解决JWT续期问题而提出的。由于JWT颁布，意味着在指定时间内都能够通行，这就可能会存在问题：

1）如果有效期过长，服务器失去掌控力，在这期间无法让用户失效。

2）如果有效期过短，频繁重新登录影响用户体验。

3）中心化管理用户状态，每次解析JWT Token后，还需要去中心化进行对比，增加了认证的耗时，违背了初衷。

双token分为`access_token`和`refresh_token`。一般`access_token`的有效期可以设置为10分钟，`refresh_token`的有效期可以设置为7天。用户每次请求都用`access_token`，如果前端发现请求401，也就是过期了，就用`refresh_token`去重新申请一个`access_token`。继续请求。

这里的关键在于，`refresh_token`申请`access_token`的时候，用户是无感知的，前后端的框架自动去更新这个新的`access_token`。

还有一个点在申请`access_token`的时候，后端这时候会去校验用户的状态等问题，如果发现用户被禁用了，就申请不到token了。



## 中心化存储管理token

由于JWT由去中心化的特性，为了能够控制上下线、主动下线、登录续期等功能，依然可进行中心化管理。

依赖Redis中心化管理uid->token信息，确保一个uid只有一个有效token。用户登录后，每次认证都解析出uid，并请求Redis进行token比对，并且异步判断有效期小于一天时，进行续期。

为啥不用uuid做token呢，既然都是redis中心存储，用uuid还可以少一次解析？

如果用uuid，前端每次请求除了带上uuid，还需要带上uid。

单纯用uuid的话，黑客可能会不断遍历uuid来撞库，如果此时碰巧装上有关联的在线用户，从而造成信息泄露。

而如果将uuid和uid一起比对，那篇uuid碰巧撞到了登录的用户，还需要确保是相同的uid，此时概率就下降了很多。

用jwt的话，刚好包含了uid，前端传起来也方便，因此就这么选择了。


{{< collapse "代码实现demo" >}}
```JAVA
public interface LoginService {


    /**
     * 校验token是不是有效
     *
     * @param token
     * @return
     */
    boolean verify(String token);

    /**
     * 刷新token有效期
     *
     * @param token
     */
    void renewalTokenIfNecessary(String token);

    /**
     * 登录成功，获取token
     *
     * @param uid
     * @return 返回token
     */
    String login(Long uid);

    /**
     * 如果token有效，返回uid
     *
     * @param token
     * @return
     */
    Long getValidUid(String token);

}
```

{{< /collapse >}}




---END---


> 参考文献
> [1] https://blog.csdn.net/weixin_40208575/article/details/101868259