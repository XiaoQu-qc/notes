### 1实现见D/Algotithm

### 2.hash表
zset还维护一个hash表结构，key为元素的字符串，value为score，以O（1）查找某个元素的score

### 3.为什么使用跳表而不是其他数据结构
<img width="583" height="489" alt="638ce7647bbe86e187a0c3083f714131" src="https://github.com/user-attachments/assets/a9113628-52f5-4c56-95bc-998565aea878" />

### 4.和实现有点不同的地方

```
typedef struct zskiplistNode {
    sds ele;                      // 成员字符串
    double score;                 // 分值
    struct zskiplistNode *backward; // 后退指针（只有 L1）
    struct zskiplistLevel *level[];  // 柔性数组，长度 = 实际随机层数
} zskiplistNode;

struct zskiplistLevel {
    struct zskiplistNode *forward; // 本级后继
    unsigned int span;             // 跨越节点数，用于 RANK 命令
};

```
比如zrange 要找到排行第5的数据，仍然是从最高层索引开始找，如果没有span，不知道现在跳到第几名了
