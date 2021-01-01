---
layout: post
title: Raft 一致性共识算法（二）—— 领导者选举
categories: raft 
description: 
keywords: 一致性共识 分布式 raft 领导者选举
---

我们希望团队有一个负责人，可以放心的把事情交给他，他来保证这件事落地成功与否。 

所以我们需要一套选举机制，选出大家公认的负责人，并且同一时刻有且仅会有一个负责人。

# 选出负责人

我们希望选举过程公平公正公开，符合标准的普通人都可以成为候选人，而只有获得过半数投票的候选人，才能上任成为负责人。

并且在负责人负责的任期内，只要能正常工作，没有生病，离职等情况发生，就会一直是负责人，不会发生选举过程。

>任期：每个负责人负责的时间周期内称为一个任期

**团队中的三种角色：普通人，候选人，负责人**

* 普通人与选举过程无关，普通人发现当前没有负责人时，会主动提议参与选举，成为候选人。

* 候选人只有发生选举过程时才存在，候选人受到过半数其他成员的投票，就会成为负责人。如果已经选举出负责人，则会恢复为普通人。

* 负责人在选举过程中诞生，负责人上任后会告诉其他成员自己是负责人，并且自此由自己来接收外部请求，对外交付。

目前你手下的团队有三个成员：小智，小障，大脸猫。

**谁会通过选举成为负责人呢？**

一开始他们三位都是普通人，小智率先发现当前没有负责人，于是立马成为候选人参与选举，投自己一票，且向小障和大脸猫发起上任请求。

小障和大脸猫收到小智的请求后，把票投给了他。

小智获得三票，超过半数，于是顺理成章上任为负责人，开始负责对外交付。

![](/images/posts/Raft/投票1.png)

**如果一开始，小智和小障的动作都比较快，同时发现当前没有负责人了，选举过程会发生什么呢？**

小智和小障都发现没有负责人，这时候他们之间还没有互相沟通过，所以会同时成为候选人参与选举，投自己一票，并向其他两位发起上任请求。

![](/images/posts/Raft/投票2.png)

小智和小障现在发现都想成为负责人，且自己的一票都已经投过自己了，所以大脸猫的票至关重要。

任意一方只要票数过半，即可胜任。因为在时间上总会有先来后到，所以速度快的人优先获取胜利。

这时候大脸猫先收到了小智的请求，投了他一票，之后又收到了小障的请求。因为在当前任期已经投过票了，所以小障没有收到投票

* 小智 2 票（自己一票 + 大脸猫一票）
* 小障 1 票（自己一票）

![](/images/posts/Raft/投票3.png)

所以小智上任。

>小智上任后，会不间断发送信息告诉其他人，现在我是负责人，不要企图篡位 
>
>其他人知道现在有负责人了，就不会发起投票选举。

**如果小智上任后，离职了怎么办？**

小智离职了，就不会发送"权威保证"的信息给小障和大脸猫。

小障发现现在没有负责人，于是主动变身候选人，发起投票，提议自己为负责人，因为此时小智离职，所以不会收到小智的响应，只能收到大脸猫的响应。

获得两票，成功成为负责人。

![](/images/posts/Raft/投票4.png)

现在小障开始发送"权威保证"的信息，确保自己是负责人。

**如果大脸猫和小障同时发起投票怎么办？**

就像上文中小智和小障都发现没有负责人的流程一样，每个人都会先投自己一票，然后征求其他人的票，因为此时没有其他人，所以在这次选举过程中，
每个人都只有一票，不满足过半原则，没有负责人胜出。

![](/images/posts/Raft/投票5.png)

这种情况怎么办呢？

我们可以开启下一轮选举，重新发起投票。但是给每个候选人都留一个"冷静思考"的时间，这个时间随机不固定，总有的人很坚定，这个时间非常短，又会发起投票，
推举自己为负责人，有的人可能会犹豫不决，这时候收到了其他人的上任请求，就答应推举别人了。

Raft 中应对这种情况的机制是**随机超时**机制，会在之后中说明。

这样就能保障，即使一个人离职，整套机制仍然可以正常运转。

如果此时小智找不到新工作，又回来继续上班了，怎么办？如果团队又招聘了新同事怎么办呢？

这些问题都要考虑，不过这不属于选举过程，先不着急暂且一放待下回分解。

我们先总结一下，选举流程的核心点都有什么呢？

1. 权威保证：领导者定期发送权威信息，确定自己领导地位
2. 自驱原则：普通人每隔一段时间，就确认下当前任期有没有人负责，没有的话就主动提议下期选举开始，并发起投票
3. 过半原则：只有收到过半数的选票，才能上任为领导者
4. 超时机制：如果有多个候选人，选票被瓜分，则开启下一轮选举


# Raft 中的实现

Raft 中对应普通人，候选人和负责人三种角色的分别是 follower，candidate 和 leader。

当服务启动的时候，处于 follower 状态
```
func NewRaft(···) {
	// Initialize as a follower.
	r.setState(Follower)
    ···
}
```

follower 如果收到来自 leader 或者 candidate 的有效 RPC 请求时就会保持 follower 的状态

```
func (r *Raft) runFollower() {
    ···
    case rpc := <-r.rpcCh:
        r.processRPC(rpc)
    ···
}
```

leader 也会发送周期性的心跳(不含日志的 AppendEntries RPC)，给所有的 follower 来确保自己不会下任
```
func (r *Raft) runLeader()
    func (r *Raft) startStopReplication()
        func (r *Raft) replicate(s *followerReplication)
            func (r *Raft) heartbeat(s *followerReplication, stopCh chan struct{})
---
// heartbeat is used to periodically invoke AppendEntries on a peer to ensure they don't time out.
func (r *Raft) heartbeat(s *followerReplication, stopCh chan struct{}) {
    ···
	req := AppendEntriesRequest{
		RPCHeader: r.getRPCHeader(),
		Term:      s.currentTerm,
		Leader:    r.trans.EncodePeer(r.localID, r.localAddr),
	}
    ···
    r.trans.AppendEntries(s.peer.ID, s.peer.Address, &req, &resp)
    ···
}
```

如果一个 follower 一段时间(timeout)没有收到消息，它就会假定 leader 失效并推举自己为 Candidate，开始新一轮的选举

```
func (r *Raft) runFollower() {
    ···
    case <-heartbeatTimer:
        // Restart the heartbeat timer
        heartbeatTimer = randomTimeout(r.conf.HeartbeatTimeout)
    
        // Check if we have had a successful contact
        lastContact := r.LastContact()
        if time.Now().Sub(lastContact) < r.conf.HeartbeatTimeout {
            continue
        }
    
        // Heartbeat failed! Transition to the candidate state
        lastLeader := r.Leader()
        r.setLeader("")
    
        ···
            r.setState(Candidate)
            return
        }
    ···
}

```

为了开始新一轮选举，candidate 它会先给自己投一票，并提高自己当前的 term。然后并行向集群中的其他 server 发出 RequestVote RPC

```
func (r *Raft) runCandidate() {
    ···
	// Start vote for us, and set a timeout
	voteCh := r.electSelf()
    ···
}

func (r *Raft) electSelf() <-chan *voteResult {
    ···
	// Increment the term
	r.setCurrentTerm(r.getCurrentTerm() + 1)
    ···
    // For each peer, request a vote
    for _, server := range r.configurations.latest.Servers {
        if server.Suffrage == Voter {
            askPeer(··)
        }
    }
}

```

candidate 会保持这个状态，直到下面三种事情之一发生

**1. 赢得选举**

当 candidate 收到了集群中相同 term 的多数节点的赞成票时就会选举成功，收到多数节点的选票可以确保在一个 term 内至多只能有一个 leader 选出。
```
func (r *Raft) runCandidate() {
    ···
    votesNeeded := r.quorumSize()
    ···
		case vote := <-voteCh:
            ···
			// Check if the vote is granted
			if vote.Granted {
				grantedVotes++
			}
			// Check if we've become the leader
			if grantedVotes >= votesNeeded {
				r.setState(Leader)
				r.setLeader(r.localAddr)
				return
			}
    ···
}
```

每一个 server 在给定的 term 内至多只能投票给一个 candidate，先到先得。
```
func (r *Raft) requestVote(rpc RPC, req *RequestVoteRequest) {
    ···
	// Check if we have voted yet
	lastVoteTerm, err := r.stable.GetUint64(keyLastVoteTerm)
	lastVoteCandBytes, err := r.stable.Get(keyLastVoteCand)
    ···
	// Check if we've voted in this election before
	if lastVoteTerm == req.Term && lastVoteCandBytes != nil {
		return
	}
    ···
}
```
一旦一个 candidate 赢得选举，它就会成为 leader，它之后会发送心跳消息来建立自己的权威，并阻止新的选举。

**2. 另一个 server 被确定为 leader**

在等待投票的过程中，candidate 可能收到来自其他 server 的 AppendEntries RPC，声明它才是leader。

如果 RPC 中的 term 比自己当前的小，将会拒绝这个请求并保持 candidate 状态。

如果 RPC 中的 term 大于等于 candidate 的 current term，candidate 就会认为这个 leader 是合法的并转为 follower状态。

```
func (r *Raft) runCandidate() 
    case rpc := <-r.rpcCh:
        func (r *Raft) processRPC(rpc RPC) 
            func (r *Raft) appendEntries(rpc RPC, a *AppendEntriesRequest)
---
 
func (r *Raft) appendEntries(rpc RPC, a *AppendEntriesRequest) {
    ···
	// Ignore an older term
	if a.Term < r.getCurrentTerm() {
		return
	}

	// Increase the term if we see a newer one, also transition to follower
	// if we ever get an appendEntries call
	if a.Term > r.getCurrentTerm() || r.getState() != Follower {
		// Ensure transition to follower
		r.setState(Follower)
		r.setCurrentTerm(a.Term)
		resp.Term = a.Term
	}
    ···
}

```
**3. 没有获胜者产生，等待选举超时**

candidate 没有选举成功或者失败，如果许多 follower 同时变成 candidate，选票就会被瓜分，形不成多数派。

这种情况发生时，candidate 将会超时并触发新一轮的选举，提高 term 并发送新的 RequestVote RPC。
```
func (r *Raft) runCandidate() {
    ···
    case <-electionTimer:
        // Election failed! Restart the election. We simply return,
        // which will kick us back into runCandidate
        r.logger.Warn("Election timeout reached, restarting election")
        return
    ···
}
```

然而如果不采取其他措施的话，选票将可能会被再次瓜分。

Raft 使用随机选举超时来确保选票被瓜分的情况很少出现而且出现了也可以被很快解决。election timeout 的值会在一个固定区间内随机的选取(比如150-300ms)。
```
func (r *Raft) runCandidate() {
    ···
	electionTimer := randomTimeout(r.conf.ElectionTimeout)
    ···
}
```

这使得在大部分情况下仅有一个 server 会超时，它将会在其他节点超时前赢得选举并发送心跳。

candidate 在发起选举前也会重置自己的随机 election timeout，也可以帮助减少新的选举轮次内选票瓜分的情况。

这样就保障了，有且只会有一个领导者，当领导者不可用时，会有新的领导者上任。

有了领导者之后，我们接下来看，如何处理请求消息。
