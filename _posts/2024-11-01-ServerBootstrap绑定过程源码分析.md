---
title: ServerBootstrap绑定过程源码解析
date: 2024-11-01 20:30:00 +0800
categories: [网络编程, Netty]
tags: [netty]     
---

# ServerBootstrap绑定过程源码解析

服务端引导类代码的写法一般都是类似下面这种写法

```java
//无参的EventLoopGroup构造器默认线程数为2倍核心数，也可以通过启动参数：io.netty.eventLoopThreads来设置
NioEventLoopGroup bossGroup = new NioEventLoopGroup();
NioEventLoopGroup workerGroup = new NioEventLoopGroup();

try{
		ServerBootstrap serverBootstrap = new ServerBootstrap();
		serverBootstrap
				//设置childGroup=workerGroup，父类AbstractBootstrap.group=bossGroup
        .group(bossGroup,workerGroup)
        //基于参数提供的具体Channel实现类class来初始化一个利用反射机制创建具体Channel的实例工厂（ReflectiveChannelFactory），并设置到父类AbstractBootstrap.channelFactory域
        .channel(NioServerSocketChannel.class)
        //添加channel选项到父类AbstractBootstrap.options，后面会用来设置父channel也就是NioServerSocketChannel。如果要设置子channel的选项则应该使用childOption方法。
        .option(ChannelOption.SO_BACKLOG,100)
        //这里使用一个**特殊的ChannelHandler子类：ChannelInitializer**来设置ServerBootstrap.childHandler
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel socketChannel) throws Exception {
                socketChannel.pipeline()
                    .addLast(new LineBasedFrameDecoder(Integer.MAX_VALUE))
                    .addLast(new LineEncoder())
                    .addLast(new ChannelInboundHandlerAdapter(){
                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                            ByteBuf byteBuf = (ByteBuf) msg;
                            String received = byteBuf.toString(CharsetUtil.UTF_8);
                            log.info("received msg: {} from: {}",received,ctx.channel());
                            byteBuf.release();
                            ctx.writeAndFlush(received);
                        }

                        @Override
                        public void channelActive(ChannelHandlerContext ctx) throws Exception {
                            ctx.writeAndFlush("Hi! What can i do for you?");
                        }
                    });
            }
		    });
		ChannelFuture future = serverBootstrap.bind(8688).sync();
		future.channel().closeFuture().sync();
}catch(Exception e){
		log.error("server error",e);
}finally{
		bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

上面代码首先通过调用ServerBootstrap的相关方法设置了一些属性值（具体见上面代码注释）。具体这些属性值设置后有什么作用需要深入跟踪后面的bind方法。这个方法实际实在父类AbstractBootstrap中定义的

```java
//... class AbstractBootstrap ...

public ChannelFuture bind(int inetPort) {
    return this.bind(new InetSocketAddress(inetPort));
}

public ChannelFuture bind(SocketAddress localAddress) {
    this.validate();
    return this.doBind((SocketAddress)ObjectUtil.checkNotNull(localAddress, "localAddress"));
}
```

validate()方法主要完成一些参数的校验，不是核心逻辑。我们主要关注doBind方法

```java
//... class AbstractBootstrap ...
private ChannelFuture doBind(final SocketAddress localAddress) {
		//重点关注...
    final ChannelFuture regFuture = this.initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    } else if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        //重点关注...
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
	          public void operationComplete(ChannelFuture future) throws Exception {
	              Throwable cause = future.cause();
	              if (cause != null) {
	                  promise.setFailure(cause);
	              } else {
                    promise.registered();
                        AbstractBootstrap.doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            }
        });
        return promise;
    }
}
```

doBind方法中我们重点关注`this.initAndRegister()`和`doBind0(regFuture, channel, localAddress, promise)。其中doBingd0方法用于将底层socket绑定到服务器监听端口上`。

`initAndRegister()`代码如下：

```java
//... class AbstractBootstrap ...

final ChannelFuture initAndRegister() {
        Channel channel = null;

        try {
		        //这里使用前面提到ReflectiveChannelFactory通过反射的方式创建具体的channel实例
            channel = this.channelFactory.newChannel();
            //重点关注...
            this.init(channel);
        } catch (Throwable var3) {
            Throwable t = var3;
            if (channel != null) {
                channel.unsafe().closeForcibly();
                return (new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE)).setFailure(t);
            }

            return (new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE)).setFailure(t);
        }
        
				//重点关注...
        ChannelFuture regFuture = this.config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        return regFuture;
    }
```

首先调用channelFactory的newChannel方法创建具体的channel实例。具体channel类型由`ServerBootStrap.channel(NioServerSocketChannel.class)`方法参数指定。

然后重点关注`this.init(channel)`和`ChannelFuture regFuture = this.config().group().register(channel);`前者用来配置子连接（下面具体介绍），后者用来讲父连接（ServerSocketChannel)注册到bossGroup中

`this.init(channel)`方法由子类ServerBootStrap实现：

```java
//... class ServerBootstrap ...

void init(Channel channel) {
		//为channel设置选项和属性
    setChannelOptions(channel, this.newOptionsArray(), logger);
    setAttributes(channel, this.newAttributesArray());
    
    //channel构造方法里会创建DefaultChannelPipeline实例并赋值给实例属性：pipeline。
    //这里通过pipeline()方法获取该实例
    ChannelPipeline p = channel.pipeline();
    
    final EventLoopGroup currentChildGroup = this.childGroup;
    final ChannelHandler currentChildHandler = this.childHandler;
    final Map.Entry<ChannelOption<?>, Object>[] currentChildOptions = newOptionsArray(this.childOptions);
    final Map.Entry<AttributeKey<?>, Object>[] currentChildAttrs = newAttributesArray(this.childAttrs);
    final Collection<ChannelInitializerExtension> extensions = this.getInitializerExtensions();
    p.addLast(new ChannelHandler[]{new ChannelInitializer<Channel>() {
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = ServerBootstrap.this.config.handler();
            if (handler != null) {
                pipeline.addLast(new ChannelHandler[]{handler});
            }

						**//重点关注这里,为监听channel添加ServerBootstrapAcceptor**
            ****ch.eventLoop().execute(new Runnable() {
                public void run() {
                    pipeline.addLast(new ChannelHandler[]{new ServerBootstrapAcceptor(ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs, extensions)});
                }
            });**
        }
    }});
    if (!extensions.isEmpty() && channel instanceof ServerChannel) {
        ServerChannel serverChannel = (ServerChannel)channel;
        Iterator var9 = extensions.iterator();

        while(var9.hasNext()) {
            ChannelInitializerExtension extension = (ChannelInitializerExtension)var9.next();

            try {
                extension.postInitializeServerListenerChannel(serverChannel);
            } catch (Exception var12) {
                Exception e = var12;
                logger.warn("Exception thrown from postInitializeServerListenerChannel", e);
            }
        }
    }

}
```

init方法中最重要的部分是为父Channel(用于监听服务端端口接受客户端连接)设置了
ChannelInboundHandlerAdapter:**ServerBootstrapAcceptor** 

其主要作用是对建立连接的子Channel（SocketCHannel）设置其选项、属性、处理器链并将其注册到工作线程worker中

再来看一下代码：`doBind0(regFuture, channel, localAddress, promise);`

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image.png)

最终等待channel成功注册到eventLoop后调用channel的bind方法

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%201.png)

继续跟踪代码

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%202.png)

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%203.png)

由于引导过程是运行在主线程而不在NioServerSocketChannel所注册的EventLoop线程上，所以这里调用safeExecue方法最终会触发EventLoop线程开始启动处理事件循环并执行当前任务：next.invokeBind(localAddress,promise);

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%204.png)

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%205.png)

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%206.png)

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%207.png)

当线程开始循环处理事件时将会从任务队列中取出上述绑定任务并进行执行。最终会调用headerContext的bind方法

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%208.png)

追踪DefaultChannelPipeline的fireChannelActive方法

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%209.png)

最终调用到headerContext的channelActive方法如下

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%2010.png)

ctx.fireChannelActive()方法会从下一个ChannelInboundHandler开始触发依次调用channelActive（）。重点关注`readIfIsAutoRead()`方法

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%2011.png)

对于NioServerSocketChannel来说isAutoRead()=true,因此会执行该channel的read方法

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%2012.png)

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%2013.png)

上述代码从channelPipeline尾部出发（不包括tail）向前找到最近的一个支持处理read事件的outBoundHandler，对于此引导程序则是HeadContext。然后调用其read方法

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%2014.png)

上述read方法会调用channel的beginRead方法

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%2015.png)

![image.png](/assets/images/ServerBootstrap%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image%2016.png)

由于当前创建的NioServerSocketChannel实例的readInterestOp=SelectionKey.OP_ACCEPT。所以上面代码最终执行的效果是使得当前channel在selector上注册了对获取连接事件感兴趣。后续有客户端连接过来时将能被当前channel上的ChannelHandler（具体来讲是前面channel初始化时设置的：**ServerBootstrapAcceptor** ）正确处理。