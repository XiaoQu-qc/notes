### 1.AQS的核心设计哲学

乐观尝试：在任何可能的地方尝试快速获取
减少阻塞：尽可能避免线程进入完全阻塞状态
分层控制：外层粗粒度控制，内层细粒度控制
在正式加入队列之前，多次尝试快速获取，给足机会（直接cas改变state， setExclusiveOwnerThread(current)）

Node head被修饰为volatile，确保可见性，比如初始化方法tryInitializeHead()方法中，线程A 和B可能同时进入该方法初始化队列，如果A对head已经做了初始化，B需要立刻看到所以head用volatile修饰

### 2.回顾lock（）方法的调用过程，AQS的关键数据结构Node以及成员变量
