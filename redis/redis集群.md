### 1.若哈希槽不是由自身节点负责，就返回MOVED重定向，在redis cluster中，谁来计算key的槽位，谁来定位对应的节点或者说确定发送给哪个节点
而集群节点的slot槽表是不断变化的，定位节点
客户端本地有一份固定的槽表（slot → (ip,port)），来源：
第一次连接时 CLUSTER SLOTS 或 CLUSTER NODES 返回；
后续遇到 MOVED/ASK 时增量更新

### 2.redis集群的广播实际上是挑选N个节点进行单播，gossip，为什么不用广播？
1. 技术限制：广播包无法跨越网络边界
 3. 可扩展性（Scalability）问题：无法支撑大规模集群
这是最核心的原因。

广播的消息量与集群规模呈 O(n²) 关系：虽然不是严格的数学上的O(n²)，但可以这么理解：一次集群状态变化，就需要发送 n-1 个消息（一个节点发给其他所有节点）。如果集群有1000个节点，一次变更就是999次发送。这被称为“全互联”（All-to-All）通信，其开销随着节点数量增加呈平方级增长，是无法承受的。
Gossip的消息量与集群规模呈 O(n) 关系：Gossip协议每个周期只固定发送固定数量（如c条）消息。无论集群是10个节点还是1000个节点，每个节点每秒钟都只和大致相同数量的邻居通信。总消息量是 n * c，是线性的，这使得它可以轻松扩展到数千个节点。
4. 网络环境的现实：拥抱而非对抗“不可靠”

对网络分区（Network Partition）的鲁棒性：假设网络发生分裂，集群被切成两半。广播无法穿过分区。而在分区内部，Gossip协议依然可以正常工作，维持小集群的内部状态一致。当网络恢复时，来自两个分区的节点通过Gossip通信能够很快地发现对方并同步状态，最终合并成一个一致的集群。
容忍节点故障：即使部分节点临时下线，当它们重新上线后，只需要与任意一个活着的节点进行一次Gossip握手，就能迅速获取到错过期间的所有集群变更。而广播模式下，掉线的节点就永远错过了那条广播。

### 3.节点通信或者说gossip握手的通用步骤
时间轴：[节点X下线]------[错过若干变更]------[节点X重新上线]

节点X重新上线后的步骤：
1. 启动 启动Gossip协议线程（本来就在运行）
2. 从本地配置中读取已知的集群节点列表
3. 随机选择一个存活节点Y建立连接
4. 发送PING消息，其中包含：
   - 节点X自己的基本信息
   - 节点X当前认为的集群状态（可能是过时的）
   
5. 节点Y回复PONG消息，其中包含：
   - 节点Y的基本信息  
   - 完整的/更新的集群拓扑信息
   - 所有的Slot映射变更历史
   - 其他节点的最新状态
  
### 4.
<img width="481" height="313" alt="c02477512a941c97799c70b1937c4b35" src="https://github.com/user-attachments/assets/499d931d-d3e8-4147-a210-582d66ffbfc3" />

### 5.clusterNode数据结构
<img width="448" height="240" alt="image" src="https://github.com/user-attachments/assets/df0e4c45-4407-4e50-aea6-df8d43c18ccc" />

### 6.槽表变更（故障转移、重新分片）会走 集群总线 + epoch 版本号，确保所有节点先达成一致后才会对外生效，因此任一节点给出的 MOVED 信息都全局有效。
### 7.
```
typedef struct clusterState {
   clusterNode *myself;  /* 当前节点的clusterNode信息 */
   ....
   dict *nodes;          /* name到clusterNode的字典 */
   ....
   clusterNode *slots[CLUSTER_SLOTS]; /* slot 和节点的对应关系*/
   ....
} clusterState;

typedef struct clusterNode {} 这个nodes就是某个redis节点视角下它维护的全局状态
```

### 8.redis 哨兵架构中，客户端订阅哨兵的switch-master频道感知主节点的切换，那么在redis cluster架构中，redis客户端怎么知道主节点掉线，并且连接到顶替的主节点的呢
<img width="501" height="394" alt="37ffee230b1cb5ccdadec9c437d258b1" src="https://github.com/user-attachments/assets/a0d26491-d90a-4d29-8597-d46a268477c4" />

### 9.redis客户端和服务器之间建立的tcp连接是长连接吗，redis集群架构下，每次查询对应一个redis节点，那么岂不是要和所有的redis建立tcp长连接？


### 10.随机周期性发送PING消息，每次挑选更长时间未收到其 PONG 消息的节点
