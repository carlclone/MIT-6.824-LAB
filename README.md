
All labs and assignment for the course

- [x] Lab1 MapReduce
  - [x] WordCount test 
  - [x] Indexer test 
  - [x] Map parallelism test 
  - [x] Reduce parallelism test 
  - [x] Task timeout redistribute test 
  - [x] Crash Test 
  
- [x] Lab2 Raft
  - [x] Part 2A 
    - [x] Initial Election 
    - [x] ReElection with network failure 


  - [x] Part 2B
    - [x] TestBasicAgree2B 最理想情况下的客户端请求 ,日志复制
    - [x] TestRPCBytes2B 保证一次agreement 每个 peer只发送一次 RPC
    - [x] TestFailAgree2B 
    - [x] TestFailNoAgree2B 
    - [x] TestConcurrentStarts2B 
    - [x] TestRejoin2B 
    - [x] TestBackup2B 
    - [x] TestCount2B 优化,减少一次选举需要的 RPC 次数

  - [x] Part 2C
    - [x] Some2cPersistSimpleTest
    - [x] TestFigure82C
    - [x] TestUnreliableAgree2C
    - [x] TestFigure8Unreliable2C *****
    - [x] TestUnreliableChurn2C *****
    - [x] TestReliableChurn2C


- [ ] Lab3 KV Raft
- [ ] Lab4 Sharded KV


---


### Lab2

2020/06/04 ~ 2020/06/07

为了解决TestFigure8Unreliable2C跑 100 次不能稳定通过,以至于不敢做 lab3,隔了一个月的时间,开始了 lab2 第三次重写,用回了第一次写时候的思路,event-driven,只不过这次把需要并发和不能并发的逻辑理清楚了,每个 peer 有 3 个线程,一个选举超时线程,一个心跳线程,一个主事件线程(主要是心跳,投票请求和响应的处理)

说一下需要并发的逻辑: 群发心跳包,群发选票 , 但是发送之前的参数准备是不能并发的,需要加锁(和主事件线程互斥)

然后是TestFigure8Unreliable2C这个 case, 必须优化日志复制的逻辑,否则跑 100 次的通过率会很低,有 commited 超时限制 ,  需要实现 fast backup (快速回退): 当 follower 发现 prevLogTerm 不一致的时候,会在本机的 log 中找到该 term 第一个 log 的 index, 发给 leader . 如果没有 , 那么 leader会把该 follower 的 nextIndex 置为 leader 的 log 中该 term 的第一个 index , 实现快速回退一个任期的 log , 而不是一个一个回退



### FailAgree 和 FailNoAgree 的场景

摘抄自https://pdos.csail.mit.edu/6.824/notes/l-raft2.txt

```
how can logs disagree after a crash?
  a leader crashes before sending last AppendEntries to all
    S1: 3
    S2: 3 3
    S3: 3 3
  worse: logs might have different commands in same entry!
    after a series of leader crashes, e.g.
        10 11 12 13  <- log entry #
    S1:  3
    S2:  3  3  4
    S3:  3  3  5

Raft forces agreement by having followers adopt new leader's log
  example:
  S3 is chosen as new leader for term 6
  S3 sends an AppendEntries with entry 13
     prevLogIndex=12
     prevLogTerm=5
  S2 replies false (AppendEntries step 2)
  S3 decrements nextIndex[S2] to 12
  S3 sends AppendEntries w/ entries 12+13, prevLogIndex=11, prevLogTerm=3
  S2 deletes its entry 12 (AppendEntries step 3)
  similar story for S1, but S3 has to back up one farther
  ```

### Lab2

2020/06/04 ~ 2020/06/07

分支在`rewrite_lab2_0604`

为了解决TestFigure8Unreliable2C跑 100 次不能稳定通过,以至于不敢做 lab3,隔了一个月的时间,开始了 lab2 第三次重写,用回了第一次写时候的思路,event-driven,只不过这次把需要并发和不能并发的逻辑理清楚了,每个 peer 有 3 个线程,一个选举超时线程,一个心跳线程,一个主事件线程(主要是心跳,投票请求和响应的处理)

说一下需要并发的逻辑: 群发心跳包,群发选票 , 但是发送之前的参数准备是不能并发的,需要加锁(和主事件线程互斥)
不能并发的逻辑:  除开并发逻辑之外的基本都是 , 主要有心跳,选票请求和响应, 选举超时事件 , 客户端发起的agreement

然后是TestFigure8Unreliable2C这个 case, 必须优化日志复制的逻辑,否则跑 100 次的通过率会很低,有 commited 超时限制 ,  需要实现 fast backup (快速回退): 当 follower 发现 prevLogTerm 不一致的时候,会在本机的 log 中找到该 term 第一个 log 的 index, 发给 leader . 如果没有 , 那么 leader会把该 follower 的 nextIndex 置为 leader 的 log 中该 term 的第一个 index , 实现快速回退一个任期的 log , 而不是一个一个回退

总算没有烂尾..........


### Lab1

Lab1 用了 3 天时间,没什么难度,就不做总结了… 不像 Lab2 一个 bug 就是3天


### 资料

[课表](https://pdos.csail.mit.edu/6.824/schedule.html)


某个场景的paper
https://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p85.pdf

 
## 思考记录

加锁意义不明 每一段代码都要能说出为什么存在 意义是什么

加锁的粒度可以很好的参考acid的3个情况 write write. Read write  write read 看你是要避免什么问题来决定锁的粒度 比如两个线程加一 write write问题

并发编程的设计不能直接把语言转换过来 会出现糟糕的设计 实际上任何都是 最好定义好状态机 ，要用上一切最好的流程工具了 画图定义伪代码
