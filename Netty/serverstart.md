## Netty服务端启动分析
```java
private ChannelFuture doBind(final SocketAddress localAddress) {
        //初始化Channel，并向Selector注册Accpet事件
        final ChannelFuture regFuture = this.initAndRegister();
        final Channel channel = regFuture.channel();
        if(regFuture.cause() != null) {
            return regFuture;
        } else if(regFuture.isDone()) {
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            final AbstractBootstrap.PendingRegistrationPromise promise = new AbstractBootstrap.PendingRegistrationPromise(channel, null);
            regFuture.addListener(new ChannelFutureListener() {
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if(cause != null) {
                        promise.setFailure(cause);
                    } else {
                        promise.executor = channel.eventLoop();
                    }
                    //绑定端口
                    AbstractBootstrap.doBind0(regFuture, channel, localAddress, promise);
                }
            });
            return promise;
        }
    }
```
1. 设置启动类参数，最重要的就是设置channel
2. 创建server对应的channel，创建各大组件，包括ChannelConfig,ChannelId,ChannelPipeline,ChannelHandler,Unsafe等
3. 初始化server对应的channel，设置一些attr，option，以及设置子channel的attr，option，给server的channel添加新channel接入器，并出发addHandler,register等事件
4. 调用到jdk底层做端口绑定，并触发active事件，active触发的时候，真正做服务端口绑定

