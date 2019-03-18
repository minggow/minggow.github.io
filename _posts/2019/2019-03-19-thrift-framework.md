---
layout: post
title: Thrift框架概览
category: other
tags: [other]
no-post-nav: true
---


# 1 前言
我们对于Thrift或许不陌生了，在平时的开发中都会或多或少听到关于Thrift的博客或者文档。提到Thrift往往和“性能”、“跨语言”、“RPC”等一些关键字关联在一起，那到底Thrift到底是什么呢？也许我们很难用一句话说的清楚，但通过一些描述Thrift的名词，我们对Thrift有了初步的了解，但是Thrift到底是什么？它又是如何做到“高性能”、“跨语言跨平台”的呢？带着这些疑问，我们去了解Thrift的底层实现。
*** 
# 2 Thrift是什么
- 英文释义
节俭；节约
- 官网定义
> Thrift is a lightweight, language-independent software stack with an associated code generation mechanism for RPC. Thrift provides clean abstractions for data transport, data serialization, and application level processing. The code generation system takes a simple definition language as its input and generates code across programming languages that uses the abstracted stack to build interoperable RPC clients and servers.

**thrift 是一种轻量级软件技术栈，通过代码生成的原理来实现跨语言的支持。Thrift 提供了数据传输、数据序列化和应用层处理方面的简洁抽象。 代码生成系统提供了简单地语言定义作为输入，生成跨语言支持的代码, 这些代码使用抽象栈来构建服务器和客户端都可使用的。**

* Thrift特点
	- 高性能，相比其他优秀的RPC框架，其性能毫不逊色
	- 跨语言，支持20+语言，几乎涵盖所有主流语言
	- 轻量级，不需要多余的配置，几乎开箱即用

* 发展历程
	- 最初由facebook开发用作系统间各语言之间的RPC通信。 
	- 2007年由facebook贡献到apache基金，08年5约进入apache孵化器

# 3 Thrift协议层级结构
![Thrift架构图](/assets/images/2019/03/thrift01.png)

**Thrift框架实际上实现了C/S通信模型**

- 通过代码生成工具，生成客户端和服务端代码(可以为不同语言)，实现跨语言支持
- 生成的代码主要完成数据结构化解析、发送和接收，通过processor调用服务端处理逻辑
- TProtocal为协议层，主要实现各种格式的序列化协议，如二进制、JSON和压缩格式等
- TTransport为传输层，主要实现了阻塞IO和非阻塞IO的实现
- 底层IO传输，主要使用socket、http等一些传输协议
 
# 4 Thrift框架
**Thrift的核心组件, 主要包含以下几个方面**

- IDL服务描述组件,负责完成跨平台和跨语言(针对不同语言完成了Server层和Client代码的生成)
- TServer和Client，服务端和客户端组件的实现
- TProtocal 协议和解编码组件
- TTransport 传输组件
- TProcessor 服务调用组件，完成对服务实现的调用

## 4.1 Thrift Server
* Thrift Server的职责是将Thrift支持的各种特性结合起来。
	- 创建传输Transport并为Transport创建输入或输出TProtocal
	- 创建基于输入或输出的处理器processor(process调用服务端业务实现)
	- 等待连接建立并将数据交给处理器processor，处理完成返回client
* Thrift服务端的实现，目前主要有TSimpleServer、TNonblockingServer、THsHaServer、TThreadPoolServer、TThreadSelectorServer的实现，当前生产环境中主要使用的是TThreadPoolServer的实现。

### 4.1.1 TSimpleServer
![TSimpleServer工作流程](/assets/images/2019/03/thrift02.svg)

TSimpleServer的工作模式最简单地阻塞IO，一次只能接收和处理一个Socket连接，效率比较低，生产中并不会使用这种Server的实现

### 4.1.2 TNonblockingServer

非阻塞服务模式实现，对所有客户端的调用几乎是公平，该服务模式采用的是单线程工作，但采用的时NIO的实现方式。
![TNonBlockingServer工作流程](/assets/images/2019/03/thrift03.svg)

- 该工作模式效率提升主要体现在IO多路复用上, 采用nio同时监听多个socket的状态变化
- 仍然采用单线程顺序执行，在业务处理复杂和耗时的情况下，效率仍然是不高的

### 4.1.3 THsHaServer
半同步半异步模式，THsHaServer是TNonblockingServer的子类，因为TNonblockingServer仍然采用一个县城完成socket的监听和业务处理，效率相对较低。THsHaServer引入了线程池专门进行业务处理

![THsHaServer工作流程](/assets/images/2019/03/thrift04.svg)

- 主线程只读取数据，业务处理交给线程池完成处理，主线程效率大大提升
- 主线程仍然要对所有的socket监听和读取，当并发大和发送数据较多的情况下，监听的socket请求不能及时接受

### 4.1.4 TThreadPoolServer

TThreadPoolServer模式采用阻塞socket方式工作，主线程负责阻塞监听新socket，业务处理交给线程池处理

![TThreadPoolServer工作流程](/assets/images/2019/03/thrift05.svg)

- 线程池模式中，数据读取和业务处理都交给线程池处理，主线程只负责监听，因此在并发量较大情况下也能及时接受
- 线程池处理模式，比较适合服务端能够预知多少客户端并发的情况，这样每个请求都能够及时处理，性能也相对理想
- 线程池模式的处理能够受限于线程池的工作能力，在高并发情况下，新的请求只能够排队等待

### 4.1.5 TThreadSelectorServer
ThreadSelectorServer是目前Thrift提供的最高级的工作模式，其内部主要的工作流程如下

- 一个accept thread线程对象，专门用于处理监听socket新连接
- 若干个selector thread线程对象，专门用于处理业务socket上得IO，所有网络读写都由selector thread完成
- 一个负载均衡器(SelectorThreadLocadBalancer)，主要用于accept thread接收到新socket请求时，决定分配请求到selector thread
- ExecutorService工作线程池，用于业务处理，在selector thread 读取socket请求数据，交给业务线程池具体执行

![TThreadSelectorServer](/assets/images/2019/03/thrift06.svg)

- 专门的accept thread用于接收新socket请求，可以接受大量的请求
- socket请求经过负载均衡器分散到selector thread，可以应对io读写较大的情况
- executor工作线程池，具体执行业务逻辑，可以发挥服务端最大的工作能力

## 4.2 TTransport 
- TTransport传输层提供了和网络之间交互的读写抽象，这使得Thrift能够将底层传输和系统其他部分(例如序列化和反序列化)分离开来。
- Transport暴露的接口主要有open、close、read、write、flush等
- 除了Transport提供的上卖弄接口，Thrift提供了用于接收和创建原始对象的ServerTransport接口，主要用于服务端为传入的链接创建新的传输对象。open、listen、accept和close等
- 同时Thrift还提供了文件传输和HTTP传输等传输实现

### 4.2.1 客户端Transport实现
* 客户端的传输实现主要分为两类，阻塞传输实现和非阻塞传输实现
* 阻塞传输实现主要在TIOStreamTransport和TSocket中实现
	- TIOStreamTransport是最常用的传输层实现，它通过一个输入流和输出流实现了传输层的所有操作，其和Java的结构完美兼容(Java实现了各种IO流)
	- TSocket是通过Socket完成对Thrift传输实现，是客户端Socket连接实现和服务端传输的连接实现
* 阻塞传输相关类TNonblockingTransport(接口定义)和TNonblockingSocket(java nio中SocketChannel的包装实现)
* THttpClient是http的传输实现，主要用于服务端是HTTP服务，作为thrift的客户端的请求读取实现

### 4.2.2 服务端Transport实现
* TServerSocket是通过包装ServerSocket的传输实现，是一种阻塞的传输实现
* TNonblockserServerSocket是一种通过包装nio的ServerSocketChannel的实现，基础传输还是ServerSocket

### 4.2.3 缓存传输实现
* TMemoryInputTransport 封装了字节数组byte[]作为输入流的封装，从系统缓冲区读取数据，不支持写缓存。TMemoryBuffer则通过TByteArrayOutputStream作为输出流的封装，支持缓存读也支持往缓冲区写入数据。
* TFrameTransport是一种缓冲的Transport实现，它通过在每个消息前都有一个4个字节的帧消息来保证每次读取完整的消息
	- 封装TMemoryInputTransport作为输入流、TByteArrayOutputStream作为输出流,作为内存缓冲区的封装
	- TFrameTransport的flush执行时，会先写4byte的消息头，然后写入消息体
	- 在读取消息时，也会先读取4byte的长度，然后在读取消息体
* TFastFramedTransport是一种内存利用率更高的内存读写实现，它使用自动增长的`byte[](长度不够时才new)`，而不是每次都new一个byte[],从而提升了内存的使用率。其余实现和TFramedTransport一样，也会有消息头作为帧来记录消息的长度

### 4.2.4 其他传输实现介绍
* TFileTransport 文件传输实现，基于Event的异步实现
* TZlibTransport 基于zlib库的解压缩传输实现，通过压缩减少网络传输
* TSaslTransport 是基于Simple Authentication Security Layer的认证实现

### 4.2.5 传输层实现总结

* Thrift的传输层采用装饰器模式实现了包装IO流，可以通过包装流和节点流的概念区分各种Transport实现
* 节点流表示自身采用byte[]提供IO读写的实现，包装流表示封装类其他传输实现提供IO的读写
* 包装流主要是TFrame的传输实现，其实现是在写完消息flush时，回家上4byte的消息头，读消息的时候也会读取4byte的消息头
* Thrift协议和具体的传输对象绑定，协议使用具体的Transport来实现数据的读取

## 4.3 TProtocol
协议抽象定义了将内存数据映射到有线格式的机制。换句话说，协议规定了数据类型如何使用底层传输对自身进行编码/解码。因为，协议实现了管理编码方案并负责(反)序列化。这里指的序列化协议的例子包含JSON、XML、纯文本、紧凑二进制等。
Thrift实现的协议如下：

- 二进制，字段的长度和类型编码为字节码
- 压缩实现 [THRIFT-110](https://issues.apache.org/jira/browse/THRIFT-110)  
- JSON实现

### 4.3.1 TBinaryProtocol
是一种字节流读取的实现，String类型读取是通过nio实现，其余类型通过原生数据直接读取实现。核心代码如下：

```
  public ByteBuffer readBinary() throws TException {
    int size = readI32();

    checkStringReadLength(size);

    if (trans_.getBytesRemainingInBuffer() >= size) {
      ByteBuffer bb = ByteBuffer.wrap(trans_.getBuffer(), trans_.getBufferPosition(), size);
      trans_.consumeBuffer(size);
      return bb;
    }

    byte[] buf = new byte[size];
    trans_.readAll(buf, 0, size);
    return ByteBuffer.wrap(buf);
  }

  private void checkStringReadLength(int length) throws TProtocolException {
    if (length < 0) {
      throw new TProtocolException(TProtocolException.NEGATIVE_SIZE,
                                   "Negative length: " + length);
    }
    if (stringLengthLimit_ != NO_LENGTH_LIMIT && length > stringLengthLimit_) {
      throw new TProtocolException(TProtocolException.SIZE_LIMIT,
                                   "Length exceeded max allowed: " + length);
    }
  }
```

### 4.3.2 TCompactProtocol
TCompactProtocol协议作为TBinaryProtocol协议的升级强化版，都作为二进制编码传输方式，采用了一种乐器MIDI文件的编码方法。详细描述参见 [THRIFT-110](https://issues.apache.org/jira/browse/THRIFT-110)

1. ZigZag——有符号数编码

编码前 | 编码后
---|---
0 | 0 
-1 | 1
1 | 2
-2 | 3 
2 | 4 
-3 | 5

其效果等效于正数等于原先 * 2，负数变正数

32bits int =  (i << 1) ^ (i >> 31), 64bits long = (l << 1) ^ (l >> 63)

2. VLQ——编码压缩
A variable-length quantity (VLQ) 是一种通用编码，使用任意数量的二进制八位字节（8bit字节）来表示一个任意大的整数，其没定义为MIDI格式以节省空间资源。这种编码也被用于表示表式扩展音乐格式（XMF）中。即VLQ本质上就是用一个无符号的最大128来表示一个无符号的整数，并增加了一个第八位来表示字节是否继续。
即一字节的最高位（MHB）为标志位，不参与具体的内容，意思数值的大小仅仅有其它七位来表示。当最高位bit为1时，表示下一个byte也是该数值的内容（下一个byte的低七位bits）;当最高位bit为0时，下一个byte不参与其中。通过这样的方式，而不是int固定的4个bytes,long 8个bytes来讲，对于小数，能节约不少的空间大小；但凡事有利有弊，当数值比较大时，就要占用更多的空间，例如较大的int ,需要5bytes,较大的long需要10bytes.
编码假定八位位组（八位字节），其中最高有效位（MSB）（通常也称为符号位）被保留以指示是否有另一个VLQ八位组

VLQ 八位字节

| 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|---|---|
|2^7|2^6|2^5|2^4|2^3|2^2|2^1|2^0|
|A | Bn  |

如果A是0，那么这是整数的最后一个VLQ八位字节。如果A是1，则接下来是另一个VLQ字节。
B是7位数字[0x00,0x7F]，n是VLQ八位字节的位置，其中B 0是最不重要的。VLQ八位组在流中首先排列得最重要

**两种编码的结合** 

当VLQ编码遇到负数时，例如：long -1; 0XFFFFFFFFFFFFFFFF,就需要10bytes了，通过和ZigZag的结合，把负数转变相应的正数。当正数，负数的 |数值|较小时，都可以通过两者的结合，有效的压缩占用的空间大小。但同上，数值较大不可避免的占用比平常正常编码更多的空间。

 

106903转化为VLQ字节码例子

![VLQ字节码转换](/assets/images/2019/03/thrift07.png)

其他转换例子

Integer | Variable-length quantity
---| ---
0x00000000 | 0x00
0x0000007F | 0x7F
0x00000080 | 0x81 0x00
0x00002000 | 0xC0 0x00
0x00003FFF | 0xFF 0x7F
0x00004000 | 0x81 0x80 0x00
0x001FFFFF | 0xFF 0xFF 0x7F
0x00200000 | 0x81 0x80 0x80 0x00
0x08000000 | 0xC0 0x80 0x80 0x00
0x0FFFFFFF | 0xFF 0xFF 0xFF 0x7F



writeVarint32实现实现

```
private void writeVarint32(int n) throws TException { 
    int idx = 0; 
    while (true) { 
      if ((n & ~0x7F) == 0) { 
        temp[idx++] = (byte)n; 
        // writeByteDirect((byte)n); 
        break; 
        // return; 
      } else { 
        temp[idx++] = (byte)((n & 0x7F) | 0x80); 
        // writeByteDirect((byte)((n & 0x7F) | 0x80)); 
        n >>>= 7; 
      } 
    } 
    trans_.write(temp, 0, idx);
}
```

> [variable-length-quantiry](https://en.wikipedia.org/wiki/Variable-length_quantity)

### 4.3.3 TJSONProtocal实现

- TJSONProtocol 和 TSimpleJSONProtocol 两种实现。
- 实现比较简单，不再赘述。

## 4.4 Processor
- Processor封装了从输入流中读取数据并写入输出流的能力。
- 输入流和输出流由协议对象表示，处理结构非常接单

```
interface TProcessor {
    bool process(TProtocol in, TProtocol out) throws TException
}
```

- 服务响应的处理器由编译器生成的代码，并由服务端业务实现。
- 处理器实际上是从线路（通过协议输入流）读取数据，然后委托给处理程序(用户实现执行)
- 处理程序结果，通过线路（通过协议输出流），写入响应中，客户端得到结果

### 4.4.1 TBaseProcessor实现

```
public abstract class TBaseProcessor<I> implements TProcessor {
  private final I iface;
  private final Map<String,ProcessFunction<I, ? extends TBase>> processMap;

  protected TBaseProcessor(I iface, Map<String, ProcessFunction<I, ? extends TBase>> processFunctionMap) {
    this.iface = iface;
    this.processMap = processFunctionMap;
  }

  public Map<String,ProcessFunction<I, ? extends TBase>> getProcessMapView() {
    return Collections.unmodifiableMap(processMap);
  }

  @Override
  public boolean process(TProtocol in, TProtocol out) throws TException {
    TMessage msg = in.readMessageBegin();
    ProcessFunction fn = processMap.get(msg.name);
    if (fn == null) {
      TProtocolUtil.skip(in, TType.STRUCT);
      in.readMessageEnd();
      TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
      out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
      x.write(out);
      out.writeMessageEnd();
      out.getTransport().flush();
      return true;
    }
    fn.process(msg.seqid, in, out, iface);
    return true;
  }
}
```
### 4.4.2 TBaseAsyncProcessor

```
public class TBaseAsyncProcessor<I> implements TAsyncProcessor, TProcessor {
    protected final Logger LOGGER = LoggerFactory.getLogger(getClass().getName());

    final I iface;
    final Map<String,AsyncProcessFunction<I, ? extends TBase,?>> processMap;

    public TBaseAsyncProcessor(I iface, Map<String, AsyncProcessFunction<I, ? extends TBase,?>> processMap) {
        this.iface = iface;
        this.processMap = processMap;
    }

    public Map<String,AsyncProcessFunction<I, ? extends TBase,?>> getProcessMapView() {
        return Collections.unmodifiableMap(processMap);
    }

    public boolean process(final AsyncFrameBuffer fb) throws TException {

        final TProtocol in = fb.getInputProtocol();
        final TProtocol out = fb.getOutputProtocol();

        //Find processing function
        final TMessage msg = in.readMessageBegin();
        AsyncProcessFunction fn = processMap.get(msg.name);
        if (fn == null) {
            TProtocolUtil.skip(in, TType.STRUCT);
            in.readMessageEnd();
            if (!fn.isOneway()) {
              TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
              out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
              x.write(out);
              out.writeMessageEnd();
              out.getTransport().flush();
            }
            fb.responseReady();
            return true;
        }

        //Get Args
        TBase args = fn.getEmptyArgsInstance();

        try {
            args.read(in);
        } catch (TProtocolException e) {
            in.readMessageEnd();
            if (!fn.isOneway()) {
              TApplicationException x = new TApplicationException(TApplicationException.PROTOCOL_ERROR, e.getMessage());
              out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
              x.write(out);
              out.writeMessageEnd();
              out.getTransport().flush();
            }
            fb.responseReady();
            return true;
        }
        in.readMessageEnd();

        if (fn.isOneway()) {
          fb.responseReady();
        }

        //start off processing function
        AsyncMethodCallback resultHandler = fn.getResultHandler(fb, msg.seqid);
        try {
          fn.start(iface, args, resultHandler);
        } catch (Exception e) {
          resultHandler.onError(e);
        }
        return true;
    }

    @Override
    public boolean process(TProtocol in, TProtocol out) throws TException {
        return false;
    }
}

```
以上两种Processor的实现细节都在 FrameBuffer 和 AsyncFrameBuffer

* FrameBuffer是Thrift NIO服务器端的一个核心组件，它一方面承担了NIO编程中的缓冲区的功能，另一方面还承担了RPC方法调用的职责。
	- 实现了客户端和服务端交互的状态机
	- 管理读取帧的大小和帧数据，将其作为一个包装数据进行数据传递，然后将响应数据写会客户端
	- 在这个过程中，它管理为客户端管理翻动选中key区域数据的读写
* AsyncFrameBuffer是FrameBuffer的子类，主要功能和FrameBuffer，主要实现了异步的处理器的读写


# 5 Thrfit 服务过程解析

## 5.1 Server端

HelloServiceServer启动过程和客户端调用过程

![Server端调用过程](/assets/images/2019/03/thrift08.png)

**过程详解** 

1. 程序调用TheadPoolServer的serve方法后，server进入阻塞监听状态，阻塞在TServerSocket的accept方法上
2. 当接收到客户端的调用请求，服务端创建新线程处理请求，原线程再次进入阻塞状态
3. 新线程中同步TBinaryProtocol协议读取消息内容，调用HelloServerImpl的helloVoid方法，并将helloVoid_result中传回客户端

## 5.2 Client端

**HelloServiceClient调用过程和接收返回结果过程** 

![Client端调用过程](/assets/images/2019/03/thrift09.png)

1. 程序调用Hello.Client的helloVoid方法
2. 在helloVoid中通过send_helloVoid发送对服务端请求，通过recv_helloVoid方法接收对服务请求后返回的结果
3. 

# 6 Thrift性能对比
- 对比对象 Thrift 二进制协议和压缩协议、谷歌ProtoBuf、RMI、REST-JSON和REST-XML
- 对比维度，传输数据大小、运行时间和CPU负载

## 6.1 传输大小对比

### 6.1.1 对比结果

![Thrift-Size-Comparing](/assets/images/2019/03/thrift10.png)

### 6.1.2 结果汇总

Method	| Size*	| % Larger than TCompactProtocol
---|---|---
Thrift — TCompactProtocol |	278 |	N/A
Thrift — TBinaryProtocol |	460 |	65.47%
Protocol Buffers** |	250 |	-10.07%
RMI (using Object Serialization for estimate)**	| 905 | 225.54%
REST — JSON |	559 |	101.08%
REST — XML |	836 |	200.72%

### 6.1.3 结论 
与RMI和基于XML的REST相比，Thrift在其有效负载的大小方面具有明显的优势。来自Google的protobuf协议实际上是相同的，因为协议缓冲区数量不包含消息传递开销。

### 6.2.1 运行对比

![Thrift-Runtime_Comparing](/assets/images/2019/03/thrift11.png)

### 6.2.2 结果汇总

Server | CPU %	| Avg Client CPU % | 	Avg Wall Time
---|---|---|---
REST — XML |	12.00%	| 80.75% |	05:27.45
REST — JSON |	20.00%	| 75.00% |	04:44.83
RMI |	16.00%	| 46.50%	| 02:14.54
Protocol Buffers |	30.00% |	37.75% |	01:19.48
Thrift — TBinaryProtocol |	33.00% |	21.00% | 01:13.65
Thrift — TCompactProtocol |	30.00% |	22.50% | 01:05.12

* 平均时间不包括第一次运行，以考虑服务器预热。数字越小越好。
* WALL TIME 挂钟时间，实际运行时间，包括其他进行时间片占用和IO阻塞时间

### 6.2.3 结果分析

1. 在WALL-TIME上，REST和RMI的表现明显不如Thift，如TCompactProtocol花费不到20%的时间比REST-XML传输测试一样的数据。可见二进制相比文本传输具备更高的性能，所有尽管RMI有效负载比Thrift大很多，但也是明显优于基于JSON的REST服务的
2. 虽然Thrift和Protocol Buffers在服务端CPU占比最高，但REST客户端CPU占比最高。RPC框架会降负载平衡放在服务端和客户端，协议缓冲区在其中平衡其负载。服务端一些额外的CPU开销是编解码转化引起的。

## 6.3 对比结论

1. Thrift是一个强大的RPC框架库，用于创建可从多种语言中调用的高性能服务，比较适合跨语言的内部服务远程调用，在速度和性能上都非常优秀。
2. 如果是外部调用就会面临安全方面的考验，在防火墙一侧还需要面临端口打开过多的问题。

> [Thrift Features and Non-features](http://jnb.ociweb.com/jnb/jnbJun2009.html)

# 7 Thrift 未来展望
## 7.1 好的方面
- 一站式解决方案，相比protobuf，对于搭建thrift服务和调用，几乎开箱即用
- 特性丰富，相比其他的rpc。thrift一开始就支持了丰富的数据类型，几乎满足各种业务场景的需要
- 性能优异，可能不是性能最高的，但排名一定靠前的，基本不会成为系统瓶颈
- 很多开源项目都支持thrift，如hbase、hive等都有对外标准的thrift多语言支持
- 对于早期互联网公司，thrift是一个很好地开箱即用的解决方案，性能优异，也经过了时间的检验

## 7.2 坏的方面
- 基本没有官方文档，有些代码注释都不完整，相关的书籍也很少见到，总之缺乏权威的文档和学习资料
- 前期有一些坑，坑了一些用户。0.6 到 0.7 不兼容，坑了大批早期用户
- 开源社区不够活跃，bug fix也不是非常积极
- facebook 开源了表现更好的fbthrift，让我们怎么选？
