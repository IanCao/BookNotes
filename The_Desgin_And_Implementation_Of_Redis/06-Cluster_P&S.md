## 集群
一个Redis集群通常是由多个节点(node)组成，而节点之间是通过 `Cluster meet <ip> <port>` 来进行握手。

在集群模式之下，redis会继续使用`redisServer`结构来保存客户端的状态，至于集群中的节点则是通过`clusterNode`和`clusterLink`结构进行保存。

### 集群的数据结构
`clusterNode`保存了一个节点的当前状态，比如创建时间，节点名称等等信息
![](http://7n.caoyixiong.top/20201220222838.png)

而通过`clusterLink`保存了连接其他节点所需的有关信息。
![](http://7n.caoyixiong.top/20201220222911.png)

> RedisClient 和 clusterLink的结构的异同？
>
> redisClient结构和clusterLink结构中都有自己的套接字描述符和输入,输出缓冲区
> 但是redisClient中的套接字和缓冲区都是用于连接客户端的
> 而clusterLink中则是用于连接其他节点的

并且每个节点中都保存着一个`clusterState`结构，这个结构记录了从当前节点的视角下，整个集群目前所处的状态，具体信息如下：
![](http://7n.caoyixiong.top/20201220223657.png)

如下展示了一个拥有三个节点的集群的其中一个节点的`clusterState`数据
![](http://7n.caoyixiong.top/20201220223728.png)

**`cluster meet`的实现流程**
> 节点A 发送 `cluster meet` 消息给节点B
 - 节点A首先会创建一个节点B的`clusterNode`并添加到自己的`clusterState.nodes`字典里面；
 - 然后发送meet消息给节点B
 - 节点B接收到数据后，也会创建个节点A的`clusterNode`到自己的`clusterState`里面；
 - 节点B向节点A返回一个Pong消息
 - 节点A收到了Pong消息之后，会再给节点B发送一个Ping消息
 - 自此，同步结束
 - 同步结构以后，节点A会将节点B的信息通过`Gossip`协议传播给集群中的其他节点，让其他节点也跟节点B进行握手
![](http://7n.caoyixiong.top/20201220224714.png)

### 槽指派
Redis通过分片的方式来保存数据库中的键值对；集群的整个数据库被分为16384个槽(slot)，Redis中的每个键都属于这个16384个槽中的一个，集群中的每个节点可以处理0个或者最多16384个槽
如果当前Redis的16384个槽都有节点在处理，则集群处于上线状态，相反则是下线状态
 
redis通过`clusterNode`中的`slots`属性和`numslot`属性记录了节点负责处理哪部分槽
```
struct clusterNode {
   //.....
   unsigned char slots[16384/8];
   int numslots;
   //.....
}
```

slots属性是一个二进制位数组，数组长度为 16384/8= 2048个字节，共16384位
![](http://7n.caoyixiong.top/20201220230915.png)
如果slots数组在某个索引下的值为1，则代表当前节点处理此槽，否则代表不处理此槽

`clusterNode`中的`slots`数组记录了当前节点所负责的slot

`clusterState`中使用了一个16483长度的`clusterNode`数组记录了槽的指派信息
 
```
struct clusterState {
   // ....
    clusterNode *slots[16384];
   // ....
}
```
 - 如果`slots[i]`指向NULL，则代表当前槽为指派给任何节点
 - 如果`slot[i]`指向某个节点，则代表当前槽为指向的节点处理
 
**clusterState中记录slots的优势**
 - 如果要查找某个槽是被哪个节点处理，通过O(1)的时间复杂度就可以找到

虽然clusterState中记录了所有槽的归属，但是clusterNode中的slots数据仍然有必要存在，
因为在数据同步的时候，节点就不需要遍历`clusterState`的所有slot来找自己负责的部分给其他节点了，也是通过空间来换取时间。

### 在集群中执行命令
当客户端向节点发出与数据库键有关的命令时，接收命令的节点会计算出命令要的数据库键属于哪个槽
 - 如果所属的槽恰好就是当前节点，则会直接执行命令
 - 如果键所在的槽不是当前节点，则当前节点会给客户端返回一个MOVE错误，指引客户端重定向到正确的节点，并再次发送之前想要执行的命令。
 >  若当前键所在的槽不是当前节点，当前节点就会从`clusterState`中找到所在槽的节点的ip和port，并将这些信息包装成MOVE错误返回给客户端

![](http://7n.caoyixiong.top/20201220232422.png)

#### 节点数据库的实现
节点数据库只能使用0号数据库，而单机数据库则没有这个限制

redis除了会将键值对保存在数据库里面之外，节点还会用`clusterState`结构中的`slots_to_keys`跳表来保存槽和键之间的关系
```
struct clusterState {
  // ....
  zskiplist *slots_to_keys;
  // ....
} clusterState;
```

`slots_to_keys`跳表的每个节点的分值`score`都是一个槽号，每个节点的成员(member)都是一个数据库键；
- 每当节点往数据库中添加一个新的键值对时，节点就会将这个键以及键的槽号关联到`slots_to_keys`跳跃表中
- 每当节点从数据库中删除一个键值对时，也会将键从跳表中删除
![](http://7n.caoyixiong.top/20201220234046.png)