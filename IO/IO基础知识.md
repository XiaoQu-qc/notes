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
上面没修改过的代码，只能对一个clinetIO,一个客户端连接guo
---
