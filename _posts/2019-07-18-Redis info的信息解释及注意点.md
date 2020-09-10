---
layout:     post
title:      Redis info的信息解释及注意点
subtitle:   Redis info的信息解释及注意点
date:       2019-07-18
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - redis
    - redis info
---

> [这里展示效果好](http://river0314.lofter.com/post/1d03f335_efda21c1)

Info是获取单机信息的命令，主要是9 块的信息，也可以单独使用 info server 这样的命令来获取部分信息，一下为 info (all) 的结果：

##### Server(服务器信息)
```conf
redis_version:3.0.0                             #redis服务器版本
redis_git_sha1:00000000                 #Git SHA1
redis_git_dirty:0                                   #Git dirtyflag
redis_build_id:6c2c390b97607ff0    #redis build id
redis_mode:cluster                             #运行模式，单机或者集群
os:Linux 2.6.32-358.2.1.el6.x86_64 x86_64 #redis服务器的宿主操作系统
arch_bits:64                                        #架构(32或64位)
multiplexing_api:epoll                       #redis所使用的事件处理机制
gcc_version:4.4.7                               #编译redis时所使用的gcc版本
process_id:12099                              #redis服务器进程的pid
run_id:63bcd0e57adb695ff0bf873cf42d403ddbac1565  #redis服务器的随机标识符(用于sentinel和集群)
tcp_port:9021                               #redis服务器监听端口
uptime_in_seconds:26157730   #redis服务器启动总时间，单位是秒
uptime_in_days:302                   #redis服务器启动总时间，单位是天
hz:10                               #redis内部调度（进行关闭timeout的客户端，删除过期key等等）频率，程序规定serverCron每秒运行10次。
lru_clock:14359959      #自增的时钟，用于LRU管理,该时钟100ms(hz=10,因此每1000ms/10=100ms执行一次定时任务)更新一次。
config_file:/redis_cluster/etc/9021.conf  #配置文件路径
```

##### Clients(已连接客户端信息)
```conf
connected_clients:1081       #已连接客户端的数量(不包括通过slave连接的客户端)
client_longest_output_list:0 #当前连接的客户端当中，最长的输出列表，用clientlist命令观察omem字段最大值
client_biggest_input_buf:0   #当前连接的客户端当中，最大输入缓存，用clientlist命令观察qbuf和qbuf-free两个字段最大值
blocked_clients:0                  #正在等待阻塞命令(BLPOP、BRPOP、BRPOPLPUSH)的客户端的数量
```

##### Memory(内存信息)
```conf
used_memory:327494024                 #由redis分配器分配的内存总量，以字节为单位
used_memory_human:312.32M       #以人类可读的格式返回redis分配的内存总量
used_memory_rss:587247616         #从操作系统的角度，返回redis已分配的内存总量(俗称常驻集大小)。这个值和top命令的输出一致
used_memory_peak:1866541112    #redis的内存消耗峰值(以字节为单位) 
used_memory_peak_human:1.74G #以人类可读的格式返回redis的内存消耗峰值
used_memory_lua:35840                  #lua引擎所使用的内存大小(以字节为单位)
mem_fragmentation_ratio:1.79          #used_memory_rss和used_memory之间的比率，小于1表示使用了swap，大于1表示碎片比较多
mem_allocator:jemalloc-3.6.0            #在编译时指定的redis所使用的内存分配器。可以是libc、jemalloc或者tcmalloc
```

##### Persistence(rdb和aof的持久化相关信息)
```conf
loading:0                                                  #服务器是否正在载入持久化文件
rdb_changes_since_last_save:28900855 #离最近一次成功生成rdb文件，写入命令的个数，即有多少个写入命令没有持久化
rdb_bgsave_in_progress:0                 #服务器是否正在创建rdb文件
rdb_last_save_time:1482358115        #离最近一次成功创建rdb文件的时间戳。当前时间戳 - rdb_last_save_time=多少秒未成功生成rdb文件
rdb_last_bgsave_status:ok                  #最近一次rdb持久化是否成功
rdb_last_bgsave_time_sec:2               #最近一次成功生成rdb文件耗时秒数
rdb_current_bgsave_time_sec:-1        #如果服务器正在创建rdb文件，那么这个域记录的就是当前的创建操作已经耗费的秒数
aof_enabled:1                                        #是否开启了aof
aof_rewrite_in_progress:0                    #标识aof的rewrite操作是否在进行中
aof_rewrite_scheduled:0              

**rewrite任务计划，当客户端发送bgrewriteaof指令，如果当前rewrite子进程正在执行，那么将客户端请求的bgrewriteaof变为计划任务，待aof子进程结束后执行rewrite**

aof_last_rewrite_time_sec:-1            #最近一次aof rewrite耗费的时长
aof_current_rewrite_time_sec:-1      #如果rewrite操作正在进行，则记录所使用的时间，单位秒
aof_last_bgrewrite_status:ok             #上次bgrewriteaof操作的状态
aof_last_write_status:ok                    #上次aof写入状态
aof_current_size:4201740                #aof当前尺寸
aof_base_size:4201687                   #服务器启动时或者aof重写最近一次执行之后aof文件的大小
aof_pending_rewrite:0                      #是否有aof重写操作在等待rdb文件创建完毕之后执行?
aof_buffer_length:0                            #aof buffer的大小
aof_rewrite_buffer_length:0             #aof rewrite buffer的大小
aof_pending_bio_fsync:0                 #后台I/O队列里面，等待执行的fsync调用数量
aof_delayed_fsync:0                         #被延迟的fsync调用数量
```

##### Stats(一般统计信息)
```conf
total_connections_received:209561105 #新创建连接个数,如果新创建连接过多，过度地创建和销毁连接对性能有影响，说明短连接严重或连接池使用有问题，需调研代码的连接设置
total_commands_processed:2220123478  #redis处理的命令数
instantaneous_ops_per_sec:279                 #redis当前的qps，redis内部较实时的每秒执行的命令数
total_net_input_bytes:118515678789          #redis网络入口流量字节数
total_net_output_bytes:236361651271       #redis网络出口流量字节数
instantaneous_input_kbps:13.56                 #redis网络入口kps
instantaneous_output_kbps:31.33              #redis网络出口kps
rejected_connections:0                                  #拒绝的连接个数，redis连接个数达到maxclients限制，拒绝新连接的个数
sync_full:1                                                        #主从完全同步成功次数
sync_partial_ok:0                                           #主从部分同步成功次数
sync_partial_err:0                                          #主从部分同步失败次数
expired_keys:15598177                               #运行以来过期的key的数量
evicted_keys:0                                               #运行以来剔除(超过了maxmemory后)的key的数量
keyspace_hits:1122202228                         #命中次数
keyspace_misses:577781396                    #没命中次数
pubsub_channels:0                                      #当前使用中的频道数量
pubsub_patterns:0                                       #当前使用的模式的数量
latest_fork_usec:15679                                #最近一次fork操作阻塞redis进程的耗时数，单位微秒
migrate_cached_sockets:0                         #
```

##### Replication(主从信息，master上显示的信息)
```conf
role:master                              #实例的角色，是masteror slave
connected_slaves:1              #连接的slave实例个数
slave0:ip=192.168.64.104,port=9021,state=online,offset=6713173004,lag=0 #lag从库多少秒未向主库发送REPLCONF命令
master_repl_offset:6713173145  #主从同步偏移量,此值如果和上面的offset相同说明主从一致没延迟
repl_backlog_active:1                  #复制积压缓冲区是否开启
repl_backlog_size:134217728    #复制积压缓冲大小
repl_backlog_first_byte_offset:6578955418  #复制缓冲区里偏移量的大小
repl_backlog_histlen:134217728   #此值等于master_repl_offset - repl_backlog_first_byte_offset,该值不会超过repl_backlog_size的大小
```

##### Replication(主从信息，slave上显示的信息)
```conf
role:slave                                       #实例的角色，是master or slave
master_host:192.168.64.102       #此节点对应的master的ip
master_port:9021                         #此节点对应的master的port
master_link_status:up                  #slave端可查看它与master之间同步状态,当复制断开后表示down
master_last_io_seconds_ago:0  #主库多少秒未发送数据到从库?
master_sync_in_progress:0        #从服务器是否在与主服务器进行同步
slave_repl_offset:6713173818   #slave复制偏移量
slave_priority:100                         #slave优先级
slave_read_only:1                        #从库是否设置只读
connected_slaves:0                     #连接的slave实例个数
master_repl_offset:0         
repl_backlog_active:0                 #复制积压缓冲区是否开启
repl_backlog_size:134217728   #复制积压缓冲大小
repl_backlog_first_byte_offset:0 #复制缓冲区里偏移量的大小
repl_backlog_histlen:0           #此值等于 master_repl_offset - repl_backlog_first_byte_offset,该值不会超过repl_backlog_size的大小
```

##### CPU(CPU计算量统计信息)
```conf
used_cpu_sys:96894.66             #将所有redis主进程在核心态所占用的CPU时求和累计起来
used_cpu_user:87397.39           #将所有redis主进程在用户态所占用的CPU时求和累计起来
used_cpu_sys_children:6.37     #将后台进程在核心态所占用的CPU时求和累计起来
used_cpu_user_children:52.83 #将后台进程在用户态所占用的CPU时求和累计起来

##### Commandstats(各种不同类型的命令的执行统计信息)
cmdstat_get:calls=1664657469,usec=8266063320,usec_per_call=4.97  

**call每个命令执行次数,usec总共消耗的CPU时长(单位微秒),平均每次消耗的CPU时长(单位微秒)**
```

##### Cluster(集群相关信息)
```conf
cluster_enabled:1   #实例是否启用集群模式
```

##### Keyspace(数据库相关的统计信息)
```conf
db0:keys=194690,expires=191702,avg_ttl=3607772262 #db0的key的数量,以及带有生存期的key的数,平均存活时间
```

以上信息链接：[csdn 文章](https://blog.csdn.net/mysqldba23/article/details/68066322)


**需要额外注意：**

**instantaneous_ops_per_sec** 这个选项，如果 qps 过高，可以考虑通过 monitor 指令快速观察一下究竟是哪些 key 访问比较频繁，从而在相应的业务上进行优化，以减少 IO 次数。monitor 指令会瞬间吐出来巨量的指令文本，所以一般在执行 monitor 后立即 ctrl+c中断输出。

**info clients** 即当前多少客户端链接，如果有过多链接或者异常的链接，可以使用client list 命令查看链接列表，而同时rejected_connections这个被拒绝的链接请求数量也很重要，如果实际需要更多链接，可以调整maxclients 参数

**Replicationrepl_backlog_size** 信息，这个就是复制积压缓冲区大小，他非常重要，它严重影响到主从复制的效率。当从库因为网络原因临时断开了主库的复制，然后网络恢复了，又重新连上的时候，这段断开的时间内发生在 master 上的修改操作指令都会放在积压缓冲区中，这样从库可以通过积压缓冲区恢复中断的主从同步过程。

**积压缓冲区是环形的，后来的指令会覆盖掉前面的内容。**如果从库断开的时间过长，或者缓冲区的大小设置的太小，都会导致从库无法快速恢复中断的主从同步过程，因为中间的修改指令被覆盖掉了。这时候从库就会进行全量同步模式，非常耗费 CPU 和网络资源。 如果有多个从库复制，积压缓冲区是共享的，它不会因为从库过多而线性增长。如果实例的修改指令请求很频繁，那就把积压缓冲区调大一些，几十个 M 大小差不多了，如果很闲，那就设置为几个 M。**也可以通过查看sync_partial_err变量的次数来决定是否需要扩大积压缓冲区，它表示主从半同步复制失败的次数。**


最后恰饭 [阿里云全系列产品/短信包特惠购买 中小企业上云最佳选择 阿里云内部优惠券](https://www.aliyun.com/minisite/goods?userCode=0amqgcs9)
