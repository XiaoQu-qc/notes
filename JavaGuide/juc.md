### 1.synchronized 块和实例方法
```
public void increment() {
        synchronized (this) { // 使用 synchronized 块
            count++;
        }
    }
```
和
```
public synchronized void increment() {
 count++;
}
```
效果是一样的，但是推荐使用 synchronized 块，因为它提供了更细粒度的控制，使得在方法中只把可能发生并发冲突的代码加锁，而如果把整个方法加锁，会影响并发的程度
synchronized只能作用于对象上，synchronized加在成员方法上相当于synchronized (this)，加在静态方法上相当于synchronized (this.class)也就是对该类的类对象加锁
### 2.双重校验锁实现对象单例（线程安全）
```
public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public  static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```
#### (1) 
```
public  static Singleton getUniqueInstance() {

    if (uniqueInstance == null) {
      uniqueInstance = new Singleton();
    }
    return uniqueInstance;
    }
```
在单线程情况下没问题，单线程情况下同时调用getInstance这个方法肯定不行，因为A,B线程可能前后都通过判断，这样new了两侧
#### （2）去掉第一个判断条件
```
public  static Singleton getUniqueInstance() {
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        return uniqueInstance;
    }
```
是可以的，只是如果单例对象已经存在，也要加锁，会影响并发程度，事实上，如果单例对象存在，直接获取就行，就像读操作一样
#### （3）去掉第二个if条件
```
public  static Singleton getUniqueInstance() {
        if (uniqueInstance == null) {
            synchronized (Singleton.class) {
                    uniqueInstance = new Singleton();
            }
        }
        return uniqueInstance;
    }
```
不行，原因和（1）一样
### 3.有锁的情况下读写和volatile实现无锁读写
有两个线程，一个线程修改资源，一个线程读取资源，理想情况是比如哪怕线程2先启动一点点，那么他应该读到的是修改前的资源，而如果不加锁，有可能读操作没执行完，第一个线程修改完了，这就产生了数据并发的问题。
而我们可以加锁，也可以使用volatile（换一种情况，修改线程先启动）,类似于数据库的并发事务带来的问题中的不可重复读，可能读到另一个并发事务（线程）修改前的数据，也可能读到修改后的数据。
多线程情况下，可以区分这个key是不存在还是value=null吗
if (map.containsKey("key")) {  // 检查
    map.get(key);    // 操作
}else{
sout(没有这个key)
}
if (!map.containsKey("key")) {  // 检查键是否存在
    map.put("key", null);       // 插入 null 值
}

### 4.现代cpu调度策略
现代CPU的调度策略远比“时间片轮转”（Round-Robin）复杂，是否采用时间片轮转取决于操作系统和具体场景。也就是说cpu并不是像课本上学的那样分到时间片就从就绪态变为运行态，时间片用完就再进入就绪态，然后这样频繁的切换。
就是说现在的多线程cpu设计都是基于线程切换的，但是不一定是基于时间片切换。

### 5.jvm线程调度视角下锁发生了什么
“拿不到锁”时线程的状态变化
线程调用 lock()/mutex.lock() 失败 → 内核把它从 运行队列（runqueue） 移到 锁的等待队列（sleep/wait queue），状态设为 TASK_UNINTERRUPTIBLE 或 TASK_INTERRUPTIBLE → 当前 CPU 立即重新调度，挑下一个可运行线程。
此时原线程不再持有 CPU，也不参与调度，直到被唤醒。
“切换”由谁触发
这次切换不是“时间片到期”，而是 线程主动阻塞 触发的。调度器只是响应这一事件，与固定时间片无关。
因此：
拿不到锁 → 线程阻塞 → 立刻让出 CPU → 调度器换另一线程 这一流程成立，属于“事件驱动切换”而非时间片切换。

### 6.cas算法可以减少上下文切换，但真的能提升效率吗，一般都是cas+自旋的情况下，它确实不切换线程，但是它占用着cpu并且做无意义的等待自选？
CAS+自旋在**“临界区极短 + 竞争可控”的场景下能显著减少上下文切换，从而提升效率；
在“临界区长或竞争激烈”的场景下，空转的 CPU 时间反而成为最大开销，应退化为阻塞锁。
因此，现代库/内核都会根据实时统计信息动态选择**自旋还是阻塞，而不是一刀切。

### 7.synchronize 底层monitor实现

### 8.线程池中阻塞队列的阻塞怎么理解
线程池中阻塞队列的阻塞机制解析
线程池中的阻塞队列(BlockingQueue)的"阻塞"特性是其核心功能，主要体现在以下两个方面：


线程池的 execute/submit 永远非阻塞
队列本身不会阻塞提交者；想真正排队就必须让队列足够大，或者自己在外层用信号量/阻塞队列做二次节流。

线程池中阻塞队列的阻塞机制解析
线程池中的阻塞队列(BlockingQueue)的"阻塞"特性是其核心功能，主要体现在以下两个方面：

1. 当队列满时的生产者阻塞
当线程池的工作队列已满，并且线程池中的线程数已达到最大线程数时：提交新任务的线程(生产者)会被阻塞阻塞方式取决于线程池的拒绝策略：
默认情况下(使用ThreadPoolExecutor.CallerRunsPolicy)，提交任务的线程会尝试执行该任务使用BlockingQueue的put()方法时，线程会一直阻塞直到队列有空间
// 示例：使用put方法时的阻塞行为
blockingQueue.put(task); // 如果队列满，调用线程会在此阻塞
2. 当队列空时的消费者阻塞
当工作线程(消费者)从空队列中获取任务时：工作线程会被阻塞，等待新任务到来这种阻塞是自动的，通过BlockingQueue.take()方法实现

阻塞队列的实现原理阻塞队列通常使用以下机制实现阻塞：

ReentrantLock锁：保证线程安全
Condition条件变量：
notFull条件：队列未满时唤醒生产者
notEmpty条件：队列非空时唤醒消费者
实际线程池中的应用
在ThreadPoolExecutor中：

核心线程通过getTask()方法从队列获取任务
使用poll(keepAliveTime, TimeUnit)实现超时等待，避免核心线程永久阻塞
非核心线程在空闲超时后会被回收
理解阻塞队列的阻塞机制对于合理配置线程池参数(如队列容量、线程数等)非常重要，可以避免系统资源耗尽或任务处理效率低下的问题

### 9.线程频繁阻塞/唤醒（网络 I/O、锁竞争）是不是会导致上面说的自愿让出：线程调用 sleep, wait, read, epoll_wait, yield 等主动挂起，网络 I/O、锁竞争分别对应着那个方法的调用呢
| 场景           | 应用层代码写法／API                                                  | 实际挂起线程的系统调用/指令（自愿让出的触发点）                                                                                                                                              |
| ------------ | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **网络 I/O**   | `read()`, `InputStream.read()`, `channel.read()`, `recv()` 等 | 最终落到内核 `read(2)` / `recvfrom(2)`，缓冲区无数据 → 内核把线程状态置为 `TASK_INTERRUPTIBLE`，调度器将其移出运行队列。                                                                                 |
| **显式 sleep** | `Thread.sleep(long)`                                         | JVM→Linux：`nanosleep(2)`；Windows：`Sleep` → 线程进入定时睡眠队列。                                                                                                                |
| **锁竞争**      | `synchronized(obj){}` 或 `ReentrantLock.lock()`               | HotSpot：锁膨胀为“重量级锁”后，竞争失败的线程在 `ObjectMonitor::EnterI` 里调用 `pthread_cond_wait`（Linux）或 `WaitForSingleObject`（Windows）→ 线程挂起；唤醒时由 `pthread_cond_signal` / `SetEvent` 触发。 |
| **显式 wait**  | `obj.wait()`                                                 | 同上，走 `ObjectMonitor::wait` → `pthread_cond_wait`。                                                                                                                     |
| **epoll 等待** | `Selector.select()` / `epoll_wait()`                         | 用户态→内核：`epoll_wait(2)`，无就绪事件 → 线程挂起；事件到达后由软中断/唤醒路径将其重新放入运行队列。                                                                                                         |
| **yield**    | `Thread.yield()`                                             | 内核：`sched_yield(2)`；线程只是让出剩余时间片，仍保持 `TASK_RUNNING`，调度器立即挑下一个线程，**不会进入睡眠状态**。                                                                                          |
### 10.A线程执行的是计算型任务，B线程执行iO型任务，为了充分利用cpu的资源，是不是可以将IO任务通过一些手段变为非阻塞执行的，这样它不会占用cpu，这时A上cpu执行，这么理解对吗
方向对，但细节需要澄清。
阻塞 I/O 与 CPU 的关系
如果 B 线程使用传统的阻塞 I/O（BIO），在等数据时确实会“挂起”——线程被操作系统挂到等待队列，不再占用 CPU 时间片，CPU 可以立刻调度其他线程（例如 A 线程）运行。因此，把 I/O 变成“非阻塞”并不是为了让出 CPU，因为阻塞本身就已经让出 CPU 了。
非阻塞/异步 I/O 的意义
把 I/O 改为非阻塞（NIO）、I/O 多路复用（select/poll/epoll）或异步 I/O（AIO）的真正好处是：
• 减少线程数量：一个线程可以并发处理很多 I/O 事件，不必为每个连接都开一个阻塞线程；
• 降低上下文切换开销：线程少了，切换就少；
• 提高吞吐：当数据到达时，CPU 可以立即被回调/事件处理逻辑使用，而不必重新唤醒沉睡的阻塞线程。
因此
• 如果只有 A（计算）和 B（阻塞 I/O）两个线程，B 阻塞时 CPU 已经能被 A 占用，不存在“浪费”。
• 真正需要把 I/O 做成非阻塞的场景是：高并发、大量连接、大量 I/O 任务，想节省线程、减少切换、提高整体吞吐，而不仅仅是“让出 CPU”。
总结：阻塞 I/O 本身就会让出 CPU；非阻塞/异步 I/O 的价值在于节省线程和减少切换，而不是避免 CPU 被“空占”。

### 11.使用线程池有什么好处
(1)提交任务后一般可以直接使用已经在线程池中创建好的线程，无需等待线程创建
(2)线程池里的线程是可以重复利用的。一旦线程完成了某个任务，它不会立即销毁，而是回到池子里等待下一个任务。这就避免了频繁创建和销毁线程带来的开销。
(3)提高线程的可管理性，防止内存溢出

### 12.reentranlock和thread join ，wait的关系
The returned {@link Condition} instance supports the same
     * usages as do the {@link Object} monitor methods ({@link
     * Object#wait() wait}, {@link Object#notify notify}, and {@link
     * Object#notifyAll notifyAll}) when used with the built-in
     * monitor lock.
<img width="492" height="241" alt="3c764902e73c279b90d3cefccd97df4e" src="https://github.com/user-attachments/assets/2b8dc8fa-2f33-4917-aefd-90687ca589fa" />

synchronized 的实现：是类似的
<img width="550" height="391" alt="c85bffa84b258ef261e2fc7ff4a9e8f7" src="https://github.com/user-attachments/assets/14cc339b-93bc-44a7-9cbb-b8fe7678908e" />

当线程尝试进入 synchronized 块时，如果无法获取锁，它会被放入对象的阻塞队列中。
锁释放后，阻塞队列中的线程会被唤醒，尝试获取锁。
