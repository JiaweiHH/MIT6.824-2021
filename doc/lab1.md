## Lab1 MapReduce

第一个实验中 Worker 只需要做一件事：在空闲的时候向 Coordinator 请求任务并执行

Coordinator 需要负责：

1. 给 Worker 分配任务
2. 记录已分配任务的超时情况，对于超时的任务重新分配
3. 协调任务，只有当所有的 Map 任务完成之后才可以分配 Reduce 任务

### Worker

Worker 的主循环是判断分配得到的是 Map Task 还是 Reduce Task

```go
for {
  ...
  switch reply.Type {
    case MAP:
      map_for_reduce := do_map(reply.FileName, reply.Index, reply.NReduce, mapf)
      // 提交任务
      submit_task(reply.Index, reply.Type, map_for_reduce)
    case REDUCE:
      do_reduce(reply.FileNames, reply.Index, reducef)
      // 提交任务
      submit_task(reply.Index, reply.Type, nil)
  }
}
```

对于 Map Task，在执行完毕后会将 `KeyValue` 散列到不同的中间文件中，供 Reduce 任务读取，中间文件的命名使用实验提示的 `mr-map_index-reduce_index`

这里我会在提交任务的时候将 ReduceTaskIndex 到 ReduceTaskFiles 的映射提交给 Coordinator，Coordinator 会在预先准备的 Reduce Task 中增加这些文件

对于 Reduce Task，只需要读取文件并调用 `reducef` 即可

### Coordinator

首先是 Coordinator 的定义

```go
type Coordinator struct {
  _stage int // 已经完成的阶段

  _mapTaskNum         int // map 任务的数量
  _reduceTaskNum      int // reduce 任务数量
  _finishedMapTask    int // 已完成的 map 数量
  _finishedReduceTask int // 已完成的 reduce 数量

  _mapTasks    []Task // 分发的 map task
  _reduceTasks []Task // 分发的 reduce task

  _availTasks chan Task // 可用的任务

  _mutex sync.Mutex // mutex
}
```

Coordinator 比较复杂，我这里定义了几个状态

- NONE, 表示 Coordinator 还在处理 Map Task 的阶段
- MAP, 表示 Coordiantor 已经处理完所有的 Map Task，开始处理 Reduce Task 的阶段
- REDUCE, 表示 Coordinator 已经完成了整个 MapReduce 任务

`channel` 用来和 Worker 同步可以拿取的任务

然后是 `Task` 的定义，不管是 Reduce Task 还是 Map Task 都抽象成 `Task`

```go
type Task struct {
  _index      int       // 任务编号
  _type       int       // 表示任务的类型
  _inputFile  string    // map 任务参数
  _inputFiles []string  // reduce 任务参数
  _deadLine   time.Time // 任务截止时间
  _state      bool      // 1: finish, 0: running
}
```

> 这里的定义其实还可以优化，并不是那么完美

Coordinator 提供的几个 RPC 调用：

1. `AskTask`, Worker 申请任务
2. `SubmitTask`, Worker 提交完成的任务

每当有一个任务被 Worker 拿去的时候，就会更新它的 `_deadLine` 表示截止时间，并且会把这个任务放入 `[]_mapTasks` 或 `[]_reduceTasks` 中。在 Coordinator 初始化的时候启动了一个协程，它会不定期的轮询检查有没有 Timeout Task，如果有就重新放回 `channel` 中

所有的 Map Task 都完成的时候，就会把预先初始化好的 Reduce Task 放入 channel

> early exit test 中会调用 `wait -n`，在 MacOS 下会失败，需要在测试脚本中改成 `wait`