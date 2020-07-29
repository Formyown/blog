---
title: Netty获取并检查Websocket握手请求
toc: true
tags: [Netty, Websocket]
categories: [Netty, Websocket]
author: Formyown
date: 2020-07-29 12:00:00
---

在使用Netty开发Websocket服务时，通常需要解析来自客户端请求的URL、Headers等等相关内容，并做相关检查或处理。本文将讨论两种实现方法。

# 方法一：基于HandshakeComplete自定义事件
> 特点：使用简单、校验在握手成功之后、失败信息可以通过Websocket发送回客户端。

## 1.1 从netty源码出发
一般地，我们将netty内置的`WebSocketServerProtocolHandler`作为Websocket协议的主要处理器。通过研究其代码我们了解到在本处理器被添加到`Pipline`后`handlerAdded`方法将会被调用。此方法经过简单的检查后将`WebSocketHandshakeHandler`添加到了本处理器之前，用于处理握手相关业务。

我们都知道Websocket协议在握手时是通过HTTP(S)协议进行的，那么这个`WebSocketHandshakeHandler`应该就是处理HTTP相关的数据的吧？

下方代码经过精简，放心阅读😄

```java
package io.netty.handler.codec.http.websocketx;

public class WebSocketServerProtocolHandler extends WebSocketProtocolHandler {

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {
        ChannelPipeline cp = ctx.pipeline();
        if (cp.get(WebSocketServerProtocolHandshakeHandler.class) == null) {
            // Add the WebSocketHandshakeHandler before this one.
            cp.addBefore(ctx.name(), WebSocketServerProtocolHandshakeHandler.class.getName(),
                    new WebSocketServerProtocolHandshakeHandler(serverConfig));
        }
        //...
    }
}
```

我们来看看`WebSocketServerProtocolHandshakeHandler`都做了什么操作。

`channelRead`方法会尝试接收一个`FullHttpRequest`对象，表示来自客户端的HTTP请求，随后服务器将会进行握手相关操作，此处省略了握手大部分代码，感兴趣的同学可以自行阅读。

可以注意到，在确认握手成功后，`channelRead`将会调用两次`fireUserEventTriggered`，此方法将会触发其他（在此处理器之后）的处理器中名为的`serEventTriggered`方法。其中一个方法传入了`WebSocketServerProtocolHandler`对象，此对象保存了HTTP请求相关信息。那么解决方案逐渐浮出水面，通过监听自定义事件即可实现检查握手的HTTP请求。

```java
package io.netty.handler.codec.http.websocketx;

/**
 * Handles the HTTP handshake (the HTTP Upgrade request) for {@link WebSocketServerProtocolHandler}.
 */
class WebSocketServerProtocolHandshakeHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(final ChannelHandlerContext ctx, Object msg) throws Exception {
        final FullHttpRequest req = (FullHttpRequest) msg;
        if (isNotWebSocketPath(req)) {
            ctx.fireChannelRead(msg);
            return;
        }

        try {

            //...
                
            if (!future.isSuccess()) {
                
            } else {
                localHandshakePromise.trySuccess();
                // Kept for compatibility
                ctx.fireUserEventTriggered(
                        WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE);
                ctx.fireUserEventTriggered(
                        new WebSocketServerProtocolHandler.HandshakeComplete(
                                req.uri(), req.headers(), handshaker.selectedSubprotocol()));
            }
        } finally {
            req.release();
        }
    }
}

```
## 1.2 解决方案

下面的代码展示了如何监听自定义事件。通过抛出异常可以终止链接，同时可以利用`ctx`向客户端以Websocket协议返回错误信息。因为此时握手已经完成，所以虽然这种方案简单的过分，但是效率并不高，耗费服务端资源。

```java
private final class ServerHandler extends SimpleChannelInboundHandler<DeviceDataPacket> {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        
        if (evt instanceof WebSocketServerProtocolHandler.HandshakeComplete) {
            // 在此处获取URL、Headers等信息并做校验，通过throw异常来中断链接。
        }
        super.userEventTriggered(ctx, evt);
    }
}

```
## 1.3 ChannelInitializer实现
附上Channel初始化代码作为参考。
```java
private final class ServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel ch) {
        ch.pipeline()
                .addLast("http-codec", new HttpServerCodec())
                .addLast("chunked-write", new ChunkedWriteHandler())
                .addLast("http-aggregator", new HttpObjectAggregator(8192))
                .addLast("log-handler", new LoggingHandler(LogLevel.WARN))
                .addLast("ws-server-handler", new WebSocketServerProtocolHandler(endpointUri.getPath()))
                .addLast("server-handler", new ServerHandler());
    }
}

```

# 方法二：基于新增安全检查处理器
> 特点：使用相对复杂、校验在握手成功之前、失败信息可以通过HTTP返回客户端。
## 2.1 解决方案
编写一个入站处理器，接收`FullHttpMessage`消息，在Websocket处理器之前检测拦截请求信息。下面的例子主要做了四件事情：
1. 从HTTP请求中提取关心的数据
2. 安全检查
3. 将结果和其他数据绑定在Channel
4. 触发安全检查完毕自定义事件

```java
public class SecurityServerHandler extends ChannelInboundHandlerAdapter {
    private static final ObjectMapper json = new ObjectMapper();

    public static final AttributeKey<SecurityCheckComplete> SECURITY_CHECK_COMPLETE_ATTRIBUTE_KEY =
            AttributeKey.valueOf("SECURITY_CHECK_COMPLETE_ATTRIBUTE_KEY");

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if(msg instanceof FullHttpMessage){
            //extracts device information headers
            HttpHeaders headers = ((FullHttpMessage) msg).headers();
            String uuid = Objects.requireNonNull(headers.get("device-connection-uuid"));
            String devDescJson = Objects.requireNonNull(headers.get("device-description"));
            //deserialize device description
            DeviceDescription devDesc = json.readValue(devDescJson, DeviceDescriptionWithCertificate.class);
            //check ......

            //
            SecurityCheckComplete complete = new SecurityCheckComplete(uuid, devDesc);
            ctx.channel().attr(SECURITY_CHECK_COMPLETE_ATTRIBUTE_KEY).set(complete);
            ctx.fireUserEventTriggered(complete);
        }
        //other protocols
        super.channelRead(ctx, msg);
    }
    @Getter
    @AllArgsConstructor
    public static final class SecurityCheckComplete {
        private String connectionUUID;
        private DeviceDescription deviceDescription;
    }
}
```
在业务逻辑处理器中，可以通过组合自定义的安全检查事件和Websocket握手完成事件。例如，在安全检查后进行下一步自定义业务检查，在握手完成后发送自定义内容等等，就看各位同学自由发挥了。
```java
 private final class ServerHandler extends SimpleChannelInboundHandler<DeviceDataPacket> {

    public final AttributeKey<DeviceConnection> 
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof SecurityCheckComplete){
            log.info("Security check has passed");
            SecurityCheckComplete complete = (SecurityCheckComplete) evt;
            listener.beforeConnect(complete.getConnectionUUID(), complete.getDeviceDescription());
        }
        else if (evt instanceof WebSocketServerProtocolHandler.HandshakeComplete) {
            log.info("Handshake has completed");
            SecurityCheckComplete complete = ctx.channel().attr(SecurityServerHandler.SECURITY_CHECK_COMPLETE_ATTRIBUTE_KEY).get();
            DeviceDataServer.this.listener.postConnect(complete.getConnectionUUID(),
                    new DeviceConnection(ctx.channel(), complete.getDeviceDescription()));
        }
        super.userEventTriggered(ctx, evt);
    }
}
```
## 2.2 ChannelInitializer实现
附上Channel初始化代码作为参考。
```java
private final class ServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel ch) {
        ch.pipeline()
                .addLast("http-codec", new HttpServerCodec())
                .addLast("chunked-write", new ChunkedWriteHandler())
                .addLast("http-aggregator", new HttpObjectAggregator(8192))
                .addLast("log-handler", new LoggingHandler(LogLevel.WARN))
                .addLast("security-handler", new SecurityServerHandler())
                .addLast("ws-server-handler", new WebSocketServerProtocolHandler(endpointUri.getPath()))
                .addLast("packet-codec", new DataPacketCodec())
                .addLast("server-handler", new ServerHandler());
    }
}
```
## 总结
上述两种方式分别在握手完成后和握手之前拦截检查；实现复杂度和性能略有不同，可以通过具体业务需求选择合适的方法。

Netty增强了责任链模式，使用`userEvent`传递自定义事件使得各个处理器之间减少耦合，更专注于业务。但是、相比于流动于各个处理器之间的"主线"数据来说，`userEvent`传递的"支线"数据往往不受关注。通过阅读Netty内置的各种处理器源码，探索其产生的事件，同时在开发过程中加以善用，可以减少冗余代码。另外在开发自定义的业务逻辑时，应该积极利用`userEvent`传递事件数据，降低各模块之间代码耦合。