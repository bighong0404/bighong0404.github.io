# RocketMQ-消息发送逻辑

## 1. 同步阻塞等待响应的发送\(批量\)消息的底层实现逻辑

1\) 构建`ResponseFuture`对象, 通过channel向**Broker**发送数据包, 并把`ResponseFuture`对象放入`responseTable`中, 这个concurrentHashMap对象维护所有`ResponseFuture`对象. 2\) `responseFuture.waitResponse(timeoutMillis)` 通过闭锁`await`阻塞等待. 内部实现是闭锁 3\) netty客户端收到Broker发来的数据包, 通过类型识别是响应数据包, 交由`NettyClientHandler`处理. handler根据数据包中的`requestId`从获取到`responseTable`获取对应的`ResponseFuture`对象, 回写响应数据, 把`responseFuture`对象从map中移除, 若有回调, 则执行回调, 否则通过闭锁唤醒上一步中阻塞等待响应的线程.

```java
/*====================
NettyRemotingAbstract.java 超类
====================*/
    public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request,
        final long timeoutMillis)
        throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
        final int opaque = request.getOpaque();

        try {
            final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis, null, null);
            //往responseTable保存responseFuture, 后面handler会获取出来, 然后回写数据
            this.responseTable.put(opaque, responseFuture);
            final SocketAddress addr = channel.remoteAddress();
          //往channel发数据, 并添加响应的监听器
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    if (f.isSuccess()) {
                        //响应成功
                        responseFuture.setSendRequestOK(true);
                        return;
                    } else {
                        responseFuture.setSendRequestOK(false);
                    }

                    responseTable.remove(opaque);
                    responseFuture.setCause(f.cause());
                    //响应失败
                    responseFuture.putResponse(null);
                    log.warn("send a request command to channel <" + addr + "> failed.");
                }
            });
            //在这里阻塞, 等待响应
            RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
            //... 省略
            return responseCommand;
        } finally {
            this.responseTable.remove(opaque);
        }
    }
```

看下`ResponseFuture.java`, 通过闭锁`countDownLatch`来实现阻塞同步

```java
    // 阻塞等待响应
    public RemotingCommand waitResponse(final long timeoutMillis) throws InterruptedException {
        this.countDownLatch.await(timeoutMillis, TimeUnit.MILLISECONDS);
        return this.responseCommand;
    }
    //回写响应
    public void putResponse(final RemotingCommand responseCommand) {
        this.responseCommand = responseCommand;
        //通过闭锁唤醒阻塞住的线程
        this.countDownLatch.countDown();
    }
```

在netty的处理handler中, 处理broker返回的数据包, 调用`putResponse(cmd)`来回写结果, 并通过闭锁来唤醒线程

```java
/*====================
NettyRemotingClinet.java
====================*/
        Bootstrap handler = this.bootstrap.group(this.eventLoopGroupWorker).channel(NioSocketChannel.class)
            //省略各种option()
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline pipeline = ch.pipeline();
                    //... 省略
                    pipeline.addLast(
                        defaultEventExecutorGroup,
                        new NettyEncoder(),
                        new NettyDecoder(),
                        new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()),
                        new NettyConnectManageHandler(),
                        //注册处理数据的handler
                        new NettyClientHandler());
                }
            });

    //内部类
    class NettyClientHandler extends SimpleChannelInboundHandler<RemotingCommand> {

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
            processMessageReceived(ctx, msg);
        }
    }
/*====================
NettyRemotingAbstract.java 超类
====================*/
    public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
        final RemotingCommand cmd = msg;
        if (cmd != null) {
            switch (cmd.getType()) {
                case REQUEST_COMMAND:
                    //处理broker请求的数据包
                    processRequestCommand(ctx, cmd);
                    break;
                case RESPONSE_COMMAND:
                     //处理broker响应的数据包
                    processResponseCommand(ctx, cmd);
                    break;
                default:
                    break;
            }
        }
    }

//处理borker响应的数据包
public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd) {
        final int opaque = cmd.getOpaque();
        //从responseTable中获取responseFuture
        final ResponseFuture responseFuture = responseTable.get(opaque);
        if (responseFuture != null) {
            responseFuture.setResponseCommand(cmd);

            responseTable.remove(opaque);
            //回调
            if (responseFuture.getInvokeCallback() != null) {
                executeInvokeCallback(responseFuture);
            } else {
                //回写响应数据
                responseFuture.putResponse(cmd);
                responseFuture.release();
            }
        } else {
            log.warn("receive response, but not matched any request, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
            log.warn(cmd.toString());
        }
    }
```

## 2. 无返回的单条消息发送的底层实现实现逻辑

1\) `request.markOnewayRPC();`标记这个请求是Oneway模式, 服务端服务端在返回数据包时候识别到请求是oneway, 则不写回数据包 2\) 从`semaphoreOneway`oneway信号量中获取许可, 信号量许可值默认65535. 3\) 获取许可成功, 在往通道写数据`channel.writeAndFlush(request)`时候添加监听器`addListener`, 监听器的`operationComplete`中`release`了信号量. 在数据包被发送完成后交被执行线程调用. 4\) 获取许可失败, 则打日志或者返回异常

```java
/*====================
NettyRemotingAbstract.java 超类
====================*/
    public void invokeOnewayImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
        //设置请求是Oneway模式, Broker端处理request的方法中, 判断请求是否是oneway, 不是则发回response
        request.markOnewayRPC();
        boolean acquired = this.semaphoreOneway.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
        if (acquired) {
            final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreOneway);
            try {
                channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture f) throws Exception {
                        once.release();
                        if (!f.isSuccess()) {
                            log.warn("send a request command to channel <" + channel.remoteAddress() + "> failed.");
                        }
                    }
                });
            } catch (Exception e) {
                once.release();
                log.warn("write send a request command to channel <" + channel.remoteAddress() + "> failed.");
                throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
            }
        } else {
            if (timeoutMillis <= 0) {
                throw new RemotingTooMuchRequestException("invokeOnewayImpl invoke too fast");
            } else {
                String info = String.format(
                    "invokeOnewayImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d",
                    timeoutMillis,
                    this.semaphoreOneway.getQueueLength(),
                    this.semaphoreOneway.availablePermits()
                );
                log.warn(info);
                throw new RemotingTimeoutException(info);
            }
        }
    }

    // handler中处理请求数据的方法
    public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {

      // 省略获取request processor, channelContext代码 ... 
                       if (!cmd.isOnewayRPC()) {
                            if (response != null) {
                                // 不oneway模式, 写回requestid
                                response.setOpaque(opaque);
                                // 标记是response类型
                                response.markResponseType();
                                try {
                                    // 响应数据写回客户端
                                    ctx.writeAndFlush(response);
                                } catch (Throwable e) {
                                    log.error("process request over, but response failed", e);
                                    log.error(cmd.toString());
                                    log.error(response.toString());
                                }
                            } else {
                                // 是oneway模式, 不做任何处理
                            }
                        }

    //省略异常处理代码....

}
```

## 3. 异步带回调的单条消息发送的底层实现逻辑

1\) 异步消息交由异步线程池去执行发送. 2\) 跟无返回的消息发送一样, 会从`semaphoreAsync`异步信号量中获取许可, 信号量许可值默认65535. 3\) 其他过程跟同步发送的逻辑一样, 只是handler在处理返回数据包的时候, 若有回调, 则执行回调, 否则通过闭锁唤醒上一步中同步阻塞等待响应的线程.

```java
/*====================
DefaultMQProducerImpl.java 
====================*/
    public void send(final Message msg, final SendCallback sendCallback, final long timeout)
        throws MQClientException, RemotingException, InterruptedException {
        final long beginStartTime = System.currentTimeMillis();
        //获取异步sender线程池
        ExecutorService executor = this.getAsyncSenderExecutor();
        try {
            //把异步请求msg丢入线程池处理
            executor.submit(new Runnable() {
                @Override
                public void run() {
                    long costTime = System.currentTimeMillis() - beginStartTime;
                    if (timeout > costTime) {
                        try {
                            sendDefaultImpl(msg, CommunicationMode.ASYNC, sendCallback, timeout - costTime);
                        } catch (Exception e) {
                            sendCallback.onException(e);
                        }
                    } else {
                        sendCallback.onException(
                            new RemotingTooMuchRequestException("DEFAULT ASYNC send call timeout"));
                    }
                }

            });
        } catch (RejectedExecutionException e) {
            throw new MQClientException("executor rejected ", e);
        }
    }


/*====================
NettyRemotingAbstract.java 超类
====================*/
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
        final InvokeCallback invokeCallback)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
        long beginStartTime = System.currentTimeMillis();
        final int opaque = request.getOpaque();
        //从[异步信号量]获取许可, 该值最大65535, 跟linux服务器的文件句柄值设置一致. 实现控流
        boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
        if (acquired) {
            //把[异步信号量]对象封装, 在超时或者handler执行后, release()许可. 
            final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                throw new RemotingTooMuchRequestException("invokeAsyncImpl call timeout");
            }
            //传入invokeCallback回调对象, handler处理响应时候会被执行
            final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
            //往responseTable保存responseFuture, 后面handler会获取出来, 然后回写数据
            this.responseTable.put(opaque, responseFuture);
            try {
                channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture f) throws Exception {
                        if (f.isSuccess()) {
                            responseFuture.setSendRequestOK(true);
                            return;
                        }
                        requestFail(opaque);
                        log.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                    }
                });
            } catch (Exception e) {
                //其中释放信号量许可
                responseFuture.release();
                log.warn("send a request command to channel <" + RemotingHelper.parseChannelRemoteAddr(channel) + "> Exception", e);
                throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
            }
        } else {
          //获取不到许可, 输出日志或者异常
            if (timeoutMillis <= 0) {
                throw new RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast");
            } else {
                String info =
                    String.format("invokeAsyncImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d",
                        timeoutMillis,
                        this.semaphoreAsync.getQueueLength(),
                        this.semaphoreAsync.availablePermits()
                    );
                log.warn(info);
                throw new RemotingTimeoutException(info);
            }
        }
    }
```

