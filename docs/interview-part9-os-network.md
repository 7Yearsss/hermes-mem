# 计算机网络与操作系统高频面试题

---

## 1. TCP三次握手：为什么需要三次？ISN和SYN超时怎么处理？

### 题目
面试官：客户端向服务器发送SYN后一直没收到服务器的SYN+ACK，此时会发生什么？TCP为什么要设计成三次握手而不是两次？

### 核心答案
**三次握手的本质：同步双方序列号**

| 阶段 | 客户端 | 服务器 |
|------|--------|--------|
| 第一次 | 发送SYN，seq=x（客户端ISN） | — |
| 第二次 | — | 收到SYN，发送SYN+ACK，seq=y（服务器ISN），ack=x+1 |
| 第三次 | 收到SYN+ACK，发送ACK，seq=x+1，ack=y+1 | 收到ACK，ESTABLISHED |

**为什么不是两次？**
- 两次握手只能让**一方**确认对方的接收能力和发送能力都正常
- 两次握手无法同步双方的初始序列号（ISN）
- 历史教训：两次握手会导致**旧SYN包**被服务器误认为新连接，浪费服务器资源

**ISN（Initial Sequence Number）作用：**
- 随机生成，避免**旧连接的报文被误认为新连接的合法数据**
- 防御SYN Flood攻击（但不够，需要SYN Cookie）

**SYN超时处理（以Linux为例）：**
- 默认超时时间：63秒（指数退避：1s→2s→4s→8s→...→64s）
- 超时后**最多重传5次**（CentOS 6默认），之后报错"Connection timed out"
- 代码中表现为：`connect()` 系统调用阻塞直到超时或失败

### 追问方向
- **SYN Cookie**：如何用Cookie防御SYN Flood？Cookie是如何计算和验证的？
- **半连接队列**：listen backlog参数有什么用？SYN Flood攻击打到的是哪个队列？
- **如果第三次握手丢了**：服务器会怎样？客户端会怎样？

### 避坑提示
- 别说"三次是为了确认双方都在线"——这太浅，正确答案是**同步序列号**
- 不要混淆ISN和ACK序号的关系：ACK = seq + 1
- Linux内核参数 `tcp_syn_retries`（客户端重传次数）和 `tcp_synack_retries`（服务器重传次数）是两个不同的参数，面试官常追问

---

## 2. TCP四次挥手：为什么需要四次？TIME_WAIT和2MSL是什么？

### 题目
面试官：TCP四次挥手能不能变成三次？如果客户端大量处于TIME_WAIT状态该怎么处理？

### 核心答案
**四次挥手的本质：双方独立关闭发送通道**

```
客户端                        服务器
  |                            |
  |----------- FIN ------------>|  客户端：我发完了（关闭写端）
  |<---------- ACK ------------|  服务器：知道了
  |                            |
  |<---------- FIN ------------|  服务器：我发完了
  |----------- ACK ------------>|  客户端：知道了
  |                            |
  (客户端进入TIME_WAIT)        (服务器进入CLOSED)
```

**为什么不能合并成三次？**
- TCP是**全双工**通信，每一方向的关闭是独立的
- 客户端发送FIN只表示"客户端不发送数据了"，但服务器可能还在发数据
- 合并FIN+ACK？不能，因为ACK可以立即发，但服务器的数据可能还没发完

**TIME_WAIT状态（客户端）：**
- 持续时间 = **2MSL**（Maximum Segment Lifetime，通常=60秒）
- MSL是报文在网络中最大存活时间，业界一般定义为**60秒或120秒**
- 2MSL = 一次单向最大存活 + 一次ACK返回的存活

**TIME_WAIT存在的两个理由：**
1. **可靠关闭**：让迟到的FIN包能被正确丢弃，避免旧连接的包被新连接误收
2. **让旧连接的端口复用更安全**：确保四元组相同的旧连接所有报文都消失在网络中

**CLOSING状态**：双方同时发送FIN（同时关闭），收到对方的FIN后先进入CLOSING，再收到ACK才进入TIME_WAIT → 这种情况很少见但面试会问。

### 追问方向
- **大量TIME_WAIT**：原因是什么（频繁短连接）？如何优化（`tcp_tw_reuse`=1、`tcp_tw_recycle`=1、设置SO_LINGER为0、扩大本地端口范围）？
- **服务器会有TIME_WAIT吗**：主动关闭方才会有。服务器主动关闭也会有TIME_WAIT，所以生产环境中服务器也要处理
- **`tcp_tw_reuse`和`tcp_tw_recycle`的区别**：recycle会重用于新连接，recycle依赖对端 timestamp，reuse依赖协议规范

### 避坑提示
- 2MSL不是120秒就固定了——Linux内核用 `net.ipv4.tcp_fin_timeout` 控制，默认60秒
- 说"TIME_WAIT是60秒"不准确，准确说法是"取决于`tcp_fin_timeout`配置，默认为60秒"
- SO_REUSEADDR 选项解决的是"服务器重启后端口仍被TIME_WAIT占用"的问题，不是解决客户端TIME_WAIT的

---

## 3. TCP状态机：画图说明各状态的含义和转换条件

### 题目
面试官：服务器进程崩溃时，TCP连接处于什么状态？客户端发送数据会怎样？

### 核心答案
**完整TCP状态转换图（简化版）：**

```
CLOSED
   | 主动打开
   v
SYN_SENT ←---------------→ SYN_RCVD
   |    \                 /  |
   |     \   同时打开    /   |
   |      \             /    |
   |       CLOSING ←---       |
   |         |                 |
   |    收到ACK/超时           | 收到FIN
   |         |                 v
   |         |           LAST_ACK
   |         |                 |
   |         |                 | 收到ACK
   |         |                 v
   |    TIME_WAIT             CLOSED
   |         |
   |         | 2MSL超时
   |         v
   |       CLOSED
   |                       
ESTABLISHED ←------------------+
   |                           |
   | 收到FIN                   | 发送FIN
   v                           v
FIN_WAIT_1 ←--- FIN_WAIT_2 --→ CLOSE_WAIT
   |                           |
   | 收到ACK                   | 收到对端FIN，发送ACK
   | (主动端关闭)               | (被动端关闭)
   v                           |
FIN_WAIT_2                     v
   |                      CLOSING（同时关闭）
   | 收到对端FIN
   v
TIME_WAIT
```

**关键状态含义：**
- **ESTABLISHED**：双方正常通信
- **FIN_WAIT_1**：主动关闭方发了FIN，等待对方ACK
- **FIN_WAIT_2**：主动关闭方发了FIN，收到ACK，等待对方FIN
- **CLOSE_WAIT**：被动关闭方收到FIN并发了ACK，等待本进程调用close()
- **LAST_ACK**：被动关闭方发了FIN，等待对方ACK
- **CLOSING**：双方同时关闭，两边都发了FIN

**服务器进程崩溃的情况：**
- 进程崩溃 → 内核发送FIN → 触发四次挥手
- 此时连接处于**CLOSE_WAIT**（服务器）状态
- 客户端发送数据 → 服务器内核回复**RST**（因为没有进程在监听）

### 追问方向
- **`close()` vs `shutdown()`** 对状态机的影响：close()会导致"发送FIN并立即删除TCB"，shutdown()可以只关闭一半
- **RST报文**：什么情况会发RST？收到RST后连接直接进入CLOSED，不走四次挥手

### 避坑提示
- 背状态机图时注意箭头方向，不要画反
- "同时打开"进入SYN_SENT状态，"同时关闭"进入CLOSING状态，这两个特殊路径一定要知道

---

## 4. TCP可靠性：确认机制、重传超时、快速重传、SACK

### 题目
面试官：TCP如何知道某个数据包丢了？重传机制是怎样的？RTO是怎么计算的？

### 核心答案
**确认机制（ACK累积确认）：**
- TCP采用**累积ACK**，ack=N表示"N之前的所有数据我都收到了"
- ACK可能 delay（ Delayed ACK），最多等40ms

**重传超时（RTO，Retransmission Timeout）：**
- 发送方发包后启动定时器，超时未收到ACK就重传
- **RTT（Round Trip Time）测量**：`RTT = 收到ACK时间 - 发包时间`
- **RTO计算（Jacobson算法）**：
  ```
  SRTT = (1-α) × SRTT + α × RTT      // 平滑RTT
  RTTVAR = (1-β) × RTTVAR + β × |RTT - SRTT|  // 平滑偏差
  RTO = SRTT + 4 × RTTVAR
  ```
  - α = 1/8，β = 1/4（经典值）
  - Linux实现：`RTO = SRTT + 4 × RTTDEV`

**快速重传（Fast Retransmit）：**
- 发送方收到**3个重复ACK**（即同一个seq的ACK收到4次），不等到超时就重传
- 触发条件：3个冗余ACK说明"这个包之后的东西都收到了，唯独缺这个包"

**SACK（Selective Acknowledgment，选择性确认）：**
- 基础ACK只能确认连续数据的上限
- SACK选项允许接收方告诉发送方"我收到了哪些不连续的块"
- 格式：`[left, right]` 区间
- 发送方收到SACK后就知道哪些包真正丢了，只重传丢失的部分

### 追问方向
- **TLP（Tail Loss Probe）**：为什么要在超时时额外发一个probe包？
- **RTO计算的 Karn's Algorithm**：重传后收到ACK如何判断是哪个包的ACK？
- **伪超时（Spurious Timeout）**：网络抖动导致的虚假重传如何处理？

### 避坑提示
- RTO不是固定值，是**动态计算**的，不要说"RTO是200ms"
- 说"超时时间是200ms"是错的，要说"Linux默认RTO最小值是200ms，实际是动态计算的"
- 快速重传只看**ACK数量**，不看**时间**

---

## 5. TCP流量控制：滑动窗口、窗口收缩、零窗口探测

### 题目
面试官：如果接收方窗口收缩到0，发送方会怎么做？Zero WindowProbe包里面有什么？

### 核心答案
**滑动窗口机制：**
- 发送方维护：`SND.NXT`（下一个要发送的字节）、`SND.UNA`（最早未被确认的字节）
- 可用窗口大小 = `SND.WND`（对方接收窗口大小）
- 接收方维护：`RCV.NXT`（期望收到的下一个字节）

**窗口收缩：**
- 接收方通知更小的窗口值，发送方必须遵守
- 极端情况：接收方窗口=0，发送方**不能再发送任何数据**（除了一种情况）

**零窗口探测（Zero Window Probe）：**
- 发送方窗口为0时，会**周期性**发送探测包（1字节的零窗口探测）
- 探测间隔：**超时重传**，探测间隔随RTO指数增长（Linux默认最小5秒）
- Probe包是带1字节数据的包，用来触发ACK并获取最新窗口值
- 即使窗口为0，**探测包本身必须发送**（因为要获取窗口信息）

**持续计时器（Persistence Timer）：**
- 专门处理零窗口问题
- 探测包超时未收到响应就继续探测
- 直到窗口变为非0或连接关闭

### 追问方向
- **窗口膨胀（Window Scaling）**：接收窗口最大值只有65535字节？RFC 1323的窗口膨胀因子了解吗？
- **糊涂窗口综合征（SWS）**：小窗口不发送数据的死锁问题如何解决？（Nagle算法、延迟ACK）

### 避坑提示
- 不要把"零窗口探测"和"Keep-Alive"混淆——两者目的完全不同
- 说"窗口大小是固定的"是错误的，窗口大小是**动态协商**的（Window Scaling）
- 接收方窗口收缩是**正常的**，但发送方要正确处理，不能直接丢弃未发送的数据

---

## 6. TCP拥塞控制：慢启动、拥塞避免、快恢复，cwnd是怎么变化的？

### 题目
面试官：cwnd从1开始增长，假设不限制，能无限增长吗？快恢复阶段收到3个冗余ACK和超时分别怎么处理？

### 核心答案
**拥塞窗口（cwnd，congestion window）：**
- 发送方维护的另一个窗口，`可用发送窗口 = min(cwnd, rwnd)`
- 体现**网络承受能力**

**四个阶段：**

```
慢启动（Slow Start）
  cwnd = 1 MSS
  每收到一个ACK → cwnd += 1 MSS   // 指数增长
  何时退出：cwnd ≥ ssthresh

拥塞避免（Congestion Avoidance）
  每收到一个ACK → cwnd += MSS²/cwnd  // 线性增长（近似）
  每经过一个RTT → cwnd += 1 MSS

快恢复（Fast Recovery）← 快重传触发
  ssthresh = cwnd / 2
  cwnd = ssthresh + 3 MSS
  每收到一个冗余ACK → cwnd += 1 MSS
  收到新ACK → 进入拥塞避免

超时（Timeout）→ 最严厉的惩罚
  ssthresh = cwnd / 2
  cwnd = 1 MSS
  重新慢启动
```

**ssthresh（慢启动阈值）：**
- cwnd < ssthresh：慢启动（指数）
- cwnd ≥ ssthresh：拥塞避免（线性）

**快重传 vs 超时的区别：**
- 3个冗余ACK：说明网络**勉强能到达**，部分数据包丢了但后续到了
- 超时：说明网络**严重拥塞**，所有后续包可能都没到

### 追问方向
- **BBR（CUBIC是Linux默认）**：BBR和Reno/CUBIC的区别是什么？为什么BBR能更快利用高带宽？
- **PRR（Proportional Rate Reduction）**：快恢复时如何更平滑地恢复发送速率？
- **`tcp_congestion_control` 参数**：生产环境如何调整？

### 避坑提示
- 别说"cwnd最大65535"——这是古老的限制，现代用SACK和窗口膨胀可以很大
- 快恢复时"cwnd = ssthresh + 3"中的3是有道理的：代表3个数据包仍在网络中
- 超时后cwnd重置为1 MSS，但ssthresh不是1 MSS，是原来的一半

---

## 7. TCP vs UDP区别：面向连接 vs 无连接、可靠性、流控、效率

### 题目
面试官：TCP和UDP的核心区别是什么？什么场景用UDP更好？

### 核心答案

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接性 | 面向连接（三次握手四次挥手） | 无连接（直接发数据报） |
| 可靠性 | 可靠（确认、重传、排序） | 不可靠（不确认、不重传） |
| 顺序性 | 有序（序列号保证） | 无序（可能乱序） |
| 流量控制 | 有（滑动窗口） | 无 |
| 拥塞控制 | 有（慢启动等） | 无 |
| 首部开销 | 20-60字节 | 8字节固定 |
| 效率 | 相对低（头部大、确认开销） | 高（头部小、无连接建立） |
| 传输形式 | 字节流 | 数据报（Datagram） |

**TCP适用场景：** HTTP/HTTPS、SSH、SMTP、文件传输等需要可靠传输的应用

**UDP适用场景：**
- DNS查询（请求小，响应也小，可接受丢包）
- 实时音视频（直播、视频会议）：容忍一定丢包，实时性 > 可靠性
- QUIC协议：基于UDP实现可靠传输 + 多路复用
- 广播/多播通信

### 追问方向
- **TCP的"假连接"问题**：NAT、超时导致连接看似存在但对端已不存在
- **UDP如何实现可靠**：QUIC、Reliable UDP（RUDP）如何用UDP模拟TCP的可靠性？
- **QUIC vs TCP+TLS**：QUIC的优势是什么？

### 避坑提示
- 不要说"UDP不可靠所以没用"——很多高实时性场景UDP是唯一选择
- TCP是字节流协议，不是数据报协议——UDP才是数据报协议
- TCP头部最小20字节，UDP固定8字节，不要记混

---

## 8. UDP应用：DNS、QUIC、视频直播为什么用UDP？

### 题目
面试官：为什么DNS查询用UDP而不是TCP？QUIC协议是怎么用UDP实现可靠传输的？

### 核心答案
**DNS为什么用UDP：**
- DNS查询请求通常**小于512字节**，一个UDP包就能装下
- TCP三次握手+挥手开销太大，DNS需要快速响应
- DNS服务器压力大：用TCP的话服务器要维护连接状态，开销巨大
- DNS的可靠性通过**应用层重试**来实现：查询没响应就重发

**DNS什么时候用TCP：**
- 响应数据超过512字节（zone transfer时）
- 主从DNS同步
- IXFR（增量zone transfer）

**QUIC协议（基于UDP）：**
Google设计的协议，解决TCP的多个痛点：
- **0-RTT握手**：首次连接0-RTT，再次连接0-RTT（TLS 1.3类似）
- **多路复用**：不同Stream独立，不阻塞（解决TCP队头阻塞问题）
- **连接迁移**：连接以Connection ID标识，换IP/端口不需要重建连接
- **可靠传输**：QUIC层自己实现ACK、重传、顺序保证，不依赖TCP

**QUIC如何实现可靠：**
- 每个Stream有自己的Sequence Number
- 数据包加密，丢失检测后触发该包内所有Stream的数据重传
- Fast Open：首次握手时即可发送业务数据

**视频直播为什么用UDP：**
- **实时性 > 可靠性**：视频直播丢几帧不影响体验，但卡顿影响体验
- TCP重传会导致延迟累积（ARQ机制），不适合实时流
- UDP没有拥塞控制，不会因网络拥塞主动降速
- 现代直播用UDP+RTP/RTMP，在应用层做丢包补偿（Forward Error Correction）

### 追问方向
- **RTP/RTMP**：RTP是怎么利用UDP的？时间戳和序列号的作用？
- **QUIC的队头阻塞**：QUIC如何解决Stream之间的队头阻塞？和TCP对比
- **FEC（前向纠错）**：视频直播中如何用FEC减少重传？

### 避坑提示
- QUIC不是替代TCP，而是解决TCP的特定问题（队头阻塞、握手延迟）
- "视频直播用UDP所以不可靠"——这是误解，是**在应用层实现了部分可靠性**
- DNS用UDP的端口是53，但超过512字节的响应会切换到TCP

---

## 9. HTTP 1.0/1.1/2.0/3.0区别：长连接、Pipeline、多路复用、头部压缩

### 题目
面试官：HTTP/2解决了HTTP/1.1的哪些问题？多路复用具体是怎么实现的？

### 核心答案

| 版本 | 关键特性 | 核心问题 |
|------|----------|----------|
| HTTP/1.0 | 短连接（每个请求单独TCP） | 每次请求都要握手，延迟高 |
| HTTP/1.1 | **长连接（keep-alive）**、Pipeline | 队头阻塞（HOL blocking） |
| HTTP/2.0 | **多路复用**、HPACK头部压缩、服务器推送 | 仍受TCP队头阻塞影响 |
| HTTP/3.0 | **QUIC（UDP）**、0-RTT、快速重启 | 初期高丢包时不如HTTP/2 |

**HTTP/1.1 长连接：**
- 默认开启`Connection: keep-alive`
- 一个TCP连接可以发多个HTTP请求
- 但**响应必须顺序返回**（队头阻塞）

**HTTP/1.1 Pipeline：**
- 客户端可以发多个请求不等待响应（管线化）
- 但服务器必须**按顺序返回**响应
- 基本没被广泛使用（浏览器不支持）

**HTTP/2.0 多路复用：**
- 在**一个TCP连接**上，通过**Stream ID**同时传输多个Stream
- 每个Stream内有自己的请求/响应
- Stream之间**互不阻塞**
- 解决了应用层的队头阻塞，但**没有解决TCP层的队头阻塞**

**HTTP/2.0 HPACK 头部压缩：**
- 静态表（常用头部+索引）
- 动态表（本次连接中出现的头部）
- Huffman编码
- 避免重复传输冗长的Header

**HTTP/3.0（QUIC）：**
- 基于UDP，避免TCP队头阻塞
- 0-RTT/1-RTT握手
- 连接迁移（Connection Migration）

### 追问方向
- **TCP队头阻塞**：HTTP/2的Stream不会互相阻塞，但丢包会阻塞所有Stream——为什么？
- **HPACK原理**：静态表和动态表如何配合工作？
- **HTTP/2的服务器推送**：浏览器收到推送资源后如何处理（缓存）？

### 避坑提示
- HTTP/2多路复用≠多个TCP连接，是**一个TCP连接上的多个Stream**
- 说"HTTP/2完全解决了队头阻塞"是错的——只解决了应用层队头阻塞，TCP层队头阻塞仍在
- HTTP/3不是IETF标准了——HTTP/3于2022年6月正式成为RFC 9114

---

## 10. HTTP常见状态码：2xx/3xx/4xx/5xx含义，304缓存、502/503/504

### 题目
面试官：502和504有什么区别？304是怎么实现缓存的？

### 核心答案
**2xx 成功：**
- 200 OK：请求成功
- 201 Created：资源创建成功（POST后）
- 204 No Content：成功但无返回体（用于DELETE）
- 206 Partial Content：部分内容（断点续传、Range请求）

**3xx 重定向：**
- 301 Moved Permanently：**永久**重定向（缓存）
- 302 Found：**临时**重定向（不缓存）
- 304 Not Modified：**缓存未修改**（见下方）
- 307 Temporary Redirect：临时重定向，但方法不变
- 308 Permanent Redirect：永久重定向，方法不变

**4xx 客户端错误：**
- 400 Bad Request：请求语法错误
- 401 Unauthorized：未认证（需要登录）
- 403 Forbidden：已认证但无权限
- 404 Not Found：资源不存在
- 405 Method Not Allowed：方法不支持
- 408 Request Timeout：请求超时
- 429 Too Many Requests：请求过于频繁

**5xx 服务器错误：**
- 500 Internal Server Error：服务器内部错误
- 501 Not Implemented：功能未实现
- 502 Bad Gateway：网关错误（**上游服务器返回了无效响应**）
- 503 Service Unavailable：服务不可用（过载、维护）
- 504 Gateway Timeout：网关超时（**上游服务器响应超时**）

**304 Not Modified 缓存机制：**
```
客户端请求：
  If-Modified-Since: <上次缓存的Last-Modified时间>
  If-None-Match: <上次缓存的ETag值>

服务器判断：
  资源未变化 → 304 + 空body
  资源已变化 → 200 + 新内容
```
- 304响应没有body，节省带宽
- ETag比Last-Modified更精确（时间戳 vs 哈希）

**502 vs 504 的区别：**
- 502：上游服务器**返回了错误响应**（比如nginx代理到后端，后端崩溃返回HTML错误页）
- 504：上游服务器**超时没响应**（nginx等待后端，后端处理太慢）

### 追问方向
- **302 vs 307 vs 308**：307和308分别解决了302什么问题？
- **429限流**：服务端如何实现？Redis token bucket？

### 避坑提示
- 301和302的区别在于"是否缓存"，面试常考但容易说错
- 401是"未认证"，403是"认证了但没权限"——两者容易混淆
- 502不代表后端服务器一定挂了，可能是后端返回了nginx不认识的响应格式

---

## 11. HTTPS原理：SSL/TLS握手过程、RSA/ECDHE密钥交换、数字证书

### 题目
面试官：HTTPS握手过程是怎样的？ECDHE和RSA握手的区别是什么？

### 核心答案
**TLS握手（RSA密钥交换，1-RTT）：**
```
Client                                        Server
  |                                              |
  | ① ClientHello（支持的TLS版本、加密套件、random）  |
  |--------------------------------------------> |
  |                                              |
  | ② ServerHello（选定版本+加密套件+server_random） |
  | ③ Certificate（服务器证书链）                   |
  | ④ ServerHelloDone                            |
  |<-------------------------------------------- |
  |                                              |
  | ⑤ ClientKeyExchange（premaster secret，用公钥加密）|
  | ⑥ ChangeCipherSpec                            |
  | ⑦ Finished（摘要验证）                        |
  |--------------------------------------------> |
  |                                              |  ⑥ ChangeCipherSpec
  |                                              |  ⑦ Finished
  |<-------------------------------------------- |
  |                                              |
  Application Data <==========================>  |
```

**ECDHE密钥交换（TLS 1.3默认，前向安全）：**
- 不使用RSA做密钥交换，使用DH/ECDH
- 双方各生成临时DH私钥，交换公钥
- 不需要等服务器Certificate，**可以并行发送应用数据（0-RTT）**
- 最重要的区别：**ECDHE支持前向安全（Forward Secrecy），RSA不支持**
- 前向安全：即使私钥泄露，过去的通信也无法被解密

**数字证书：**
- 证书内容：`主体信息 + 公钥 + 有效期 + 签名`
- 签名：`CA私钥`对`证书内容哈希`加密
- 证书链：Root CA → Intermediate CA → Server Certificate
- 客户端验证：使用CA公钥验证Intermediate CA签名 → 验证Server Certificate签名 → 检查有效期/域名/CRL/OCSP

### 追问方向
- **TLS 1.3的改进**：1-RTT变0-RTT、废弃3DES/RC4/MD5/SHA1、简化握手
- **CRL和OCSP**：证书吊销检查的区别？OCSP stapling怎么优化？
- **中间人攻击**：HTTPS能防止中间人攻击吗？什么情况下防不住？

### 避坑提示
- 说"HTTPS使用RSA加密"是**过时**的——TLS 1.3默认用ECDHE，RSA只用于签名
- 证书验证失败不等于"连接被劫持"，可能是用户配置问题（自签名证书）
- HTTPS只加密传输内容，证书域名验证是防中间人的关键

---

## 12. HTTPS连接建立过程：TCP三次握手 + TLS握手，总耗时优化

### 题目
面试官：一次完整的HTTPS请求需要多少个RTT？有什么优化手段？

### 核心答案
**完整连接建立耗时（RSA密钥交换 + TLS 1.2）：**
- TCP三次握手：1 RTT
- TLS握手（1-RTT）：1 RTT
- **总计：2 RTT**（纯握手，还不含HTTP请求/响应）

**TLS 1.3的改进：**
- TLS 1.3握手：**1 RTT**（减少一次）
- TLS 1.3 0-RTT：恢复会话时直接发送加密数据（依赖之前建立的session）

**总耗时分析：**
```
TCP握手（1 RTT） + TLS握手（1 RTT） + HTTP请求/响应（1 RTT） = 3 RTT（首次）
```

**优化手段：**

| 优化手段 | 节省RTT | 原理 |
|----------|---------|------|
| TLS Session Resumption | 减少1 RTT | 复用之前会话的密钥 |
| OCSP Stapling | 减少1 RTT | 服务器直接提供吊销状态 |
| HTTP/2 | 减少连接数 | 多路复用，多个请求共用1个连接 |
| CDN | 减少物理距离 | 就近接入 |
| TLS 1.3 | 1-RTT变0-RTT | ECDHE + Early Data |
| 预加载（Preconnect） | 减少DNS+TCP | `<link rel="preconnect">` |

**HTTP/2 + HTTPS：**
- HTTP/2可以复用同一连接
- 多个请求可以并行发出（多路复用）
- 首部压缩（HPACK）减少数据传输量

### 追问方向
- **Session ID vs Session Ticket**：会话恢复的两种方式，区别是什么？
- **False Start**：TLS False Start是什么？允许在握手完成前发送加密数据？
- **OCSP Stapling原理**：服务器从哪里获取OCSP响应？

### 避坑提示
- 不要把"减少RTT"和"减少延迟"混淆——RTT取决于物理距离，优化只能减少次数，不能改变光速
- TLS 1.3的0-RTT有**重放攻击**风险，某些场景不适合开启
- OCSP Stapling需要服务器主动获取OCSP响应，不是浏览器主动去查的

---

## 13. DNS解析过程：递归/迭代查询、hosts文件、CDN解析

### 题目
面试官：访问一个域名，DNS解析的完整流程是什么？CDN如何通过DNS实现就近访问？

### 核心答案
**DNS解析完整流程：**

```
1. 浏览器检查自身DNS缓存 → 有则返回
2. 浏览器检查OS DNS缓存（nscd） → 有则返回
3. 系统调用 gethostbyname() → 查询 /etc/resolv.conf 中的本地DNS服务器
4. 本地DNS服务器（递归解析器）开始查询：
   a. 查根域名服务器（全球13组）→ 拿到 .com TLD服务器地址
   b. 查 .com TLD服务器 → 拿到 example.com 权威服务器地址
   c. 查 example.com 权威服务器 → 拿到 www.example.com 的IP
5. 本地DNS缓存结果（TTL时间内有效）
6. 返回给浏览器
```

**递归查询 vs 迭代查询：**
- **递归查询**：客户端 → 本地DNS服务器，本地DNS服务器负责查到结果返回
- **迭代查询**：DNS服务器之间，各级服务器只返回"我知道的下一级地址"
- 客户端到本地DNS服务器是**递归**，DNS服务器之间是**迭代**

**hosts文件：**
- 最高优先级：`/etc/hosts`（Linux）或 `C:\Windows\System32\drivers\etc\hosts`
- 优先级高于DNS缓存
- 用于本地开发、劫持（恶意修改hosts）、测试

**CDN的DNS解析流程：**
```
用户 → 访问 www.example.com
  → 本地DNS → example.com权威DNS
  → 权威DNS返回 CDN智能DNS的地址
  → CDN DNS根据用户IP → 返回最近CDN节点的IP
  → 用户直接访问CDN节点（缓存内容就近分发）
```
- 核心：**权威DNS返回的IP不是源站IP，而是CDN节点的IP**
- CDN DNS会查看请求者的**来源IP**，返回最近节点的IP
- 有时会返回多个IP，让客户端做**负载均衡**

### 追问方向
- **DNS TTL**：权威DNS的TTL设置过低/过高各有什么问题？
- **DNSoverHTTPS（DoH）**：浏览器直接发起HTTPS DNS查询，防DNS劫持
- **DNS缓存投毒/欺骗攻击**：Kaminsky攻击的原理和防御

### 避坑提示
- 说"本地DNS服务器就是递归DNS"——对，但要注意它可能转发到上游DNS
- CDN就近访问的核心是**DNS层面决策**，不是BGP路由
- DNS记录类型：A（IPv4）、AAAA（IPv6）、CNAME（别名）、MX（邮件）——要分清楚

---

## 14. Cookie vs Session：区别、分布式Session、CSRF攻击

### 题目
面试官：Cookie存在哪里？Session存在哪里？如果服务器是集群，如何处理Session？

### 核心答案
**Cookie：**
- 存储在**客户端浏览器**（每个Cookie大小限制~4KB）
- 通过HTTP Header传输：`Set-Cookie` / `Cookie`
- 属性：`Name/Value/Path/Domain/Expires/Max-Age/HttpOnly/Secure/SameSite`
- `HttpOnly`：JS无法读取（防XSS）
- `Secure`：仅HTTPS传输
- `SameSite`：防CSRF（Strict/Lax/None）

**Session：**
- 存储在**服务器端**（内存、Redis、数据库）
- 通过Cookie中的`SessionID`关联
- 每个Session有TTL（超时时间）

**分布式Session的几种方案：**

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| Session复制 | 集群内同步Session | 简单 | 占用内网带宽 |
| Session绑定（Sticky Session） | 同一用户路由到同一节点 | 简单 | 节点故障会丢失 |
| Session集中存储（Redis） | 所有节点从Redis读Session | 高可用、跨节点 | 依赖Redis |
| Cookie中存储Session数据 | 签名+加密存客户端 | 无存储压力 | 带宽开销、安全风险 |

**CSRF攻击（Cross-Site Request Forgery）：**
- 原理：用户登录A网站后，访问恶意网站B，B自动发起对A的请求（带Cookie）
- 防御手段：
  - `SameSite` Cookie：Strict（完全禁止跨站）/Lax（GET允许，POST限制）
  - CSRF Token：服务端生成随机Token，表单中携带，验证一致性
  - 验证码/密码确认：关键操作需要用户二次验证
  - 检查Referer/Origin Header

### 追问方向
- **Session Fixation攻击**：攻击者预先设置SessionID，用户登录后攻击者用同一个SessionID劫持
- **JWT vs Session**：JWT存在哪里？和Session的区别？JWT被盗怎么办？
- **Cookie大小限制**：为什么Cookie限制4KB？如何绕过（Session Storage/IndexedDB）？

### 避坑提示
- Cookie是客户端存储，Session是服务端存储——这是核心区别
- 说"Session存在Redis里所以不用Cookie"——错，SessionID还是通过Cookie传输的
- CSRF和XSS的区别：CSRF是利用用户已登录状态，XSS是偷用户的信息/权限

---

## 15. 输入URL到页面展示全过程：DNS/TCP/TLS/HTTP/DOM解析/渲染

### 题目
面试官：从输入URL到页面展示，完整过程中发生了什么？DNS解析是哪一步做的？

### 核心答案
**完整过程（以HTTPS为例）：**

```
1. URL解析
   → 协议（https）、域名（www.example.com）、端口（默认443）、路径（/page）

2. DNS解析
   → 浏览器缓存 → 系统缓存 → 本地DNS递归解析 → 获取IP

3. 建立TCP连接（三次握手）
   → 根据IP建立TCP连接到:443端口

4. TLS握手（HTTPS）
   → 客户端/服务端协商加密参数 → 验证证书 → 交换密钥

5. HTTP请求
   → 浏览器发送HTTP请求（GET /page HTTP/1.1, Host: www.example.com）

6. 服务器处理
   → 反向代理/应用服务器/数据库 → 生成响应

7. HTTP响应
   → 服务器返回HTML（以及CSS/JS的链接）

   === 此时首字节（TTFB）时间点 ===

8. 浏览器解析HTML（Parser）
   → DOM树构建（HTML Parser）

9. 遇到CSS → 并行下载CSS（不阻塞DOM构建）
   → CSSOM（CSS Object Model）

10. 遇到JS → 阻塞HTML解析（除非defer/async）
    → JS执行（JS Engine）

11. DOM + CSSOM → Render Tree（渲染树）

12. Layout（布局/重排）
    → 计算每个节点的位置和大小

13. Paint（绘制）
    → 将节点绘制到图层

14. Composite（合成）
    → 合成图层，GPU加速

15. 页面展示
```

**关键时间点：**
- **TTFB（Time To First Byte）**：从发请求到收到第一个字节，衡量服务器响应速度
- **DOMContentLoaded**：HTML解析完成，DOM树构建完毕
- **Load**：所有资源（CSS/JS/图片）加载完成
- **First Paint / First Contentful Paint**：首次有内容绘制

### 追问方向
- **DNS预解析**：` <link rel="dns-prefetch">` 如何让DNS解析提前？
- **TCP预连接**：`preconnect` 如何提前建立TCP连接？
- **渲染阻塞**：CSS和JS谁阻塞渲染？defer和async的区别？

### 避坑提示
- DNS解析在TCP连接建立**之前**——但现代浏览器会并行做DNS和TCP
- JS阻塞HTML解析是**必须的**，因为JS可能修改DOM
- CSS不会阻塞DOM构建，但会阻塞渲染（因为要等CSSOM）

---

## 16. TCP粘包半包：分包策略、Netty处理

### 题目
面试官：TCP粘包是什么？应用层如何处理？Netty是怎么解决的？

### 核心答案
**TCP粘包/半包的本质：**
- TCP是**字节流**协议，没有消息边界
- 应用层写入的数据可能被**合并**（粘包）或**拆分**（半包）
- 根本原因：**TCP只保证顺序和可靠，不保证应用层消息边界**

**粘包原因：**
- 发送方写入数据小于TCP缓冲区大小，合并发送
- 接收方读取数据时没有一次读完

**半包原因：**
- 发送方数据大于TCP缓冲区，分拆发送
- 一次recv()只读取了部分数据

**分包策略（应用层解决方案）：**

| 策略 | 原理 | 适用场景 |
|------|------|----------|
| 固定长度 | 每个消息固定N字节，不够补0 | 简单，但浪费带宽 |
| 分隔符 | 消息尾部加特殊分隔符（如`\n`） | 简单，但数据内容不能包含分隔符 |
| 长度字段 | 头部记录后续数据长度 | 最通用，推荐使用 |
| 消息格式 | Length + Type + Data + Checksum | 完整协议（Protobuf、Thrift） |

**Netty处理方案：**
```java
// 固定长度分包
ch.pipeline().addLast(new FixedLengthFrameDecoder(1024));

// 分隔符分包（\r\n）
ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, 
    Unpooled.copiedBuffer("\r\n", CharsetUtil.UTF_8)));

// 长度字段分包（自定义协议）
ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(
    1024,  // maxFrameLength
    0,     // lengthFieldOffset（长度字段偏移）
    4,     // lengthFieldLength（长度字段长度）
    0,     // lengthAdjustment
    4));   // initialBytesToStrip

// 自定义编码器/解码器
ch.pipeline().addLast(new MyMessageCodec());
```

### 追问方向
- **为什么UDP不存在粘包**：UDP是数据报协议，每个datagram天然有边界
- **LengthFieldBasedFrameDecoder参数详解**：offset、length、adjustment怎么配合？
- **Length Preamble**：自定义协议常用格式 `[4字节长度][数据]`

### 避坑提示
- "TCP粘包"是**应用层概念**，TCP本身没有包的概念
- 不是说Netty能消除粘包——Netty提供了**分包工具**，开发者需要正确选择和使用
- 半包问题在高并发下尤其明显：一个TCP连接上同时有多个消息时，必须有明确的消息边界

---

## 17. Socket编程：accept/connect/send/recv、Java NIO SocketChannel

### 题目
面试官：ServerSocket accept()、Socket connect() 分别在三次握手哪个阶段返回？Java NIO的Selector模型是什么？

### 核心答案
**Socket基础API与三次握手/四次挥手的关系：**

| 系统调用 | 底层TCP状态 | 说明 |
|----------|-------------|------|
| `socket()` | — | 创建socketfd |
| `bind()` | — | 绑定地址+端口 |
| `listen()` | LISTEN | 监听，backlog队列生效 |
| `accept()` | — | **从已完成队列取一个连接**，阻塞直到有连接 |
| `connect()` | SYN_SENT → ESTABLISHED | 发起连接（三次握手） |
| `send()/write()` | ESTABLISHED | 发送数据 |
| `recv()/read()` | ESTABLISHED | 接收数据 |
| `close()` | FIN_WAIT → CLOSED | 关闭连接（四次挥手） |

**accept() 在三次握手完成后才返回：**
- 客户端connect()成功 = 三次握手完成 = 连接进入已完成队列
- accept()从已完成队列取走连接

**Java NIO SocketChannel（Reactor模型）：**

```java
// 服务端NIO：单线程Selector
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.socket().bind(new InetSocketAddress(8080));
server.configureBlocking(false);  // 非阻塞
server.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // 阻塞直到有事件就绪
    Set<SelectionKey> keys = selector.selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isAcceptable()) {
            // 处理新连接
            SocketChannel client = ((ServerSocketChannel) key.channel()).accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            // 处理读事件
            SocketChannel ch = (SocketChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(1024);
            int read = ch.read(buf);
            // ...
        }
    }
    keys.clear();
}
```

**关键概念：**
- `OP_ACCEPT`：服务器监听socket可accept
- `OP_READ`：socket可读（收到数据）
- `OP_WRITE`：socket可写（发送缓冲区有空间）
- `OP_CONNECT`：客户端connect完成

### 追问方向
- **阻塞 vs 非阻塞 accept()**：设置non-blocking后，accept()在没有连接时返回什么？
- **epoll相比Selector的优势**：Selector每事件要遍历所有key，epoll直接返回就绪列表
- **多线程Reactor**：主Reactor处理Accept，多个子Reactor处理IO——Netty的线程模型

### 避坑提示
- listen(backlog) 的backlog是**已完成握手队列**的最大长度，不是半连接队列
- accept()是阻塞的，但如果ServerSocketChannel设为非阻塞，则返回null（没有连接时）
- NIO的read()返回0不代表对方关闭连接——返回-1才是

---

## 18. 进程 vs 线程：区别、进程间通信

### 题目
面试官：进程和线程的区别是什么？线程崩溃了进程会崩溃吗？

### 核心答案
**进程 vs 线程的本质区别：**

| 维度 | 进程 | 线程 |
|------|------|------|
| 资源拥有 | 独立地址空间（代码/数据/堆/栈/文件描述符等） | 共享进程资源（地址空间、文件描述符） |
| CPU调度 | 进程是调度的基本单位 | 线程是CPU调度的基本单位 |
| 通信方式 | IPC（管道/消息队列/信号量/共享内存） | 直接读写共享内存（需同步） |
| 创建/销毁开销 | 大（需要复制整个地址空间） | 小（只需创建线程栈和TCB） |
| 崩溃影响 | 其他进程不受影响 | 同一进程的所有线程都崩溃（因为共享地址空间） |
| 切换开销 | 大（切换页表、刷新TLB） | 小（不需要切换地址空间） |

**线程崩溃 ≠ 进程崩溃：**
- 如果是**其他线程**崩溃，不会导致进程崩溃
- 如果是**主线程**崩溃且没有处理，进程会被内核杀死（收到SIGKILL）
- Java中：非主线程抛未捕获异常 → JVM调用`ThreadGroup.uncaughtException` → 主线程不受影响

**Linux下的"线程"：**
- Linux不区分进程和线程，**线程是共享地址空间的轻量级进程**
- `clone()` 时传入 `CLONE_VM` 标志创建线程，否则创建进程

**进程间通信（IPC）：**

| 方式 | 原理 | 特点 |
|------|------|------|
| 管道（Pipe） | 内核缓冲区，单向字节流 | 只能亲缘进程间通信（父子） |
| 命名管道（FIFO） | 有路径名的管道文件 | 可以非亲缘进程通信 |
| 消息队列 | 内核消息链表，按类型获取 | 有消息边界，可按优先级 |
| 信号量（Semaphore） | 计数器，PV原子操作 | 用于进程同步 |
| 共享内存 | 直接映射同一块物理内存 | 最快，裸并发访问需配合信号量 |
| 套接字（Socket） | 网络/本地socket文件 | 最通用，支持跨机器 |
| 信号（Signal） | 异步通知（SIGKILL/SIGSEGV等） | 用于异常处理 |

### 追问方向
- **共享内存的同步**：共享内存本身不提供同步，谁来同步？（信号量 / Mutex）
- **管道 vs 消息队列**：管道为什么只能单向？消息队列如何实现双向？
- **mmap映射文件**：共享内存的一种形式，mmap和shmget的区别？

### 避坑提示
- "线程是轻量级进程"是Linux的视角，Windows/Solaris的线程是真实概念——但面试通常以Linux为准
- 进程崩溃不会影响其他进程，线程崩溃**可能**影响进程（如果是主线程或导致SIGSEGV）
- 管道写端关闭后，读端read()返回0（读到EOF）；读端关闭后，写端会收到SIGPIPE

---

## 19. 线程状态：NEW/RUNNABLE/BLOCKED/WAITING/TIMED_WAITING/TERMINATED

### 题目
面试官：画一个Java线程状态转换图，BLOCKED和WAITING的区别是什么？

### 核心答案
**Java线程六状态（`java.lang.Thread.State`）：**

```
                    ┌─────────────────────┐
                    │       NEW            │  (创建，start()未调用)
                    └──────────┬──────────┘
                               │ start()
                               v
                    ┌─────────────────────┐
                    │     RUNNABLE         │  (就绪+运行，JVM眼中都在RUNNABLE)
                    │ (READY + RUNNING)    │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          v                    v                    v
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│    BLOCKED      │  │    WAITING      │  │ TIMED_WAITING   │
│ (等待synchronized)│  │  (wait()/join())│  │ (sleep/to wait) │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         v                    v                    v
         └──────────┌─────────┴────────────────────┘
                    v
         ┌─────────────────────┐
         │     TERMINATED       │  (run()执行完毕)
         └─────────────────────┘
```

**RUNNABLE的内部细分（面试常考）：**
- Java的RUNNABLE = **就绪状态（Ready）** + **运行状态（Running）**
- 状态转换：JVM调度 ↔ 操作系统调度
- 对OS来说RUNNABLE都是"可运行"，JVM内部区分ready和running

**BLOCKED vs WAITING 的核心区别：**

| 状态 | 进入条件 | 唤醒方式 | 示例 |
|------|----------|----------|------|
| BLOCKED | 等待**synchronized锁** | 获得锁 | `synchronized`块 |
| WAITING | 等待**其他线程执行完**（无超时） | notify/notifyAll/interrupt | Object.wait(), Thread.join() |
| TIMED_WAITING | 等待（有超时） | 超时/ interrupt | Thread.sleep(n), wait(n), join(n) |

**WAITING的典型场景：**
- `Object.wait()`：等待其他线程notify
- `Thread.join()`（无参数）：等待目标线程结束
- `LockSupport.park()`：等待获取许可证

**TIMED_WAITING的典型场景：**
- `Thread.sleep(n)`
- `Object.wait(n)`
- `Thread.join(n)`
- `Lock.tryLock(n, TimeUnit)`

### 追问方向
- **BLOCKED的线程占用CPU吗**：不占用，BLOCKED线程在等待锁，不被调度
- **WAITING状态如何避免死锁**：park/unpark机制、Condition.await()
- **线程状态和操作系统线程状态的关系**：Java的NEW/WAITING/TIMED_WAITING在OS层面是什么？

### 避坑提示
- Java线程状态和OS线程状态不是一一对应的——尤其RUNNABLE在OS层面只有Running和Ready
- BLOCKED是**主动请求锁失败**的状态，不是等待I/O
- 调用sleep()进入TIMED_WAITING，但**不释放已持有的锁**——这是sleep和wait的根本区别

---

## 20. 进程调度算法：FCFS/SJF/时间片轮转/优先级/多级反馈队列

### 题目
面试官：SJF调度算法有什么问题？Linux用的是哪种调度算法？

### 核心答案
**常见调度算法：**

| 算法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| FCFS（先来先服务） | 按到达顺序 | 简单，无偏见 | 短作业等长作业（Convoy effect） |
| SJF（最短作业优先） | 选下次CPU执行时间最短的 | 平均等待时间最优 | 饥饿（长作业永远等）、需要预估未来 |
| 时间片轮转（Round Robin） | 每个进程执行一个时间片 | 公平，响应快 | 上下文切换开销，吞吐量低 |
| 优先级调度 | 优先级高的先执行 | 重要任务优先 | 饥饿，低优先级可能饿死 |
| 多级反馈队列（MLFQ） | 多队列+优先级+时间片动态调整 | 兼顾响应时间和吞吐量 | 实现复杂 |

**Linux调度器（O(n) → O(1) → CFS）：**
- **O(n)调度器**（2.4）：每次调度遍历所有进程，n为进程数
- **O(1)调度器**（2.6早期）：每个优先级一个队列，选红黑树最左节点，固定时间
- **CFS（Completely Fair Scheduler）**（2.6.23+）：
  - 用红黑树按**vruntime（虚拟运行时间）**排序
  - 目标：每个进程获得等比例的CPU时间
  - `vruntime += delta_exec * (NICE_0_LOAD / weight)`
  - weight由nice值决定，nice越大weight越小

**CFS的调度延迟（调度周期）：**
- 不是固定时间片，而是 `sched_latency_ns / nr_running`
- 默认 `sched_latency_ns = 6ms`
- 10个进程 → 每个进程每6ms最多运行0.6ms
- 最小粒度 `min粒度 = 0.75ms`，防止频繁切换

**多级反馈队列（MLFQ）示例：**
```
队列0（最高优先级）→ 时间片=8ms
队列1（中优先级）  → 时间片=16ms  
队列2（低优先级）  → FCFS（时间片很长或无限）

规则：
1. 新进程进入队列0
2. 队列0时间片用完 → 降到队列1
3. 队列1时间片用完 → 降到队列2
4. 队列2用完 → 回到队列1
```

### 追问方向
- **实时调度**：SCHED_FIFO、SCHED_RR、Linux如何保证硬实时？
- **CFS的红黑树**：vruntime最小在树最左，插入/删除是O(log n)
- **Linux load average**：1分钟/5分钟/15分钟的负载分别反映什么？

### 避坑提示
- SJF的"最短"指的是**下一次CPU burst时间**，不是总执行时间
- 时间片太短 → 上下文切换开销大；太长 → 响应差（不可兼得）
- CFS的"公平"是**比例公平**，不是每个进程同等时间——是nice值加权的CPU比例

---

## 21. 死锁：四个必要条件、预防/避免/检测/解除、银行家算法

### 题目
面试官：死锁的四个必要条件是什么？银行家算法是如何避免死锁的？

### 核心答案
**死锁的四个必要条件（Cohen's Conditions）：**
1. **互斥条件**：资源只能被一个进程占用
2. **持有并等待**：进程持有资源的同时请求其他资源
3. **不可抢占条件**：已分配资源不能被强制抢走
4. **循环等待条件**：存在一个进程→资源的循环链

**四者缺一不可，全部满足才死锁。破坏任一条件即可预防死锁。**

**死锁处理策略：**

| 策略 | 做法 | 代价 |
|------|------|------|
| 预防（Prevention） | 破坏四个条件之一 | 资源利用率低，实现复杂 |
| 避免（Avoidance） | 运行时检测安全状态（银行家算法） | 有额外判断开销 |
| 检测（Detection） | 允许死锁发生，定期检测并解除 | 定时检测开销 |
| 解除（Recovery） | 杀死进程/回滚/资源抢占 | 可能导致数据不一致 |

**银行家算法（Banker's Algorithm）：**
- 核心：每次资源分配前，检查分配后系统是否仍处于**安全状态**
- 安全状态：存在一个**安全进程序列** `<P1, P2, ..., Pn>`，每个进程的资源需求都能被满足
- 算法步骤：
  ```
  1. 计算Allocation、Max、Need矩阵
  2. Work = Available
  3. 找Need[i] ≤ Work的进程 → 完成后Work += Allocation[i]
  4. 重复直到所有进程完成或找不到 → 判断是否安全
  ```

**解除死锁的方法：**
- 撤销进程（Kill）：杀死环路中某个进程
- 资源剥夺：挂起进程，剥夺其资源
- 回滚：检查点回滚（需要进程检查点机制）

### 追问方向
- **哲学家就餐问题**：死锁经典问题，如何解决？（资源排序、允许最多4人同时拿筷子、 asymmetry）
- **活锁（Livelock）**：两个线程一直谦让但都没进展——和死锁的区别？
- **死锁检测频率**：多久检测一次？开销和死锁概率如何权衡？

### 避坑提示
- "破坏循环等待条件"最常用——通过**资源分配顺序排序**（所有进程按固定顺序请求锁）
- 银行家算法是**死锁避免**，不是死锁检测——两者区别要分清
- 死锁检测可以在每次资源分配时做，也可以定时做

---

## 22. 内存管理：分页/分段、页面置换算法（FIFO/LRU/Clock）、页面抖动

### 题目
面试官：FIFO页面置换算法为什么Belady异常？LRU如何用有限硬件实现？

### 核心答案
**分页 vs 分段：**

| 维度 | 分页 | 分段 |
|------|------|------|
| 划分方式 | 固定大小页（4KB等） | 变量长度段（代码段/数据段/堆/栈） |
| 逻辑地址 | [页号, 页内偏移] | [段号, 段内偏移] |
| 物理地址 | [页框号, 页内偏移] | [基址+段内偏移] |
| 碎片问题 | 内部碎片（页内未用完） | 外部碎片（段间空闲） |
| 共享 | 按页共享（代码可共享） | 按段共享（需要段边界对齐） |
| 保护 | 页级权限控制 | 段级权限（R/W/X） |

**页面置换算法：**

| 算法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| FIFO | 淘汰最早进入的页面 | 简单 | Belady异常，命中率不一定随帧数增加 |
| LRU（最近最少用） | 淘汰最久未使用的 | 命中率接近OPT | 实现成本高（需要时间戳或栈） |
| Clock（第二次机会） | FIFO + 引用位，引用位=1则给第二次机会 | 近似LRU，成本低 | 不考虑使用频率 |
| LFU（最不经常使用） | 淘汰访问次数最少的 | 考虑频率 | 对热点数据友好，但实现复杂 |

**Belady异常（Belady's Anomaly）：**
- 现象：增加物理页帧数，缺页率反而上升
- 原因：FIFO的页面进入顺序与未来访问顺序不一致
- 例外：FIFO会出现，LRU不会出现（LRU有栈属性）

**Clock算法（改进的FIFO）：**
- 每个页面有一个**引用位（R）**和**修改位（M）**
- 淘汰时：扫一遍页面，R=1则清0继续，R=0则淘汰
- 第二次扫描才能决定是否淘汰，给了"第二次机会"

**页面抖动（Thrashing）：**
- 进程频繁缺页，CPU时间大量用于页面置换
- 原因：物理内存不足，进程工作集 > 物理内存
- 解决：增加物理内存、减少并发进程数、合理的页面置换算法

### 追问方向
- **工作集模型（Working Set）**：进程的活跃页面集合 = 避免抖动的关键
- **LRU近似实现**：用双向链表维护access时间——但硬件开销大；如何用硬件辅助？
- **NUMA架构下的页面置换**：本地内存和远程内存的代价不同

### 避坑提示
- 不是说"分页比分段好"——两者结合（段页式）最常见
- Belady异常只发生在FIFO上，LRU/Clock都不会
- "抖动"和"颠簸（thrashing）"是同一概念

---

## 23. 虚拟内存：swap空间、MMU、TLB命中、段页式管理

### 题目
面试官：虚拟内存解决了什么问题？TLB Miss后MMU怎么找物理地址？

### 核心答案
**虚拟内存的核心价值：**
1. **地址空间隔离**：每个进程有独立虚拟地址空间，互不干扰
2. **物理内存抽象**：程序可以使用比物理内存更大的地址空间
3. **内存共享**：共享库/代码页可以映射到多个进程的虚拟空间
4. **按需分配**：页面按需加载，不是一次性全部加载

**MMU（Memory Management Unit）地址转换流程：**

```
虚拟地址（VA） → MMU → 物理地址（PA）

分页模式：
  VA = [VPN（虚拟页号）, offset（页内偏移）]
  PA = [PFN（物理页框号）, offset（相同）]
  
  MMU查页表 → 获取PFN → 组合offset → PA
  页表项(PTE)：[PFN, 有效位, 访问权限, 脏位, 使用位]

TLB（Translation Lookaside Buffer）：
  MMU内部的硬件缓存（最近使用的页表项）
  TLB命中 → 直接获取PFN，不需要查页表
  TLB miss → 查页表 → 更新TLB → 获取PFN
```

**TLB命中/未命中：**
- TLB是CPU内部的高速缓存（16-256项，典型）
- TLB miss开销：数十到数百个时钟周期
- 现代CPU有**L1 TLB**（指令/数据分离）、**L2 TLB**

**段页式管理：**
```
虚拟地址 → 段号 → 段表（段基址+段长） → 得到线性地址
线性地址 → 页号 → 页表 → 得到物理页框号
物理地址 = 页框号 + 页内偏移
```
- 段式管理：程序按逻辑段组织（代码/数据/堆/栈）
- 页式管理：物理内存按固定大小页框分配
- **段页式 = 两者结合**，先段后页
- Linux只用了分页，**没有使用段式管理**（段机制存在但基址=0）

**Swap空间：**
- 当物理内存不足，将不活跃页面换出到磁盘swap分区
- swap读写比物理内存慢**10000倍以上**
- `vm.swappiness` 参数控制使用swap的倾向（0-100，越高越积极换出）

### 追问方向
- **多级页表**：为什么需要多级？优点？Linux几级页表（48位虚拟地址→4级）？
- **大页（HugePages）**：Transparent HugePages vs 标准HugePages，解决什么问题？
- **TLB shootdown**：多核CPU修改页表时，如何刷新其他CPU的TLB？

### 避坑提示
- TLB miss不是"查内存页表"，是"查CPU缓存/内存中的页表"——MMU硬件做这件事
- Linux的虚拟地址空间是：用户空间（0-3GB）+ 内核空间（3-4GB）——32位系统
- swap空间用完会导致OOM Killer杀掉进程——不是系统崩溃，但会丢数据

---

## 24. Linux进程间同步：互斥锁、信号量、PV操作、生产者消费者

### 题目
面试官：用信号量实现一个生产者消费者模型，互斥锁和信号量有什么区别？

### 核心答案
**互斥锁（Mutex）vs 信号量（Semaphore）：**

| 维度 | Mutex | Semaphore |
|------|-------|-----------|
| 用途 | 保护**一个**资源的互斥访问 | 控制**多个**资源的并发访问 |
| 等待方式 | 阻塞直到获得锁 | PV原子操作，计数>0才能减 |
| 所有权 | 有（只有持有者能释放） | 无（谁都可以V） |
| 初始值 | 1（二进制信号量） | N（资源数量） |

**信号量的PV操作（荷兰语：Passeren=通过，Vrijgeven=释放）：**
```c
struct semaphore {
    int value;         // 资源计数
    wait_queue_head_t q; // 等待队列
};

void P(struct semaphore *s) {   // wait()
    atomic_dec(&s->value);
    if (s->value < 0) {
        // 加入等待队列，阻塞
    }
}

void V(struct semaphore *s) {   // signal()
    atomic_inc(&s->value);
    if (s->value <= 0) {
        // 唤醒等待队列中的一个进程
    }
}
```

**生产者消费者模型（伪代码）：**
```c
#define N 100
struct semaphore mutex = 1;   // 互斥访问缓冲区
struct semaphore empty = N;   // 空槽数量
struct semaphore full = 0;    // 满槽数量

void producer() {
    while (1) {
        item = produce();
        P(&empty);      // 等空槽
        P(&mutex);       // 进入临界区
        put_item(item);
        V(&mutex);       // 离开临界区
        V(&full);        // 增加满槽
    }
}

void consumer() {
    while (1) {
        P(&full);        // 等有产品
        P(&mutex);       // 进入临界区
        item = get_item();
        V(&mutex);       // 离开临界区
        V(&empty);       // 增加空槽
        consume(item);
    }
}
```

**Linux中的同步原语：**
- **Mutex（互斥锁）**：`pthread_mutex_t`，支持进程/线程间
- **Semaphore（信号量）**：`pthread_mutex_t`，计数信号量
- **Spinlock（自旋锁）**：忙等待，锁持有时间极短时使用（内核）
- **RWLock（读写锁）**：读多写少场景，写时独占
- **Barrier（屏障）**：等待所有线程到达后再继续

### 追问方向
- **Futex（Fast Userspace Mutex）**：Linux用户态互斥锁如何工作？为什么要先自旋再阻塞？
- **乐观锁 vs 悲观锁**：CAS（Compare-And-Swap）是乐观锁实现，适用于读多写少
- **RCU（Read-Copy-Update）**：写时复制，读者无需加锁

### 避坑提示
- `P(&mutex)` 和 `V(&mutex)` 的顺序很重要——**先P资源，再P互斥**，顺序反了会死锁
- 自旋锁在用户态可以用CAS实现，但Linux内核中的自旋锁有禁用抢占的考量
- futex结合了自旋（短等待）和阻塞（长等待），避免无意义的上下文切换

---

## 25. I/O模型：阻塞/非阻塞/多路复用/异步I/O，epoll高效原因

### 题目
面试官：select/poll/epoll三者的区别是什么？epoll为什么高效？

### 核心答案
**五种I/O模型（Unix网络编程）：**

| 模型 | 等待数据就绪 | 数据复制 | 阻塞点 |
|------|-------------|----------|--------|
| 阻塞I/O（Blocking） | 阻塞 | 阻塞 | recv() |
| 非阻塞I/O（Non-blocking） | 轮询（忙等） | 阻塞 | recv() |
| I/O多路复用（select/poll/epoll） |阻塞（但在多路上） | 阻塞 | select() |
| 信号驱动I/O（SIGIO） | 不阻塞（SIGIO通知） | 阻塞 | 无（异步） |
| 异步I/O（POSIX AIO / io_uring） | 完全不阻塞 | 不阻塞（内核完成） | 无 |

**select的问题：**
- **fd数量限制**：默认1024（`FD_SETSIZE`）
- **每次调用都要把所有fd从用户态拷贝到内核态**
- **线性扫描所有fd**（O(n)），即使只有1个fd就绪也要扫描全部
- fd_set不可重用，每次都要重新设置

**poll的改进：**
- 用`pollfd`数组代替fd_set，**没有fd数量限制**（但受制于系统ulimit）
- 同样是线性扫描
- 解决了select的fd数量限制，但没解决O(n)扫描问题

**epoll的高效原因（三板斧）：**

1. **红黑树存储文件描述符**（O(log n)增删查）
   - `epoll_create()` 创建eventpoll结构，内含红黑树
   - `epoll_ctl(ADD)` 将fd加入红黑树
   - `epoll_ctl(DEL)` 从红黑树删除

2. **回调就绪列表**（O(1)通知）
   - fd就绪时，**内核回调**将fd放入rdlist（就绪列表）
   - 不需要线性扫描

3. **只返回就绪fd**（O(k)，k=就绪数）
   - `epoll_wait()` 返回rdlist内容（就绪的fd）
   - 不复制未就绪的fd到用户态

**epoll两种触发模式：**
- **LT（Level Triggered，默认）**：状态未改变就一直通知；简单但可能重复通知
- **ET（Edge Triggered）**：状态改变才通知一次；高效但编程复杂，需要一次性读完所有数据

**epoll + Reactor模型（高性能服务器）：**
```c
int epfd = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN, .data.fd = listenfd };
epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);

while (1) {
    int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == listenfd) {
            // 处理新连接
        } else if (events[i].events & EPOLLIN) {
            // 处理读事件
        }
    }
}
```

**异步I/O（io_uring，Linux 5.1+）：**
- 真正的异步I/O：提交请求后立即返回，内核完成后通知
- 两条环形缓冲区：Submission Queue（SQ）和 Completion Queue（CQ）
- 零拷贝（io_uring可支持）

### 追问方向
- **epoll惊群问题（Thundering Herd）**：多个进程/线程等待同一个fd，只有一个被唤醒？Linux如何解决？
- **LT vs ET的编程差异**：ET模式下recv()必须循环直到读完，否则不会再通知——为什么？
- **io_uring vs epoll**：异步I/O和事件驱动的本质区别？

### 避坑提示
- epoll是**事件通知机制**，不是异步I/O——数据还是要同步读写
- 说"epoll是异步的"是错误的；io_uring才是真正的异步I/O
- 边沿触发（ET）模式下，**一次性必须读完所有数据**，否则剩下的数据不会再次触发EPOLLIN——这是ET编程中最容易踩的坑

---

*本文档共25道高频面试题，涵盖计算机网络与操作系统核心知识点。建议配合项目实战经验准备面试答案。*
