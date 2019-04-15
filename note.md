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

### 实时通信
- 实现方式
    - Ajax轮训
        - 浏览器定时询问,不需要刷新浏览器
    - Long pull
        - 阻塞模式
    - websocket
        - 持久化协议         
- WSServerInitialzer
```$xslt
    
    pipeline.addLast("HttpServerCodec",new HttpServerCodec())

    //对写大数据流的支持
    pipeline.addLast(new ChunkedWriteHandler())
    
    //对HTTPMessage进行聚合,FullHttpRequest或FullHttpResponse
    //几乎都会用到次handler
    pipeline.addLast(new HttpObjectAggregator())
    
    //======以上用于支持HTTP协议
    
    // websocket服务器处理的协议,用于指定客户端连接的路由
    // 心跳保持 close ping pong
    // 以frames进行传输
    pipeline.addLast(new WebSocketServerProtocolHandler("/ws"))
    
    //自动以助手类
    pipeline.addLast(null)
``` 
- ChatHandler <TextWebSocketFrame>
```$xslt

    // 用于记录和管理所有客户端的channel
    private static ChannelGroup clients = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE); 
    
    //TextWebSocketFrame 处理文本 
    channelRead0(ctx,msg){
        //获取客户端消息
        String content = msg.text();
        clients.writeAndFlush(new TextWebSocketFrame(content));           
    }
    
    //建连时获取channel
    public handlerAdded
        clients.add(ctx.channel());
        
    public handlerRemoved
        //当触发handlerRemoved channelGroup会自动移除channel
        clients.remove(ctx.channel())
```
### Springboot整合
- users表
    - 大小头像
    - qrcode 用户扫描加好友
    - cid clientId 手机的id
    
- friends_request
    - send_user_id
    - accept_user_id
    - request_time

- my_friends
    - my_user_id
    - my_friend_user_id

- chat_msg
    - send_user_id
    - accept_user_id
    - msg
    - sign_flag 未读 已读
    - create_time
    
- mybatis逆向生成 从表自动生成对象
    - 配置generatorConfig.xml
    - jdbcConnection
    - pojo位置
    - mapper位置
    - mapper与java映射
    - table
    - 运行GeneratorDisplay.java
                
- Springboot搭建
    - @ComponentScan
    - @MapperScan
    - WSServer修改
    ```$xslt
       @Component
       public class WSServer             
       
       private static class SingletionWSServer {
            static final WSServer instance = new WSServer();
       }
       
       public static WSServer getInstance() {
            return SingletionWSServer.instance;
       }
       
       public void start() {
            this.future = server.bind(8088);
       }
    ```   
    - NettyBooter启动
    ```$xslt
        @Component
        publice class NettyBooter implements ApplicationListener<ContextRefreshedEvnet>
        
        @Override
        public void onApplicaitonEvent(event){
            WSServer.getInstance().start()
        }

    ```
### 文件服务器
- 第三方 七牛云
- FastDFS
    - C语音实现
    - Tracker Server
        - 维护Storage Server状态
        - 返回可用Storage Server给客户端
        
    - Storage Server
        - 文件写入
        - 返回文件相对路径 
    
    - 客户端
        - 保存文件路径       