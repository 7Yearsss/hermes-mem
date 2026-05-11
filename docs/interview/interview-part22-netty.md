# Netty 网络编程面试题（第22期）

> 适用岗位：Java 后端 / 网络通信 / 高并发架构  
> 建议准备时长：每题 5~8 分钟，理解原理 + 动手调试

---

## 1. Netty 线程模型：Reactor 单线程 / 多线程 / 主从多线程

### 题目
描述 Netty 的线程模型演进过程，解释 Reactor 单线程模型、多线程模型、主从多线程模型的区别，以及 Netty 实际使用的是哪一种？

### 核心答案

Netty 源自 Reactor 模式，线程模型经历了三次演进：

**① Reactor 单线程模型**

```
accept → read → decode → compute → encode → send
        └─────────── one thread ───────────┘
```

所有连接管理、IO 读写、业务处理全部由**一个 NIO 线程**完成。优点是模型简单，缺点是一旦业务逻辑耗时长（比如复杂计算、DB 查询），整个线程被阻塞，后续连接全部等待。适用于极少数连接且业务极轻的场景。

**② Reactor 多线程模型**

```
bossGroup (1 thread)  →  accept
workerGroup (N threads) →  read/write/business
```

bossGroup 只负责 accept（通常 1 个 NIO 线程），workerGroup 持有多个 NIO 线程（默认 CPU 核数 × 2）处理 IO 读写和业务逻辑。但业务处理仍然在 workerGroup 的 EventLoop 中，若业务阻塞会拖累整个 EventLoop。

**③ 主从多线程模型（Netty 默认）**

```
bossGroup (N threads)  →  accept →  注册到 workerGroup
workerGroup (M threads) →  read/write/business
```

bossGroup 可以配置多个线程（通常 1~2 个）处理 accept，workerGroup 负责 IO 事件和管道流水线执行。主从分离使 acceptor 不会因为业务处理阻塞影响新连接建立。

Netty 实际默认使用的即**主从多线程模型**。

---

### 追问方向

- bossGroup 和 workerGroup 线程数量如何配置？有哪些调优参数？
- EventLoop 与线程的绑定关系是什么？一个 Channel 的一生由哪个线程管理？
- 如果业务逻辑中有阻塞调用（如 RPC、DB），该如何处理？

### 避坑提示

- ❌ 不要回答"Netty 用的是单线程模型"——Netty 从来不是单线程。
- ❌ 混淆 bossGroup 和 workerGroup 的职责：boss 只 accept，worker 处理 IO 和业务。
- ❌ 说"主从多线程就是多线程模型"——没有说清楚主（acceptor）和从（worker）的职责分工。

---

## 2. ChannelHandler 与 ChannelPipeline：入站出站处理器、流水线机制

### 题目
解释 ChannelHandler 的分类（入站 / 出站）、ChannelPipeline 的双向链表结构，以及 ChannelHandlerContext 在流水线中的作用。

### 核心答案

**入站处理器（ChannelInboundHandler）**：处理数据从外部流入 Channel 的事件链。

```
ChannelPipeline 入站方向（自上而下）：

HeadContext → LoggingHandler → MyDecoder → BusinessHandler → ... → TailContext
```

入站事件顺序：channelRegistered → channelActive → channelRead → channelReadComplete 等。

**出站处理器（ChannelOutboundHandler）**：处理数据从内部写出到外部的事件链。

```
ChannelPipeline 出站方向（自下而上）：

TailContext ← LoggingHandler ← MyEncoder ← BusinessHandler ← ... ← HeadContext
```

出站事件顺序：bind / connect / write / flush / close 等。

**ChannelPipeline 双向链表**：

每个 Channel 拥有一个 Pipeline，Pipeline 内部是 `ChannelHandlerContext` 构成的双向链表。HeadContext 关联 HeadHandler（Tcb 和 Wcb），TailContext 关联 TailHandler。添加处理器时可以指定在 Head、Tail 之前或之后插入。

**ChannelHandlerContext**：

每个 Handler 持有自己的 Context，Context 封装了下一个 Context 的引用（next / prev）。通过 `ctx.write(msg)` 时会自动向后传播（从当前节点向 tail），通过 `ctx.channel().write(msg)` 则从 tail 向 head 传播。

---

### 追问方向

- @Sharable 注解的作用是什么？不标注 @Sharable 会怎样？
- 业务 Handler 放在 Pipeline 的哪个位置最合理？编码器、解码器、业务处理器顺序如何安排？
- Handler 的 exceptionCaught 方法什么时候被调用？

### 避坑提示

- ❌ 混淆入站和出站的传播方向。
- ❌ 以为 Pipeline 中 Handler 顺序无所谓——顺序直接影响处理流程。
- ❌ 不理解 Context 的传播机制，答不出 `ctx.write()` 和 `channel().write()` 的区别。

---

## 3. ByteBuf：直接内存 vs 堆内存、池化技术（PooledArena）

### 题目
说明 ByteBuf 的两种内存模式（堆缓冲区 / 直接缓冲区）的区别，以及 Netty 池化内存管理的原理。

### 核心答案

**堆缓冲区（HeapByteBuf）**

```
byte[] array = byteBuf.array();
```

数据存储在 JVM 堆内存，分配/回收由 GC 管理，分配速度快。但数据通过 Socket 发送时，需要先拷贝到堆外内存（OS 级别 sendfile 之前），再多一次 GC 开销。

**直接缓冲区（DirectByteBuf）**

```
ByteBuf directBuf = Unpooled.directBuffer();
```

数据存储在堆外内存（OS 直接管理的内存），通过本地 IO 操作时无需中间拷贝，性能更高。但分配和回收成本较高（依赖 `ByteBuffer.allocateDirect()`）。

最佳实践：**IO 读写用 DirectByteBuf，业务处理用 HeapByteBuf**。Netty 的 `#readBytes()` 等操作会自动将 DirectBuf 转换为 HeapBuf。

**池化技术（PooledArena）**

Netty 4.x 引入 PooledArena（jemalloc 算法）实现 ByteBuf 池化：

- 按大小分为多个 SmallSubpageArea（< 512B）和 NormalPage（>= 512B）缓存区。
- 每个 EventLoop 线程绑定一个 PoolArena，避免跨线程竞争。
- 避免频繁 allocate/deallocate 导致的 GC 压力和内存碎片。
- 通过 `-Dio.netty.allocator.type=pooled` 开启（默认 pooled）。

**重要参数**：
- `MAX_CAPACITY`：最大容量限制（默认 Integer.MAX_VALUE）
- `writerIndex` / `readerIndex`：读写指针，分离式设计避免读写竞争

---

### 追问方向

- 如何选择？什么场景下用 HeapBuffer？
- PooledArena 的内存何时释放？引用计数归零时发生了什么？
- ByteBuf 的扩容机制是怎样的？动态扩容量如何计算？

### 避坑提示

- ❌ 说"直接内存更快所以全部用 DirectBuffer"——直接内存分配慢，小消息频繁分配反而更慢。
- ❌ 忽略 ByteBuf 的引用计数，导致内存泄漏。
- ❌ 说不清 Pooled 和 Unpooled 的区别。

---

## 4. 零拷贝：CompositeByteBuf / FileRegion

### 题目
Netty 为什么要实现零拷贝？它通过哪些手段实现？CompositeByteBuf 和 FileRegion 的原理是什么？

### 核心答案

传统网络 IO 的数据拷贝次数：
```
磁盘 → 内核缓冲区 → 用户缓冲区 → 内核缓冲区 → 网卡驱动
        ①           ②            ③            ④
```
共 **4 次拷贝**。

**Netty 零拷贝**（实际上是**近零拷贝**，消除不必要的 CPU 拷贝）：

**① 堆缓冲区避免中间拷贝**
使用 DirectByteBuf 时，本地 IO 可以直接使用，无需从堆内拷贝到堆外。

**② CompositeByteBuf**
将多个 ByteBuf 组合成逻辑上的单个缓冲区，无需将多个小 ByteBuf 合并成一个大缓冲区（避免一次 memcpy）。

```java
CompositeByteBuf composite = Unpooled.compositeBuffer();
composite.addComponents(true, headerBuf, contentBuf);
```

`true` 表示自动释放已添加组件。组合后仍然是一个逻辑 ByteBuf，但底层物理内存不必连续。

**③ FileRegion（Linux sendfile）**

```java
FileRegion region = new DefaultFileRegion(file.getChannel(), 0, file.length());
ctx.write(region);
```

FileRegion 将文件内容从 PageCache 直接发送到 Socket，绕过用户空间，全程只有 **2 次 DMA 拷贝**（磁盘→内核缓冲区→网卡），无 CPU 参与。Java NIO 的 `FileChannel.transferTo()` 即对应 Linux `sendfile(2)` 系统调用。

**④ 包装而非拷贝**
`Unpooled.wrappedBuffer(byte[])` 将 byte[] 包装为 ByteBuf，不发生内存拷贝。

---

### 追问方向

- 零拷贝是否意味着完全不需要 CPU 拷贝？
- FileRegion 的使用限制是什么？为什么有时需要自定义 ChunkedFileHandler？
- CompositeByteBuf 内存是否会被 GC 管理？

### 避坑提示

- ❌ 将"零拷贝"理解为"完全不需要任何拷贝"，忽视 DMA 拷贝。
- ❌ 混淆 FileRegion 和普通 ByteBuf 的使用方式。
- ❌ 在文件传输场景不使用 FileRegion/ChunkedStream 导致性能问题。

---

## 5. Netty 心跳：IdleStateHandler、读空闲 / 写空闲 / 全部空闲

### 题目
Netty 如何实现心跳检测？IdleStateHandler 的工作原理是什么？如何区分读空闲、写空闲和全部空闲？

### 核心答案

**IdleStateHandler** 是 Netty 提供的空闲状态检测处理器，位于 `io.netty.handler.timeout` 包。

```java
// 5秒没有收到消息 → 读空闲（READER_IDLE）
// 5秒没有发送消息 → 写空闲（WRITER_IDLE）
// 7秒既没收也没发 → 全部空闲（ALL_IDLE）
ch.pipeline().addLast(new IdleStateHandler(5, 5, 7, TimeUnit.SECONDS));
```

**三种空闲状态**：
- `READER_IDLE`：链路存活但长时间无数据接收，说明对方可能崩溃或网络中断
- `WRITER_IDLE`：链路存活但长时间无数据发送，说明本端发送方出现故障
- `ALL_IDLE`：双向均无数据交互，检测整个连接是否仍然活跃

**触发机制**：
IdleStateHandler 在每次事件循环（EventLoop）中检查当前时间与上一次读写事件的差值，超过阈值则触发对应 IdleStateEvent。

**处理方式**：
在业务 Handler 中覆写 `userEventTriggered()` 方法：

```java
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    if (evt instanceof IdleStateEvent) {
        IdleStateEvent idleEvent = (IdleStateEvent) evt;
        switch (idleEvent.state()) {
            case READER_IDLE:
                // 超过时间没收到消息，关闭连接或重连
                ctx.channel().close();
                break;
            case WRITER_IDLE:
                // 超过时间没发送消息，发送心跳
                ctx.writeAndFlush(Unpooled.copiedBuffer("HEARTBEAT".getBytes()));
                break;
            case ALL_IDLE:
                // 双方均无数据，关闭连接
                break;
        }
    }
    super.userEventTriggered(ctx, evt);
}
```

---

### 追问方向

- IdleStateHandler 的检测时间精度如何？是否会导致频繁定时任务？
- 如何在多路复用场景下实现每个连接独立的空闲检测？
- 心跳和重连如何配合使用？

### 避坑提示

- ❌ 将 IdleStateHandler 和 Handler timeout 混淆（如 ReadTimeoutHandler）。
- ❌ 不重写 `userEventTriggered` 方法或忘记调用 `super`，导致事件无法传播。
- ❌ 忽视 IdleStateHandler 默认触发的是抛出异常而非关闭连接。

---

## 6. Netty 编解码：ByteToMessageCodec、ReplayingDecoder

### 题目
Netty 编解码框架的核心接口是什么？ByteToMessageCodec 和 ReplayingDecoder 的区别和使用场景是什么？

### 核心答案

**编解码核心接口**

- 编码器：`MessageToByteEncoder<I>` — 将业务对象编码为字节
- 解码器：`ByteToMessageDecoder<I>` — 将字节流解码为业务对象
- 组合编解码：`ByteToMessageCodec<I>` — 同时包含编码和解码能力

**ByteToMessageDecoder（常规解码器）**

```java
public class MyDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        // 需要自己判断 in.readableBytes() 是否足够
        if (in.readableBytes() < 4) return; // 数据不足，等待更多数据
        int length = in.readInt();
        if (in.readableBytes() < length) {
            // 长度不够，需要回退 readerIndex（或等待下次）
            return;
        }
        byte[] body = new byte[length];
        in.readBytes(body);
        out.add(new MyMessage(body));
    }
}
```

需要**手动判断数据是否完整**，不完整时必须回退读取位置。

**ReplayingDecoder<S>（回退解码器）**

```java
public class MyReplayingDecoder extends ReplayingDecoder<Void> {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // in.readInt() 在数据不足时自动抛出 REPLAY 异常，Netty 会自动重试
        int length = in.readInt();
        byte[] body = new byte[length];
        in.readBytes(body);
        out.add(new MyMessage(body));
    }
}
```

`ReplayingDecoder` 内部包装了 `ByteBuf`，在读取字节数不足时自动抛出 `Error`（非异常）并暂停读取，直到数据完整后重新调用。**优点是代码简洁，缺点是有一定的性能开销**（异常捕获和回退逻辑）。

**ByteToMessageCodec**

```java
public class MyCodec extends ByteToMessageCodec<MyMessage> {
    @Override
    protected void encode(ChannelHandlerContext ctx, MyMessage msg, ByteBuf out) {
        // 编码逻辑
    }
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 解码逻辑
    }
}
```

同时处理编解码，适合协议对象需要双向转换的场景。

---

### 追问方向

- ReplayingDecoder 的 S（状态机）参数何时使用？
- ByteToMessageDecoder 的 `channelReadComplete` 中 `ctx.fireChannelReadComplete()` 和 `in.release()` 的关系？
- 如何避免 TCP 粘包半包导致的解码失败？

### 避坑提示

- ❌ 不释放 `ByteBuf` 导致内存泄漏（`in.release()` 或 `ReferenceCountUtil.release(msg)`）。
- ❌ ReplayingDecoder 中捕获了 `Exception` 而非 `Error`，导致状态机无法正常工作。
- ❌ 混淆 ReplayingDecoder 和 LengthFieldBasedFrameDecoder 的职责。

---

## 7. TCP 粘包半包：固定长度 / 分隔符 / 长度字段、LengthFieldBasedFrameDecoder

### 题目
什么是 TCP 粘包和半包？Netty 提供了哪些解决方案？LengthFieldBasedFrameDecoder 的配置参数有哪些？

### 核心答案

**TCP 粘包**：多个业务包被合并成一个 TCP 段发送（发送方 NSADA / 接收方 Nagle 算法优化）

**TCP 半包**：一个业务包被拆分成多个 TCP 段（MTU 限制、滑动窗口）

**三种解决策略**：

**① 固定长度（FixedLengthFrameDecoder）**

```java
ch.pipeline().addLast(new FixedLengthFrameDecoder(128)); // 每128字节一个完整消息
```

适用于定长协议，缺点是浪费带宽。

**② 分隔符（DelimiterBasedFrameDecoder）**

```java
ByteBuf delimiter = Unpooled.copiedBuffer("&&".getBytes());
ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, delimiter));
```

以特殊字符作为消息边界，适用于文本协议（如 HTTP 头的换行）。

**③ 长度字段（LengthFieldBasedFrameDecoder）** ⭐最常用

```java
ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(
    1024,        // maxFrameLength 单帧最大长度
    0,           // lengthFieldOffset 长度字段偏移量
    4,           // lengthFieldLength 长度字段本身字节数
    0,           // lengthAdjustment 长度调整值
    4,           // initialBytesToStrip 帧头剥离字节数
    true         // failFast 长度超限立即失败
));
```

**LengthFieldBasedFrameDecoder 参数详解**：

假设协议格式为：`[2字节长度][1字节类型][N字节数据]`

```java
LengthFieldBasedFrameDecoder(65535, 0, 2, 0, 2)
// offset=0：从头开始读2字节长度
// lengthFieldLength=2：长度字段占2字节
// lengthAdjustment=-2：读取长度后，需要向前偏移2字节（因为长度字段本身占2字节）
// initialBytesToStrip=2：剥除长度字段，不传给后续Handler
```

**典型场景**：
- `lengthFieldOffset=0, lengthFieldLength=4`：最常见，头4字节是长度
- `lengthFieldOffset=1`：跳过1字节（如版本号）再读长度
- `lengthAdjustment` 用于处理长度字段包含或不包含 Header 的情况

---

### 追问方向

- 粘包半包和 Nagle 算法有什么关系？如何关闭 Nagle？
- 如何处理变长消息且长度字段位于消息末尾的场景？
- lengthAdjustment 为负数的原理是什么？

### 避坑提示

- ❌ 不理解为什么 TCP 会粘包——因为 TCP 是流协议，不是消息协议。
- ❌ 在同一个 Pipeline 中叠加多个 FrameDecoder 导致解码混乱。
- ❌ 设置 maxFrameLength 过小导致正常大消息被截断。

---

## 8. Netty 编码器：ByteToMessageEncoder、MessageToByteEncoder

### 题目
Netty 编码器的继承体系是什么？MessageToByteEncoder 和 MessageToMessageEncoder 的区别？

### 核心答案

**编码器继承体系**

```
ChannelEncoder (interface)
  └── MessageToMessageEncoder<I>  → 对象→对象，如 Java Object → MyPOJO
  └── MessageToByteEncoder<I>      → 对象→字节，如 MyMessage → ByteBuf
```

**MessageToByteEncoder**

```java
// 将业务对象编码为 ByteBuf，通过 Socket 发送
public class MyEncoder extends MessageToByteEncoder<MyMessage> {
    @Override
    protected void encode(ChannelHandlerContext ctx, MyMessage msg, ByteBuf out) {
        out.writeInt(msg.getLength());
        out.writeBytes(msg.getBody());
    }
}
```

**MessageToMessageEncoder**

```java
// 将一种对象格式转换为另一种对象格式，不涉及字节操作
// 例如：内部对象 → JSON 字符串
public class JsonEncoder extends MessageToMessageEncoder<Object> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Object msg, List<Object> out) {
        String json = JsonUtils.toJson(msg);
        out.add(Unpooled.copiedBuffer(json.getBytes()));
    }
}
```

**编码器的传播机制**：

`ctx.write(msg)` 在编码器链中从 Head 向 Tail 传播，编码完成后最终到达 HeadContext 调用底层 Socket 发送。编码器不调用 `ctx.fireChannelRead()`，而是直接输出 ByteBuf。

**编码器的 write 传播**：

```java
ctx.write(msg)   // 从当前 Handler 向 Tail 传播
ctx.writeAndFlush(msg) // 写完立即刷新
```

---

### 追问方向

- 为什么编码器通常放在 Pipeline 前部（靠近 Head）？
- 编码完成后 ByteBuf 由谁负责释放？
- 如何避免同一个消息被编码多次（重复调用 encode）？

### 避坑提示

- ❌ 混淆编码器和解码器的方向。
- ❌ 在 encode 方法中忘记写长度字段或分隔符，导致对方解码失败。
- ❌ 编码后的 ByteBuf 未正确设置 readerIndex（从头写）和 writerIndex（数据结尾）。

---

## 9. Netty 解码器：ByteToMessageDecoder、io.netty.buffer.ByteBuf 类型

### 题目
Netty 解码器的核心抽象是什么？ByteToMessageDecoder 的 decode 方法如何与 ChannelInboundHandlerAdapter 配合工作？

### 核心答案

**解码器继承体系**

```
ByteToMessageDecoder (抽象类)
  └── ReplayingDecoder<S>          — 自动回退解码
  └── LengthFieldBasedFrameDecoder — 基于长度字段的帧解码
  └── DelimiterBasedFrameDecoder   — 基于分隔符的帧解码
  └── FixedLengthFrameDecoder      — 固定长度帧解码

ChannelInboundHandlerAdapter
  └── ByteToMessageDecoder         — 继承此类以自动加入入站链
```

**ByteToMessageDecoder 核心逻辑**：

```java
public abstract class ByteToMessageDecoder extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {
            // 将字节流传入 cumulation（累积缓冲区）
            ByteBuf in = (ByteBuf) msg;
            cumulation = cumulation == null ? in : Pooled.append(cumulation, in);
            callDecode(ctx, cumulation);
        } else {
            ctx.fireChannelRead(msg); // 非 ByteBuf 直接传播
        }
    }
}
```

解码器将收到的 ByteBuf **累积**到 `cumulation` 中（解决半包问题），然后调用子类的 `decode()` 方法尝试解码出完整消息。解码完成后**必须释放 cumulation 中已消费的部分**。

**cumulation 释放机制**：

在 `channelReadComplete` 中，Netty 会释放累积缓冲区的已读部分，`retain()` 未读部分。解码器不需要手动释放累积 ByteBuf（Netty 内部管理），但**如果 decode 中创建了新的 ByteBuf 分片**，需要负责释放。

**常用 ByteBuf 类型**：

| 类型 | 用途 |
|------|------|
| `PooledHeapByteBuf` | 池化堆内存 |
| `PooledDirectByteBuf` | 池化直接内存 |
| `UnpooledHeapByteBuf` | 非池化堆内存 |
| `UnpooledDirectByteBuf` | 非池化直接内存 |
| `CompositeByteBuf` | 组合缓冲区 |

---

### 追问方向

- cumulation 何时会被释放？什么情况下需要手动释放？
- `callDecode` 和 `decode` 的关系是什么？为什么需要分两次调用？
- 如何判断 cumulation 中的数据是否被完整消费？

### 避坑提示

- ❌ 在 decode 方法里忘记 `in.readBytes()` 消费数据，导致死循环（一直解码同一数据）。
- ❌ 将 decode 方法中的局部变量 ByteBuf 泄漏出去（被其他 Handler 持有），但解码器侧释放了它。
- ❌ 混淆 channelRead 和 decode 的调用时机。

---

## 10. Netty 引导：Bootstrap / ServerBootstrap、Channel 初始化

### 题目
描述 Bootstrap 和 ServerBootstrap 的区别，以及 Channel 初始化的完整流程。

### 核心答案

**客户端 Bootstrap**

```java
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(workerGroup)            // 只需一个 EventLoopGroup
         .channel(NioSocketChannel.class)
         .option(ChannelOption.TCP_NODELAY, true)
         .handler(new ChannelInitializer<NioSocketChannel>() {
             @Override
             protected void initChannel(NioSocketChannel ch) {
                 ch.pipeline().addLast(new MyHandler());
             }
         });
ChannelFuture f = bootstrap.connect("127.0.0.1", 8080).sync();
```

**服务端 ServerBootstrap**

```java
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(bossGroup, workerGroup)  // 两个 EventLoopGroup
               .channel(NioServerSocketChannel.class)
               .option(ChannelOption.SO_BACKLOG, 1024)        // ServerSocket 级别
               .childOption(ChannelOption.TCP_NODELAY, true)   // SocketChannel 级别
               .childHandler(new ChannelInitializer<NioServerSocketChannel>() {
                   @Override
                   protected void initChannel(NioServerSocketChannel ch) {
                       ch.pipeline().addLast(new MyHandler());
                   }
               });
ChannelFuture f = serverBootstrap.bind(8080).sync();
```

**关键区别**：

| 配置项 | Bootstrap | ServerBootstrap |
|--------|-----------|----------------|
| EventLoopGroup | 1 个（worker） | 2 个（boss + worker） |
| Channel 类型 | NioSocketChannel | NioServerSocketChannel |
| option | SocketChannel 选项 | ServerSocketChannel 选项 |
| childOption | 无 | accepted SocketChannel 选项 |

**Channel 初始化流程**：

```
bind() → initAndRegister() → new Channel() → init() → register()
                                    ↓
                         unsafe = channel.newUnsafe()
                         pipeline = channel.newPipeline()
                         fireChannelRegistered()  // 向 Pipeline 触发 channelRegistered 事件
```

**init(channel) 中做了什么**：

1. 设置 Options 和 Attributes
2. 将 config 中的 Handler 加入 Pipeline（通过 `pipeline.addLast(config.handler())`）
3. 执行用户自定义的 `initChannel`（一次性调用）

---

### 追问方向

- bossGroup 和 workerGroup 的线程数如何配置？各承担什么角色？
- `sync()` 和 `addListener()` 的区别是什么？
- Channel 的 attribute 和 option 的区别是什么？

### 避坑提示

- ❌ 混淆 Bootstrap 和 ServerBootstrap 的 API（如在客户端使用 `childOption`）。
- ❌ 服务端忘记调用 `bind()` 而直接调用 `connect()`。
- ❌ 不理解 `sync()` 会阻塞当前线程，导致死锁（如果在 EventLoop 中调用 sync）。

---

## 11. Netty Future / Promise：ChannelFuture、sync / await、addListener

### 题目
Netty 的 Future 和 Java 原生 Future 有何不同？ChannelFuture 的同步（sync/await）和异步（addListener）用法是什么？

### 核心答案

**Java 原生 Future vs Netty Future vs Netty Promise**

| 特性 | Java Future | Netty Future | Netty Promise |
|------|------------|--------------|--------------|
| 阻塞等待 | `get()` 阻塞 | `sync()/await()` 阻塞 | `sync()/await()` 阻塞 |
| 异步回调 | 不支持 | `addListener()` | `addListener()` |
| 主动完成 | 不支持 | 只读 | 可写（setSuccess） |
| 取消 | `cancel()` | 不支持 | 不支持 |

**Netty ChannelFuture**

ChannelFuture 表示一个异步 IO 操作的未来结果：

```java
Channel ch = ...;
ChannelFuture future = ch.close();  // 非阻塞，立即返回
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) {
        System.out.println("连接已关闭, success=" + future.isSuccess());
    }
});
```

**sync() vs await()**

- `sync()`：阻塞等待结果，**会抛出异常**，且会**打断当前线程的中断状态**
- `await()`：阻塞等待结果，**不抛出异常**，只是简单的等待

实战中通常用 `sync()`，因为连接失败时需要感知异常。

```java
// 错误用法
bootstrap.connect(host, port).sync(); // OK

// 在 EventLoop 线程中 sync 会死锁
// 因为 sync 会阻塞当前线程等待 EventLoop，而 EventLoop 本身需要处理这个任务
// 应该用 await 或 addListener
```

**addListener 异步方式**（推荐在 EventLoop 中使用）：

```java
bootstrap.connect(host, port).addListener((ChannelFuture future) -> {
    if (future.isSuccess()) {
        System.out.println("连接成功");
    } else {
        System.err.println("连接失败: " + future.cause());
    }
});
```

---

### 追问方向

- 为什么在 EventLoop 线程中不能调用 sync/await？
- GenericFutureListener 的调用线程是什么？IO 完成线程还是用户线程？
- ChannelFuture 和 FutureTask 的关系是什么？

### 避坑提示

- ❌ 在 EventLoop 线程中调用 sync() 导致死锁（Netty 最常见的面试坑之一）。
- ❌ 不释放 ChannelFuture 或忘记添加 listener 导致回调丢失。
- ❌ 混淆 sync() 和 await() 的区别，在异常处理场景误用 await()。

---

## 12. Netty ByteBuf 引用计数：retain / release、ReferenceCounted

### 题目
Netty 4 引入的引用计数机制是什么？ByteBuf 的 retain 和 release 如何配对使用？ReferenceCounted 接口有哪些方法？

### 核心答案

Netty 4.x 引入了**引用计数**来精确管理 ByteBuf 的生命周期，避免提前释放或泄漏。

**核心接口**

```java
public interface ReferenceCounted {
    int refCnt();           // 当前引用计数
    ReferenceCounted retain();         // 引用计数 +1
    ReferenceCounted retain(int increment); // 引用计数 +increment
    ReferenceCounted touch(Object hint);   // 记录最后访问位置（用于调试泄漏）
    boolean release();                 // 引用计数 -1，归零时释放内存
    boolean release(int decrement);    // 引用计数 -decrement
}
```

**引用计数规则**

- 新创建的 ByteBuf：`refCnt() = 1`
- 调用 `retain()`：`refCnt++`
- 调用 `release()`：`refCnt--`，归零时立即释放内存
- **谁 retain 了，谁必须 release**

**ByteBuf 传递中的引用计数**

```
业务Handler读取 → retain() → 传递给下一个Handler → 由下一个Handler负责release
```

示例：

```java
// Handler A
ByteBuf buf = ...;
ctx.write(buf);        // 引用计数不变，写入 Pipeline
// 注意：write 不会改变 refCnt，release 仍然是 Handler A 的责任

// 正确做法
ByteBuf buf = ...;
ctx.write(buf);        // 发送给下一个 Handler
// buf 仍然被 A 持有

// 或者
ByteBuf buf = ...;
ctx.write(buf.retain());  // retain 后传递出去，自己不再持有
```

**release 的位置**

通常在以下位置 release：
- `channelRead()` 方法最后（如果消息没有传递下去）
- `exceptionCaught()` 中（出现异常时立即释放）
- `handlerRemoved()` 中（Handler 从 Pipeline 移除时）

**release 的常见陷阱**：

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        // ❌ 直接传递但没有 release（如果后续无人 release 则泄漏）
        ctx.fireChannelRead(buf);  // 这里会触发下一个 Handler 的 channelRead
        
        // ✓ 正确做法：传递时 retain，让下一个 Handler 负责 release
        ctx.fireChannelRead(buf.retain());
        
        // ✓ 或者自己负责 release
        try {
            // process buf
        } finally {
            buf.release();
        }
    }
}
```

---

### 追问方向

- release 归零后再次调用 release 会发生什么？
- compositeByteBuf 和 slice() 返回的 ByteBuf 如何管理引用计数？
- 泄漏检测级别有哪些？如何定位泄漏？

### 避坑提示

- ❌ 混淆 retain 和 release 的配对关系，导致计数错误。
- ❌ 在 fireChannelRead 时传递了 ByteBuf 但没有 retain，导致提前释放。
- ❌ ByteBuf 经过编码器后认为已经释放，但实际上编码器可能 retain 了引用。

---

## 13. Netty 编解码器链：@Sharable、ChannelHandlerContext 传递

### 题目
@Sharable 注解的作用是什么？什么样的 Handler 可以标注 @Sharable？ChannelHandlerContext 在编解码链中如何传递？

### 核心答案

**@Sharable 注解**

```java
@Sharable
public class MyDecoder extends ByteToMessageDecoder { }
```

标注 @Sharable 表示此 Handler **实例可以被添加到多个 Pipeline 中**（多个 Channel 共享同一个 Handler 实例）。未标注时，尝试重复添加到 Pipeline 会抛出 `IllegalStateException: Already added to a pipeline`。

**@Sharable 使用条件**：

满足以下任一条件才能标注 @Sharable：
1. Handler 是**无状态**的（不持有实例字段，或字段是线程安全的，如 AtomicLong）
2. Handler 不需要每次 channelRead 创建新实例
3. Handler 的实例字段不会因为并发访问导致数据错乱

常见 @Sharable 场景：
- `LoggingHandler`（无状态，单纯打印）
- `SslHandler`（内部已有线程安全机制）

非 @Sharable 场景：
- 持有 `ByteBuf` 累积缓冲区的解码器
- 持有用户 session 状态的对象

**ChannelHandlerContext 传递机制**

Pipeline 中事件传播依赖 ChannelHandlerContext：

```java
// 入站：从 Head 向 Tail
ctx.fireChannelRead(msg);  // 调用 ctx 的下一个 InboundHandler

// 出站：从 Tail 向 Head
ctx.write(msg);             // 从当前 Handler 向 Head 传播
```

关键理解：
- `ChannelHandlerContext` 是 Handler 与 Pipeline 之间的桥梁
- 传递时 current handler → ctx.fireChannelRead → next handler（当前 Handler 不会再次收到）
- 如果要在当前 Handler 重新处理，需要调用 `ctx.fireChannelRead(msg)` 之外的方法（如 `invokeChannelRead`）

**@Sharable 与 Pipeline 改造**

```java
// 单例 Handler（无状态）—— 可以 @Sharable
@Sharable
public class LoggingHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println(msg);
        ctx.fireChannelRead(msg); // 必须传递，否则流水线中断
    }
}

// 多实例 Handler（有状态）—— 不能 @Sharable
public class MyStatefulDecoder extends ByteToMessageDecoder {
    private int count = 0; // 实例状态，不能共享
}
```

---

### 追问方向

- 为什么 SslHandler 可以是 @Sharable 而普通解码器不行？
- 不标注 @Sharable 但同一个实例添加到两个 Channel 会发生什么？
- `ctx.fireChannelRead()` 和 `super.channelRead()` 的区别是什么？

### 避坑提示

- ❌ 在有状态 Handler 上标注 @Sharable 导致并发问题。
- ❌ 在 channelRead 中调用 `ctx.fireChannelRead()` 后又 `super.channelRead()` 导致消息重复处理。
- ❌ 以为 @Sharable 只是性能优化——它首先是线程安全承诺。

---

## 14. Netty 内存泄漏：ResourceLeakDetector、四级泄漏级别

### 题目
Netty 的内存泄漏是如何检测的？ResourceLeakDetector 的四级检测级别分别是什么？如何在生产环境排查泄漏？

### 核心答案

**ResourceLeakDetector** 是 Netty 内置的泄漏检测器，基于 JDK 的 `PhantomReference`（虚引用）实现。

**检测原理**：

每当创建 ByteBuf 时，如果开启了泄漏检测，Netty 会创建一个与 ByteBuf 关联的 `PhantomReference`，并注册到 `ResourceLeakDetector` 的引用队列中。如果 ByteBuf 被 GC 回收但引用队列中没有任何 release 调用记录，说明发生了泄漏。

**四级检测级别**

通过 `-Dio.netty.leakDetection.level` 设置：

| 级别 | 行为 | 性能影响 | 适用场景 |
|------|------|---------|---------|
| `DISABLED` | 完全关闭 | 无 | 生产环境（极端性能敏感） |
| `SIMPLE` | 抽样 1% ByteBuf，报告首次泄漏 | ~5% | 生产环境（默认） |
| `ADVANCED` | 抽样 1% ByteBuf，报告泄漏点和引用链 | ~10%~25% | 调试 / 测试环境 |
| `PARANOID` | 检测 **每一个** ByteBuf，报告引用链 | 极高（每次分配均有开销） | 本地开发调试 |

**生产环境推荐**：`SIMPLE`，泄漏率 < 1% 时几乎无感知。

**泄漏报告示例**

```
LEAK: ByteBuf.release() was not called before it's garbage-collected.
Recent access records: 
  #0
      Hint: 'channelRead0' will release the ByteBuf referenced by the object at this access.
      at MyHandler.channelRead0(MyHandler.java:18)
```

**如何防止泄漏**

```java
// 1. 总是使用 try-finally
ByteBuf buf = ctx.alloc().buffer();
try {
    // use buf
} finally {
    buf.release();
}

// 2. 使用 ReferenceCountUtil 自动释放
ReferenceCountUtil.releaseLater(msg);

// 3. 在 exceptionCaught 中释放
@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    ctx.close();
}
```

---

### 追问方向

- 为什么泄漏在开发环境难以复现？
- 如何在测试用例中验证无内存泄漏？
- 泄漏检测级别是否可以通过代码动态切换？

### 避坑提示

- ❌ 生产环境开启 PARANOID 级别——性能会严重下降。
- ❌ 以为关闭泄漏检测（DISABLED）就不会泄漏——它只是不检测，不是没有泄漏。
- ❌ 在 fireChannelRead 之前没有 retain 就 release 了 ByteBuf。

---

## 15. Netty 连接管理：connect / disconnect / close、Channel.isActive()

### 题目
Netty 中 Channel 的 connect、disconnect、close 有什么区别？Channel.isActive() 在不同状态下返回什么？

### 核心答案

**三种操作的区别**

| 操作 | 方法 | 触发事件 | 底层行为 |
|------|------|---------|---------|
| 连接 | `Channel.connect()` | `fireChannelActive` | 发起 TCP 连接（三次握手） |
| 断开 | `Channel.disconnect()` | `fireChannelInactive` | 发送 FIN 包（优雅关闭） |
| 关闭 | `Channel.close()` | `fireChannelInactive` + `fireChannelUnregistered` | 彻底关闭 Socket |

**disconnect vs close**：

- `disconnect()`：尝试优雅关闭（SO_LINGER=0 设为后可能直接 RST），触发 `channelInactive`，但 Channel 仍可重新 connect。底层调用 `Java NIO SocketChannel.disconnect()`。
- `close()`：强制关闭，释放资源，Channel 无法再使用。底层调用 `Java NIO SocketChannel.close()`。

**Channel.isActive() 的状态返回值**

```java
NioSocketChannel.isActive():
  - 已连接（TCP 连接建立） → true
  - 未连接                 → false
  - 连接中（ConnectListener 等待中）→ false

NioServerSocketChannel.isActive():
  - 已绑定端口              → true
  - 未绑定                  → false
```

**完整的 Channel 生命周期**

```
channelRegistered() → channelActive() → (数据交互) → channelInactive() → channelUnregistered()
```

**最佳实践**

```java
// 检测连接是否存活
if (ch.isActive()) {
    ch.writeAndFlush(msg);
}

// 主动关闭并等待完成
ch.close().sync();

// 注册关闭监听器
ch.closeFuture().addListener((ChannelFutureListener) future -> {
    // 清理资源
});
```

---

### 追问方向

- 主动关闭连接时，closeFuture 和 sync 的关系是什么？
- 如何区分对端强制关闭（收到 RST）和优雅关闭（收到 FIN）？
- Channel 的 closeFuture 和 ChildChannel 的 closeFuture 有什么区别？

### 避坑提示

- ❌ 混淆 disconnect 和 close 的语义，导致连接无法正确释放。
- ❌ 在 channelInactive 后继续使用 Channel 写数据（此时 Channel 已不活跃）。
- ❌ 不注册 closeFuture 监听器导致资源清理不及时。

---

## 16. Netty 与 Tomcat 对比：线程模型、性能差异、适用场景

### 题目
Netty 和 Tomcat 的核心区别是什么？各自的线程模型和适用场景有何不同？

### 核心答案

**架构定位对比**

| 维度 | Tomcat | Netty |
|------|--------|-------|
| 定位 | Web Servlet 容器 | 底层网络通信框架 |
| 协议层 | HTTP/WebSocket 等应用层 | TCP/UDP/自定义协议 |
| 线程模型 |BIO(旧) / NIO(8.x+) / APR| Reactor 多线程 |
| 生态 | Servlet API、JSP、Spring MVC | 需要自行处理编解码、业务逻辑 |
| 上手难度 | 低（开箱即用） | 高（需要理解 Pipeline、ByteBuf） |

**Tomcat 线程模型（Tomcat 8+ NIO）**

```
Acceptor(N) ──→ Poller(M) ──→ RequestHandler(Worker) ──→ Servlet Thread
   (accept)     (selector)     (处理请求)               (业务逻辑)
```

Tomcat 的 Worker 线程池处理 Servlet 业务逻辑，Connector 和 ProtocolHandler 管理 IO 线程。

**Netty 线程模型**

```
BossGroup ──→ accept ──→ WorkerGroup (EventLoop) ──→ Pipeline ──→ Handler
 (线程N)      (多线程)    (线程数=CPU*2)              (单线程绑定)   (业务逻辑)
```

一个 EventLoop 绑定一个线程，该 EventLoop 上所有 Channel 的 Pipeline 都在同一个线程执行。

**性能对比**

| 场景 | Netty | Tomcat NIO |
|------|-------|-----------|
| 长连接（TCP/WebSocket） | ✅ 极优 | ⚠️ 可用但不推荐 |
| 短连接 HTTP | ⚠️ 需要自行处理协议 | ✅ 极优 |
| 高并发（百万连接） | ✅ 天生支持 | ⚠️ 受 Servlet 规范限制 |
| 自定义协议 | ✅ 灵活 | ❌ 不支持 |
| 开发效率 | ❌ 需要大量 boilerplate | ✅ 极快 |

**适用场景**

- **Netty**：游戏服务器、物联网网关、推送服务、RPC 框架（如 Dubbo）、金融协议通信、音视频传输
- **Tomcat**：传统 Web 应用、REST API、Spring MVC、Servlet 标准业务

---

### 追问方向

- Dubbo 使用 Netty 作为通信层的原因是什么？
- 为什么 Tomcat 不适合长连接推送场景？
- gRPC 和 Netty 的关系是什么？

### 避坑提示

- ❌ 将 Tomcat 和 Netty 放在同一层比较——它们解决的是不同层次的问题。
- ❌ 认为 Netty 可以完全替代 Tomcat——HTTP 场景下 Tomcat 更完善。
- ❌ 忽视 Netty 的学习曲线，在简单 Web 场景使用 Netty。

---

## 17. Netty 心跳检测：IdleStateHandler、EventExecutorGroup

### 题目
Netty 的 IdleStateHandler 是如何与 EventExecutorGroup 结合实现定时心跳检测的？EventExecutorGroup 的原理是什么？

### 核心答案

**EventExecutorGroup** 是 Netty 提供的"可在指定时间后触发回调"的执行器接口，常用于实现定时任务：

```java
// 每 30 秒执行一次
ScheduledFuture<?> future = eventExec.schedule(() -> {
    System.out.println("定时任务");
}, 30, TimeUnit.SECONDS);
```

**IdleStateHandler 内部原理**

```java
// IdleStateHandler 构造函数
public IdleStateHandler(
    long readerIdleTime,      // 读空闲时间
    long writerIdleTime,      // 写空闲时间  
    long allIdleTime,         // 全部空闲时间
    TimeUnit unit
) {
    // 内部创建一个 ScheduledFuture 绑定到 EventLoop 的 EventExecutorGroup
    // 在 schedule 时会检查当前时间与上次事件的差值
}
```

IdleStateHandler 在每次 IO 事件（read/write）时记录时间戳，EventLoop 的定时任务定期检查这些时间戳差值，超过阈值则触发 IdleStateEvent。

**EventExecutorGroup 与 EventLoop 的关系**

```
EventLoop extends EventExecutorGroup
             extends ScheduledExecutorService
```

EventLoop 本身就实现了 EventExecutorGroup，所以 IdleStateHandler 直接使用 Channel 所在 EventLoop 的定时器，无需额外线程池。

**IdleStateHandler 的定时检测机制**

```java
// 内部使用 SingleThreadEventExecutor.schedule()
ScheduledFuture<?> scheduledFuture = eventLoop.schedule(
    new Runnable() { 
        public void run() {
            // 检查 readerIdleTime / writerIdleTime / allIdleTime
            // 超过阈值则 fireUserEventTriggered(new IdleStateEvent(...))
        }
    },
    5, TimeUnit.SECONDS  // 每5秒检查一次（不是空闲超时值）
);
```

注意：`readerIdleTime=5` 不代表每 5 秒检测一次，而是**每 N 秒检测一次**（N 通常等于所有 IdleTime 的最小值），检测到空闲超过阈值才触发。

---

### 追问方向

- IdleStateHandler 的定时任务在 EventLoop 中执行，会不会影响 IO 性能？
- 如何同时支持多个 Channel 的独立空闲检测而不相互干扰？
- 如果 EventLoop 线程被业务阻塞，IdleStateHandler 的定时还能正常触发吗？

### 避坑提示

- ❌ 以为 IdleStateHandler 在独立线程运行——它依赖 EventLoop 的定时调度。
- ❌ EventLoop 被阻塞时，IdleStateHandler 的检测也会延迟。
- ❌ 不理解为什么读写空闲触发时间可能不精确。

---

## 18. Netty HTTP 协议开发：HttpServerCodec / HttpObjectAggregator / FullHttpRequest

### 题目
在 Netty 中开发 HTTP 服务端需要哪些编解码器？FullHttpRequest 和 FullHttpResponse 的使用方法是什么？

### 核心答案

**HTTP 服务端常用编解码器组合**

```java
public class HttpPipelineFactory extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) {
        ch.pipeline()
            // HTTP 请求解码 / 响应编码
            .addLast("codec", new HttpServerCodec())
            // 将多个 HttpObject 聚合成完整的 FullHttpRequest/FullHttpResponse
            // 解决 HTTP 分块传输（chunked）问题
            .addLast("aggregator", new HttpObjectAggregator(65536))
            // 业务 Handler
            .addLast("handler", new HttpRequestHandler());
    }
}
```

**HttpServerCodec** = HttpRequestDecoder + HttpResponseEncoder

```java
// 同时处理 HTTP 请求和响应
// - HttpRequestDecoder: ByteBuf → HttpRequest
// - HttpResponseEncoder: HttpResponse → ByteBuf
```

**HttpObjectAggregator** 将分段消息聚合：

```
Chunk 1: HttpContent
Chunk 2: HttpContent
Chunk 3: LastHttpContent
        ↓ HttpObjectAggregator
FullHttpRequest (包含完整 body)
```

**FullHttpRequest 处理示例**

```java
public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) {
        // 获取请求信息
        String uri = request.uri();
        HttpMethod method = request.method();
        HttpHeaders headers = request.headers();
        
        // 读取 body
        ByteBuf content = request.content();
        byte[] body = new byte[content.readableBytes()];
        content.readBytes(body);
        
        // 构造响应
        FullHttpResponse response = new DefaultFullHttpResponse(
            HttpVersion.HTTP_1_1,
            HttpResponseStatus.OK,
            Unpooled.copiedBuffer("Hello".getBytes())
        );
        response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain");
        response.headers().set(HttpHeaderNames.CONTENT_LENGTH, response.content().readableBytes());
        
        ctx.writeAndFlush(response);
    }
}
```

**Pipeline 配置顺序（重要）**

```
HttpServerCodec → HttpObjectAggregator → BusinessHandler
     (解码)         (聚合)                (业务)
```

如果 HttpObjectAggregator 在 HttpServerCodec 之前，会导致 ByteBuf 未被解码而报错。

---

### 追问方向

- 为什么需要 HttpObjectAggregator？什么场景下不需要？
- HTTP/1.1 Keep-Alive 和 pipeline HTTP 的区别是什么？
- 如何处理 WebSocket 升级请求（Upgrade 头）？

### 避坑提示

- ❌ 忘记设置 Content-Length 或使用 chunked 编码导致客户端无法判断响应结束。
- ❌ FullHttpResponse 使用完毕后未 release（它也实现了 ReferenceCounted）。
- ❌ 在同一个 Pipeline 中同时使用 FullHttpRequest 和 HttpRequest（分不清是否已聚合）。

---

## 19. Netty WebSocket 开发：握手、帧类型、Keep-Alive

### 题目
描述 Netty WebSocket 开发的完整流程：握手、帧类型、双向通信、Keep-Alive 如何实现？

### 核心答案

**WebSocket 完整开发流程**

**① 服务端 Pipeline 配置**

```java
ch.pipeline()
    .addLast(new HttpServerCodec())
    .addLast(new HttpObjectAggregator(65536))
    .addLast(new WebSocketServerProtocolHandler("/ws", null, true))  // 启用 Ping/Pong
    .addLast(new WebSocketFrameHandler());
```

**② 握手（HTTP → WebSocket）**

WebSocketServerProtocolHandler 自动处理：
1. 客户端发送 HTTP Upgrade 请求（包含 `Connection: Upgrade` 和 `Sec-WebSocket-Key`）
2. 服务端返回 `101 Switching Protocols`，包含 `Sec-WebSocket-Accept`
3. 握手完成后，Pipeline 切换到 WebSocket 模式，HTTP Codec 从 Pipeline 移除

```java
// WebSocketServerProtocolHandler 构造函数参数
WebSocketServerProtocolHandler(String websocketPath, String subprotocols, boolean allowExtensions)
// allowExtensions=true 允许压缩等扩展
// 握手完成后，会自动触发 WebSocketHandshakeComplete 事件
```

**③ WebSocket 帧类型**

Netty 定义的帧类型（继承 `WebSocketFrame`）：

| 帧类型 | 类 | 用途 |
|--------|-----|------|
| Binary | `BinaryWebSocketFrame` | 二进制数据 |
| Text | `TextWebSocketFrame` | 文本数据 |
| Continuation | `ContinuationWebSocketFrame` | 分片数据 |
| Ping | `PingWebSocketFrame` | 心跳请求（自动回复 Pong） |
| Pong | `PongWebSocketFrame` | 心跳响应 |
| Close | `CloseWebSocketFrame` | 关闭握手 |

**④ 业务 Handler 示例**

```java
public class WebSocketFrameHandler extends SimpleChannelInboundHandler<WebSocketFrame> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, WebSocketFrame frame) {
        if (frame instanceof TextWebSocketFrame) {
            String text = ((TextWebSocketFrame) frame).text();
            System.out.println("收到: " + text);
            // 广播给所有连接
            ctx.channel().writeAndFlush(new TextWebSocketFrame("回复: " + text));
        } else if (frame instanceof CloseWebSocketFrame) {
            ctx.channel().close();
        }
    }
    
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt == WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE) {
            System.out.println("WebSocket 握手完成");
        }
    }
}
```

**⑤ Keep-Alive / 心跳机制**

开启 `WebSocketServerProtocolHandler(..., true)` 后，Netty 自动处理 Ping/Pong：
- 收到 Ping → 自动回复 Pong（无需业务代码处理）
- 若要自定义心跳（发送业务层心跳消息），使用 `PingWebSocketFrame` / `PongWebSocketFrame`

**⑥ 分片传输（Chunked WebSocket）**

```java
// 大数据分片发送
ByteBuf data = ...;
ctx.write(new TextWebSocketFrame(true, WebSocketScheme.WSS, data)); // isLast=true 表示最后一个分片
```

---

### 追问方向

- WebSocket 握手中的 Sec-WebSocket-Key/Sec-WebSocket-Accept 是如何计算的？
- WebSocket 断开连接时，什么情况下需要发送 Close 帧？
- 如何限制 WebSocket 连接数量和消息大小？

### 避坑提示

- ❌ 混淆 WebSocket close 帧和 TCP RST——需要遵循 WebSocket 关闭协议。
- ❌ 忘记在 Pipeline 中移除 HTTP Codec，导致 WebSocket 连接无法建立。
- ❌ TextWebSocketFrame 和 ByteBuf 的引用计数管理混淆。

---

## 20. Netty 实战问题：内存溢出、CPU 占用 100%、连接超时排查

### 题目
生产环境中 Netty 服务出现内存溢出、CPU 占用 100%、连接超时三类问题，请描述排查思路和工具。

### 核心答案

**① 内存溢出（OOM / Full GC）**

**排查步骤**：

1. 添加 JVM 参数开启 GC 日志
   ```
   -Xlog:gc*:file=gc.log -XX:+HeapDumpOnOutOfMemoryError
   ```

2. 使用 `jstat -gcutil <pid> 1000` 观察 GC 情况
   - FGC（Full GC）频繁且 Old 区快速增长 → ByteBuf 未及时 release
   - Eden 区增长过快 → Channel 频繁创建但未正确关闭

3. 开启 Netty 泄漏检测
   ```
   -Dio.netty.leakDetection.level=PARANOID
   ```

4. 使用 MAT / VisualVM 分析堆转储文件（.hprof）
   - 查找 `PooledByteBufAllocator` 的 `PoolThreadCache` 或 `PoolArena`
   - ByteBuf 对象数量异常 → 某 Handler 未 release

5. 代码层检查
   - `channelRead()` 中 ByteBuf 是否每次都 release
   - `exceptionCaught()` 中是否有遗漏的 release
   - 连接池是否设置了最大连接数限制

**典型泄漏模式**：

```java
// 错误：msg 没有向下传递也没有释放
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        process(buf);  // 处理完后直接丢弃，buf 未 release
    }
}

// 正确
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ReferenceCountUtil.release(msg); // 或传递时 retain
}
```

---

**② CPU 占用 100%**

**排查步骤**：

1. 确认是哪个线程占用 CPU
   ```bash
   top -Hp <pid>  # 找到占用最高的线程 ID
   printf '%x\n' <tid>  # 转换为十六进制
   jstack <pid> | grep <hex_tid>  # 查看该线程的堆栈
   ```

2. 常见 Netty CPU 高占用原因：
   - **EventLoop 线程阻塞**：业务 Handler 执行了同步阻塞操作（RPC、DB、Thread.sleep），导致 EventLoop 被阻塞
     ```
     jstack 显示 "NioEventLoop.run()" 中大量 "Unsafe.park"
     ```
   - **大量定时任务**：IdleStateHandler + 高频 schedule 导致 EventLoop 过度调度
   - **正则表达式回溯**：解码器中使用了低效正则，触发回溯攻击
   - **ConstantPool 竞争**：大量动态创建 Handler 类导致 ClassLoader 压力

3. 解决方案
   - 将阻塞业务拆到独立线程池：`ctx.executor().execute(() -> { ... })`
   - 或使用 Netty 的 `UnorderedThreadPoolEventExecutor`
   - 使用 `io.netty.leakDetection.level` 排除泄漏因素
   - 检查正则表达式使用 `Pattern.matches()` 而非 `Matcher.find()`

---

**③ 连接超时（connect timeout / response timeout）**

**排查步骤**：

1. 服务端问题排查
   ```bash
   netstat -anp | grep <port>  # 查看连接状态
   # ESTABLISHED: 正常
   # TIME_WAIT: 连接正常关闭
   # CLOSE_WAIT: 对端关闭但本端未 close()
   # SYN_SENT: 发起连接但未收到 ACK
   ```

2. 客户端问题排查
   - EventLoop 是否被阻塞（`sync()` 死锁）
   - 连接超时配置：`Bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)`
   - Channel 是否已 active：`if (!ch.isActive()) return`

3. 网络层问题
   - 查看 Netty 日志中是否有 `Connection reset by peer`
   - 检查 `SO_BACKLOG` 是否过小导致连接队列溢出
   - 检查防火墙 / NAT 超时导致连接被提前断开

4. 心跳配合解决
   - 配置 `IdleStateHandler` 检测空闲连接
   - 启用 TCP keepalive：`childOption(ChannelOption.SO_KEEPALIVE, true)`
   - 配置合理的重连策略（指数退避）

---

### 追问方向

- OOM 发生时如何dump堆栈？如何判断是 Netty ByteBuf 泄漏还是业务对象泄漏？
- EventLoop 阻塞时如何定位是哪个 Handler 导致的？
- 高并发场景下，如何预防 Netty 连接被打满？

### 避坑提示

- ❌ 发现 OOM 就直接加内存，不排查根因——ByteBuf 泄漏导致加再多内存也会耗尽。
- ❌ CPU 100% 时只重启服务，不分析线程堆栈——问题会复发。
- ❌ 连接超时时只调大超时时间，不排查是否是真的网络问题还是 EventLoop 阻塞。
- ❌ 使用 `thread` 命令而非 `top -Hp` 查看线程 CPU——无法定位具体线程。

---

## 附录：面试高频关键词速查

| 关键词 | 关联题目 |
|--------|---------|
| Reactor 模型 | #1 线程模型 |
| ChannelPipeline 双向链表 | #2 Handler 管道 |
| PooledArena 池化 | #3 ByteBuf |
| sendfile / FileRegion | #4 零拷贝 |
| IdleStateHandler | #5/#17 心跳 |
| LengthFieldBasedFrameDecoder | #7 粘包半包 |
| @Sharable | #13 编解码链 |
| ResourceLeakDetector | #14 内存泄漏 |
| sync/await 死锁 | #11 Future |
| refCnt retain/release | #12 引用计数 |
| WebSocket 握手 | #19 WebSocket |
| EventLoop 阻塞 | #20 实战问题 |
