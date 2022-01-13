### Redis 集群介绍

Redis 集群允许在多个 Redis 节点间共享数据。

Redis 集群在 network partition 的情况下提供一定程度的高可用。也就是说，当某些节点无法访问的情况下，集群中的其他节点仍然可以继续工作。只有当大部分的 master 节点也无法访问的情况下，集群才会停止工作。

Redis 集群的优势：

- automatically split your dataset among multiple nodes
- continue operations when a subset of the nodes are experiencing failures or are unable to communicate with the rest of the cluster

### Redis 集群的 TCP 端口

Redis 集群中的每个节点都保持两个 TCP 连接。一个 TCP 端口用来服务 clients，比如 `6379`，称作 `data port`。另一个端口也称作总线端口 `bus port`。集群的 `bus port` 通过 `data port` 加上 10000 获取，比如 `16379`，或者你也可以在 config 中修改它。

`bus port` 是节点与节点之间沟通用的，它们之间的沟通采用了一个二进制协议。它的用处包括 **failure detection**, **configuration update**, **failover authorization** 等等。确保你的防火墙开通了这两个端口。

### Redis 集群和 Docker

Currently Redis Cluster does not support `NATted` environments and in general environments where IP addresses or TCP ports are remapped.

> NAT: Network address translation (NAT) 网络地址转换 在[计算机网络](https://zh.wikipedia.org/wiki/%E8%A8%88%E7%AE%97%E6%A9%9F%E7%B6%B2%E7%B5%A1 "计算机网络")中是一种在 IP[数据包](https://zh.wikipedia.org/wiki/%E5%B0%81%E5%8C%85 "数据包")通过[路由器](https://zh.wikipedia.org/wiki/%E8%B7%AF%E7%94%B1%E5%99%A8 "路由器")或[防火墙](https://zh.wikipedia.org/wiki/%E9%98%B2%E7%81%AB%E7%89%86 "防火墙")时重写来源[IP 地址](https://zh.wikipedia.org/wiki/IP%E5%9C%B0%E5%9D%80 "IP地址")或目的 IP 地址的技术。NAT is a method of mapping an IP address space into another by modifying network address information in the IP header of packets while they are in transit across a traffic routing device.

Docker 使用了一种端口映射的技术 `port mapping`: 在 Docker 容器中运行的程序可以将容器内的端口映射到容器的宿主机的端口上。为了使 Docker 兼容 Redis 集群，你必须用 Docker 的 `host networking mode`网络模式。

> `--net=host` 启动 Docker 容器时的选择 network driver 为 host：remove network isolation between the container and the Docker host, and use the host's networking directly.

### Redis 集群和 data sharding

Redis Cluster 不使用 consistent hashing, 而是使用了 `hash slot` 哈希槽的概念。

在 Redis 集群中由 16384 个哈希槽，每个 key 通过 CRC16 校验后对 16384 取模来决定放置哪个槽。

集群的每个节点负责一部分 hash 槽,举个例子,比如当前集群有 3 个节点,那么:

- 节点 A 包含 0 到 5500 号哈希槽.
- 节点 B 包含 5501 到 11000 号哈希槽.
- 节点 C 包含 11001 到 16384 号哈希槽.

> CRC16 算法 (Cyclic Redundancy Check) 循环冗余校验是一种根据网络数据包或者计算机文件等数据产生剪短固定位数校验码的一种信道编码技术，主要用来检测或校验数据传输或者保存后可能出现的错误。它是利用出发及余数的原理来做错误侦测的。

这种结构很容易添加或者删除节点. 比如如果我想新添加个节点 D, 我需要把节点 A, B, C 中的一些哈希槽放到节点 D 上. 如果我想移除节点 A,需要将 A 中的槽移到 B 和 C 节点上,然后将没有任何槽的 A 节点从集群中移除即可. 由于从一个节点将哈希槽移动到另一个节点并不会停止服务,所以无论添加删除或者改变某个节点的哈希槽的数量都不会造成集群不可用的状态.

Redis Cluster supports multiple key operations as long as all the keys involved into a single command execution (or whole transaction, or Lua script execution) all belong to the same hash slot.

- The user can force multiple keys to be part of the same hash slot by using a concept called _hash tags_ .

> Hash tag 是在 key 中使用`{}`来确保不同的 key 会被放进相同的哈希槽中，比如 `this{foo}key` 和 `another{foo}key` 这两个 key 中，只有 hash tag `{}` 包起来的部分 `foo` 会被 hashed，所以这两个 key 虽然不同，但是他们的 hash 值是一样的，所以一定会在同一个哈希槽中。

### Redis 集群的主从 master slave 模式

为了确保高可用（即使某些节点宕机了，其他节点仍然能够工作）。Redis 集群使用了 `master-slave` 的模式，也就是说每一个节点都有 N-1 个 replica node。

比如在我们的集群中有 3 个 master 节点 A,B,C，如果节点 B 宕机了，那集群就无法继续工作，因为哈希槽 5501 ~ 11000 之间的数据无法访问了。

所以，当我们创建集群的时候，给每一个节点添加一个 replica node，也就是 A1, B1, C1，3 个从节点。这样即使 B 节点宕机了，集群会 promote 节点 B1，把他提拔为新的 master 节点。但是如果 B 和 B1 同时宕机了，那么集群还是会无法工作。

### Redis 集群的一致性保证

Redis 集群无法保证强一致性，**this means that under certain conditions it is possible that Redis Cluster will lose writes that were acknowledged by the system to the client**.

Redis Cluster can lose writes is because it uses asynchronous replication.

当 client 写对 redis 集群数据的时候

1. Your client writes to the master B
2. The master B replies OK to your client
3. The master B propogates the write to its replicas B1, B2 and B3

如你所见，主节点 B 不会等待从节点 B1,B2,B3 的响应，而是直接返回给 client，否则会导致 latency。So if your client writes something, B acknowledges the write, but crashes before being able to send the write to its replicas, one of the replicas (that did not receive the write) can be promoted to master, losing the write forever.

This is **very similar to what happens** with most databases that are configured to **flush data to disk every second**, 当然你可以通过 force the database to flush data to disk before replying to the client 来提升一致性，但是这也会导致很严重的性能下降。我们必须在性能和一致性之间做出选择。

> Basically, there is a trade-off to be made between performance and consistency. 其实也就是 CAP 中的 C 和 A 之间的抉择。

Redis 集群也支持配置成 synchronous writes，会大大降低写入丢失的风险。但是，即便如此，Redis 集群也没有保证强一致性，在更复杂的场景下，仍然可能出现丢失了写入操作的节点被选为 master。

比如以下场景，有 6 个节点组成的集群 A,B,C,A1,B1,C1 其中 3 个 master 节点 3 个 replica 节点，加上一个 client Z1。

如果网络出现 partition，一个 partition 中是 Z1 和 B，另一个 partition 中是 A,C,A1,B1,C1。

Z1 仍然能写入到节点 B 中，此时集群中因为 B 无法访问，B1 被选为 master，当集群恢复以后 Z1 对节点 B 的写入就永远丢失了。

> 注意， 在 network partition 出现期间， 客户端 Z1 可以向主节点 B 发送写命令的这个时间窗口是有限制的， 这一时间限制称为节点超时时间（node timeout）， 是 Redis 集群的一个重要的配置选项：

### Redis 集群配置参数

在 `redis.conf` 中引入了一些 Redis Cluster 相关的参数

- `cluster-enabled <yes/no>` 如果 yes 的话，在 redis instance 中开启集群模式，否则这个 instance 会以 standalone 模式启动。
- `cluster-config-file <filename>` 指定一个配置文件，这是 Redis 集群自动保存集群配置（集群状态）的地方，每当集群配置发生变化，会写入这个配置文件。当 redis 实例重启的时候就可以读取这个配置文件恢复状态。这个文件记录了集群中有哪些节点，它们的状态等等。
- `cluster-node-timeout <milliseconds>` 集群中的节点 unavailable 的最大时间窗口，超过这个时间，节点会被当做 failing。如果一个 master 节点在这个时间窗口内一直无法访问到其他的 master 节点，集群会认为它 fail 了，它会停止接受请求，集群会 promote 它的 replica 作为 master。
- `cluster-slave-validity-factor <factor>` If the value is positive, a maximum disconnection time is calculated as the _node timeout_ value multiplied by the factor provided with this option, and if the node is a replica, it will not try to start a failover if the master link was disconnected for more than the specified amount of time. For example, if the node timeout is set to 5 seconds and the validity factor is set to 10, a replica disconnected from the master for more than 50 seconds will not try to failover its master.
- `cluster-migration-barrier <count>` Minimum number of replicas a master will remain connected with, for another replica to migrate to a master which is no longer covered by any replica. See the appropriate section about replica migration in this tutorial for more information.

* `cluster-require-full-coverage <yes/no>` : If this is set to yes, as it is by default, the cluster stops accepting writes if some percentage of the key space is not covered by any node. If the option is set to no, the cluster will still serve queries even if only requests about a subset of keys can be processed.
* `cluster-allow-reads-when-down <yes/no>` : If this is set to no, as it is by default, a node in a Redis Cluster will stop serving all traffic when the cluster is marked as failed, either when a node can't reach a quorum of masters or when full coverage is not met. This prevents reading potentially inconsistent data from a node that is unaware of changes in the cluster. This option can be set to yes to allow reads from a node during the fail state, which is useful for applications that want to prioritize read availability but still want to prevent inconsistent writes. It can also be used for when using Redis Cluster with only one or two shards, as it allows the nodes to continue serving writes when a master fails but automatic failover is impossible.
