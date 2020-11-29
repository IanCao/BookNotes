## Replication
在Redis中，用户可以通过执行`SLAVEOF`命令或者设置`slaveof`选项，让一个服务器去复制另一个服务器，即Redis的主从复制

## 旧版复制功能的实现
Redis的复制功能分为 同步(`sync`)和命令传播(`command propagate`)两个操作：
 - 同步操作用于将从服务器的数据库状态更新至主服务器当前所处的数据库状态
 - 命令传播操作则用于在主服务器的数据库状态被修改，导致主从服务器的数据库状态出现不一致时，让主从服务器的数据库重新会回到一致状态

具体流程：
 1. 从服务器向主服务器发送`SYNC`命令
 2. 主服务器接收到`SYNC`命令后开始执行`BGSAVE`生成RDB文件，同时用一个缓冲区来记录从现在开始执行的所有写命令
 3. 主服务器的`BGSAVE`执行完成后，会将RDB文件和临时的缓冲区都发送给从服务器，从服务器再进行同步信息操作
 4. 同步操作完成以后，后续主服务器接收到的所有的写命令都会发送给从服务器，即命令传播
 

### 旧版复制功能的缺陷
Redis的主从复制有如下两种场景：
 1. 初次复制：即从服务器没有复制过任何主服务器
 2. 断线重连后的复制：因为每次从服务器都要主服务器的全量数据，如果断线时间段，则这样就比较浪费资源

## 新版复制功能的实现
新版使用`PSYNC`来代替`SYNC`命令进行主从同步
`PSYNC`分为：
  - 完全重同步：跟旧版的是一样的，主服务器执行`BGSAVE`生成RDB
  - 部分重同步：则是出于从服务器断线重连的情况，如果条件允许，主服务器可以将主从服务器断开连接期间执行的写命令发送给从服务器

### 部分重同步的实现
 - 主服务器的复制偏移量和从服务器的复制偏移量
    - 主从服务器都会维护一个复制偏移量
      - 主服务器每次向从服务器传播N个字节的数据后，就将自己的复制偏移量的值加上N。
      - 从服务器每次收到主服务器传播来的N个字节的数据后，就将自己的复制偏移量的值加上N
 - 主服务器的复制积压缓冲区
    - 复制积压缓冲区是由主服务器维护的一个固定长度先进先出队列，默认大小为1MB。
    - 当主服务器在进行命令传播时，它不仅会将写命令发送给所有从服务器，还会将写命令入队到复制积压缓冲区里面。
    - 当从服务器重连上主服务器时，会将其的复制偏移量发送给主服务器，主服务如果判断当前的offset是在复制积压缓冲区的话，就会进行部分同步；否则就需要进行完整同步操作   
 - 服务器的运行ID
    - 每个Redis服务器在启动的时候都会生成一个随机的运行ID，在主从同步的时候，从服务器会将自己的RunId发送给主服务，主服务器会将这个信息存储下来，当再有从服务器连接上主服务器，主服务器会先判断当前的RunId是否存在，如果存在，则尝试进行部分同步；如果不存在，则进行完整同步
    
![](http://7n.caoyixiong.top/20201129180240.png)

### 心跳检测
在命令传播阶段，从服务器默认回以每秒一次的频率向主服务器发送命令：
`REPLCONF ACK <replication_offset>`
其中`replication_offset`是从服务器当前的复制偏移量。
发送`REPLCONF ACK`命令对于主从服务器有三个作用：
 - 检测主从服务器的网络连接状态
   - 如果主服务器超过一秒钟没有收到从服务器发来的`REPLCONF ACK`命令，那么主服务器就知道主从服务器之间的连接出现了问题
 - 辅助实现`min-slaves`选项
   - Redis的`min-slaves-to-write`和`min-slaves-max-lag`两个选项可以防止主服务在不安全的情况下执行写命令
     即如果我们在主服务器设置如下：
       ```
        min-slaves-to-write 3
        min-slaves-max-lag 10 
       ```
     如果从服务器的数量少于3个，或者三个从服务器的延迟都大于等于10s，主服务器将拒绝执行写命令
 - 检测命令丢失
   - 如果因为网络问题，主服务器给从服务器的命令丢失了，就可通过从服务器的返回值中得知，并在缓冲区找到对应的数据，重新给到从服务器
   
   
## Sentinel
Sentinel 是由一个或者多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器，以及这些主服务器属下的所有的从服务器。
![](http://7n.caoyixiong.top/20201129182348.png)

Sentinel本质上只是一个运行在特殊模式下的Redis服务器，但是其与普通的Redis服务器又不同；

Sentinel实例启动时，会创建两个连向主服务器的异步网络连接：
 - 一个是命令连接，这个链接专门用于向主服务器发送命令，并接受命令回复
 - 另一个是订阅连接，这个连接专门用于订阅主服务器的`_sentinel_:hello`频道。
 
### 获取主服务器信息
Sentinel默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送INFO命令，并通过分析INFO命令的回复来获取主服务器的当前信息
 - 一方面是主服务器资深的信息
 - 另一方面则是主服务器属下的所有从服务器的信息。
 
当Sentinel与一个主服务器或者从服务器建立起订阅连接之后，Sentinel就会通过订阅连接，向服务器发送如下命令：
 `SUBSCRIBE _sentinle_:hello`，Sentinel对于此频道的订阅会一直持续到Sentinel与服务器的连接断开为止。

对于监视同一个服务器的多个Sentinel来说，一个Sentinel发送的信息会被其他Sentinel接收到，这些信息会被用于更新其他Sentinel对发送Sentinel的认知（实现了Sentinel之间的服务发现），也会被用于更新其他Sentinel对被监视服务器的认知。

当Sentinel通过频道信息发现了一个新的Sentinel节点之后，不仅会更新自己的sentinels字典中的信息，也会创建连向这个Sentinel的命令连接，最终监视统一主服务器的多个Sentinel将形成互相连接的网。
![](http://7n.caoyixiong.top/20201129190341.png)

**Redis哨兵的三个定时任务，Redis哨兵判定一个Redis节点故障不可达主要就是通过三个定时监控任务来完成的：**
 - 每隔10秒每个哨兵节点会向主节点和从节点发送"info replication" 命令来获取最新的拓扑结构 
  ![](https://img-blog.csdn.net/20181002105353886?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L251b21pemhlbmRlNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
 - 每隔2秒每个哨兵节点会向Redis节点的_sentinel_:hello频道发送自己对主节点是否故障的判断以及自身的节点信息，并且其他的哨兵节点也会订阅这个频道来了解其他哨兵节点的信息以及对主节点的判断
  ![](https://img-blog.csdn.net/20181002105430450?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L251b21pemhlbmRlNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
 - 每隔1秒每个哨兵会向主节点、从节点、其他的哨兵节点发送一个 “ping” 命令来做心跳检测
  ![](https://img-blog.csdn.net/2018100210544956?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L251b21pemhlbmRlNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
### 检测主观下线状态
默认情况下，Sentinel会每一秒向与它创建了命令连接的实例(主服务器，从服务器，其他Sentinel)发送Ping命令，并通过实例返回的Ping命令回复来判断实例是否在线。
Sentinel配置中的`down-after-milliseconds`选项中指定了Sentinel判断实例进入主观下线所需的时间长度。

这个配置不仅会被Sentinel用来主服务器的主观下线状态，也会用于判断从服务器的主观下线状态。

如果不同Sentinel配置的时间不同则其判断服务器主观下线的时间也不同。

### 选举领头Sentinel
所有客观认为主服务器已经下线的Sentinel发送vote命令，如果接收的服务器还没有收到其他的vote命令，则会将自己的局部领头Sentinel设置为其，否则则拒接
然后最终判断当前Sentinel的获得的票数是否大于配置，如果大于则等于 （一半加一）的票数，即成功当选。
### 选举新的服务器
![](http://7n.caoyixiong.top/20201129192413.png)

当新的主服务器出现的时候，领头Sentinel就需要让从服务器归属于当前新的主服务器，具体是向从服务器发送`SLAVEOF`命令来实现。
当旧的主服务器重新上线之后，Sentinel也会向其发送`SLAVEOF`命令，让其成为新主服务器的从服务器
![](https://img-blog.csdn.net/20181002130008223?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L251b21pemhlbmRlNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)