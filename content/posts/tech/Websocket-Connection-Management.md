---
title: "[Tech]IM项目中的Websocket连接管理"
date: 2025-01-24T23:28:38+08:00
lastmod: 2025-01-24T23:28:38+08:00
author: ["Fubos"]

categories:
- tech

tags:
- IM
- websocket连接

keywords:
- IM
- websocket连接

description: "不仅对于IM通讯系统，在很多实际业务中都有用到服务端主动推送web的场景，此时选择合适的连接类型对于业务的高效运行有着重要影响。" # 文章描述，与搜索优化相关
summary: "在我们实际的IM项目中，我们使用了websocket进行连接管理，并根据连接的特性指定了前后端消息的类型以及根据请求内容获取用户真实ip，并使用心跳包进行用户下线感知，对用户请求路由处理等。" # 文章简单描述，会展示在主页
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


不仅对于IM通讯系统，在很多实际业务中都有用到服务端主动推送web的场景，比如小红点推送提醒，新消息提醒，审批流程提醒等。

目前常见的推送方案有

## 1）短轮询

web端不停地间隔一段时间想服务端发一个HTTP请求，如果有新消息，就会在某次请求中返回。

![短轮询实现原理](/posts_imgs/short_polling.png)

比如OA系统，用户需要收到小红点，审批流程提醒等信息，为了方便，直接采用每秒1次的请求，等待后端返回数据。

适用场景：

1.扫码登录：短时间内频繁查询二维码状态；

2.小OA系统：客户端使用量不大的情况下可以使用。

缺点：

1.存在大量无效请求，浪费服务器资源

2.服务端请求压力过大，万人群聊频繁访问，上万并发服务扛不住。

## 2）长轮询

长轮询相比于短轮询，最大的改进在于：

1.短轮询模式下，服务端不管本轮有没有新的消息产生，都会马上响应并返回，而长轮询模式当本次请求没有获取新消息时，并不会马上结束返回，而是在服务端“悬挂（hang）”，等待一段时间。

2.如果在等待的这段时间内服务端有新消息产生也就是新数据准备好，就能马上响应返回，当超过等待时间后返回等待超时。

![长轮询实现原理](/posts_imgs/long_polling.png)

这意味着web端的请求超时时长可以设置得长一些。

优点：

1.大幅降低短轮询中客户端高频无用的轮询导致的网络开销和功耗开销。

2.降低了服务端处理请求的QPS。

缺点：

1.存在无效请求：长轮询在超时时间内没有获取到消息后，会结束返回，因此仍然没有完全解决客户端“无效”请求的问题。

2.服务端压力大：服务端悬挂请求，只是降低了入口请求的QPS，并没有减少对后端资源轮询的压力，假如有1000个请求在等待消息，可能意味着有1000个线程在不断轮询消息资源，只是轮询转移到了后端。

## 3）WebSocket长连接

实现原理：客户端和服务器之间维持一个TCP/IP连接，全双工通道。

![websocket实现原理](/posts_imgs/websocket_connection.png)

websocket能够弥补以上两种方法的缺点，唯一的缺点是实现起来较为复杂，需要管理链接。

### WebSocket 协议与 HTTP 的关系

- WebSocket 的连接通过 HTTP/1.1 或 HTTP/2 发起，但成功建立连接后，通信不再依赖 HTTP 协议。
- HTTP 的作用仅限于：
  - 协议升级握手。
  - 通过初始的 HTTP 请求携带参数（例如认证信息、路径等）。


### WebSocket的连接过程

连接开始时，客户端使用HTTP协议和服务端升级协议，当升级完成之后，后续数据交换遵循WebSocket协议。
注意：因为原始的http连接升级后使用ws进行连接，采用加密传输后，当我们使用https协议进行连接时，在nginx配置时，需要将ws修改成wss。

{{< collapse "请求头内容" >}}

```java
GET /chat HTTP/1.1
Host: example.com:8000
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```
{{< /collapse >}}

查看Request Header，其中的关键字段是Upgrade，Connection，这相当于告诉Apache、Nginx等服务器：注意，现在我发起的是WebSocket协议，不再使用原先的HTTP。

其中Sec-WebSocket-Key可以看成请求id。


{{< collapse "响应头内容">}}
```java
HTTP/1.1 101 Switching Protocols // 表示服务器同意客户端的协议升级请求，HTTP 状态码 101 意味着协议切换。
Upgrade: websocket // 表示服务器支持将协议升级为 WebSocket 协议。
Connection: Upgrade // 确认当前连接将被用于协议升级。
Sec-WebSocket-Accept: HSmrc0s... // 服务器根据客户端的 Sec-WebSocket-Key 计算生成的验证值，用于确认握手的合法性。用于告诉客户端愿意发起一个WebSocket连接。
Sec-WebSocket-Protocol: chat // 表示服务器同意使用客户端请求的子协议 chat。该字段可选，是 WebSocket 子协议协商的一部分。
```
{{< /collapse >}}

#### 为什么在响应头中需要添加 `Sec-WebSocket-Accept`？

- **验证握手的完整性**：
  - 通过返回 `Sec-WebSocket-Accept`，服务器可以证明它正确接收了客户端的握手请求，并进行了响应。
  - 客户端在收到该值后可以验证其有效性，确保连接确实升级到 WebSocket 协议。
- **防止伪造握手请求**：
  - 由于 `Sec-WebSocket-Accept` 的计算依赖于客户端的随机值和协议定义的固定字符串，因此其他恶意客户端或中间人无法轻易伪造握手响应。


### 代码实现
支持websocket的容器很多，常见的有1）通过tomcat实现websocket，2）通过netty实现websocket。在我们的项目中，采用netty来实现。
{{< collapse "使用Netty代码实现websocket">}}

```java
public class NettyWebSocketServer {
    public static final int WEB_SOCKET_PORT = 8090;
    public static final NettyWebSocketServerHandler NETTY_WEB_SOCKET_SERVER_HANDLER = new NettyWebSocketServerHandler();
    // 创建线程池执行器
    private EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    private EventLoopGroup workerGroup = new NioEventLoopGroup(NettyRuntime.availableProcessors());

    /**
     * 启动 ws server
     *
     * @return
     * @throws InterruptedException
     */
    @PostConstruct
    public void start() throws InterruptedException {
        run();
    }

    /**
     * 销毁
     */
    @PreDestroy
    public void destroy() {
        Future<?> future = bossGroup.shutdownGracefully();
        Future<?> future1 = workerGroup.shutdownGracefully();
        future.syncUninterruptibly();
        future1.syncUninterruptibly();
        log.info("关闭 ws server 成功");
    }

    public void run() throws InterruptedException {
        // 服务器启动引导对象
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 128)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .handler(new LoggingHandler(LogLevel.INFO)) // 为 bossGroup 添加 日志处理器
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        ChannelPipeline pipeline = socketChannel.pipeline();
                        //30秒客户端没有向服务器发送心跳则关闭连接
//                        pipeline.addLast(new IdleStateHandler(30, 0, 0));
                        // 因为使用http协议，所以需要使用http的编码器，解码器
                        pipeline.addLast(new HttpServerCodec());
                        // 以块方式写，添加 chunkedWriter 处理器
                        pipeline.addLast(new ChunkedWriteHandler());
                        /**
                         * 说明：
                         *  1. http数据在传输过程中是分段的，HttpObjectAggregator可以把多个段聚合起来；
                         *  2. 这就是为什么当浏览器发送大量数据时，就会发出多次 http请求的原因
                         */
                        pipeline.addLast(new HttpObjectAggregator(8192));
                        //保存请求头
                        pipeline.addLast(new MyHeaderCollectHandler());
                        /**
                         * 说明：
                         *  1. 对于 WebSocket，它的数据是以帧frame 的形式传递的；
                         *  2. 可以看到 WebSocketFrame 下面有6个子类
                         *  3. 浏览器发送请求时： ws://localhost:8090/hello 表示请求的uri
                         *  4. WebSocketServerProtocolHandler 核心功能是把 http协议升级为 ws 协议，保持长连接；
                         *      是通过一个状态码 101 来切换的
                         */
                        //这里是握手处理器，可以通过自定义握手处理器实现用protocol传参
                        pipeline.addLast(new WebSocketServerProtocolHandler("/"));

                        // 自定义handler ，处理业务逻辑
                        pipeline.addLast(NETTY_WEB_SOCKET_SERVER_HANDLER);
                    }
                });
        // 启动服务器，监听端口，阻塞直到启动成功
        serverBootstrap.bind(WEB_SOCKET_PORT).sync();
    }

}
```
{{< /collapse >}}

由于WebSocket初期是通过HTTP请求进行升级，建立双方的连接。

1）因此编解码器需要用到HttpServerCodec，

2）WebSocketServerProtocolHandler 核心功能是把 http协议升级为 ws 协议，保持长连接。在这处理器期间会抹除http相关的信息，比如请求头之类的，如果想获取相关http信息，需要在这处理器处理之前进行获取，这样以后需要的话能够提取出来。

3）HttpHeadersHandler是我们自己的处理器，我们需要赶在HTTP协议升级为WebSocket协议之前，获取用户的ip地址，然后保存到channel的附件里。

4）NettyWebSocketServerHandler是业务处理器，用于处理客户端的事件。

5）IdleStateHandler用于实现心跳检测。`IdleStateHandler` 通过定时任务检测指定时间内是否有 I/O 事件发生（读或写），如果没有，则会触发一个空闲状态事件。常用于实现心跳检测或关闭长期空闲的连接。其中输入的参数readerIdleTime表示在指定时间内没有**读操作**时触发 `READER_IDLE` 事件。writerIdleTime表示在指定时间内没有**写操作**时触发 `WRITER_IDLE` 事件。allIdleTime表示在指定时间内没有**读或写操作**时触发 `ALL_IDLE` 事件。这些参数的值为0或负数时，表示不检测读/写空闲。

#### 为什么选用netty而不是tomcat？

1.netty是使用了非阻塞（Non-blocking I/O，在java领域，也称New I/O）的I/O模型，基于事件驱动的多路复用框架，使用单线程或少量线程处理大量的并发请求。相比之下，tomcat是基于多线程的架构，每个连接都分配了一个线程，适用于处理相对较少的并发连接，最近的tomcat版本（Tomcat 8、9）引入了NIO（New I/O）模型，因此这个点在两种框架的最新版本中都用到了，netty相对于旧版本的tomcat在这方面有一定优势。

2.netty提供了丰富的功能和组件，可以灵活地构建自定义的网络应用。它具有强大的编解码器和处理器，可以轻松处理复杂的协议和数据格式。Netty的扩展性也非常好，可以根据需要添加自定义的组件。比如用Netty的pipeline方便地进行前置和后置的处理，可以用netty的心跳处理器来检查连接的状态，这些都是netty的优势。


## 获取用户真实ip

用户的ip主要从两个场景中进行获取，一个是新用户注册时，另一个是在登录连接认证时。两处都是需要在用户与服务器的socket建立连接的时候，赶在协议升级之前，保存用户的远端ip地址。
{{< collapse "用户ip获取" >}}

```java
public class MyHeaderCollectHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //如果第一次是http请求
        if(msg instanceof HttpRequest){
            HttpRequest request = (HttpRequest) msg;
            UrlBuilder urlBuilder = UrlBuilder.ofHttp(request.getUri());
            Optional<String> tokenOptional = Optional.of(urlBuilder)
                    .map(UrlBuilder::getQuery)
                    .map(k -> k.get("token"))
                    .map(CharSequence::toString);
            //当token存在时，存入ctx中
            tokenOptional.ifPresent(s -> NettyUtils.setAttr(ctx.channel(), NettyUtils.TOKEN, s));
            //将去除 URL 中的查询参数部分,只保留路径部分。确保websocket能正常响应
            request.setUri(urlBuilder.getPath().toString());
            //获取用户ip
            // 这里由于使用了nginx代理以及涉及到api接口和前端界面域名的跨域问题，因此需要获取到原始ip
            // String ip = request.headers().get("X-Rear-IP");
            String ip = request.headers().get("X-Forwarded-For");
            log.info("当前ip是:",ip);
            if (StringUtils.isBlank(ip)){
                //如果当前的ip为空，则使用远端的ip，也就是nginx的或者直连的
                InetSocketAddress address = (InetSocketAddress) ctx.channel().remoteAddress();
                ip = address.getAddress().getHostAddress();

            }
            //保存ip到channel附件中
            NettyUtils.setAttr(ctx.channel(),NettyUtils.IP,ip);
            //由于后续消息发送使用http进行发送，协议进行升级了，这些信息没有用了，将其移除。
            ctx.pipeline().remove(this);

        }

        //执行后续操作
        ctx.fireChannelRead(msg);

    }
}

```
{{< /collapse >}}
这里由于使用了nginx代理以及涉及到api接口和前端界面域名的跨域问题，因此需要获取到原始的远端用户真实ip，在配置nginx时也需要进行相应的配置。

{{< collapse "nginx配置-针对无法解析用户真实ip问题">}}

```java

http {
    ···

    server {
        listen       443 ssl;
        server_name  testapi.fun-x.top;

        # 频控生效位置
        limit_req zone=one burst=10 nodelay;

        ssl_certificate      "/etc/letsencrypt/live/testapi.fun-x.top/fullchain.pem";
        ssl_certificate_key  "/etc/letsencrypt/live/testapi.fun-x.top/privkey.pem";

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;


        #websocket
        location = /websocket {

                proxy_pass http://127.0.0.1:8090/;
                proxy_http_version 1.1;

                // 重点：获取远端ip
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;

                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;


                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_read_timeout 600s;
        }

        location / {
            proxy_pass http://127.0.0.1:8080;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            // 重点：允许跨域

            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Headers' '*';
            add_header 'Access-Control-Allow-Methods' '*';


        }

    }
}
```
{{< /collapse >}}

## 使用心跳包进行用户下线感知

如果用户突然关闭网页，是不会有断开通知给服务端的，那么服务端将永远无法感知到用户下线。因此客户端需要维持一个心跳，当指定事件没有心跳，服务端就自动断开，进行用户下线操作。

直接接入Netty的现有组件pipeline.addLast(new IdleStateHandler(30, 0, 0))可以实现30s内链接没有读请求，就主动关闭链接。因此在线时，web前端需要保持每30s发送一个心跳包。

## 用户请求路由处理

我们自己实现的处理器NettyWebSocketServerHandler接受websocket信息，根据消息类型进行路由处理。

目前的请求对websocket的依赖很低，只处理登录请求类型消息。
{{< collapse "请求类型">}}

```java
@Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
        WSBaseReq wsBaseReq = JSONUtil.toBean(msg.text(), WSBaseReq.class);
        WSReqTypeEnum wsReqTypeEnum = WSReqTypeEnum.of(wsBaseReq.getType());
        switch (wsReqTypeEnum) {
            case LOGIN:
                this.webSocketService.handleLoginReq(ctx.channel());
                log.info("请求二维码 = " + msg.text());
                break;
            case HEARTBEAT:
                break;
            default:
                log.info("未知类型");
        }
    }
```
{{< /collapse >}}
## 前后端消息协议统一

我们用websocket的目的，主要用于后端推送前端，前端能用http就尽量用http。这样做的好处是，利用http丰富的拦截器，注解，请求头等功能，可以更好地实现或者收口我们想要的功能，尽量对websocket的依赖降到最低。

前后端交互用的是json串，里面通过type标识此次的事件类型。

前端的请求主要包括

1.请求登录二维码

发送type=1从后端请求一个登录二维码

2.心跳包

前端连接websocket后，需要**10**s发送一次心跳包消息。

后端返回的主要包括
{{< collapse "后端返回消息类型">}}

```java
package com.fubos.mallchat.common.websocket.domain.enums;
public enum WSRespTypeEnum {
    LOGIN_URL(1, "登录二维码返回", WSLoginUrl.class),
    LOGIN_SCAN_SUCCESS(2, "用户扫描成功等待授权", null),
    LOGIN_SUCCESS(3, "用户登录成功返回用户信息", WSLoginSuccess.class),
    MESSAGE(4, "新消息", WSMessage.class),
    ONLINE_OFFLINE_NOTIFY(5, "上下线通知", WSOnlineOfflineNotify.class),
    INVALIDATE_TOKEN(6, "使前端的token失效，意味着前端需要重新登录", null),
    BLACK(7, "拉黑用户", WSBlack.class),
    MARK(8, "消息标记", WSMsgMark.class),
    RECALL(9, "消息撤回", WSMsgRecall.class),
    APPLY(10, "好友申请", WSFriendApply.class),
    MEMBER_CHANGE(11, "成员变动", WSMemberChange.class),
    ;
...
}
```
{{< /collapse >}}



> 参考文献

> [1] https://www.yuque.com/snab/mallchat/skb0r8tesr7yitvf

> [2] https://www.cnblogs.com/qiqi715/p/13138589.html

---END---