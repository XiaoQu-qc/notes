### 1.BIO性能上的问题
连接数和线程数量相同，虽然io大多数时间都是在阻塞，但是这么多线程对象创建资源消耗也很恐怖

### 2.CS模式下的IO模型
<img width="576" height="296" alt="cdb3952fb97710d67802f3d5ca082005" src="https://github.com/user-attachments/assets/a157875e-e94d-4048-a803-016c3b2be5a2" />

### 3.
<img width="646" height="313" alt="822e52c1f7cad44949d65871e18ac2d9" src="https://github.com/user-attachments/assets/76b817c0-aa84-43be-b69c-ee522af0db37" />

##### （1）
体现IO阻塞的地方是在br.readLine（）。server监听9999端口，等待client端sokect连接，即使连接成功了，client端数据可能还没有发送，所以server在readLine处IO阻塞，直到数据被拷贝到用户内存空间（数据准备好），该线程从阻塞状态回到可调度状态
数据准备好在这里是指，client发送后，并且server内核对发来的数据处理拷贝到用户态内存空间。

##### (2)
if((msg = br.readLine()) != null)if改为while同时client也改为while就实现了cs的多发多收

###### （3）服务端连接多个客户端，引入多线程的机制
<img width="679" height="384" alt="895c18f389024a13852482d5e6901446" src="https://github.com/user-attachments/assets/29ec1cf2-2172-471e-9616-1e0b41ceea7e" />
上面没修改过的代码，只能对一个clinetIO.一个客户端连接过来，server创建一个thread处理与该客户端的IO任务，性能上的问题见1.写的

---
### 4.为了解决要开启过多线程的问题，引入了线程池来解决，正好线程池的优点得以发挥
其中可能会涉及到线程的并发，线程池要用单例对象，并且要防止并发重复创建

### 5.BIO模式下的端口转发
进一步如果server处理某一个clientsocket线程拿到了所有的注册（在线）的client端的socket，它可以通过其他socket的output流转发出去,这就是网络聊天的基本原理

### 6.可以利用一个socket实现C和S的全双工通信
<img width="547" height="141" alt="725eb1a4076830371965feeea2ce6f8b" src="https://github.com/user-attachments/assets/a5cd692d-02a0-493a-8ce0-eb2b6fcc023a" />

### 7.dos.write和dis.read各自的阻塞原因
<img width="549" height="279" alt="ec7312f03f56e447b2f3bb2c66497e00" src="https://github.com/user-attachments/assets/2461ecd8-ab94-46cd-873d-2897912a42c9" />
<img width="589" height="462" alt="48bd31a575581b3f0e3f310cd95760d9" src="https://github.com/user-attachments/assets/c335c6e5-2281-4742-a3d8-340722f548fd" />
如上图，server端socket 输入流阻塞的原因是TCP receive buffer为空引起的阻塞，client端输出流阻塞的原因是TCP send buffer满

### 8.TCP send buffer 和代码中定义的buffer不是一回事，前者是内核态的buffer，后者是用户态的buffer
read是从TCP send buffer读到用户态的内存中，write操作相反

### 9.内核read操作
<img width="441" height="211" alt="79cd39c56242fc835dea2cdd6d276aa8" src="https://github.com/user-attachments/assets/532599a6-27aa-4839-85a5-f09b9b017b77" />

### 8.问题与解答
User: client端中的while代码块每次阻塞读取满一个buffer后，解除阻塞，-》得到调度重新运行，然后socket输出流把buffer中的数据阻塞写出，此时server端的阻塞的read操作收到了刚刚的bufffer中的内容。。。我说的对吗。在传输整个文件的过程中，阻塞是断断续续的对吗
Kimi: 你理解的大方向基本正确，但可以更精确一些。下面按“一次循环”把阻塞点、唤醒时机和 CPU 调度关系拆开说：

------------------------------------------------
一次 while 循环的时间线（BIO，默认 TCP 阻塞模式）

1. client 线程  
   `int len = is.read(buffer)`  
   ├─ 本地磁盘数据已到 → 立即返回 len>0  
   └─ 磁盘慢 → 线程在内核置为 `TASK_INTERRUPTIBLE`，**调度器把 CPU 给别的任务**；DMA 完成后唤醒 → 重新抢到 CPU 才继续

2. 紧接着  
   `dos.write(buffer, 0, len)`  
   ├─ 发送窗口足够 → 只是把数据拷进 socket send buffer，立即返回  
   └─ 发送窗口满 → 线程再次阻塞（`TASK_INTERRUPTIBLE`），**CPU 立即被调度走**；等到 TCP ACK 推进窗口后内核唤醒

3. server 线程  
   `int len = dis.read(buffer)`  
   ├─ socket recv buffer 里已有 ≥1 字节 → 立即返回，把可用数据拷满用户 buffer  
   └─ 暂无数据 → 阻塞，**让出 CPU**；当 client 的 TCP 报文到达后，内核把数据塞进 recv buffer 并唤醒 server 线程

------------------------------------------------
关键结论

- 阻塞是**断断续续**的——每轮循环都可能出现 **“磁盘读阻塞”→“写阻塞”→“对端读阻塞”** 三段，但哪一段真阻塞取决于当时缓冲区状态。  
- 只要缓冲区未满/非空，相应 `read/write` 会**立即返回**，此时**不会发生上下文切换**。  
- 一旦真阻塞，内核**立刻** `schedule()` 把 CPU 让给别的就绪任务；被阻塞线程**不再占用时间片**，直到数据到达/窗口打开才被唤醒重新参与调度。

所以：  
“每次阻塞读取满一个 buffer → 解除阻塞 → 写出时又可能阻塞 → server 读再阻塞” 这个链条**在逻辑上成立**，但是否**真让出 CPU** 要看每次系统调用时对应缓冲区是否刚好不能满足，**阻塞点是“按需”出现的，不是必然连续发生**。
User: dos.write和dis.read是一对同步阻塞操作对吗，无论哪一条语句先被执行到，，它都要等另一个语句，然后同步进行数据传输，我理解的对吗
Kimi: 不是“严格一对一同步”的阻塞对。

1. 发送端 `dos.write()`  
   只把数据拷进 **TCP send buffer** 就返回；只要缓冲区有空位，**不必等对方 `read()`**。

2. 接收端 `dis.read()`  
   只要 **TCP recv buffer** 里有数据就可立即返回；**不必等对方正在 `write()`**。

3. 实际现象  
   - 写端连续 `write()` 若干次后，send buffer 满才阻塞；  
   - 读端可能一次性把之前所有数据全读上来；  
   - 也可能写端一次小数据，读端分多次才读完。

因此它们是 **“TCP 缓冲区”** 上的生产/消费关系，**不是同步阻塞对**；  
哪条语句先执行、是否阻塞，完全取决于 **缓冲区空/满状态**，而**不依赖对方当前是否在读写**。
User: 这里while中间，具体操作是怎么样的呢

read操作读满一个buffer，dos.write再把write整体写出？
Kimi: 不是“必须读满”才写，也不是“整体写出”。

while 循环里真正的步骤是：

1. `is.read(buffer)`  
   把**“最多 1024 字节”**读进 `buffer`；  
   返回值 `len` 告诉你**实际读了多少**（可能 1~1024，读到文件尾就返回 -1 退出循环）。

2. `dos.write(buffer, 0, len)`  
   把刚才读到的**前 len 字节**原样送进 TCP 发送缓冲区；  
   写完就进入下一次循环，**缓冲区内容立刻被覆盖重用**。

3. 循环继续  
   重复 1→2→1→2……直到文件读完。

因此：  
- **读多少就写多少**，读不满 1024 也立即写；  
- **不是攒够 1024 才一次性整体发出**；  
- 网络另一端可以**逐批收到**这些数据块，块大小取决于本轮实际 `len`。
User: 你刚刚说“发送端 dos.write(buffer)
只把数据拷进 TCP send buffer 就返回；只要缓冲区有空位，不必等对方 read()。” TCP send buffer和作为参数的buffer是同一个东西吗
Kimi: 完全不是同一块内存。

1. 用户态 buffer  
   你 `new byte[1024]` 出来的数组，位于**Java 堆**（或 DirectBuffer 的堆外内存），属于**进程用户空间**。

2. TCP send buffer  
   内核为这条 socket 维护的**内核态**内存区域，大小由 `net.core.wmem_default` / `wmem_max` 等参数决定，**对用户代码不可见**。

3. `dos.write()` 做的事  
   把用户 buffer 里的内容**复制**到内核的 TCP send buffer，然后立即返回；  
   真正的 TCP 分段、重传、流量控制都由内核在后面处理，**与用户 buffer 生命周期无关**。

4. 关系  
   用户 buffer → 拷贝 → 内核 send buffer → TCP/IP 协议栈 → 网卡  
   写完你的数组就可以被重用或 GC，**不会**被 TCP 直接拿来发送。

一句话：  
**用户 buffer 只是“货源”，send buffer 是“仓库”；写操作=把货搬进仓库，仓库怎么发车跟你无关。**
User: dos.write(buffer)这一步是阻塞的还是非阻塞的，因为你刚刚的描述“发送端 dos.write()
只把数据拷进 TCP send buffer 就返回；只要缓冲区有空位，不必等对方 read()”这句话像是非阻塞
Kimi: `dos.write(buffer)` 在 **BIO（阻塞 I/O）** 模型下是**阻塞的**。

---

### 为什么刚刚说“拷进 TCP send buffer 就返回”？

这是**理想情况**下的行为：  
当 **TCP send buffer 有足够空间**时，内核会**立即完成拷贝**，然后 `write()` 系统调用返回，用户线程继续执行——**这次调用没有阻塞**。

---

### 但它是“**可能阻塞**”的：

当 **TCP send buffer 已满**（例如网络慢、对端接收慢），内核无法立即接收更多数据时：

- `write()` 会**阻塞当前线程**（进入 `TASK_INTERRUPTIBLE` 状态）；  
- 直到 TCP 发送窗口推进、缓冲区有空位，内核才唤醒线程，`write()` 返回。

---

### 总结一句话：

> `dos.write(buffer)` 是 **BIO 的阻塞写**，它**不等待对方 read()**，但**会等待内核 TCP send buffer 有空位**；  
> 缓冲区未满时“看起来像非阻塞”，**本质仍是阻塞调用**——**是否真阻塞取决于当时内核缓冲区的状态**。
User: 那我类推一下，这里server端开启线程处理socket，dis.read(buffer)的阻塞原因也不是因为tcp的数据没准备好，而是因为TCP receive buffer 内容拷贝到用户内存分buffer时因为某些原因的阻塞对吗
Kimi: 不对。  
在 BIO 的 `dis.read(buffer)` 这一行，**唯一能让线程阻塞的点**就是：

> **TCP receive buffer 里目前没有哪怕 1 个字节的数据**  
> → 内核把线程置为 `TASK_INTERRUPTIBLE` 并立即 `schedule()` 出去；  
> 直到对端 TCP 报文到达且通过校验后，内核才把数据放进 receive buffer，再唤醒线程，**真正开始拷贝**。

拷贝过程本身（内核 → 用户 buffer）只做一次内存复制，**耗时只有几微秒**，远不到需要“阻塞”的程度；  
除非你把 buffer 设成几十兆这种极端值，否则**不会因“拷贝慢”而阻塞**。

因此 BIO 的 `read` 阻塞原因**只有一个**：**receive buffer 空，没数据可读**；  
一旦内核端有数据，**拷贝立即完成并返回**，不会因为拷贝动作再额外阻塞。
User: receive buffer 空的原因时tcp网络传输引起的对吗
Kimi: 对，**receive buffer 空的根因就是“TCP 传输还没把数据搬进来”**，细分下去有 3 类典型场景：

1. 网络还在路上  
   报文尚未到达本机网卡 → 内核协议栈自然没有数据可放。

2. 本机已收包但尚未排好序  
   TCP 严格按序号交付；若收到的是「后序」段而前面序号缺失，**会暂存但不提交**到 receive buffer，直到缺口被补齐。

3. 对端压根还没 send / 发送窗口被卡住  
   - 发送方应用没调用 `write()` 或自身 send buffer 满，导致对端 TCP 无法继续发段；  
   - 或者发送方收到「零窗口」通告，被迫停发。

> 总结：  
> receive buffer 空 → 内核无数据可交付 → 阻塞 `read()`；  
> 而“没数据”99% 是因为 **TCP 层面还没收到/还不能交付**，不是内核拷贝慢。
User: 你刚刚说“直到对端 TCP 报文到达且通过校验后，内核才把数据放进 receive buffer，再唤醒线程，真正开始拷贝”。可是结合之句话“等待数据从内核空间拷贝到用户空间的这段时间里，线程依然是阻塞的，直到在内核把数据拷贝到用户空间”。我认为应该是把receive buffer拷贝到用户态内存之后才唤醒线程，因为此时用户态内存数据没有准备好，
Kimi: 内核的“阻塞/唤醒”粒度是**“数据是否就绪”**，而不是**“拷贝是否完成”**。

1. 数据到达 → 内核先把它放进 receive buffer（此时线程仍在睡眠）。  
2. **只要 receive buffer 里出现 ≥1 字节可读**，内核立即把线程标记为 **RUNNABLE** 并插入就绪队列；  
   线程被调度回来后，**再做 memcpy** 把数据从内核空间复制到用户提供的 buffer，然后系统调用返回。  
   这段拷贝**极短（微秒级）**，且**不重新阻塞**——只是占用 CPU 的常规指令序列。

因此：
- 阻塞点是 “**没数据**” → 线程睡；  
- 唤醒点是 “**有数据可读**” → 线程醒；  
- 醒后把数据拷给用户 buffer 是**立即完成的尾巴工作**，不会再让线程进入睡眠状态。

> 所以“直到拷贝完成才唤醒”是**误解**；唤醒发生在**数据就绪**时，拷贝只是唤醒后一瞬间的 CPU 操作。
点击链接查看和 Kimi 的对话 https://www.kimi.com/share/d3md6kp8bjvtau6mm8ng
