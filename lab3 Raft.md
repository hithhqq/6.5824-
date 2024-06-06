# lab3 Raft

lab3的内容是实现Raft算法，Raft是分布式系统中的一种强Leader模型的一致性共识算法，和Paxos一样高效，并且比Paxos更容易理解。我觉得这个实验是整个6.5824最难的实验，不仅要完全理解Raft的逻辑，还要考虑高并发场景下可能出现的问题，比如避免死锁，减少锁争用。我觉得这个实验的细节非常多，稍微不慎有个地方没有处理好就会出来很多bug，下一个实验还是在这个实验的基础上建立的，也就是说这个实验的bug如果没有完全修复，在下个实验仍然会暴露出来。我强烈建议在做实验之前要多读几遍Raft的[原论文](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)。

## lab3A

### 1 任务描述

实现Raft `Leader选举`和`心跳监测`（通过发送AppendEntries RPC请求但不带日志条目）。第 2A 部分的目标是选举单个领导者，在没有失败时让 Leader 继续担任;如果旧 Leader 失败或旧 Leader 的数据包丢失，则新 Leader 可以接管。运行`go test -run 2A`以测试 2A 代码。

### 2 实现思路

通过阅读论文和lab文档，我们可以知道3A的要求是这样的：

- 在运行时`Leader`不断给`Follower`发送`心跳`，`Follower`收到之后回复，这个心跳是通过AppendEntries RPC实现的，只不过其中的Entries[]是空的
- 选举
  - 指定一个选举超时时间，当选举超时时，`Follwer`转换为`Candidate`并向其它所有`Server`发送投票请求，并且会先为自己投票，并增加任期
  - 每一个收到投票请求的`Server`，会判断目标`Server`是否满足当`Leader`的条件，满足则投票
  - 每一个`Server`在一个`任期`中只能投票`一次`（保证最多只会有一名Candidate能够赢得选举）
  - 如果`超时选举`的`Server`收到了超过`一半`的`Server`给它投票，那么它就会成为`Leader`
  - 成为`Leader`之后，就会在给其他的Server发送周期性心跳`不携带日志条目的AppendEntries RPC`来维持它的权威性，收到心跳的Server更新自己的状态
  - 如果一个`Follower`在一段时间里没有收到`Leader`发过来的任何消息，则它假设当前没有Leader并且发起选举来选择一个新的Leader
  - 为了防止许多Follower都在同一时间成为Candidate而导致投票被瓜分一直没有Candidate可以成为Leader，会将每一个Server的选举超时时间随机化。

### 3 具体细节

- 首先要按照论文补全`Raf`t、`RequestVoteArgs`、`RequestVoteReply`结构体，在Raft里还需要额外定义一个变量来判断是否选举超时，因为lab指导书里提到time.Timer很难使用正确，所以我们在这里要先使用`time.Time`，用`time.Since`来判断是否超时需要发起选举，同时还要定义`心跳间隔时间`和`选举超时时间`，这两个时间的定义需要根据课程里教授提到的`合理定制`，不然一些测试可能会超时，论文里还提到了要随机化超时时间，所以要自己定义一个`获取随机数`的函数。在选举完成之后要根据自己所得票数判断是否当选Leader，所以还要新增一个voteCount的变量和一把锁voteMu来保护voteCount。
- 补全`GetState`函数(这个函数只需要判断当前Server是不是Leader），因为发送的心跳是不携带日志条目的AppendEntries，所以还要创建`AppendEntriesArgs`和`AppendEntriesReply`结构体，参数在论文中也已给出。
- 在`make`函数中把Raft新添加的属性初始化。
- 补全RequestVote函数，这是Server判断是否需要给发起选举的Candidate投票的RPC处理函数，首先要判断rf的term和args的term的关系，如果rf的term大于args的term则返回false，说明args的term已经过期了，如果rf的term小于args的term，则要更新rf的term，并把rf的votefor置为-1，因为rf可能在它的term给别的server投票了，同时要把role更新为Follower。接着进行判断请求选举的server的log是否和当前的server一样新，论文里给出的一样新的定义是args的最后一条log的term>当前server的最后一条log的term，或者args的最后一条log的term=当前server的最后一条log的term并且args的最后一条log的索引>=当前server的最后一条log的索引。如果是一样新则给同意投票，否则拒绝投票。
- 补全tricker函数，tricker的作用是监测选举超时，如果超时发起新一轮选举，并刷新超时时间。
- 定义Elect函数，Elect函数是启动选举的函数，先把自己的term自增，然后把role转换为Candidate，接着给自己投票，并将voteCount置为1，然后向每一个Server发送RPC请求，拿到请求之后的结果，判断对方是否给自己投票，如果是则增加voteCount并判断voteCount是否大于server数量的一半，如果大于则将自己的role转换为Leader并开始周期性的发送心跳。
- 定义发送心跳的函数，如果该server没有被kill，则会在心跳间隔到了之后给其它所有server发送不带有log的AppendEntries，发送完之后刷新自己的心跳间隔。
- 定义handleAppendEntries函数，这个函数是发送AppendEntriesRPC请求，并处理返回结果的函数。因为发送的心跳没有log所以只需要判断reply的Term和当前server的term的关系，如果reply的Term大于当前server的term说明当前server的term已经过期，外面已经进行了新的选举，所以要把自己的term改为reply的term并把role改为Follower，并把voteFor置为-1，然后重置自己的选举超时时间。
- 定义AppendEntries函数，这个函数是RPC处理函数，因为发送的心跳没有log，所以在这个lab里这个函数的实现也比较简单，只需要判断目标server发来的args的term和自己term的关系，如果args的term小于自己的term，说明目标的term已经过期，不对这个args进行处理，将reply的success值为false并返回，如果args的term大于自己的term，则把自己的改为args的term，同时把自己的votefor改为-1，把role改为Follower。

#### 4 总结

到了这里lab3A基本上已经完成了，这个lab容易出错的地方不算很多，下面是一些要注意的点：

- 访问临界区的资源要注意上锁，同时也要注意解锁，如果对锁使用不熟练可以用一把大锁保平安
- voteFor和term是绑定的如果term改了则votefor也要改
- 如果出现bug了要多使用lab提供的Dprintf来打印日志进行Debug，使用时要记得把Debug置为true
- 要记得使用go test -race来测试有哪些地方存在线程安全问题
