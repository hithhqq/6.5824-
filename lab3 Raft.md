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
- 补全`RequestVote`函数，这是Server判断是否需要给发起选举的Candidate投票的RPC处理函数，首先要判断rf的term和args的term的关系，如果rf的term大于args的term则返回false，说明args的term已经过期了，如果rf的term小于args的term，则要更新rf的term，并把rf的votefor置为-1，因为rf可能在它的term给别的server投票了，同时要把role更新为Follower。接着进行判断请求选举的server的log是否和当前的server一样新，论文里给出的一样新的定义是args的最后一条log的term>当前server的最后一条log的term，或者args的最后一条log的term=当前server的最后一条log的term并且args的最后一条log的索引>=当前server的最后一条log的索引。如果是一样新则给同意投票，否则拒绝投票。
- 补全`tricker`函数，tricker的作用是监测选举超时，如果超时发起新一轮选举，并刷新超时时间。
- 定义`Elect`函数，Elect函数是启动选举的函数，先把自己的term自增，然后把role转换为Candidate，接着给自己投票，并将voteCount置为1，然后向每一个Server发送RPC请求，拿到请求之后的结果，判断对方是否给自己投票，如果是则增加voteCount并判断voteCount是否大于server数量的一半，如果大于则将自己的role转换为Leader并开始周期性的发送心跳。
- 定义发送心跳的函数，如果该server没有被kill，则会在心跳间隔到了之后给其它所有server发送不带有log的AppendEntries，发送完之后刷新自己的心跳间隔。
- 定义`handleAppendEntries`函数，这个函数是发送AppendEntriesRPC请求，并处理返回结果的函数。因为发送的心跳没有log所以只需要判断reply的Term和当前server的term的关系，如果reply的Term大于当前server的term说明当前server的term已经过期，外面已经进行了新的选举，所以要把自己的term改为reply的term并把role改为Follower，并把voteFor置为-1，然后重置自己的选举超时时间。
- 定义`AppendEntries`函数，这个函数是RPC处理函数，因为发送的心跳没有log，所以在这个lab里这个函数的实现也比较简单，只需要判断目标server发来的args的term和自己term的关系，如果args的term小于自己的term，说明目标的term已经过期，不对这个args进行处理，将reply的success值为false并返回，如果args的term大于自己的term，则把自己的改为args的term，同时把自己的votefor改为-1，把role改为Follower。

### 4 总结

到了这里lab3A基本上已经完成了，这个lab容易出错的地方不算很多，下面是一些要注意的点：

- 作为RPC参数的结构体需要把成员变量的首字母大写

- 访问临界区的资源要注意上锁，同时也要注意解锁，如果对锁使用不熟练可以用一把大锁保平安
- `voteFor`和`term`是绑定的如果term改了则votefor也要改
- 如果出现bug了要多使用lab提供的`Dprintf`来打印日志进行Debug，使用时要记得把Debug置为true
- 要记得使用`go test -race`来测试有哪些地方存在线程安全问题
- 有些bug可能一次测试不会出现，建议写个shell进行多次测试

## lab3B

### 1 任务描述

实现raft中的日志复制和提交，同时也要进行Leader和Follower的日志一致性检查。

### 2 实现思路

- ##### Leader视角

  1. leader收到client的命令后，会将其构造成一个日志项，将当前节点的currentTerm作为日志项的term，并将其追加到自己的log中。

  2. Leader之后会给所有的server发送AppendEntries RPC将log复制到所有的节点，AppendEntries RPC需要增加PrevLogIndex、PrevLogTerm以供follower校验，其中PrevLogIndex、PrevLogTerm由nextIndex确定。

  3. 如果RPC返回了成功，则更新matchIndex和nextIndex，同时寻找一个满足过半的matchIndex[i] >= N的索引位置N，将其更新为自己的commitIndex，并提交直到commitIndex部分的日志项

  4. 如果RPC返回了失败，且伴随的Term更大，表示自己已经不是Leader了，将自身的角色转换为Follower，并更新currentTerm和voteFor，重启计时器。

  5. 如果RPC返回了失败，且伴随的Term和自己的currentTerm相同，则有可能是Follower的日志和Leader的日志有矛盾，将nextIndex自减重试。

- ##### Follower视角

  1. Follower收到client的命令后，会将leader返回给client。
  2. Follower收到AppendEntries RPC后，通过PrevLogIndex、PrevLogTerm可以判断出leader认为自己log的结尾位置是否和Leader的log匹配，如果不匹配，返回false并不执行操作。
  3. 如果上述位置的信息匹配，则需要判断插入位置之后是否有冲突的日志项，如果有，则将log中冲突的内容清除。
  4. 将RPC中的日志项追加到log中。
  5. 根据RPC的传入参数更新commitIndex，并提交直到commitIndex部分的日志项。

### 3 具体细节

- 要先补全Start函数，先判断该server是不是Leader如果不是直接返回false，如果是，则需要把命令添加到日志里，并返回true，最后一条日志的索引和当前任期。
- 然后在labA的基础上补全sendHeartBeats函数，如果发给erver的args的PrevLogIndex小于log的长度，则说明有log需要发送给这个server，把需要发送的log添加在args的Entries里。
- 然后补全AppendEntries函数，如果args的Entries不为空，则需要根据args.PrevLogIndex来进行一致性检查，我们使用的是课上教授讲的优化的nextIndex回退方法，需要在AppendEntriesReply的结构体加上XLen（返回Follower的日志长度）,XIndex（返回Follower和Leader出现矛盾的log的索引）,XTerm（返回Follower和Leader出现矛盾的log的任期）这三个成员，逻辑如下：
  - 如果args.PrevLogIndex大于该Follower的日志长度，则需要设置eply.XTerm = -1告知Leader，并将reply.XLen设置为自身日志长度，让Leader把Follower的日志长度之后的log发送给该Follower。
  - 如果该Follower在args.PrevLogIndex处的日志的term不等于args.PrevLogTerm则说明这个任期的log和Leader的log冲突，需要找到这个冲突的任期第一次出现的位置，把XIndex置为这个位置，将XTerm置为这个log的term。
  - 如果log出现了矛盾则直接返回false（表示一致性检查失败）。
  - 如果没有矛盾，则需要把该Follower在args.PrevLogIndex之后的log全部删除，并把args.Entries添加到rf的log之后。并返回true（表示一致性检查成功）。返回的结果会在下一次Leader发送心跳时进行处理。
- 最后再补全handleAppendEntries函数，对AppendEntries RPC函数返回的结果进行处理，如果一致性检查失败，则需要根据Reply返回的XLen，XTerm，XIndex将正确的Log传入args的Entries，并更新该server的nextIndex。如果一致性检查成功，则需要更新该server的nextIndex和matchedIndex，然后外层倒序遍历log直到commitIndex，内层遍历每一个server判断该server的这个log是否已经复制，遇到自己则跳过，如果一半以上的server都复制了这个log则把commitIndex加一，并通知commitChecker函数把该日志所包含的命令应用到状态机。
- 定义CommitChecker函数，同时需要在Raft结构体里定义一个条件变量，用来通知该函数，和一个管道，用来把命令传到状态机，要注意把这两个新增的成员变量在make中初始化。这个函数在rf没有被killed的时，一直在监测是否有命令需要应用到状态机。首先要判断commitIndex是否大于lastApplied，如果小于则需要释放锁并等待，否则需要把rf.lastApplied到rf.commitIndex之间的日志封装在ApplyMsg结构体里通过applych应用到状态机。这个函数在go rf.tricker之后，用一个GoRoution执行。

### 4 总结

这个lab的Hint要比labA少很多，细节处理也要更加复杂，下面是一些容易出错的点：

- 在初始化applych时一定要用make参数里穿的applych，不要自己用make初始化，我因为这个点Debug了好久。
- 在判断Leader的log和Follower的log是否有冲突时逻辑必须清晰，不然容易出现误判或者考虑不周的情况。
- 同样要多用Dprintf来打印日志。
- 要多注意锁的使用，在CommitChecker函数中，因为向applych中传数据可能会阻塞，所以要在之前先解锁，在之后再加锁，不然容易死锁。
- 有些bug可能一次测试不会出现，建议写个shell进行多次测试。

## lab3C

### 	1 任务描述

​		如果基于 Raft 的服务器重新启动，它应该在中断的地方恢复服务。这要求 Raft 在重启后，依旧能确保数据持久化。

### 	2 实现思路

​		只需要把`persist`函数和`readPersist`函数按照给出的注释补全就行了，这两个函数比较简单。

### 	3 具体细节

​		根据论文所讲，在这个lab中持久化的内容只包括：votedFor`, `currentTerm`, `log，这三个变量，所以在每次更改了这三个变量之后都需要执行一次persist函数。

### 	4 总结

​		这个lab是这四个lab中最简单的一个，只需要注意在修改上面提到的三个变量之后不要漏掉readPersist函数就可以了，如果之前的lab没有一流bug的话，这个lab还是比较容易完成的。

## lab3D

### 1 任务描述

这个lab要求实现raft中的快照功能。

### 2 实现思路

我们需要实现的SnapShot是在raft之上的service层提供的，因为raft层并不理解所谓的状态机内部状态的机制，因此有必要了解整个代码的层次结构：

![raft_diagram](C:\Users\86183\Desktop\raft_diagram.png)

上面的图是官方的指导书给出的代码层级的接口的示意图。service和raft的交互逻辑如下：

1. 位于raft层之上的Servcice，通过Start发送命令给Leader一侧的raft层，Leader raft会将日志复制给集群中的其它Follower raft，Follower raft通过applych这个管道将已经提交的包含命令的日志项向上发送给Follower侧的service。
2. 某一时刻service里日志的数量过大，为了减轻内存压力，将状态机状态封装成一个SnapShot并将请求发送给Leader一侧的raft（Follower侧的service也会存在快照操作），raft层保存SnapShot并截断自己的log数组，当通过心跳发现Follower的节点的log落后SnapShot时，通过InstallSnapshot发送给Follower，Follower保存SnapShot并将快照发送给service。
3. raft之下还存在一个持久化层Persistent Storage，也就是Make函数提供的Persister，调用相应的接口实现持久化存储和读取。

### 3 具体细节

- #### SnapShot设计

  - 因为发送SnapShot后需要截断日志，而raft结构体中的字段如commitIndex，lastApplied等，等存储的仍然是全局递增的索引，所以需要在raft结构体中要新增snapShot（快照）、lastIncludedIndex（被截断后日志的最高索引，是全局真实索引），lastIncludedTerm（lastIncludedIndex的term）。
  - 将全局真实索引称为Virtual Index， 将log切片使用的索引称为Real Index， 新定义两个函数：一个用来求Real Index，例如将commitIndex转换为真实的Index，需要将commitIndex减去rf.lastIncludedIndex，另一个用来求Virtual Index，需要将RealIndex加上rf.lastIncludedIndex。
  - 有了RealIndex和VirtualIndex，需要在代码中遵循以下的规则：
    - 访问rf.log一律使用真实的切片索引，即Real Index
    - 其余情况一律使用全局真实递增的索引，即Virtual Index

- #### SnapShot函数设计

  SnapShot函数很简单，接收service层的快照请求，并截断自己的log数组，以下几点需要注意：

  - 判断是否接受SnapShot
    - 创建SnapShot时，必须保证其index小于等于commitIndex，如果index大于commitIndex，则会有包含未提交日志的风险。快照中不应该包含未被提交的日志项。
    - 创建SnapShot时，必须保证其index小于等于lastIncludedIndex，因为这可能是一个重复的或者更旧的快照请求RPC, 应当被忽略。
  - 将snapshot保存，因为后续Follower可能需要snapshot，以及持久化时需要找到snapshot进行保存，因此此时要保存以便后续发送给Follower。
  - 除了更新lastIncludedTerm和lastIncludedindex外，还需要检查lastApplied是否位于SnapShot之前，如果是，需要调整到与index一致，因为快照中保存的日志一定是被应用的。
  - 调用persist持久化。

- #### 相关持久化函数

  - ##### persist函数

    添加快照后，调用persist时还需要编码额外的字段lastIncludedIndex和lastIncludedTerm，在调用Save函数时需要传入快照rf.snapShot

  - ##### readPersist函数

    需要额外解码字段lastIncludedIndex和lastIncludedTerm，同时注意把commitIndex和lastApplied置为lastIncludedIndex，因为快照包含的索引一定是被提交和应用的，此操作可以避免后续的索引越界问题。

- #### make函数

  初始化时，索引需要变为Virtual Index，同时也要初始化每一个server的nextIndex。

- #### InstallSnapshot RPC设计

  - ##### RPC结构体设计

    按照论文给出的描述设计就好，如果你的log索引不是从0开始的，则需要新增一个interface类型的LastIncludedCmd，用于在log的0处占位。

  - ##### InstallSnapshot RPC发起端设计

    - 在SendHeartBeats时发现PrevLogIndex  < lastIncludedIndex，表示将要发送的日志项已经被截断，需要改发送心跳为发送InstallSnapshot RPC。
    - handleAppendEntries检查心跳RPC的回复，并进行相应的回退，如果发现已经回退到lastIncludedIndex还不能满足要求，就需要把nextIndex改为rf.lastIncludedIndex，等待下次发送心跳时发送InstallSnapshot RPC。
    - 这里会有3个情况触发发送InstallSnapshot RPC:
      1. Follower的日志过短(PrevLogIndex这个位置在Follower中不存在), 甚至短于lastIncludedIndex
      2. Follower的日志在PrevLogIndex这个位置发生了冲突, 回退时发现即使到了lastIncludedIndex也找不到匹配项(大于或小于这个Xterm)
      3. nextIndex中记录的索引本身就小于lastIncludedIndex

  - ##### InstallSnapshot发送

    - 这里的实现相对简单, 只要构造相应的请求结构体即可, 但需要注意:
      1. 需要额外发生Cmd字段, 因为构造0索引时的占位日志项, 尽管其已经被包含在了快照中
      2. 发送RPC时不要持有锁
      3. 发送成功后, 需要将nextIndex设置为VirtualLogIdx(1), 因为0索引处是占位, 其余的部分已经不需要再发送了
      4. 和心跳一样, 需要根据回复检查自己是不是旧Leader

  - ##### InstallSnapshot响应

     InstallSnapshot响应需要考虑更多的边界情况:

    1. 如果是旧leader, 拒绝
    2. 如果Term更大, 证明这是新的Leader, 需要更改自身状态, 但不影响继续接收快照
    3. 如果LastIncludedIndex位置的日志项存在, 即尽管需要创建快照, 但并不导致自己措施日志项, 只需要截断日志数组即可
    4. 如果LastIncludedIndex位置的日志项不存在, 需要清空切片, 并将0位置构造LastIncludedIndex位置的日志项进行占位
    5. 需要检查lastApplied和commitIndex 是否小于LastIncludedIndex, 如果是, 更新为LastIncludedIndex
    6. 完成上述操作后, 需要将快照发送到service层
    7. 由于InstallSnapshot可能是替代了一次心跳函数, 因此需要重设定时器

### 4 总结

这个lab终于做完了，我觉得这个是最难的lab，各种bug层出不穷，看着打印的满屏日志，几乎感到绝望。。。

主要难度在于：

1. 将日志数组截断后, 需要实现全局索引和数组索引之前的转化, 需要考虑更多数组越界的边界条件
2. 在接收InstallSnapshot RPC安装快照后, lastApplied可能已经落后于快照产生时的日志索引, 是无效的日志项, 不应该被应用
3. 由于安装快照后, Leader的nextIndex在回退时可能出现索引越界, 需要考虑边界情况

个人体会是, 本Lab的核心技能是: **一定要学会从日志输出中诊断问题!!!**

索引越界问题可能直接报错，根据报错信息去分析导致越界的代码，还是比较容易找出问题所在，但是比如死锁，逻辑问题等bug就必须要靠打印日志来debug了。
