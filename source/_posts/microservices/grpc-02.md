---
title: 从微服务角度理解 gRPC
date: 2021-04-01 17:29:11
categories: 
	- [gRPC]
tags:
  - microservices
author: Jony
---


# GRPC 

grpc 本质上还是 RPC 框架，为了远程调用而产生的一个产物。所以尝试从微服务的角度是看 gRPC 。

微服务主要涉及：服务注册、服务发现、服务调用。主要的还是这三个功能，其他的熔断、限流的不过是在这三个基础上增加的新功能。

## 服务注册

服务注册主要的功能就是将用户写的服务经过扫描存储到同一个地方，然后等待用户的请求时，将对应的服务取出来执行并返回结果。所以这里首先看的是如何扫描。


简单实现一个服务：

```java
public class GreeterImpl extends HelloServiceGrpc.HelloServiceImplBase {

    @Override
    public void sayFuchGrp(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {
        String name = request.getName();
        HelloResponse response = HelloResponse.newBuilder().setReply(name + "\t" + "hahahhhaa").build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
    ....
}

    public static void main(String[] args) {
            server = ServerBuilder.forPort(port)
                    .addService(new GreeterImpl())
                    .build()
                    .start();
	}                    
```


从这里看可以发现是通过手动注册一个服务到 gRPC 的注册中心。看下 addService 的详细过程，因为主要看注册中心的功能管理，所以启动相关的分析直接忽略或者查看参考文档。通过 Debug 跟踪主要涉及三个 class 文件：`AbstractServerImplBuilder`、`ServerImplBuilder`、`InternalHandlerRegistry`，这三个文件基本已经涵盖了服务注册到注册中心的整个流程。

```java
	//  AbstractServerImplBuilder
  @Override
  public T addService(BindableService bindableService) {
    delegate().addService(bindableService);
    return thisT();
  }
  	// ServerImplBuilder
   @Override
  public ServerImplBuilder addService(BindableService bindableService) {
    return addService(checkNotNull(bindableService, "bindableService").bindService());
  }

  @Override
  public ServerImplBuilder addService(ServerServiceDefinition service) {
    registryBuilder.addService(checkNotNull(service, "service"));
    return this;
  }
	// InternalHandlerRegistry
    Builder addService(ServerServiceDefinition service) {
      System.out.println(getClass() + ",将接口注册到注册中心");
      services.put(service.getServiceDescriptor().getName(), service);
      return this;
    }
```

从代码上看与 Spring 类似最后都将实例化的服务存放到了一个 HashMap 中，`Key` 就是服务名，`Value` 就是已经构建好的实例。这里可能需要看下 bindService 触发的时候做了哪些工作，Debug 进去之后就会进入通过 `protoc` 生成的 RPC 文件中，所以这里的 BindableService 就是所有 RPC 类的父类，通过统一的抽象方法来生成一个实例，绑定所有服务接口。这里也间接的说明方法支持二级路径调用。通过代码分析得知注册中心其实就是一个 HashMap 容器，将所有服务注册到容器里面方便调用。


## 服务发现和服务调用

### 服务调用

服务注册并启动成功后就可以对外提供服务，外部响应的接口也就可以发起调用。调用代码如下：

```java
        // 创建 ManagedChannelImpl
        ManagedChannel channel = ManagedChannelBuilder.forAddress("127.0.0.1",8080).usePlaintext().build();
 		// 创建客户端 Stub     
        HelloServiceGrpc.HelloServiceBlockingStub stub  = HelloServiceGrpc.newBlockingStub(channel);
        HelloRequest request = HelloRequest.newBuilder().setName("fadsfasfsafsafsafdsf").build();
        // 发起 RPC 调用，获取响应
        HelloResponse response = stub.sayFuchGrp(request);
        System.out.println("返回结果 ==========>" +response.toString());
        channel.shutdown();
```
（忽略注释行）第一行代码就是连接远程服务，底层使用到了 Netty 的 Bootstrap。 第二第三行只是声明了调用方式和构建一个参数，真正的调用再第四行。所以主要分析第四行代码，看客户端如何连接服务端发起远程调用。

**ManagedChannel 提供了接口式的切面 ClientInterceptor，它可以拦截 RPC 客户端调用，注入扩展点，以及功能定制，方便框架的使用者对 gRPC 进行功能扩展。**


主要过程如下：
1. 客户端 Stub 调用 sayFuchGrp 发起 RPC 调用
2. 通过 DnsNameResolver 进行域名解析，然后使用负载均衡策略，选择具体实例
3. 如果和具体实例没有可用连接，则创建一个新的连接
4. 对消息做序列化，然后通过 stream 发送给服务端
5. 接收到服务端响应后做反序列化操作
6. 回掉环境阻塞的客户都按线程，获取响应

因为主要分析请求和寻址，所以主要分析步骤2、3、4。先来看下主要流程经过的类：HelloServiceGrpc、ClientCall、ForwardingManagedChannel、ManagedChannelImpl 。主要涉及的就是这三个类。
因为整体调用时异步，所以会涉及很多的上下文切换，但是核心的点还是很明显：1. 异步何时发起调用，2.异步如何获取返回结果。

### 异步何时发起调用

在 ManagedChannel 创建之后，ManagedChannel 会创建一个新的 ClientCall 实例。ClientCall 的用途是业务应用层的消息调度和处理，典型用法：
```java
ClientCall<ReqT, RespT> call = channel.newCall(method,callOptions);
GrpcFuture<RespT> responseFuture = new GrpcFuture<>(call);
startCall(call, responseListener);
call.sendMessage(req);
call.halfClose();
call.request(1);
V v = responseFuture.get();
```

在创建 call 的时候传入了两个参数，分别是 method 和 callOptions，method 是 `MethodDescriptor` 的实例，由类名就可以知道是一个方法的描述，具体看下 `MethodDescriptor` 都有什么内容：

```java
public final class MethodDescriptor<ReqT, RespT> {

  private final MethodType type; // 调用方式
  private final String fullMethodName; // 方法名称
  @Nullable private final String serviceName; // 服务名称
  private final Marshaller<ReqT> requestMarshaller; //请求序列化方式
  private final Marshaller<RespT> responseMarshaller; //响应序列化方式
  private final @Nullable Object schemaDescriptor; // 方法的模式描述符。方便服务器反射服务使用
  private final boolean idempotent;  // 返回此方法是否为幂等函数。
  private final boolean safe; // 返回此方法是否安全。
  private final boolean sampledToLocalTracing; // 是否可以将此方法的RPC采样
  private final AtomicReferenceArray<Object> rawMethodNames = new AtomicReferenceArray<>(2);
}  
```

根据字段可以判断 `MethodDescriptor` 主要存储一些调用时需要使用到的基本信息，callOptions 主要存放 RPC 调用调用时附加信息，例如超时、鉴权、长度限制和线程池等，选项可以从 `ClientCalls` 中获取常用的值。跟踪 newCall 最后调用的是新建一个 `PendingCall` 的实例。

当 Calls 实例化完成之后，就会调用 startCall 和 sendMessage 方法，startCall 后面单独分析。sendMessage 的调用主要是为了完成请求对象的序列化和 HTTP2 Frame 的初始化。sendMessage 会调用 如下代码：

```java
      if (stream instanceof RetriableStream) {
        @SuppressWarnings("unchecked")
        RetriableStream<ReqT> retriableStream = (RetriableStream<ReqT>) stream;
        retriableStream.sendMessage(message);
      } else {
        stream.writeMessage(method.streamRequest(message));
      }
      // 知道 end_stream 为 true 才会调用 flush
    if (!unaryRequest) {
      stream.flush();
    }      
```

通过方法分析，发现在 sendMessage 之前 MethdoDescriptor 将 requestMessage 转成了 InputStream 类型，看下这 InputStream 类型的实现 ProtoInputStream 接收了两个参数：MessageLite 和 Parser。Parser 好解释就是之前说的 Request 序列化工具，MessageLite 看注释大概就是 Protocol Message 对象实现的抽象接口。在基础资源受限时使用 LITE_RUNTIME ，在基础资源比较大时使用 CODE_SIZE，以牺牲性能为代价更好的压缩。具体可能需要了解 protobuf 的实现了，这里大概就这个意思吧。


消息被转为 InputStream 类型之后就会进入 stream.writeMessage 方法继续往下跟踪会到跟踪到 `MessageFramer.writeKnownLengthUncompressed` 这里大概的意思就是写出没有经过序列化已知长度和为压缩的数据，会再次将数据转存存放到一个 ByteBuffer 中，最后 ByteBuffer 中的数据格式如下：

![ByteBuffer 数据格式](/jony.github.io/images/grpc/grpc_payload.png)

将数据转存写出完成后，会调用 Message.close 方法以此释放原有的资源，所以这里整个流程看下来 sendMessage 并非 `send Message` 只是将消息进行了序列化和格式化处理。然后接下来调用的时 halfClose 方法。halfClose -> endOfMessages -> close -> commitToSink -> deliverFrame -> writeFrameInternal ->  新建 SendGrpcFrameCommand 类，将 请求 Frame 封装成自定义的 SendGrpcFrameCommand 放入写队列中。

配合流程图查看：

![ByteBuffer 数据格式](/jony.github.io/images/grpc/grpc-02-11.png)

```java
 private void writeFrameInternal(WritableBuffer frame, boolean endOfStream, boolean flush, final int numMessages) {
      Preconditions.checkArgument(numMessages >= 0);
      ByteBuf bytebuf = frame == null ? EMPTY_BUFFER : ((NettyWritableBuffer) frame).bytebuf().touch();
      final int numBytes = bytebuf.readableBytes();
      if (numBytes > 0) {
        onSendingBytes(numBytes);
        writeQueue.enqueue(new SendGrpcFrameCommand(transportState(), bytebuf, endOfStream), flush)
            .addListener(...);
      } else {
        writeQueue.enqueue(
            new SendGrpcFrameCommand(transportState(), bytebuf, endOfStream), flush);
      }
    }
```

然后由 WriteQueue 异步执行 flush 将 SendGrpcFrameCommand 写入到 Netty 的 Channel 中，调用 Channel 的 write 方法，别 NettyClientHandler 拦截到，由 NettyClientHandler 负责具体的发送操作。SendGrpcFrameCommand 的顶层实现了 WriteQueue.QueuedCommand 接口所以可以由 WriteQueue 统一调度。

这里可能会有疑问就是 SendGrpcFrameCommand 写出数据是如何被 NettyClientHandler 拦截到然后再执行真实发送接口的。上面提到是将 SendGrpcFrameCommand 写入到 Netty 的 Channel 是在哪写入的，后来又在什么地方调用的 write 方法的。看看 WriteQueue 构造方法就是在新建一个 WriteQueue 的时候就已经创建好了一个 Channel ，还记得之前说的 startCall 么，在调用 startCall 的时候如果没有连接会创建一个新的连接，这个时候创建的就是 Netty 连接自然就包含了 Channel 实例，所以这里创建 Channel 先终止，后面分析。然后继续查看 WriteQueue 异步执行之后就调用 write 方法这里才是正式调用写出接口。

```java
    @Override
    public final void run(Channel channel) {
      channel.write(this, promise);
    }

```

调用 write 接口之后会被 NettyClientHandler 拦截到，也就是 Netty pipeline 的处理过程，在这里会调用 sendGrpcFrame 方法。

```java
  private void sendGrpcFrame(ChannelHandlerContext ctx, SendGrpcFrameCommand cmd, ChannelPromise promise) {
      encoder().writeData(ctx, cmd.stream().id(), cmd.content(), 0, cmd.endStream(), promise);
  }
```

调用 Http2ConnectionEncoder writeData 方法之后就进入了最终的写出数据的流程。

### 服务发现

基本上服务注册与调用服务流程就这么多，但是中间有个环节没有分析那就是服务发现就是刚才没有分析的 startCall 流程。整个服务发现与创建连接的流程都在这里。接下来分析下 startCall 。

startCall 是在 newCall 之后被调用的但是并没有发起真正的调用，也就是说 startCall 可能在创建一个新的连接并做了很多初始化的操作。跟踪代码下来会发现主要有两个方法被调用：`(DelayedClientCall)call.start` 和 `responseListener.onStart` 。`(DelayedClientCall)call.start` 方法的注释：**启动一个调用，使用responseListener处理响应消息。它必须在这个类的任何其他方法之前被调用，除了可以在任何时候被调用的cancel。** 直接说明了这个方法是用来处理响应结果的。

```java
@Override
  public final void start(Listener<RespT> listener, final Metadata headers) {
    ...
      if (!savedPassThrough) {
        listener = delayedListener = new DelayedListener<>(listener);
      }
      ...
          realCall.start(finalListener, headers);
      ...
    
  }
```
两个参数：listen 就是用来监听返回结果的回调，Metadata 则是gRPC 自定义的一个 Header。继续 debug 查看 `(ClientStreamProvider)realCall.start` 方法。




`(UnaryStreamToFuture)responseListener.onStart` 方法最后会调用 `(ClientStreamProvider)realCall.request ->(DelayedStream)stream.request` 方法。 






NettyClientHandler 调用 Http2ConnectionEncoder 的 writeData 方法，将 Frame 写入到 HTTP2 Stream 中完成请求的发送。






# 参考文档

[gRPC 源码阅读系列](https://ilily.site/grpc-01/)



