redis 是一个基于内存的非关系型数据库，既然数据存放在内存中，那么当 redis 重启或者缓存服务器宕机的情况下，内存中的数据就会丢失，这显然不是我们想要的结果。



那么针对这两种情况，是否有办法能处理这样的情况呢？



对此 ，redis 提供了两种持久化选项将数据存储到硬盘中。一种为快照，一种为 AOF。这两种持久化方法既可以同时使用，也可以单独使用，在某些特定的情况下甚至可以两种方法都不使用，具体使用哪一种持久化方法根据用户的数据以及应用的业务场景来决定。



将内存中的数据持久化到硬盘上一个重要的原因就是为了在之后能重用数据，或者是为了防止系统故障而将数据备份到另外一个远程位置。

## 快照

快照可以将某一时刻的所有数据写入到硬盘里面。

**1.快照的配置选项**

redis 提供的快照配置选项

```properties
# 快照持久化配置选项
# 60s 内有一万次写入的情况下，redis 将自动创建快照
save 60 10000  
stop-writes-on-bgsave-error no
rdbcompression yes
dbfilename dump.rdb
# 快照文件存储的位置
dir ./   
```

根据配置，快照将会被写入 dbfilename 选项指定的文件里面，并存储在 dir 目录下面。

如果在新的快照创建完毕之前，Redis、系统、或者硬件三者之中的任意一个崩溃了，那么 Redis 将丢失最近一次创建快照之后写入的数据。

> 基于 6.0 默认的快照配置

```properties
#   In the example below the behaviour will be to save:
#   900 秒内至少有 1 个 key 被改变
#   after 900 sec (15 min) if at least 1 key changed
#   300 秒内至少有 10 个 key 被改变
#   after 300 sec (5 min) if at least 10 keys changed
#   60 秒内至少有 10000 个 key 被改变
#   after 60 sec if at least 10000 keys changed
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
dbfilename dump.rdb
dir ./
```

**2.创建快照的方式**

- （手动）客户端可以通过向 Redis 发送 BGSAVE 命令来创建一个快照。对于支持 BGSAVE 的平台来说（除开 Windows），Redis 会调用 fork 来创建一个子进程，由子进程完成快照的写入工作。
- （手动）客户端也可以向 Redis 发送一个 SAVE 命令来创建一个快照，接到 SAVE 命令后，Redis 在快照创建完毕之前，将不会响应任何其他的命令。**SAVE 命令用的不多，通常在没有足够的内存去执行 BGSAVE 的情况下才会去使用 SAVE 命令来创建快照**。
- 如果用户配置了 save 选项，当且仅当 60 秒内有 10000 次写入的情况下，Redis 会自动触发 BGSAVE 指令来创建快照。
  - 如果用户配置了多个 save 选项，那么只要有一个 save 条件满足，Redis 就会触发一次 BGSAVE 指令来创建快照。
- 当 Redis 收到 SHUTDOWN 指令时，或者接受到标准的 TERM（终止） 指令时，Redis 将不再处理其他请求，会执行 SAVE 指令，创建快照完毕后，关闭服务器。
- 当一个 Redis 服务器连接另一个 Redis 服务器，并向对方发送了一个 SYNC 命令来开始一次复制操作的时候，如果主服务器目前没有在执行 BGSAVE 操作，或者主服务器并非刚刚执行完 BGSAVE 操作，那么主服务器就会执行 BGSAVE 操作来创建快照。



> 使用快照来进行备份，切记：一旦真的发生崩溃，那么将丢失最近一次创建快照之后更改的所有数据。如果系统能接受少量数据丢失，使用快照来进行备份也不失为一种选择。
>
> SAVE 虽然会阻塞 Redis 直到快照生成完毕，但是是因为它不需要去创建子进程，所以就不会出现向 BGSAVE 一样因为创建子进程而导致 Redis 停顿，并且因为没有子进程在争抢资源，所以 SAVE 创建快照会比 BGSAVE 创建快照时间要稍微快一点。



如果丢失少量数据对应用系统来说是不可忍受的，那么推荐使用 AOF 的方式来进行持久化。



## 只追加文件（AOF）

只追加文件（append-only-life）的方式又称 AOF，它会在执行写命令时，将被执行的写命令复制到硬盘中。

**1.redis 提供的 AOF 配置选项**

```properties
# 是否打开 aof 持久化选项，no 关闭，yes 开启
appendonly no
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-precentage 100
auto-aof-rewrite-min-size 64mb
# 追加文件存储的位置
dir ./
```

简单来说，AOF 创建的方式会将被执行的命令写入到 AOF 文件的末尾，以此来记录数据发生的变化。因此在出现崩溃的情况下，Redis 只要从头到尾执行一遍 AOF 文件里面包含的所有写命令，就可以恢复 AOF 文件所记录的数据集。

AOF 通过 appendonly yes 配置选项来打开。



> 基于 redis6.0 默认的 Appending only mode

```properties
# AOF and RDB persistence can be enabled at the same time without problems.
# If the AOF is enabled on startup Redis will load the AOF, that is the file
# with the better durability guarantees.
appendonly no
# 默认的 aof 文件名称
appendfilename "appendonly.aof"

# no: don't fsync, just let the OS flush the data when it wants. Faster.
# always: fsync after every write to the append only log. Slow, Safest.
# everysec: fsync only one time every second. Compromise.
# appendfsync always
appendfsync everysec
# appendfsync no
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```



**2.appendfsync 可配置的选项值**

| 选项值   | 同步频率                                                     |
| -------- | ------------------------------------------------------------ |
| always   | 每个 Redis 写命令都要同步写入硬盘，选择该配置会严重影响 Redis 的效率 |
| everysec | 每秒执行一次同步操作，显示地将多个写命令同步到硬盘           |
| no       | 让操作系统来决定应该何时进行数据同步                         |

如果设置成 always，每个写命令都要同步写入磁盘，这种操作确实能将数据丢失的可能性将到最低，但是这种策略需要对硬盘进行大量写入，会对 Redis 的处理命令的速度大大降低。

另外高频写入也会降低磁盘的寿命。



如果要兼顾数据安全和写入性能，可以考虑使用 everysec 来进行每一秒写入。这种同步 AOF 文件的方式和不使用任何持久化特性时的性能相差不多。即使出现崩溃的情况，Redis 最多也只会丢失 1s 内产生的数据。



如果是用 no 的方式，那么 Redis 将不会再管 AOF 文件的写入，也不会再对文件做任何显示的同步，这种方式也不会对 Redis 的性能造成影响，但是系统崩溃导致使用这种选项的 Redis 服务器将会丢失不定量的数据。所以一般来说，不推荐使用 appendfsync no 这个选项。

**3.AOF 文件带来的问题**

因为频繁的在往 AOF 文件中写入数据，那么 AOF 文件的体积将会不断增大，甚至会占满硬盘容量。另外随着 AOF 文件的不断增大，还原数据所需要的时间也成倍增长。

为了解决这个问题，我们就需要对 AOF 文件进行压缩。

可以向 Redis 服务器发送 BGREWRITEAOF 命令来重写 AOF 文件，该命令会去除 AOF 文件中冗余的命令，减少 AOF 的体积。该命令和 BGSAVE 命令的原理类似，也是 fork 一个子进程来对 AOF 文件进行重写。

> BGREWRITEAOF 和 BGSAVE 都是使用了子进程，所以当重写 AOF 文件的时候，在快照持久化中产生的问题将会在重写 AOF 文件中重现。同时在删除旧的巨型 AOF 文件，也会导致操作系统无响应数秒。



通过设置 auto-aof-rewrite-percentage 选项和 auto-aof-rewrite-min-size 选项来自动执行 BGREWRITEAOF。

假设用户对 REdis 设置了配置选项 auto-aof-rewrite-percentage 100 和 auto-aof-rewrite-min-size 64mb，并且启动了 AOF 持久化，那么当 AOF 体积大于 64mb 的时候，并且 AOF 文件的体积比上一次重写之后的体积大了至少一倍（100%）的时候，Redis 将执行 BGREWRITEAOF命令。