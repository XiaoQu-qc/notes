### 1.有没有遇到印象比较深刻的问题，是怎么解决的
在更新短链接接口，在一次测试的时候发现如果要修改短链接的gid发现修改不了，原因是，我们t_link库的分库键用的是gid，比如说某个link的gid为1，分到了10号库

传的参数gid是2，想把这条记录的gid改为2，当时忽视了他们分库会分到不同的库中，如果直接由2作为key进行查询的话，他查不到10号库，那肯定不存在这条记录了，如果再gid=1中直接修改为gid=2也不行，gid=2分库后大概率不在原表中

还有就是修改数据库后会出现缓存不一致的问题，查找资料后发现，如果采用同步修改redis，在并发场景下无论先修改redis还是先修改数据库都会出现缓存不一致的问题

解决方案是**在修改数据库之前**直接删除redis

GOTO_SHORT_LINK_KEY这个跳转的缓存肯定是要删除的，因为如果是修改过期时间，并且过期了，不删除这个它可以通过这个缓存正常跳转，这是不对的

同时GOTO_IS_NULL_SHORT_LINK_KEY这个缓存也要删除，如果enablestatus本身为false，紧接着查询一次，(GOTO_IS_NULL_SHORT_LINK_KEY, fullShortUrl缓存就有记录了，我再把enablestatus改为true，如果修改操作不删除缓存，那么接下来还是查询这个fullshort

由这个GOTO_IS_NULL_SHORT_LINK_KEY, fullShortUrl跳转到pagenotefound，而事实上修改过后应该是可以跳转的

### 2.
修改短链接加分布式写锁，短链接统计加分布式读锁，如果统计方法要等写锁释放，再进行统计，这样的话restore方法会等很久，于是如果加了锁trylock，锁住了直接交给延迟队列，直接返回。

由于原本代码跳转逻辑中还会调用统计方法，然而事实上跳转逻辑本身并不应该等统计方法结束，应该把统计方法做成微服务

### 3.nurl.ink:8001/sad3j2  
这是fullshortUrl ，@GetMapping("/{short-uri}")，也是一个跳转请求，服务器收到请求后，将short-uri作为参数找到原始链接并跳转

### 4.
restore只把基本信息丢给消息队列，ShortlinkstatsSaveConsumer在真正消费时（真正对数据库操作）也是要读锁的，虽然消费时是修改，但这里加读锁是ok的，因为即使对同一个fullshorturl消费，数据库是有行锁的，保证了原子性

生产者传递的参数ShortLinkStatsRecordDTO不包括gid，（在ShortlinkstatsSaveConsumer在真正消费时时拿fullShortUrl查shortlink_goto表实时获取，因为此时在读锁范围里，消费时是需要gid的，代码中可以看到）gid 不再存储监控表，短链接修改就不再需要变更监控表大量 gid
