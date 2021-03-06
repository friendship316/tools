### 主从复制
#### 传统bin-log主从集群方式
开启过程： 
>- 主库配置文件中，指定主库serverId，开启binlog（还可以指定需要或者不需要同步的库），并在重启后开启replication slave的权限
>- 从库配置文件中，指定从库serverId，开启binlog、relay-log（还可以指定需要或者不需要同步的库），并在启动后填写主库的用户名密码等数据。

开启状态：
>- 主从复制开启后，show master status可以查看主库的同步状态，show slave status可以查看从库的同步状态
>- 从库会显示IO thread和Sql thread是running状态，其中IO thread是接收主库传来的binlog数据写入relay-log，Sql thread是将relay-log数据转成SQL写入从库。
>- 可以通过观察从库的Master_Log_File以及Read_Master_Log_Pos是否与主库一致来判断是否搭建成功

在主库中开启relay-log并填写从库的用户名密码，再开启从库的replication slave的权限，即可使两个库互为主从
#### 5.6引入的GTID集群方式
底层还是依赖bin-log，只不过主从之间不再以bin-log的偏移量为同步标准，而是一个全局事务ID。
#### 半同步复制
MySQL默认是异步复制，主线程将数据写入binlog后，即返回响应给客户端。另外单独一个dump线程异步将binblog的数据发送给从库。这种主线程和binlog同步是异步关系。显然异步关系有可能导致数据丢失，即主库挂了，但是还有数据没有同步给从库。
半同步则是主线程将数据写入binlog之后，并不是立即返回客户端响应，而是等待至少一个从库接收并写到relay log中，才会返回给客户端。MySQL在等待确认时，默认会等10秒，如果超过10秒没有收到ack，就会降级成为异步复制。半同步复制机制会造成一定程度的延迟，这个延迟时间最少是一个TCP/IP请求往返的时间，整个服务的性能是会有所下降的。
#### 并行复制
从库多个线程拉取binlog数据，降低主从复制的延迟。在从服务上设置slave_parallel_workers为一个大于0的数，然后把slave_parallel_type参数设置为LOGICAL_CLOCK，这就可以了。
但是：从binlog转为SQL这个过程还是只有一个线程。

### 高可用方案
#### MMM
两主一从，其中两主互为主从。一个主用来读写，另一个主只用于读。其中一个主故障的时候，可以切换另一个主来提供服务。
#### MHA
一主两从。当主库挂了之后，可以将从库升为master提供服务。并且会尝试从主库的binlog恢复数据。
#### MGR
是MySQL官方在5.7.17版本正式推出的一种组复制机制。主要是解决传统异步复制和半同步复制的数据一致性问题。

一个事务提交后，必须经过超过半数节点的决议并通过后，才可以提交。
MGR依靠分布式一致性协议(Paxos协议的一个变体)，实现了分布式下数据的最终一致性，提供了真正的数据高可用方案(方案落地后
是否可靠还有待商榷)。

多主模式下，客户端可以随机向MySQL节点写入数据
单主模式下，MGR集群会选出primary节点负责写请求，primary节点与其它节点都可以进行读请求处理.

优点：基本无延迟，延迟比异步的小很多支持多写模式，但是目前还不是很成熟数据的强一致性，可以保证数据事务不丢失
缺点:仅支持innodb，且每个表必须提供主键。只能用在GTID模式下，且日志格式为row格式。适用的业务场景：对主从延迟比较敏感希望对对写服务提供高可用，又不想安装第三方软件数据强一致的场景五、分库分表
### 分库分表
#### 方式
1、id取模
2、按时间
