---
title: NioServerSocketChannel初始化过程源码分析
date: 2024-11-01 20:30:00 +0800
categories: [网络编程, Netty]
tags: [netty]     
---

# NioServerSocketChannel初始化过程

## 继承结构

![image.png](/assets/images/NioServerSocketChannelInitProcess/image.png)

## 默认构造过程

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%201.png)

`newChannel(provider, family)`方法底层通过调用jdk nio相关方法创建具体的`ServerSocketChannel`实例。可见Netty的NioServerSocketChannel底层使用的是java的ServerSocketChannel。

继续调用父类构造器

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%202.png)

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%203.png)

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%204.png)

构造完成后相关对象的属性设置情况如下：

| 类名 | NioServerSocketChannel                                                     | AbstractNioChannel                             | AbstractChannel                                   |
| ---- | -------------------------------------------------------------------------- | ---------------------------------------------- | ------------------------------------------------- |
| 属性 | `config = new NioServerSocketChannelConfig(this, javaChannel().socket());` | `this.ch=ch;`//java的ServerSocketChannel实例   | `this.parent = null;`                             |
|      |                                                                            | `this.readInterestOp = SelectionKey.OP_ACCEPT` | `id = newId();` //设置channel唯一标识             |
|      |                                                                            |                                                | `unsafe = newUnsafe();` //用于调用底层channel方法 |
|      |                                                                            |                                                | `pipeline = new DefaultChannelPipeline(this);`    |

注意构造过程中会设置父类`AbstractChannel`的`pipeline`属性为`new DefaultChannelPipeline(this)` ，可见每一个channel实例都拥有自己的ChannelPipeline且构造的过程即会创建。

## 使用ServerBootStrap引导NioServerSocketChannel进行服务端监听过程中对Channel的初始化配置过程

服务端引导代码如下：

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

serverBootstrap.bind()方法内初始化channel相关代码如下：

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%205.png)

具体初始化过程如下：

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%206.png)

`channelFactory.newChannel();` 通过反射机制调用NioServerSocketChannel的默认构造器。具体构造过程如前文所述。

`init(channel)` 方法获取ServerBootstrap引导类提供的相关配置参数，并将其应用到上面创建的Channel实例上。然后向channel的pipeline通过addLast()添加一个ServerBootstrapAcceptor。

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%207.png)

ServerBootstrapAcceptor类继承ChannelInboundHandlerAdapter并在channelRead方法中完成对子channel的初始化。子channel初始化步骤和父channel类似，也是设置相关的选项、属性、handler。并将子channel绑定到worker线程。

这里需要注意向pipeline种添加handler的addLast方法（最终调用内部私有方法internalAdd如下：）

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%208.png)

截止当前channel只是完成了默认构建，registered还是默认值false。此时会调用callHandlerCallbackLater方法

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%209.png)

此方法向链表结构变量`pendingHandlerCallbackHead`尾部添加了`new PendingHandlerAddedTask(ctx)` ，`PendingHandlerAddedTask` 主要用来执行调用对应handler的`handlerAdded`方法

所以在registered=false时所有添加进pipeline的handler，在channel注册到eventloop后其handlerAdded方法都将被依次调用。

具体过程我们参考ServerBootstrap#bind()方法中将NioServerSocketChannel注册到eventloop的 代码

```java
//..AbstractBootstrap..
ChannelFuture regFuture = config().group().register(channel);
```

此时config().group()为bossGroup=new NioEventLoopGroup();

register()方法由父类MultithreadEventLoopGroup提供

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%2010.png)

next()方法从EventLoopGroup管理的多个EventLoop中轮询选择出一个，并将channel注册到该EventLoop上。并且整个channel生命周期内都不会改变。其内部实际上是调用Channel的register方法将eventLoop对象设置到Channel的eventLoop属性上：

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%2011.png)

然后设置channel的字段registered=true，并将依次调用pipeline上所有handler的handlerAdded方法。最后在pipeline上触发channelRegistered事件

![image.png](/assets/images/NioServerSocketChannelInitProcess/image%2012.png)