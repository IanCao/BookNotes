## 数据库
Redis服务器将所有数据库都保存在服务器状态`redis.h/redisServer`结构的`db`数组中，`redis`会根据服务器状态的`dbnum`属性来决定应该创建多少个数据组
默认为16个
![image.png](https://upload-images.jianshu.io/upload_images/12412504-abb3f4860096e2e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

默认情况下，redis客户端的目标数据库是0号数据库，但是可以通过`SELECT`来切换数据库

```
redis> SET msg "hello world"
OK

redis> GET msg
"hello world"

reids> SELECT 2
OK

redis[2]> GET msg
(nil)

redis[2]> SET msg "another world"
OK

redis[2]> GET msg
"another message"
```

Redis设计多个数据库的目的？

### 数据库键空间
Redis通过`redisDB`中的`dict`字典保存了数据库中的所有的键值对，这个字典称之为键空间(`key space`)

键空间和用户所见的数据库都是直接对应的：
- 键空间的键也就是数据库的键，每个键都是一个字符串对象
- 键空间的值也就是数据库的值，每个值可以是字符串对象，列表对象等等

![image.png](https://upload-images.jianshu.io/upload_images/12412504-87e06243772ee54c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中 添加/删除/更新键都是在dict结构上进行操作

#### 读写键空间时的维护操作
 - 在读取一个键的时候(读操作和写操作都要对键进行读取)，服务器会根据键是否存在来更新服务器的键空间命中(`hit`)次数或键空间不命中(`miss`)次数,
   这两个值可以通过`INFO stats`命令的`keyspace_hits`属性和`keyspace_misses`属性中查看。
 - 在读取一个键后，服务器会更新键的`LRU`(最后一次使用)时间，这个值可以用于计算键的闲置时间。
 - 服务器在读取一个键的时候，发现该键已经过期，则会先删除这个键
 
### 键生存时间
目前有四种方式设置键的生存时间
1. EXPIRE <key> <ttl> : 设置key的生存时间为ttl秒
2. PEXPIRE <key> <ttl> : 设置key的生存时间为ttl毫秒
3. EXPIREAT <key> <timestamp>: key在timestamp（秒）的时候过期
4. PEXPIREAT <key> <timestamp>: key在timestamp（毫秒）的时候过期

![image.png](https://upload-images.jianshu.io/upload_images/12412504-a3de1e313ff392ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`RedisDB`结构的`expires`字典保存了数据库中所有键的过期时间，这个字典为过期字典。
 - 过期字典的键是一个指针，这个指针指向键空间中的某个键对象(即数据库键)
 - 过期字典的值是一个`long long`类型的整数，这个整数保存了当前这个数据库键的过期时间，毫秒
 
### 键的过期策略
常规策略
 1. 定时删除：在设置键的同时，创建一个定时器来在键过期的时候删除键
    - 优点：对内存友好
    - 缺点：在CPU不友好，并且Redis的时间时间是通过无序链表实现的，查询的时间复杂度为O(N)
 2. 惰性删除：只有在下一次读取该键的时候判断该键是否过期，如果过期则删除此键
    - 优点：对CPU友好
    - 缺点：对内存不友好，会有大量过期的数据得不到剔除
 3. 定期删除：每隔一段时间，扫描一次数据库，剔除过期数据
    
`Redis`同时采用了 惰性删除和定期删除 两种策略：
 - 惰性删除的实现
   在所有读写Redis的时候都会执行`expireIfNeeded`函数，如果当前键过期，则会删除此数据
 - 定期删除的实现
   每当Redis的服务器周期性操作`serverCron`函数执行时，`activeExpireCycle`函数就会被调用，它在规定的时间内，分多次遍历服务器中的各个数据库，从数据库的`expires`
   字段中随机检查一部分键的过期时间，并且删除其过期键。
   
### AOF/RDB和复制功能对过期键的处理
RDB在生成和载入的时候都会剔除过期的键

AOF在写入的时候，如果过期键没有被剔除，则其也不会被剔除
AOF在重写的时候，会过滤过期的键

#### 复制
当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制：
即如果某一个key过期了，但是客户端向从服务器发送GET 请求，这个时候服务器发现key已经过期，但是也不会删除此key，直到接受到主服务器的DEL消息
![image.png](https://upload-images.jianshu.io/upload_images/12412504-a60ec591ef32c7ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 数据库通知
此功能可以让客户端通过订阅给定的频道或者模式，来获知数据库中键的变化，以及数据库中命令的执行情况。

## RDB
两个触发生成RDB文件的命令
  - 一个是`SAVE`:`SAVE`命令会阻塞Redis服务器进程，直到RDB文件创建完毕为止。
  - 一个是`BGSAVE`：`BGSAVE`命令会派生出一个子进程，然后由子进程负责创建RDB文件，服务器进程(父进程)继续处理命令请求。
Redis在启动时检测到RDB文件存在，它就会自动载入RDB文件。
如果Redis开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态

在`BGSAVE`命令执行期间，客户端发送的`BGSAVE`和`SAVE`命令会被服务器拒绝。
`BGSAVE`和`BGREWRITEAOF`两个命令不能同时执行：
 - 如果`BGSAVE`命令正在执行，那么客户端发送的`BGREWRITEAOF`命令会被延迟到`BGSAVE`命令执行完毕之后执行。
 - 如果`BGREWRITEAOF`命令正在执行，那么客户端发送的`BGSAVE`将会被拒绝
原因：上面两个操作都是由子进程操作，所以在操作方面没有什么冲突，不能同时执行时出于性能的考虑

### 自动间隔保存
Redis可以通过设置服务器配置的save选项，让服务器每隔一段时间自动执行一次`BGSAVE`明利
```
save 900 1 // 服务器在900s之内，对数据库进行了至少一次修改
save 300 10 // 服务器在300s之内，对数据库进行了至少10次修改
```

#### dirty计数器和lastsave属性
 - dirty计数器记录距离上一次成功执行SAVE命令或者BGSAVE命令之后，服务器对数据库状态进行了多少次修改
 - lastsave属性是一个UNIX时间戳，记录了上一次成功执行SAVE或者BGSAVE命令的时间
 
而服务器的`serverCron`会默认100ms执行一次，该函数会去检查上面的两个属性，判断是否满足条件。

