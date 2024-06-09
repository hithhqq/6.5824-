# Lab1 MapReduce

## 1 实验介绍

本次实验是实现一个建议版本的MapReduce编程框架， 官方文档在这里：[lab1文档](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)，强烈建议先阅读MapReduce的[论文](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)，这个lab的难度主要体现在设计上，实际的代码相对简单。实验需要实现的是Coordinator和Worker的设计，具体实现十分自由(~~无从下手~~)

## 2 框架解读

解读现有的框架设计是第一步。

### 2.1 代码解读

1. 阅读`src/main/mrcoordinator.go`可知： 服务进程通过`MakeCoordinator`启动了一个`Coordinator` c, `c.server()`中启用了一个协程用于接受`RPC`调用:`go http.Serve(l, nil)`, 需要注意的是, 在 Go 的 `net/http` 包中, 使用 `http.Serve(l, nil)` 启动 `HTTP` 服务器以侦听和处理请求时，服务器会为每个进来的请求自动启动一个新的协程。这意味着每个 RPC 调用都是在其自己的独立协程中被处理的，允许并发处理多个请求。因此, 我们的设计可能需要使用锁等同步原语实现共享资源的保护, 同时`Coordinator`不会主动与`Worker`通信(除非自己额外实现), 只能通过`Worker`的`RPC`通信来完成任务。同时， 当所有任务完成时， `Done`方法将会返回`false`, 从而将`Coordinator`关闭。
2. 阅读`src/main/mrworker.go` 可以得知，`mrworker.go`仅仅通过`Worker`函数来运行, 因此`Worker`函数需要完成请求任务、执行任务、报告任务执行状态等多种任务。因此可以猜测，Worker需要再这个函数里不断地轮训`Coordinator`，根据`Coordinator`的不同回复驱使当前`Worker`完成各种任务。

### 2.2 误区解读

1. `Map`、`Reduce`任务、`Coordinator`和`Worker`的关系如何? 这些任务(文中此后称为`Task`)与`Worker`是什么关系呢? 是否存在对应关系? 这些对应关系需要记录吗? 通常, 在常见的主从关系中, 主节点需要记录从节点的信息,例如线程id等表名身份的信息, 但在我们的`MapReduce`中却没有这样的必要, 因为`Worker`节点是可以扩容的, 而`Coordinator`与`Worker`之间只有传递`Task`相关信息的需求, 因此`Coordinator`只需要记录`Task`任务的状态即可, `Task`分配给`Worker`后执行可能成功或失败, 因此`Coordinator`还需要维护任务执行的时间信息, 以便在超时后重新分配任务。因此，`Map`、`Reduce`任务、`Coordinator`和`Worker`的关系可以参考下图:

   [![MapReduce_relation](https://github.com/ToniXWD/MIT6.5840/raw/master/images/MapReduce_relation.png)](https://github.com/ToniXWD/MIT6.5840/blob/master/images/MapReduce_relation.png)

   `Worker`可能在不同时间执行不同的`Task`, 也可能什么也不做(初始状态或等候所有`Map Task`完成时可能会闲置)

2. `Map`、`Reduce`任务有多少个? 如何分配?

   - `Map Task`实际上在此实验中被简化了, 每个`Map Task`的任务就是处理一个`.txt`文件, 因此`Map Task`的数量实际上就是`.txt`文件的数量。 因此, 每个`.txt`文件对应的`Map Task`需要`Coordinator`记录其执行情况并追踪。
   - `Reduce Task`的数量是`nReduce`。由于`Map Task`会将文件的内容分割为指定的`nReduce`份, 每一份应当由序号标明, 拥有这样的序号的多个`Map Task`的输出汇总起来就是对应的`Reduce Task`的输入。

3. 中间文件的格式是怎么样的? `Reduce`任务如何选择中间文件作为输入? 因为`Map Task`分割采用的是统一的哈希函数`ihash`, 所以相同的`key`一定会被`Map Task`输出到格式相同的中间文件上。例如在`wc`任务中, `Map Task 1`和`Map Task 2`输入文件中都存在`hello`这个词, `Map Task 1`中所有的`hello`会被输出到`mr-out-1-5`这个中间文件, `1`代表`Map Task`序号, `5`代表被哈希值取模的结果。那么，`Map Task 2`中所有的`hello`会被输出到`mr-out-2-5`这个中间文件。那么`Reduce Task 5`读取的就是形如`mr-out-*-5`这样的文件。

## 3 设计与实现

### 3.1 `RPC`设计

#### 3.1.1. 消息类型

,通信时首先需要确定这个消息是什么类型, 通过前述分析可知, 通信的信息类型包括:

- `Worker`请求任务
- `Coordinator`分配`Reduce`或`Map`任务
- `Worker`报告`Reduce`或`Map`任务的执行情况(成功或失败)
- `Coordinator`告知`Worker`休眠（暂时没有任务需要执行）
- `Coordinator`告知`Worker`退出（所有任务执行成功）

每一种消息类型会需要附带额外的信息, 例如`Coordinator`分配任务需要告知任务的ID, `Map`任务还需要告知`NReduce`,和输入文件名。 综上考虑, 消息类型的定义如下(`Send`和`Reply`是从`Worker`视角出发的):

```go
type MessageSend struct {
	MsgType MsgType // 消息类型
	TaskID  int // Worker回复的消息类型如MapSuccess等需要使用
}
```

```go
type MessageReply struct {
	MsgType  MsgType // 消息类型
	NReduce  int    // MapTaskAlloc需要告诉Map Task 切分的数量
	TaskID   int    // 任务Id用于选取输入文件
	TaskName string // MapSuccess专用: 告知输入.txt文件的名字
}
```

#### 3.1.2 通信函数设计

在我的设计中，`Worker`只需要有2个动作:

- 向`Coordinator`请求`Task`
- 向`Coordinator`报告之前的`Task`的执行情况

因此, `worker.go`中我设计了两个通信函数:` CallForReportStatus`，`CallForTask`

- CallForReportStatus负责在每次执行完Map任务或者Reduce任务之后，向Coordinator报告任务执行情况
- CallForTask负责向Coordinator请求任务

### 3.2 `Worker`设计

#### 3.2.1 `Worker`主函数设计

由之前的分析可以看出，`Woker`所做的内容就是不断的请求任务、执行任务和回复任务执行情况，因此，可以很容易地写出`Worker`函数:

- 外层用一个死循环，不断用CallForTask函数向Coordinator请求任务，判断Coordinator发放任务的类型，分别做响应的处理，处理之后，将处理结果发送回Coordinator。
- 任务一共有以下几种类型：
  - Map任务
  - Reduce任务
  - Wait 说明没有任务可以发放
  - ShutDown说明所有任务已完成，可以结束了

#### 3.2.2 `Map Task`执行函数

`HandleMapTask`函数是执行具体的`MapTask`, 这个函数 可以从`mrsequential.go`中取一部分代码，主要思路是，取出Coordinator发放的任务，然后用mapf进行映射，将结果按照key值排序之后，存入创建的临时文件里，完成操作后再进行原子重命名。

##### 3.24 `Reduce Task`执行函数

Reduce Task收集对应序号的中间文件，汇总后应用指定的reduce函数。实现思路和Map Task执行函数差不多。

### 3.3 `Coordinator`设计

#### 3.3.1 `TaskInfo`设计

首先需要考虑的是, 如何维护`Task`的执行信息, `Task`执行状态包括了: 未执行、执行中、执行失败、执行完成。 这里有一个很重要的问题需要考虑， 超时的任务时什么状态呢？因为在我的设计中，`Coordinator`与`Worker`是通过`RPC`来驱动彼此运行的, 当然你也可以启动一个`goroutine`间隔地检查是否超时, 但为了使设计更简单, 我们可以这样设计检查超时的方案:

1. 为每个`Worker`分配`Task`时需要记录`Task`被分配的时间戳, 并将其状态置为`running`
2. 为每个`Worker`分配`Task`, 遍历存储`TaskInfo`的数据结构, 检查每一个状态为`running`的`Task`的时间戳是否与当前时间戳差距大于`10s`, 如果是, 则代表这个`Task`超时了, 立即将它分配给当前请求的`Worker`, 并更新其时间戳
3. 如果导致`Task`超时的老旧的`Woker`之后又完成了, 结果也就是这个`Task`返回了多次执行成功的报告而已, 可忽略

> PS: `Worker`执行失败有2种, 一种是`Worker`没有崩溃但发现了`error`, 这时`Worker`会将错误报告给`Coordinator`, `Coordinator`会将其状态设置为`failed`, 另一种情况是`Worker`崩溃了, 连通知都做不到, 这就以超时体现出来, 处理好超时即可

`TaskInfo结构`如下：

```go
type taskStatus int

// Task 状态
const (
	idle     taskStatus = iota // 闲置未分配
	running                    // 正在运行
	finished                   // 完成
	failed                     //失败
)

// Map Task 执行状态
type MapTaskInfo struct {
	TaskId    int        // Task 序号
	Status    taskStatus // 执行状态
	StartTime int64      // 开始执行时间戳
}

// Reduce Task 执行状态
type ReduceTaskInfo struct {
	// ReduceTask的 序号 由数组下标决定, 不进行额外存储
	Status    taskStatus // 执行状态
	StartTime int64      // 开始执行时间戳
}

type Coordinator struct {
	// Your definitions here.
	NReduce       int                     // the number of reduce tasks to use.
	MapTasks      map[string]*MapTaskInfo //MapTaskInfo
	MapSuccess    bool                    // Map Task 是否全部完成
	muMap         sync.Mutex              // Map 锁, 保护 MapTasks
	ReduceTasks   []*ReduceTaskInfo       // ReduceTaskInfo
	ReduceSuccess bool                    // Reduce Task 是否全部完成
	muReduce      sync.Mutex              // Reduce 锁, 保护 ReduceTasks
}
```

#### 3.3.2 `RPC` 响应函数-`AskForTask`

这部分算是较为复杂的, 其逻辑如下:

1. 如果有闲置的任务(`idle`)和之前执行失败(`failed`)的`Map Task`, 选择这个任务进行分配
2. 如果检查到有超时的任务`Map Task`, 选择这个任务进行分配
3. 如果以上的`Map Task`均不存在, 但`Map Task`又没有全部执行完成, 告知`Worker`先等待
4. `Map Task`全部执行完成的情况下, 按照`1`和`2`相同的逻辑进行`Reduce Task`的分配
5. 所有的`Task`都执行完成了, 告知`Worker`退出

#### 3.3.3 `RPC` 响应函数-`NoticeResult`

这个函数只需要根据传进来的参数判断任务的执行情况，然后更改任务的状态。

#### 3.3.4 `Done`方法

这个函数只需要遍历所有的Task，判断是否全部完成。

## 4 总结

这个lab虽然只是第一个lab但是难度却一点也不低，因为这个lab的实现方式非常自由，所以最开始一度感觉无法下手，又去看了一边课，阅读了原论文，看了其他人的思路，才渐渐有了眉目。