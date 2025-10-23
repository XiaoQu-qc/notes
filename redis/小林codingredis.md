### 1.几个缓冲区的作用
复制积压缓冲区（repl_backlog）和每个从节点的 output buffer主从复制用到的
server.aof_buf缓冲区，这个是主服务器收到写命令后，要把写命令根据RESP协议序列化后存放在这，然后wrtie（）命令写到page cache内存缓冲区，再由fsync强制刷到磁盘

还有一个replication buffer
