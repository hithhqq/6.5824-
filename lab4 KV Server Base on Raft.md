# lab4 KV Server Base on Raft

## 1 实验介绍

在这个实验中，我们将在实验3的Raft函数库之上，搭建一个能够容错的key/value存储服务。你的key/value服务将有几个相同的副本，并由Raft保持一直的状态机组成只要大多数服务器在线并且网络通畅，就算其他服务器失败或者网络分区，都应该可以持续处理客户端的请求。在这个实验中，可能会暴露出lab3写的Raft遗留的问题，这个实验可以看成lab2的升级版，lab4A是 实现不需要日志压缩的kay/value服务， lab4B是在lab4A的基础上实现日志压缩的功能。

### 2 kv数据库架构

KV数据库是位于raft层之上的，或者说我们的kv数据库使用了raft库。客户端（就是代码中的clerk）调用应用层（server）的RPC，应用层收到RPC之后，会调用Start函数，Start函数会立即返回，应用层等待Raft执行完把命令提交到applych channel之后，应用层从中取出命令，开始执行，然后把结果返回给客户端。综上所述，我们可以的到这样的架构图：

![raft-kv](C:\Users\86183\Desktop\raft-kv.png)

## 3 lab4A

### 3.1 设计思路

- 由于网络问题，clerk可能向服务器发送多次RPC请求，如果服务器没有进行处理就直接发给Raft的话，Raft就会多次执行这个命令，导致线性不一致的情况发生。
- 为了避免这种情况，server就要判断某一个请求是否重复，最简单的方法就是让clerk携带一个全局递增的序列号，server需要在第一次将这个请求应用到状态机时，记录这个序列号，用以判断后续的请求是否重复。由于clerk不是并发的，server只需要记录某个clerk序列号最高的一个请求即可，序列号更低的请求不会出现，只需要考虑请求重复的场景。
- 除了记录某个clerk请求的序列号外，还需要记录执行结果，因为如果是一个重复的get请求，其返回的结果应该与其第一次发送请求时一致，否则将导致线性不一致。如果是重复的Put等改变状态机的请求，则不应该被执行。

### 3.2 具体实现

#### 3.2.1 结构体设计

在Clerk中添加用于标记请求的单调递增的序列号seq，用于标识clerk的identifer，用于记录领导者的leaderId，identifier和seq一起标记了唯一的请求

#### 3.2.2 Client RPC设计

- RPC结构体：RPC请求只需要额外携带identifier和seq，RPC回复则需要携带结果和错误信息。
- Get/Append函数
  - Get函数需要不断轮询server发送RPC请求直到发送成功，如果发送成功则把seq自增然后退出函数，出错则一直轮询
  - Append函数和Get函数的思路相同，只是RPC请求参数不一样。
  - 需要注意的是重试RPC时需要新建reply结构体，重复使用一个结构体将导致labgob报错。

#### 3.2.3 Server实现

##### 3.2.3.1 Server设计思路

根据前文分析可知，RPC handler（就是Get/Put handler)只会在raft层的commit信息到达后才能回复，因此其逻辑顺序就是：

1. 将请求封装后通过接口Start交付给raft层
   1. 如果raft层节点不是Leader，返回响应的错误
   2. 否则继续
2. 等待commit信息
   1. 信息到达，根据commit信息处理回复
   2. 超时，返回相应错误

分析到这里可知，必有一个协程在不断的接受raft的commit日志，这个协程我称为ApplyHandler协程，上面提到的重复RPC判别和处理是在ApplyHandler中进行，因为ApplyHandler是串行的，在其中处理这些日志比较安全，否则通过通道发送给`RPC handler`货条件变量唤醒RPC handler, 都存在一些并发同步的问题, 因此,，ApplyHandler需要进行重复RPC判别和处理(可能需要存储), 并将这个请求(commit log就对应一个请求)的结果返回给RPC handler

##### 3.2.3.2 RPC handler设计

RPC handler需要调用Start等待，等待Commit信息，同时还要额外注意错误的处理：

- 超时信息
- 通道关闭错误
- Leader可能过期的错误
- 不是Leader的错误

同时这里还有一个难点, 就是如果出现了重复的RPC, 他们都在等待commit信息, 那么他们的管道存储在waiCh中的key是什么呢? 如果使用Identifier或Seq, 那么必然后来的RPC会覆盖之前的管道, 可能造成错误, 因为两个重复RPC的Identifier或Seq是一样的。 这里可以巧妙地利用Start函数的第一个返回值， 其代表如果commit成功, 其日志项的索引号, 由于raft层不区分重复RPC的log, 因此这个索引号肯定是不同的, 不会相互覆盖。

##### 3.2.3.3 ApplyHandler设计

ApplyHandler是3A的最核心的部分, 其思路是:  

- 先判断log请求的Identifier和Seq是否在历史记录historyMap中是否存在, 如果存在就直接返回历史记录 
- 不存在就需要应用到状态机, 并更新历史记录historyMap 
- 如果log请求的CommandIndex对应的key在waiCh中存在, 表面当前节点可能是一个Leader, 需要将结果发送给RPC handler

这里有几大易错点:  

- 需要额外传递Term以供RPC handler判断与调用Start时相比, term是否变化, 如果变化, 可能是Leader过期, 需要告知clerk 
- 发送消息到通道时, 需要解锁 因为发送消息到通道时解锁, 所以通道可能被关闭, 因此需要单独在一个函数中使用recover处理发送消息到不存在的通道时的错误 
- 这个ApplyHandler是leader和follower都存在的协程, 只不过follower到应用到状态机和判重那里就结束了, leader多出来告知RPC handler结果的部分

## 4 lab4B

### 4.1 设计思路

简单说, lab3B就是要在底层raft的log过大时生成快照并截断日志, 从而节省内存空间, 并且快照会持久化存储到本地。因此， 原来的代码结构只需要在以下几个方面做出调整：

1. 需要再某个地方定期地判断底层raft的日志大小, 决定是否要生成快照, 生成快照直接调用我们在lab3中实现的接口Snapshot即可 
2. 由于follower的底层raft会出现无法从Leader获取log的情况, 这时Leader会发送给follower的raft层一个快照, raft层会将其上交给server, server通过快照改变自己的状态机 
3. server启动时需要判断是否有持久化的快照需要加载, 如果有就加载

### 4.2 具体实现

#### 4.2.1 快照结构

快照首先应该包含内存中的KV数据库, 也就是自己维护的map, 还应该包含对每个clerk序列号的记录信息, 因为从快照恢复后的server应该具备判断重复的客户端请求的能力, 同时也应该记录最近一次应用到状态机的日志索引, 凡是低于这个索引的日志都是包含在快照中。

#### 4.2.2 加载和生成快照

- 生成快照的函数只需要按照lab3中持久化函数的写法，将快照包含的数据Encode。
- 加载快照的函数也是按照lab3中加载持久化内容的函数，将数据Decode。

#### 4.2.3 生成快照的时机判断

由于ApplyHandler协程会不断地读取raft commit的通道, 所以每收到一个log后进行判断即可，如果大于maxraftstate的百分之九十，就生成快照。

## 5 总结

这个lab终于做完了，相较于lab3来说这个lab的实现难度还是要小一点，lab3是根基，在做这个lab的过程中，又找到了lab3很多设计不合理的地方，导致性能很低，过不了测试。最后又疯狂打印日志，分析日志，才勉强优化的够看，过的时候也是长舒一口气，在实验的实现中，还是借鉴了很多大佬的思想，自己还是太菜了。。。

