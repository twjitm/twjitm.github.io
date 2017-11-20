---
title: 游戏服务器-netty实现实时数据交互系统。
date: 2017-10-31 23:30:12
tags: [netty,游戏服务器]
toc: true
description: "first post"
layout: post
---
在联网游戏中很多都有聊天系统的存在，一个战队，一个系统，一场战斗，都是需要数据通讯，对于这么多数据交互，有的同步，异步等，采用什么设计方法才能满足这么高的要求性能呢？首选netty，是的，还有一个同胞兄弟mina，都是一个很好的基于java
NIO 的框架。
   
<!-- more -->
### netty入门
#### netty的线程模型
  事实上，Netty的线程模型与Reactor线程模型相似
##### 多线程模型
Rector多线程模型与单线程模型最大的区别就是有一组NIO线程处理IO操作，它的原理图如下：
![](/img/多线程模型.png) 
Reactor多线程模型的特点：

1）有专门一个NIO线程-Acceptor线程用于监听服务端，接收客户端的TCP连接请求；

2）网络IO操作-读、写等由一个NIO线程池负责，线程池可以采用标准的JDK线程池实现，它包含一个任务队列和N个可用的线程，由这些NIO线程负责消息的读取、解码、编码和发送；

3）1个NIO线程可以同时处理N条链路，但是1个链路只对应1个NIO线程，防止发生并发操作问题。

在绝大多数场景下，Reactor多线程模型都可以满足性能需求；但是，在极个别特殊场景中，一个NIO线程负责监听和处理所有的客户端连接可能会存在性能问题。例如并发百万客户端连接，或者服务端需要对客户端握手进行安全认证，但是认证本身非常损耗性能。
在这类场景下，单独一个Acceptor线程可能会存在性能不足问题，为了解决性能问题，产生了第三种Reactor线程模型-主从Reactor多线程模型。
##### 主从多线程模型
主从Reactor线程模型的特点是：服务端用于接收客户端连接的不再是个1个单独的NIO线程，而是一个独立的NIO线程池。Acceptor接收到客户端TCP连接请求处理完成后（可能包含接入认证等），将新创建的SocketChannel注册到IO线程池（sub reactor线程池）的某个IO线程上，由它负责SocketChannel的读写和编解码工作。Acceptor线程池仅仅只用于客户端的登陆、握手和安全认证，一旦链路建立成功，就将链路注册到后端subReactor线程池的IO线程上，由IO线程负责后续的IO操作。

它的线程模型如下图所示：
![](/img/主从多线程.png) 
利用主从NIO线程模型，可以解决1个服务端监听线程无法有效处理所有客户端连接的性能不足问题。

它的工作流程总结如下：

从主线程池中随机选择一个Reactor线程作为Acceptor线程，用于绑定监听端口，接收客户端连接；
Acceptor线程接收客户端连接请求之后创建新的SocketChannel，
1. 将其注册到主线程池的其它Reactor线程上，
2. 由其负责接入认证、IP黑白名单过滤、握手等操作；
3. 步骤2完成之后，业务层的链路正式建立，将SocketChannel从主线程池的Reactor线程的多路复用器上摘除，
4. 重新注册到Sub线程池的线程上，用于处理I/O的读写操作。
### Netty线程模型
#### Netty线程模型分类

事实上，Netty的线程模型与1.2章节中介绍的三种Reactor线程模型相似，下面章节我们通过Netty服务端和客户端的线程处理流程图来介绍Netty的线程模型。
##### 服务端线程模型
一种比较流行的做法是服务端监听线程和IO线程分离，类似于Reactor的多线程模型，它的工作原理图如下：
![](/img/netty服务器线程模型.png) 
对于简单的服务器，netty的工作流程代码如下 

        package com.twjitm.common;
        import com.twjitm.common.initalizer.WebsocketChatServerInitializer;
        import com.twjitm.common.service.ControllerService;
        import com.twjitm.common.utils.Globals;
        import io.netty.bootstrap.ServerBootstrap;
        import io.netty.channel.ChannelFuture;
        import io.netty.channel.ChannelOption;
        import io.netty.channel.EventLoopGroup;
        import io.netty.channel.nio.NioEventLoopGroup;
        import io.netty.channel.socket.nio.NioServerSocketChannel;

        import java.util.logging.LogManager;
        import java.util.logging.Logger;

	/**
	 * Created by 文江 on 2017/9/25.
	 * 长连接服务启动类
	 * 佛祖保佑！永无bug
	 */
	/*
	 */
        public class RealcomServer {
        private static RealcomServer realcomServer;
		private static Logger logger = LogManager.getLogManager().getLogger(RealcomServer.class.getName());

		public static RealcomServer getInItStance() {
        if (realcomServer == null) {
        return realcomServer = new RealcomServer();
        }
        return realcomServer;
		}
        public void startServer() {
        /*  Properties properties=new Properties();
        properties.load();*/
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
        ServerBootstrap b = new ServerBootstrap(); // (2)
        b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class) // (3)
        .childHandler(new WebsocketChatServerInitializer())  //(4)
        .option(ChannelOption.SO_BACKLOG, 128)          // (5)
        .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)
        System.out.println("WebsocketChatServer 启动了");
        
        // 绑定端口，开始接收进来的连接
        ChannelFuture f = b.bind("127.0.0.1", 8088).sync(); // (7)
        // 等待服务器  socket 关闭 。
        // 在这个例子中，这不会发生，但你可以优雅地关闭你的服务器。
        try {
        Globals.init();
        Globals.startUp();
        } catch (Exception e) {
        e.printStackTrace();
        }
        f.channel().closeFuture().sync();
        
        } catch (InterruptedException e) {
        e.printStackTrace();
        } finally {
        workerGroup.shutdownGracefully();
        bossGroup.shutdownGracefully();
        System.out.println("WebsocketChatServer 关闭了");
        }
		}
		public void stopServer() {
		}
		public void initController() {
        ControllerService.init();
		}
		public static void main(String[] args) {
        RealcomServer.getInItStance().startServer();
		}
        }        
通常情况下，服务端的创建是在用户进程启动的时候进行，因此一般由Main函数或者启动类负责创建，服务端的创建由业务线程负责完成。在创建服务端的时候实例化了2个EventLoopGroup，1个EventLoopGroup实际就是一个EventLoop线程组，负责管理EventLoop的申请和释放。

EventLoopGroup管理的线程数可以通过构造函数设置，如果没有设置，默认取-Dio.netty.eventLoopThreads，如果该系统参数也没有指定，则为可用的CPU内核数 × 2。

bossGroup线程组实际就是Acceptor线程池，负责处理客户端的TCP连接请求，如果系统只有一个服务端端口需要监听，则建议bossGroup线程组线程数设置为1。

workerGroup是真正负责I/O读写操作的线程组，通过ServerBootstrap的group方法进行设置，用于后续的Channel绑定。







