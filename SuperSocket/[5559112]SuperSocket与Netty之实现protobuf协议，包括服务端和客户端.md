今天准备给大家介绍一个c#服务器框架（[SuperSocket](https://github.com/kerryjiang/SuperSocket)）和一个c#客户端框架（[SuperSocket.ClientEngine](https://github.com/kerryjiang/SuperSocket.ClientEngine)）。这两个框架的作者是园区里面的[江大渔](http://www.cnblogs.com/jzywh/)。  首先感谢他的无私开源贡献。之所以要写这个文章是因为群里经常有人问这个客户端框架要如何使用。原因在于服务端框架的文档比较多，客户端的文档比较少，所以很多c#基础比较差的人就不懂怎么玩起来。今天就这里写一个例子希望能给部分人抛砖引玉吧。

参考资料：

SuperSocket文档 http://docs.supersocket.net/

我以前在开源中国的一部分文章：http://my.oschina.net/caipeiyu/blog


这篇文章选择 [protobuf](https://github.com/google/protobuf) 来实现,选择[protobuf](https://github.com/google/protobuf)是因为服务器有可能用的是java的netty，客户端想用SuperSocket.ClientEngine，而netty我看很多人经常用protobuf。



---------------

##一、SuperSocket服务器

新建一个项目 ProtobufServer 然后添加 SuperSocket 和  protobuf 的依赖包。

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160531102024367-1345490028.png)

添加protobuf依赖包 输入的搜索词是 Google.ProtocolBuffers
添加SuperSocket依赖包 输入搜索词是 SuperSocket，要添加两个SuperSocket.Engine 和 SuperSocket

上面的工作完成后，我们就应该来实现我们的传输协议了。传输协议打算参考netty的[ProtobufVarint32FrameDecoder.java](https://github.com/netty/netty/blob/3a9f47216143082bdfba62e8940160856767d672/codec/src/main/java/io/netty/handler/codec/protobuf/ProtobufVarint32FrameDecoder.java)

    * BEFORE DECODE (302 bytes)       AFTER DECODE (300 bytes)
    * +--------+---------------+      +---------------+
    * | Length | Protobuf Data |----->| Protobuf Data |
    * | 0xAC02 |  (300 bytes)  |      |  (300 bytes)  |
    * +--------+---------------+      +---------------+

Protobuf Data是protobuf的序列化结果。Length（Base 128 Varints）是表示Protobuf Data的长度。protobuf本身的序列号协议可以参考：https://developers.google.com/protocol-buffers/docs/encoding

我们先看一下SuperSocket的[内置的常用协议实现模版](http://docs.supersocket.net/v1-6/zh-CN/The-Built-in-Common-Format-Protocol-Implementation-Templates)看看有没有合适我们可以直接拿来用的。因为Length使用的是Base 128 Varints一种处理整数的变长二进制编码算法，所以呢内置的协议实现模板并不能直接拿来使用，所以我们只能自己来实现接口IRequestInfo和IReceiveFilter了，参考：[使用 IRequestInfo 和 IReceiveFilter 等等其他对象来实现自定义协议](http://docs.supersocket.net/v1-6/zh-CN/Implement-Your-Own-Communication-Protocol-with-IRequestInfo,-IReceiveFilter-and-etc)。

*这里说明一下：为什么protobuf明明序列化成Protobuf Data 了为什么还要再加一个Length来打包，因为tcp这个流发送会参数粘包、分包，如果不加个协议来解析会读取错误的数据而导致无法反序列化  Protobuf Data （自行谷歌 tcp 粘包、分包）*


#### ProtobufRequestInfo的实现
在实现ProtobufRequestInfo之前要先来考虑一个问题，那就是我们的传输协议是`长度+protobuf数据`，那么我们根本就无法知道获取到的protobuf数据该如何反序列化。在官方网站提供了一种解决思路：https://developers.google.com/protocol-buffers/docs/techniques#union
就是我们可以弄唯一个数据包，然后这个数据包里面必须包含一个枚举值，然后还包含了其他类型的数据包，每一个枚举值对应一个数据包，然后传送过来后，可以用分支判断来获取值。

那我们先设计一个 DefeatMessage.proto包含内容：

    import "BackMessage.proto";
    import "CallMessage.proto";
    
    message DefeatMessage {
        enum Type { CallMessage = 1; BackMessage = 2; }

        required Type type = 1;

        optional CallMessage callMessage = 2;
        optional BackMessage backMessage = 3;
    }

然后再把CallMessage和BackMessage补全

    message BackMessage {

        optional string content = 1;

    }

    message CallMessage {

        optional string content = 1;

    }
    
然后在我们的路径packages\Google.ProtocolBuffers.2.4.1.555\tools里面有两个工具protoc.exe 和 protogen.exe,我们可以执行下面的命令来生成我们的c#代码

>protoc --descriptor_set_out=DefeatMessage.protobin --proto_path=pack --include_imports pack\DefeatMessage.proto

>protogen DefeatMessage.protobin

*注意路径要自己修改*

如果有报Expected top-level statement (e.g. "message").这么一个错误，那就是你cmd的编码和proto的编码不一致，要改成一致。

相关文件：https://github.com/kotcmm/SuperSocket.ClientEngine.QuickStart/tree/master/ProtobufServer/Pack

生成完c#代码后，我们就要设计ProtobufRequestInfo了。这个比较简单，只要实现IRequestInfo接口。我们这里在实现接口带的属性外另加一个 DefeatMessage 和 DefeatMessage.Types.Type，其中DefeatMessage是为了存储我们解包完数据后反序列化出来的对象，Type是为了方便区分我们应该取出DefeatMessage里面的哪个值。

    using System;
    using SuperSocket.SocketBase.Protocol;

    namespace ProtobufServer
    {
        public class ProtobufRequestInfo : IRequestInfo
        {
            	public string Key {
                    get;
                    private set;
                }

                public DefeatMessage.Types.Type Type
                {
                    get;
                    private set;
                }

                public DefeatMessage Body { get; private set; }

                public ProtobufRequestInfo (DefeatMessage.Types.Type type, DefeatMessage body)
                {
                    Type = type;
                    Key = type.ToString();
                    Body = body;
                }
        }
    }


#### ProtobufReceiveFilter的实现

代码比较长，直接看在github上的代码[ProtobufReceiveFilter的实现](https://github.com/kotcmm/SuperSocket.ClientEngine.QuickStart/blob/master/ProtobufServer/ProtobufReceiveFilter.cs)
实现的注意点参考：[使用 IRequestInfo 和 IReceiveFilter 等等其他对象来实现自定义协议](http://docs.supersocket.net/v1-6/zh-CN/Implement-Your-Own-Communication-Protocol-with-IRequestInfo,-IReceiveFilter-and-etc)。主要是对ss里面给我们缓存的数据流进行协议解析。

1. readBuffer: 接收缓冲区, 接收到的数据存放在此数组里
2. offset: 接收到的数据在接收缓冲区的起始位置
3. length: 本轮接收到的数据的长度
4. toBeCopied: 表示当你想缓存接收到的数据时，是否需要为接收到的数据重新创建一个备份而不是直接使用接收缓冲区
5. rest: 这是一个输出参数, 它应该被设置为当解析到一个为政的请求后，接收缓冲区还剩余多少数据未被解析


        public  ProtobufRequestInfo Filter (byte[] readBuffer, int offset, int length, bool toBeCopied, out int rest)
            {
                rest = 0;
                var readOffset = offset - m_OffsetDelta;//我们重新计算缓存区的起始位置，这里要说明的是如果前一次解析还有剩下没有解析到的数据，那么就需要把起始位置移到之前最后要解析的那个位置

            CodedInputStream stream = CodedInputStream.CreateInstance (readBuffer, readOffset, length);//这个类是Google.ProtocolBuffers提供的
			var varint32 = (int)stream.ReadRawVarint32 ();//这里是计算我们这个数据包是有多长（不包含length本身）
			if(varint32 <= 0) return null;

		    var headLen = (int) stream.Position - readOffset;//计算协议里面length占用几位
            rest = length - varint32 - headLen + m_ParsedLength;//本次解析完后缓存区还剩下多少数据没有解析

			if (rest >= 0)//缓存区里面的数据足够本次解析
			{	
				byte[] body = stream.ReadRawBytes(varint32);
                DefeatMessage message = DefeatMessage.ParseFrom(body);
				var requestInfo = new ProtobufRequestInfo(message.Type,message);
				InternalReset();
				return requestInfo;
			}
			else//缓存区里面的数据不够本次解析，（名词为分包）
			{
				m_ParsedLength += length;
				m_OffsetDelta = m_ParsedLength;
				rest = 0;

				var expectedOffset = offset + length;
				var newOffset = m_OrigOffset + m_OffsetDelta;

				if (newOffset < expectedOffset)
				{
					Buffer.BlockCopy(readBuffer, offset - m_ParsedLength + length, readBuffer, m_OrigOffset, m_ParsedLength);
				}

				return null;
			}
		}


#### ProtobufAppSession 的实现

    using System;
    using SuperSocket.SocketBase;

    namespace ProtobufServer
    {
        public class ProtobufAppSession : AppSession<ProtobufAppSession,ProtobufRequestInfo>
        {
            public ProtobufAppSession ()
            {
            }
        }
    }

#### ProtobufAppServer 的实现

    using System;
    using SuperSocket.SocketBase;
    using SuperSocket.SocketBase.Protocol;

    namespace ProtobufServer
    {
        public class ProtobufAppServer : AppServer<ProtobufAppSession,ProtobufRequestInfo>
        {
            public ProtobufAppServer ()
                :base(new DefaultReceiveFilterFactory< ProtobufReceiveFilter, ProtobufRequestInfo >())
            {
            }
        }
    }

#### 服务器的实例启动实现
参考：http://docs.supersocket.net/v1-6/zh-CN/A-Telnet-Example

代码：https://github.com/kotcmm/SuperSocket.ClientEngine.QuickStart/blob/master/ProtobufServer/Program.cs

主要是接收到数据的一个方法实现，当然ss里面还带了命令模式的实现，不过这个不在本文章里面说。这里的实现了接收到不同的数据给打印出来，然后接收到CallMessage数据的话就给客户端回发一条信息

    private static void AppServerOnNewRequestReceived(ProtobufAppSession session, ProtobufRequestInfo requestInfo)
	    {
	        switch (requestInfo.Type)
	        {
                case DefeatMessage.Types.Type.BackMessage:
                    Console.WriteLine("BackMessage:{0}", requestInfo.Body.BackMessage.Content);
                    break;
                case DefeatMessage.Types.Type.CallMessage:
	                Console.WriteLine("CallMessage:{0}", requestInfo.Body.CallMessage.Content);

                    var backMessage = BackMessage.CreateBuilder()
					.SetContent("Hello I am form C# server by SuperSocket").Build();
                    var message = DefeatMessage.CreateBuilder()
                        .SetType(DefeatMessage.Types.Type.BackMessage)
                        .SetBackMessage(backMessage).Build();

                    using (var stream = new MemoryStream())
                    {

                        CodedOutputStream os = CodedOutputStream.CreateInstance(stream);

                        os.WriteMessageNoTag(message);

                        os.Flush();

                        byte[] data = stream.ToArray();
                        session.Send(new ArraySegment<byte>(data));

                    }


                    break;

            }
	    }

服务器的代码就到这里，可以编译运行起来看看有无错误。

## 二、SuperSocket.ClientEngine客户端

与服务器实现相同，先通过NuGet添加 SuperSocket.ClientEngine 和 protobuf 的依赖包。
有三个实现：

####ProtobufPackageInfo的实现

把前面实现服务器时候生成的通讯数据包拷贝过来，然后和实现服务器的ProtobufRequestInfo一样，只不过这里只是实现接口IPackageInfo而已

    using SuperSocket.SocketBase.Protocol;using SuperSocket.ProtoBase;

    namespace ProtobufClient
    {
        public class ProtobufPackageInfo : IPackageInfo
        {
            public ProtobufPackageInfo(DefeatMessage.Types.Type type, DefeatMessage body)
            {
                Type = type;
                Key = type.ToString();
                Body = body;
            }

            public string Key { get; private set; }

            public DefeatMessage Body { get; private set; }
            public DefeatMessage.Types.Type Type { get; private set; }
        }
    }
####ProtobufReceiveFilter的实现

代码：https://github.com/kotcmm/SuperSocket.ClientEngine.QuickStart/blob/master/ProtobufClient/ProtobufReceiveFilter.cs
这里的数据解析的实现与服务器的实现有点不同，不过下一个版本可能会统一，如果统一起来的话，那么以后数据解析就可以做成和插件一样，同时可以给服务器和客户端使用。

1. data：也是数据缓存区
2. rest：缓存区还剩下多少

这个实现与服务器的不同就在BufferList本身就已经有处理分包，就不需要我们自己再做处理。

    public ProtobufPackageInfo Filter(BufferList data, out int rest)
        {
            rest = 0;
            var buffStream = new BufferStream();
            buffStream.Initialize(data);

            var stream = CodedInputStream.CreateInstance(buffStream);
            var varint32 = (int) stream.ReadRawVarint32();
            if (varint32 <= 0) return default(ProtobufPackageInfo);

            var total = data.Total;
            var packageLen = varint32 + (int) stream.Position;

            if (total >= packageLen)
            {
                rest = total - packageLen;
                var body = stream.ReadRawBytes(varint32);
                var message = DefeatMessage.ParseFrom(body);
                var requestInfo = new ProtobufPackageInfo(message.Type, message);
                return requestInfo;
            }

            return default(ProtobufPackageInfo);
        }

####运行主程序实现
具体实现看：https://github.com/kotcmm/SuperSocket.ClientEngine.QuickStart/blob/master/ProtobufClient/Program.cs

这个真的没有什么好说了。运行效果如下：
![](http://images2015.cnblogs.com/blog/248834/201606/248834-20160604163400367-1194038961.png)
这里的打印信息是相对比较简单，大家可以自己下载源码来加些打印数据，让看起来更好看点。


## 三、java的Netty实现

既然前面提到了Netty，那就顺便实现一个简单的服务器来通讯看看。
使用的是Netty 4.x 参考资料：http://netty.io/wiki/user-guide-for-4.x.html

Netty的实现在网络上有超级多的例子，这里就简单的介绍一下就可以，首先先生成java的通讯包代码

>protoc --proto_path=pack --java_out=./ pack/DefeatMessage.proto

>protoc --proto_path=pack --java_out=./ pack/BackMessage.proto

>protoc --proto_path=pack --java_out=./ pack/CallMessage.proto

*这几个命令要自己灵活改变们不要死死的硬搬。*

生成的代码：https://github.com/kotcmm/SuperSocket.ClientEngine.QuickStart/tree/master/java/NettyProtobufServer/src/main/java


然后创建一个maven项目,添加Netty 和 protobuf 依赖：https://github.com/kotcmm/SuperSocket.ClientEngine.QuickStart/blob/master/java/NettyProtobufServer/pom.xml

    <dependencies>
            <dependency>
                <groupId>com.google.protobuf</groupId>
                <artifactId>protobuf-java</artifactId>
                <version>2.6.1</version>
            </dependency>
            <dependency>
                <groupId>io.netty</groupId>
                <artifactId>netty-microbench</artifactId>
                <version>4.1.0.Final</version>
            </dependency>
        </dependencies>

####ProtobufServerHandler的实现

因为Netty里面已经有帮我们实现了protobuf的解析，所以我们不需要自己实现。我们只要继承ChannelInboundHandlerAdapter然后通过channelRead就可以拿到解析好的对象，然后转换成我们自己的类型，就可以直接使用。这里同样是实现不同类型的消息打印和CallMessage消息就回复信息给客户端。

    import io.netty.channel.ChannelHandlerContext;
    import io.netty.channel.ChannelInboundHandlerAdapter;
    import io.netty.util.ReferenceCountUtil;

    /**
    * Created by caipeiyu on 16/6/4.
    */
    public class ProtobufServerHandler extends ChannelInboundHandlerAdapter {

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) { // (2)

            try {
                DefeatMessageOuterClass.DefeatMessage in = (DefeatMessageOuterClass.DefeatMessage) msg;
            if(in.getType() == DefeatMessageOuterClass.DefeatMessage.Type.BackMessage){
                System.out.print("BackMessage:");
                System.out.print(in.getBackMessage());
                System.out.flush();
            }else if(in.getType() == DefeatMessageOuterClass.DefeatMessage.Type.CallMessage){
                System.out.print("CallMessage:");
                System.out.print(in.getCallMessage());
                System.out.flush();

                DefeatMessageOuterClass.DefeatMessage out =
                        DefeatMessageOuterClass.DefeatMessage.newBuilder()
                                .setType(DefeatMessageOuterClass.DefeatMessage.Type.BackMessage)
                                .setBackMessage(BackMessageOuterClass.BackMessage
                                        .newBuilder().setContent("Hello I from server by Java Netty").build())
                                .build();

                ctx.write(out);
                ctx.flush();
            }

            } finally {
                ReferenceCountUtil.release(msg); // (2)
            }
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { // (4)
            // Close the connection when an exception is raised.
            cause.printStackTrace();
            ctx.close();
        }
    }


####ProtobufServer的实现

主要是添加已经有的编码解码和消息接收的类就可以了。

    import io.netty.bootstrap.ServerBootstrap;
    import io.netty.channel.*;
    import io.netty.channel.nio.NioEventLoopGroup;
    import io.netty.channel.socket.SocketChannel;
    import io.netty.channel.socket.nio.NioServerSocketChannel;
    import io.netty.handler.codec.protobuf.ProtobufDecoder;
    import io.netty.handler.codec.protobuf.ProtobufEncoder;
    import io.netty.handler.codec.protobuf.ProtobufVarint32FrameDecoder;
    import io.netty.handler.codec.protobuf.ProtobufVarint32LengthFieldPrepender;

    /**
    * Created by caipeiyu on 16/6/4.
    */
    public class ProtobufServer {
        public static void main(String[] args) {
            EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
            EventLoopGroup workerGroup = new NioEventLoopGroup();
            try {
                ServerBootstrap b = new ServerBootstrap(); // (2)
                b.group(bossGroup, workerGroup)
                        .channel(NioServerSocketChannel.class) // (3)
                        .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                            @Override
                            public void initChannel(SocketChannel ch) throws Exception {
                                ChannelPipeline p = ch.pipeline();
                                //解码
                                p.addLast("frameDecoder", new ProtobufVarint32FrameDecoder());
                                //构造函数传递要解码成的类型
                                p.addLast("protobufDecoder", new ProtobufDecoder(DefeatMessageOuterClass.DefeatMessage.getDefaultInstance()));
                                //编码
                                p.addLast("frameEncoder", new ProtobufVarint32LengthFieldPrepender());
                                p.addLast("protobufEncoder", new ProtobufEncoder());
                                //业务逻辑处理
                                p.addLast("handler", new ProtobufServerHandler());

                            }
                        })
                        .option(ChannelOption.SO_BACKLOG, 128)          // (5)
                        .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)

                // Bind and start to accept incoming connections.
                ChannelFuture f = b.bind(2012).sync(); // (7)

                // Wait until the server socket is closed.
                // In this example, this does not happen, but you can do that to gracefully
                // shut down your server.
                f.channel().closeFuture().sync();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                workerGroup.shutdownGracefully();
                bossGroup.shutdownGracefully();
            }

        }
    }
    
整个代码编写完成后，直接运行并打开我们前面的客户端进通讯发数据。运行结果如下：

![](http://images2015.cnblogs.com/blog/248834/201606/248834-20160604165912852-598846054.png)

    
当然这三个例子直接简单的说明如何使用框架来解决我们的问题，实际开发过程中肯定不只是我们例子的这么点东西，需要考虑的东西还很多，这里只是写一些可以运行起来的例子作为抛砖引玉。希望能给不懂的同学有点启发作用。谢谢您百忙中抽出时间来观看我的分享。

---------------------------
由于本人水平有限，知识有限，文章难免会有错误，欢迎大家指正。如果有什么问题也欢迎大家回复交流。要是你觉得本文还可以，那么点击一下推荐。
