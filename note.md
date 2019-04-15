[TOC]

### 概述
- jboss 高性能 事件驱动 异步非阻塞
- NIO
- 稳定性和伸缩性
- Netty场景
    - 高性能
    - 多线程并发领域 
        - Reactor模型
    - 异步通信领域

### IO通信
- BIO
    - 一个线程负责一个请求
    - 一请求一应答
- 伪异步IO
    - 线程池负责连接
    - M请求N应答
    - 线程池阻塞 
- NIO
    - 缓冲区Buffer
    - 通道Channel 双向
    - 多路复用器Selector
    - Selector轮训Channel
    - jdk epoll轮训,没有限制
- AIO
    - 连接注册读写函数和回调函数
    - 读写方法异步
    - 主动通知程序    
    
### Netty
- 原生NIO的缺陷
    - 入门门槛高
    - 工作量和难度大 断连 重连
    - epoll Bug    
- Netty优势
    - API简单 channelHandler扩展
    - 零拷贝
    
### WebSocket
- H5协议规范
- 握手机制 
- 客户端与服务器实时通信
- HTTP握手请求后,建立TCP连接
- 节省通信开销 轮训变推送

### Hello netty~代码 
- 初始化server
```$xslt
    //主线程组
    //定义一对线程组,用于接收客户端的连接,不做任何事情
   public static void main(String[] args) throws Exception{ 
    
    EventLoopGroup bossGroup = new NioEventLoopGroup()
    
    //从线程组,boss会把任务丢给workerGroup
    EventLoopGroup workerGroup = new NioEventLoopGroup()
    
    //netty服务器的创建,启动类
    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.group(bossGroup,workerGroup) //设置主从线程组
        .channel(NioServerSocketChannel.class)   //设置nio双向通道
        .childHandler(new HelloServerInitializer());  //子处理器 处理workGroup
    
    //启动server,8088
    ChannelFuture channelFuture = serverBootstrap.bind(8088).sync();    
    
    //监听关闭channel
    channelFuture.channel().closeFuture().sync();
    }
    
```
- 给channel增加handler
```
public class HelloServerInitializer extends ChannelInitailizer<SocketChannel>
 
  @Override
  protected void initChannel(SocketChannel channel) throws Exception   
    ChannelPipeline pipeline = channel.pipeline();
    
    //通过管道添加handler
    //HttpServerCodec netty的助手类
    //用于编解码
    pipeline.addLast("HttpServerCodec",new HttpServerCodec())
    
    //添加自定义的助手类,返回"hello netty"
    pipeline.addLast("customHandler",new CustomerHandler())
```
- 自定义handler
```$xslt
//SimpleChannelInboundHandler 相当于入站
public class CustomerHandler extends SimpleChannelInboundHandler<HttpObject> {
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx,HttpObject msg)
        throws Exception {
        
        //获取channel
        Channel channel = ctx.channel();
        if (! msg instanceof HttpRequest) {
            break;
        }
        
        System.out.println(channel.remoteAddress());
        
        //读写数据都是到缓冲区
        //定义发送数据消息
        ByteBuf content = Unpooled.copidBuffer("Hello netty~",charsetUtil.UTF_8)
        
        //构建一个http response  HTTP_1_1默认长连接 keep_alive
        FullHttpResponse response = new DefaultFullHttpResponse(httpVersion.HTTP_1_1,
                        HttpResponseStatus.OK,
                        content);
        //为响应增加数据类型和长度                
        response.headers().set(HttpHeaderNames.CONTENT_TYPE,"text/plain")
        response.headers().set(HttpHeaderNames.CONTENT_LENGTH,content.readableBytes())
        
        //把响应刷到客户端 写到缓冲区,并刷到客户端
        ctx.writeAndFlush(response);        
    }
```

- curl 可以干净的请求


### netty channel生命周期
- 查看生命周期源码
    - ideal- source- Override - 
    - channelRegistered 
    - channelUnregistered
    - channelActive
    - channelInactive
    - channelReadComplete
    - userEventTriggered 用户时间触发
    - exceptionCaught 异常处理
    - handlerAdded
    - handlerRemoved         