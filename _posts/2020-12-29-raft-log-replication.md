---
layout: post
title: Raft 一致性共识算法（三）—— 干活了兄弟们
categories: raft 
description: 
keywords: 一致性共识 分布式 raft 日志复制
---

**Raft 日志复制**

在上一章中，我们已经选择出了团队负责人。

并且当负责人离职时，会自动有队员发起选举，获得过半选票即上任成为负责人。

这样保证了无论何时（可接受的离职人数下），都会存在负责人，团队可持续对外交付。

那么团队如何对外交付，处理外部请求呢？

> 我们可以把一次请求，一次指令，一次交互，都看作是一条日志。

# 开始干活了

目前你手下的团队有三个成员：小智，小障，大脸猫。

经过选举，小智是当下的负责人，你有项目任务需要执行时，直接找小智一个人对接就可以了。

**那小智在接收到项目任务时，都会怎么处理呢？**

![](/images/posts/Raft/日志复制1.png)

你把要做的事情交代给小智，小智明确之后，负责同步到团队内其他队员：小障和大脸猫，他们明确项目确定没问题可以搞，将信息反馈给小智。

小智收到反馈后发现团队三个人，超过半数的成员都可以做，给你答复：No Problem！

小智给你答复之后，表示这件事完整的**上传下达**了，然后通知队员，开始干活了！（服务器视角就是，把刚刚的日志，提交到状态机）。

>什么是状态机？待会见下文

**那会不会存在意外情况呢？**

# 使命必达

![](/images/posts/Raft/日志复制2.png)

"大脸猫思考迟钝。"

你照旧把需要做的事情交代给小智，小智明确之后开始向小障和大脸猫同步，小障很快就答复了没问题可以搞，但是大脸猫反应迟钝了。

这时候小智需要等待大脸猫的反馈吗？

答案是不需要，因为当前小障和小智自己可以确保搞成这个项目。满足过半原则，所以小智立刻就给你答复：No Problem！

但小智答复完之后，并没有就此停止，**使命必达**小智会不断把这件事跟大脸猫念叨，直到大脸猫理解，进度跟大家对齐了为止。

![](/images/posts/Raft/日志复制3.png)


**那有没有极端情况呢？**

# 极端情况
## 兄弟们缺失

有，考虑当前除了小智之外，其他队员都不干了。

这时候项目还能继续进行下去吗？

![](/images/posts/Raft/日志复制4.png)

你照旧把需要做的事情交代给小智，小智明确之后开始向小障和大脸猫同步，但是小障和大脸猫都离职了，没有回复，小智**"等了一会儿"**发现没有收集到过半数的认可，
仅凭自己一人不能满足要求。

于是给你答复：搞不了。

## 领导者缺失

**如果小智离职了会发生什么呢？**

我们要分开看这个问题，小智在不同时期离职的情况，有微妙的不一样。

**小智离职发生在，你交代任务之前**

![](/images/posts/Raft/日志复制5.png)

小智作为领导者，领导者消失，由于此时没有领导者宣布自己领导者地位，

其他成员发现此时团队内没有领导者，则会发起选举（参见[Raft 一致性共识算法（二）—— 领导者选举](https://praline-byte.github.io/page/2020/12/20/raft-leader-election/)）。

![](/images/posts/Raft/投票4.png)

通过选举流程之后，这时候小障成为领导者。

因为你刚刚交代事情的时候，发现团队没有领导者了，你会等待一段时间直到团队出现新的领导者。

>这个时间会非常短，选举过程耗时和团队规模相关，但往往很短。但是这个时间不等也是没有办法的不是吗。

这时候小障上任领导者，你现在开始向小障交代事情。

**小智离职发生在，你交代任务之后**

如果小智发生在你交代任务之后呢？

这时候会有两种情况，第一种是你一交代完之后，小智立马原地离职，其他队员都不知道你交代的事情。

第二种是你交代完之后，小智同步了你的信息，但是反馈过程中离职。

无论哪种情况，对你来说，都是没有收到可以执行的答复。而对于其他成员来讲，均没有收到**开始干活了！**的信号，所以什么也不会发生。


# 原理实现

## 日志的组织形式

日志按下图的方式进行组织，每台服务器上的每一条日志都储存了，这条日志是**谁任期**的什么**指令**。

![](/images/posts/Raft/日志的组织形式.png)

Raft 将使用任期(term序号)和指令序号来检测数据一致性，保证每台服务器上的日志是一致的。

## 处理请求
一旦一个 leader 被选举出来，它就可以开始为客户端提供服务，处理客户端请求。
```
for r.getState() == Leader {
    select {
    case newLog := <-r.applyCh
    ···
}
```

每一个客户端请求都包含着一个待状态机执行的命令，leader 会将这个命令作为新的一条日志追加到自己的日志中。

>状态机：可以先把状态机理解为稳定可靠的数据库。
>
>服务收到请求后先以日志的形式存放，即使存放在磁盘上也可以利用顺序写的特点，很快的完成写入。
>
>一旦 raft 确认日志可以被安全提交，日志的数据就会写入到"状态机"上。

leader 接收客户端请求后，并不会立即更新状态机，而是先记录日志。

```
func (r *Raft) runLeader()
    func (r *Raft) leaderLoop() 
        case newLog := <-r.applyCh
            func (r *Raft) dispatchLogs(applyLogs []*logFuture)
            

// dispatchLog is called on the leader to push a log to disk, mark it
// as inflight and begin replication of it.
func (r *Raft) dispatchLogs(applyLogs []*logFuture) {
	···
	for idx, applyLog := range applyLogs {
        ···
		logs[idx] = &applyLog.log
        ···
	}
	// Write the log entry locally
	if err := r.logs.StoreLogs(logs); err != nil {
    ···
}
```

## 日志同步

然后并行向其他 server 发出 AppendEntries RPC 来复制日志。

```
func (r *Raft) dispatchLogs(applyLogs []*logFuture) {
	···
	// Notify the replicators of the new log
	for _, f := range r.leaderState.replState {
		asyncNotifyCh(f.triggerCh)
	}
}

func (r *Raft) replicate(s *followerReplication) {
    ···
    case <-s.triggerCh:
        lastLogIdx, _ := r.getLastLog()
        shouldStop = r.replicateTo(s, lastLogIdx) 
    ···
}
```

**当日志被安全的复制之后（过半数的 follower 复制成功）**
```
func (r *Raft) replicateTo(s *followerReplication, lastIndex uint64) (shouldStop bool)
    func updateLastAppended(s *followerReplication, req *AppendEntriesRequest)
        func (c *commitment) match(server ServerID, matchIndex uint64)
            func (c *commitment) recalculate()

// Internal helper to calculate new commitIndex from matchIndexes.
// Must be called with lock held.
func (c *commitment) recalculate() {
    ···
	quorumMatchIndex := matched[(len(matched)-1)/2]

	if quorumMatchIndex > c.commitIndex && quorumMatchIndex >= c.startIndex {
		c.commitIndex = quorumMatchIndex
		asyncNotifyCh(c.commitCh)
	}
}
```

## 日志提交

leader 就可以将日志 apply 到自己的状态机，并将执行结果返回给客户端。
```
for r.getState() == Leader {
    ···
    case <-r.leaderState.commitCh:
        r.processLogs(lastIdxInGroup, groupFutures)
    ···
}

func (r *Raft) processLogs(index uint64, futures map[uint64]*logFuture) {
    ···
    var preparedLog *commitTuple
    ···
    case preparedLog != nil:
        applyBatch := func(batch []*commitTuple) {
            select {
            case r.fsmMutateCh <- batch:
        ···
    case futureOk:
        // Invoke the future if given.
        future.respond(nil)

func (r *Raft) runFSM() {
    ···
    select {
    case ptr := <-r.fsmMutateCh:
        switch req := ptr.(type) {
        case []*commitTuple:
            commitBatch(req)
    ···
}

```

如果 follower 宕机或运行很慢，甚至丢包，leader 会无限的重试 RPC(即使已经将结果报告给了客户端)，直到所有的 follower 最终都存储了相同的日志。
```
func (r *Raft) replicateTo(s *followerReplication, lastIndex uint64) (shouldStop bool) {
    ···
    // Make the RPC call
	start = time.Now()
	if err := r.trans.AppendEntries(s.peer.ID, s.peer.Address, &req, &resp); err != nil {
		r.logger.Error("failed to appendEntries to", "peer", s.peer, "error", err)
		s.failures++
		return
	}
    ···
}

<---
func (r *Raft) replicate(s *followerReplication) {
    ···
	for !shouldStop {
		select {
        ···
		case <-randomTimeout(r.conf.CommitTimeout):
			lastLogIdx, _ := r.getLastLog()
			shouldStop = r.replicateTo(s, lastLogIdx)
        ···
		}
}
```

leader 会决定何时 apply 一条日志是安全的，这被称为 committed。Raft 确保 committed 日志是持久化的并最终被所有的状态机执行。

一旦 leader 把日志复制到了大多数节点上，就会 committed，这也意味着在此之前的所有日志都被 commit 了，包括之前其他 leader 创建的日志。

>见 **当日志被安全的复制之后（过半数的 follower 复制成功）**

leader 会追踪已经 committed 的最高的日志索引，并将这个索引放入之后的 AppendEntries RPC，以便于其他节点可以最终发现，一旦一个 follower 意识到一条日志被 committed 了，
它就会将其 apply 到自己的状态机。

```
func (r *Raft) appendEntries(rpc RPC, a *AppendEntriesRequest) {
    ···
    // Update the commit index
    if a.LeaderCommitIndex > 0 && a.LeaderCommitIndex > r.getCommitIndex() {

        idx := min(a.LeaderCommitIndex, r.getLastIndex())
        r.setCommitIndex(idx)
        
        r.processLogs(idx, nil)
    }
}

// processLogs is used to apply all the committed entries that haven't been
// applied up to the given index limit.
// This can be called from both leaders and followers.
// Followers call this from AppendEntries, for n entries at a time, and always
// pass futures=nil.
// Leaders call this when entries are committed. They pass the futures from any
// inflight logs.
func (r *Raft) processLogs(index uint64, futures map[uint64]*logFuture) {
    ···
}
```

## 日志一致性保证

Raft 日志机制可以保证不同 server 上的日志具有很高的一致性。

这不仅仅简化了系统和增强了可预测性，而且这也是确保安全的一个重要组件，Raft 通过下列特性构建日志一致性：
* 如果不同日志中的两条记录有相同的 index 和 term，他们会存有相同的命令。
* 如果不同日志中的两条记录有相同的 index 和 term，那么他们之前的记录也是完全相同的。

leader 在给定的 index 和 term 处至多只会创建一条记录，并且新的记录不会改变之前的位置，因此可以确保第一条。

第二条是通过 AppendEntries 的一致性检查实现的。当发送 AppendEntries RPC 的时候，leader 会将之前最新日志的 term 和 index 包含在其中，

```
func (r *Raft) setupAppendEntries(s *followerReplication, req *AppendEntriesRequest, nextIndex, lastIndex uint64) error {
	req.RPCHeader = r.getRPCHeader()
	req.Term = s.currentTerm
	req.Leader = r.trans.EncodePeer(r.localID, r.localAddr)
	req.LeaderCommitIndex = r.getCommitIndex()
	if err := r.setPreviousLog(req, nextIndex); err != nil {
		return err
	}
	if err := r.setNewLogs(req, nextIndex, lastIndex); err != nil {
		return err
	}
	return nil
}
```

如果 follower 没有在自己的日志中找到相同的 index 和 term，它就会拒绝这条请求。

累加效果，因此只要 AppendEntries RPC 返回成功，leader 就会知道 follower 的日志直到最新的这条都和自己一样。

## 处理日志不一致

在正常操作中，leader 和 follower 的日志是完全一致的，因此 AppendEntries 的一致性检查不会失败。然而，leader 失效可能会造成不一致的情况，这种不一致可能是多次形成的。

![](/images/posts/Raft/日志提交2.png)

> f 在 term=2 时是领导者，接收几条日志还未提交就崩溃了，但是很快又通过选举上任，此时 term=3，又接受几条日志奔溃了，日志没有提交

上图中展示了一些 follower 和 leader 日志不一样的情况。follower 可能会缺少一些日志，也可能会比当前 leader 额外多出一些日志，或者两者兼有，而且可能涉及到几个 term。

在 Raft 中，leader 通过强制 follower 复制自己的日志来解决这种不一致的情况，意味着 follower 和 leader 产生冲突的部分日志会以 leader 为准进行重写。

为了使得 follower 的日志和 leader 的日志一致，leader 必须找到自己和 follower 最后一致的日志索引，然后删掉在那之后 follower 的日志，

并将 leader 在那之后的日志全部发送给 follower。所有的这些操作都发生在 AppendEntries RPC 的一致性检查中。

```
case rpc := <-r.rpcCh:
    --> func (r *Raft) processRPC(rpc RPC)
        --> func (r *Raft) appendEntries(rpc RPC, a *AppendEntriesRequest)

func (r *Raft) appendEntries(rpc RPC, a *AppendEntriesRequest) {
    ···
        // Delete any conflicting entries, skip any duplicates
		lastLogIdx, _ := r.getLastLog()

		for i, entry := range a.Entries {
			if entry.Index > lastLogIdx {
				newEntries = a.Entries[i:]
				break
			}
			
			if err := r.logs.GetLog(entry.Index, &storeEntry); err != nil {
				···
				return
			}

			if entry.Term != storeEntry.Term {
				if err := r.logs.DeleteRange(entry.Index, lastLogIdx); err != nil {
				···
				break
			}
		}

		if n := len(newEntries); n > 0 {
			// Append the new entries
			if err := r.logs.StoreLogs(newEntries); err != nil {
				return
			}
            ···
			// Update the lastLog
			last := newEntries[n-1]
			r.setLastLog(last.Index, last.Term)
		}
···
}      
```


leader 持有针对每一个 follower 的 nextIndex 索引，代表下一条要发送给对应 follower 的日志索引。

当 leader 刚上任时，它会初始化所有 follower 的 nextIndex 值为自己最后一条日志的下一个索引。
```
func (r *Raft) startStopReplication() {
	// Start replication goroutines that need starting
	for _, server := range r.configurations.latest.Servers {
        ···
			s = &followerReplication{
				···
				currentTerm:         r.getCurrentTerm(),
				nextIndex:           lastIdx + 1,
				lastContact:         time.Now(),
				···
			}
}

```

如果 follower 的日志和 leader 的不一致，下一次 AppendEntries 的一致性检查就会失败。
```
func (r *Raft) appendEntries(rpc RPC, a *AppendEntriesRequest) {
    // Setup a response
    resp := &AppendEntriesResponse{
        ···
        Success:        false,
    }
    ···
    var prevLog Log
    if err := r.logs.GetLog(a.PrevLogEntry, &prevLog); err != nil {
        ···
        return
    }
    ···
    if a.PrevLogTerm != prevLogTerm {
        ···
        return
    }
}
```

在遭到拒绝后， leader 就会降低该 follower 的 nextIndex 并进行重试。

```
func (r *Raft) replicateTo(s *followerReplication, lastIndex uint64) (shouldStop bool) {
    ···
	if resp.Success {
        ···
    } else {
        ···
        atomic.StoreUint64(&s.nextIndex, max(min(s.nextIndex-1, resp.LastLog+1), 1))
    }
    ···
}
```

最终 nextIndex 会到达 leader 和 follower 一致的位置。

这时 AppendEntries RPC 会执行成功，并覆盖 follower 在这之后原有的日志，之后 follower 的日志会保持和 leader 一致，直到这个term结束。

如果 leader 发现自己的日志和 follower 是完全匹配的，leader 就可以发送不带有日志数据的 AppendEntries(心跳) 来节省带宽。

一旦 matchIndex 追上了 nextIndex，leader 应当开始发送日志数据。

当然上面的寻找 index 的过程可以优化减少 AppendEntries RPC。例如，当拒绝 AppendEntries 请求时，follower 可以返回发生冲突的 entry 所在的 term 以及该 term 的第一个 index，
通过这些信息，leader 可以直接跳过这个 term 中的全部 index。另外 leader 可以使用二分搜索来查找和 follower 第一个不同的日志。

实际上，这些优化不是很必要的，因为故障不很频繁，而且不太可能有太多不一致的日志。Leader 从不需要重写或者删除自己的日志。




