---
layout: post
title: Raft 一致性共识算法（四）—— 领导者变更
categories: raft 
description: 
keywords: raft 领导者变更
---

领导者变更

## 领导者变更

我们在前面介绍道，不出意外的情况下，整个团队运作良好。

当队员离职时，只要交代任务仍满足**"过半原则"**，就不影响任务的正常开展。

当领导者离职时，团队会短暂停止提供服务，直到通过自发选举的方式推举出新的领导者。

那如果我们发现，领导者不合适，不能继续胜任的时候，可以让他主动退位吗？

![](/images/posts/Raft/领导者变更.png)

假如当前你的团队还是小智，小障和大脸猫三个人，小智是领导者。

因为一段时间强度非常大的项目迭代，小智资源枯竭在崩溃边缘，你察觉到了，你发现小智马上就要被压倒了。怕他离职你决定让他缓口气，换个人
承担领导者的职责。

这时候你可以主动选择，领导者变更。

![](/images/posts/Raft/领导者变更1.png)

## 原理与实现

这一节描述了 Raft 的一个可选扩展，它允许一个节点转移自己的 leader 权给其他节点，在下面两种情况下会很有用：

1、 有时候 leader 需要主动下线。比如，它可能需要重启或者移出集群。当它下线的时候，集群在 electionTimeout 的时间内处于闲置状态，直到有一台机器成为leader，
这种不可用的情况可以通过主动转移 leader 权来避免。
2、 在一些情况下，其他的节点可能更适合于担当 leader。比如 Raft 的 leader 节点承担了全部的客户端负载，当负载很高时会影响系统的性能，
这时候 leader 可以周期性的检查集群中的 follower 是否有更适合成为 leader 的，然后将领导权转移给他。

为了转移领导权，当前 leader 会把自己的日志发送给目标节点，然后目标节点提前触发一轮选举。当前 leader 确保了目标节点拥有全部 committed 日志。下面是详细步骤：

1. 当前 leader 停止接收客户端请求

```
for r.getState() == Leader {
    select {
    case newLog := <-r.applyCh:
        if r.getLeadershipTransferInProgress() {
            r.logger.Debug(ErrLeadershipTransferInProgress.Error())
            newLog.respond(ErrLeadershipTransferInProgress)
            continue
        }
    ···
}
```
2. 当前 leader 通过复制日志将目标节点的日志更新为和自己完全一样

```
for r.getState() == Leader {
    select {
    case future := <-r.leadershipTransferCh:
    ···
        --> func (r *Raft) leadershipTransfer(...)
    ···
}

func (r *Raft) leadershipTransfer(id ServerID, address ServerAddress, repl *followerReplication, stopCh chan struct{}, doneCh chan error) {
    ···
	// Step 1: set this field which stops this leader from responding to any client requests.
	r.setLeadershipTransferInProgress(true)
	defer func() { r.setLeadershipTransferInProgress(false) }()

	for atomic.LoadUint64(&repl.nextIndex) <= r.getLastIndex() {
        ···
    }
    
    // Step 3: send TimeoutNow message to target server.
    err := r.trans.TimeoutNow(id, address, &TimeoutNowRequest{RPCHeader: r.getRPCHeader()}, &TimeoutNowResponse{})
    ···
}
```

3.当前 leader 发送一个 TimeoutNow 请求给目标节点

```
func (r *Raft) leadershipTransfer(id ServerID, address ServerAddress, repl *followerReplication, stopCh chan struct{}, doneCh chan error) {
    ···
    // Step 3: send TimeoutNow message to target server.
    err := r.trans.TimeoutNow(id, address, &TimeoutNowRequest{RPCHeader: r.getRPCHeader()}, &TimeoutNowResponse{})
    ···
}
```

这个请求会使得目标节点立刻触发超时并开启新一轮选举。它有极大可能在其他节点超时之前赢得选举，它的下一条消息将会包含新的 term 编号，导致 leader 自动下线。leader 转移完成。
```
func (r *Raft) processRPC(rpc RPC) {

	switch cmd := rpc.Command.(type) {
    ···
	case *TimeoutNowRequest:
		r.timeoutNow(rpc, cmd)
    ···
}

func (r *Raft) timeoutNow(rpc RPC, req *TimeoutNowRequest) {
	r.setLeader("")
	r.setState(Candidate)
	r.candidateFromLeadershipTransfer = true
	rpc.Respond(&TimeoutNowResponse{}, nil)
}

```

如果转移失败，之前的 leader 必须中断转移过程，并重新开始接收客户端的请求。



