# Redis Persistence

Redis 提供了多种持久化方式：

- `RDB (Redis Database)` 在指定的时间间隔创建 redis 数据的快照 snapshot
- `AOF (Append Only File)` logs 记录服务器收到的每一个写入操作，redis 重启时会重新执行这些命令，从头构建完整的数据。AOF 命令以 redis 协议追加保存每次写的操作到文件末尾.Redis 还能对 AOF 文件进行后台重写,使得 AOF 文件的体积不至于过大。
- `No persistence` 你也可以关闭持久化，完全当做内存缓存来使用
- `RDB + AOF` 你可以同时使用 RDB 和 AOF，在这种情况下, 当 redis 重启的时候会优先载入 AOF 文件来恢复原始的数据,因为在通常情况下 AOF 文件保存的数据集要比 RDB 文件保存的数据集要完整.

### RDB 的优势

- RDB 是一个非常紧凑的文件,它保存了某个时间点得数据集,非常适用于数据集的备份,比如你可以在每个小时报保存一下过去 24 小时内的数据,同时每天保存过去 30 天的数据,这样即使出了问题你也可以根据需求恢复到不同版本的数据集.
- RDB 作为一个单一文件，可以很方便传送到另一个远端数据中心或者亚马逊的 S3（可能加密），非常适用于灾难恢复.
- RDB 最大化了 Redis 的性能，因为它只需要在启动时 fork 一个子进程，父进程永远不会执行磁盘的 I/O 操作，持久化的工作由子进程完成。
- RDB 相对于 AOF 允许更快的重启。

### RDB 的缺点

- 如果你希望在 redis 意外停止工作（例如电源中断）的情况下丢失的数据最少的话，那么 RDB 不适合你.虽然你可以配置不同的 save 时间点(例如每隔 5 分钟并且对数据集有 100 个写的操作),是 Redis 要完整的保存整个数据集是一个比较繁重的工作,你通常会每隔 5 分钟或者更久做一次完整的保存,万一在 Redis 意外宕机,你可能会丢失几分钟的数据.
- RDB 需要经常 fork 子进程来保存数据集到硬盘上,当数据集比较大的时候,fork 的过程是非常耗时的,可能会导致 Redis 在一些毫秒级内不能响应客户端的请求.如果数据集巨大并且 CPU 性能不是很好的情况下,这种情况会持续 1 秒,AOF 也需要 fork,但是你可以调节重写日志文件的频率来提高数据集的耐久度.

### AOF 的优点

- 使用 AOF 会让你的 Redis 更加耐久: 你可以使用不同的 fsync 策略：无 fsync,每秒 fsync,每次写的时候 fsync.使用默认的每秒 fsync 策略,Redis 的性能依然很好(fsync 是由后台线程进行处理的,主线程会尽力处理客户端请求),一旦出现故障，你最多丢失 1 秒的数据.
- The AOF log is an append only log, so there are no seeks, nor corruption problems if there is a power outage.
- AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松。 导出（export） AOF 文件也非常简单： 举个例子， 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令， 并重启 Redis ， 就可以将数据集恢复到 FLUSHALL 执行之前的状态

### AOF 的缺点

- 对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。
- 根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）。

### 那么，我该选择哪一种呢？

有很多人选择只使用 AOF，这是非常不推荐的，since to have an RDB snapshot from time to time is a great idea for doing database backups, for faster restarts, and in the event of bugs in the AOF engine.

Note: 因为以上提到的种种原因， 未来我们可能会将 AOF 和 RDB 整合成单个持久化模型。 （这是一个长期计划。） 接下来的几个小节将介绍 RDB 和 AOF 的更多细节。

### 快照

在默认情况下， Redis 将数据库快照保存在名字为 `dump.rdb` 的二进制文件中。你可以对 Redis 进行设置， 让它在“ N 秒内数据集至少有 M 个改动”这一条件被满足时， 自动保存一次数据集。你也可以通过调用 SAVE 或者 BGSAVE ， 手动让 Redis 进行数据集保存操作。

### AOF

在配置文件中开启 AOF：

```conf
appendonly yes
```

从现在开始， 每当 Redis 执行一个改变数据集的命令时（比如 SET）， 这个命令就会被追加到 AOF 文件的末尾。这样的话， 当 Redis 重新启时， 程序就可以通过重新执行 AOF 文件中的命令来达到重建数据集的目的。

因为 AOF 的运作方式是不断地将命令追加到文件的末尾， 所以随着写入命令的不断增加， AOF 文件的体积也会变得越来越大。举个例子， 如果你对一个计数器调用了 100 次 INCR ， 那么仅仅是为了保存这个计数器的当前值， AOF 文件就需要使用 100 条记录（entry）。然而在实际上， 只使用一条 SET 命令已经足以保存计数器的当前值了， 其余 99 条记录实际上都是多余的。

为了处理这种情况， Redis 支持一种有趣的特性： 可以在不打断服务客户端的情况下， 对 AOF 文件进行重建（rebuild）。执行 BGREWRITEAOF 命令， Redis 将生成一个新的 AOF 文件， 这个文件包含重建当前数据集所需的最少命令。Redis 2.2 需要自己手动执行 BGREWRITEAOF 命令； Redis 2.4 则可以自动触发 AOF 重写， 具体信息请查看 2.4 的示例配置文件。

你可以配置 Redis 多久 will fsync data on disk.

- `appendfsync always`: fsync every time new commands are appended to the AOF. Very very slow, very safe. Note that the commands are apended to the AOF after a batch of commands from multiple clients or a pipeline are executed, so it means a single write and a single fsync (before sending the replies).
- `appendfsync everysec`: fsync every second. Fast enough (since version 2.4 likely to be as fast as snapshotting), and you may lose 1 second of data if there is a disaster.
- `appendfsync no`: Never fsync, just put your data in the hands of the Operating System. The faster and less safe method. Normally Linux will flush data every 30 seconds with this configuration, but it's up to the kernel's exact tuning.
