# This is my self practice on labs for MIT 6.824 Distributed Systems (2018) http://nil.csail.mit.edu/6.824/2018/schedule.html

## Lab1 - MapReduce

Just follow the definitions of what are Map and Reduce, below are some caveats/intakes:

1. When trigger a Goroutine, consider passing outer variable into Goroutine's arguments. e.g. pass index i inside a for loop, otherwise i will be changed
2. In common_reduce need to join all the lists of the same key
3. In schedule, if a worker is finished, can put into registerChan for reuse, but need to spawn a separate Goroutine for that without holding the WaitGroup

## Lab2 - Raft

Need to follow the Figure 2 of the paper closely, below are some caveats/intakes:

1. Entering any concurrent running code, check state first, it may no longer be leader/candidate/follower
2. It is important that log is starting with index 1 and commitIndex initialized as 0 (means nothing committed yet). And later down the code, `len(log)-1` can be passed around as log's last index without worrying it ever being -1 and commitIndex can be `len(log)-1` as well. Also check the code of passing arguments to sendHeartbeat
3. Received the RequestVote request with higher term, need to step down to follower first then proceed with the vote
4. Heartbeat and AppendEntries are not separate requests, they are the same, AppendEntries are actually heartbeats sent out. A newly updated commitIndex of leader will be sent along with periodic heartbeats so that followers can bump their commitIndex; otherwise in an alternative design that leader only send out AppendEntries request upon client's operation, followers can only bump their commitIndex upon client's next operation which may never happen or takes a very long time to happen
5. To pass TestBackup2B test case, need to implement the smart nextIndex decrease by using a hinted index from follower. This will reduce a lot of RPC calls just to decrease nextIndex 1 by 1
6. The electionTimeout needs to be randomized upon every new (re-)election (mentioned in paper but not in Figure 2), so that each server's time outs are not fixed. This can help reduce a rare case that one server always timed up first and stay being candidate, which may make the election never or take a very long time to end. Such edge case is found 2 out 100 times of TestRejoin2B test case

## Lab3 - Raft Key/Value Service

Lab3 should be fairly straight-forward, just talk to leader for requests, but during implementations, it discovers some bugs in previous Raft implementations and also got some insights for kvraft in terms of availability.

### Raft Bugs Found

1. When selecting a random leader timeout, previously, I make a new seed not realizing the `rand` is shared. This will lead to a case that all servers will get almost close timeouts (the first random number generated after re-seeding). The fix is to remove re-seeding at all, just use default seeding, in this way, each time server generates a timeout, it is the next generated random number

2. In lab2's raft implementation, I did not introduce the applyIndex at all, whenver something can be commited, I send to applyCh right away to apply. But in kvraft, I found sometimes, there can be mulitple goroutines trying to publish to applyCh concurrently, leading to out of order apply. I can make the change to publish to applyCh in a blocking way, but it has the potential of blocking the whole raft from progress if there are not kvraft consuming the applyCh or slow to consuming; and in some cases, deadlocks happen between kvraft and raft. The raft paper introduces an applyIndex that is apply changes in a seperate goroutine, this can gurantee the applyCh will be published in sequence, if no one is consuming the apply, that is fine, that goroutine just block there without holding the progress of commit (commit is actually just updating the commitIndex)

### KVRaft Implemenation Insights

1. When client's request comes in, it cannot block there forever, if server finds it is no longer the leader, it will send back to client to retry other servers. This means the KVRaft needs to periodically check if it is leader or not.

2. When apply state machine changes, server must reply back to client even if it is found not to be the leader. This is safe because in Raft, once a change applied, it cannot be changed. If we fail to reply back to the client, if the server quickly becomes leader after the state machine application, that client will wait for the reply forever until this server loses leader. Actually it is the timing between request goroutine, apply goroutine and leader check goroutine, not being leader in apply goroutine does not mean not-leader in request goroutine, it can be declared leader in next leader check goroutine before request goroutine polls again.

3. To solve apply-at-most-once, the cache is filtering duplicate requests in state machine application stage instead of request stage. We cannot put cache and filter at request stage, because one request may be applied but fails to respond to the client and the client will retry the same request to other servers where the same request is not cached, leading to duplicate application. Design the cache in state machine application stage will allow duplicate requests go into Raft logs, but just don't apply twice in state machine application. For Get request, the key will be cached so that it will just reply to the client the current value of the key

4. In Raft protocol, the leader cannot commit changes of previous term, but in some of the partition tests, all clients are talking to the leader with old term but the leader is at new term, all clients will not get response back and pending there forever until test case time out after 10 minutes. To solve this, the leader needs to commit a no-op message in new term so that old termed changes can be commited and applied. I cannot make the change in Raft layer, because lab2's test cases won't allow it. So I made the chagne in KVRaft layer, that is whenever it is found to be the leader with a new term, submit an no-op message to Raft. 

### KVRaft Log Compaction Insights

1. lastApplied should not be changed immediately during Snapshot or InstallSnapshot, lastApplied is updated only after the change/snapshot has been applid to KVRaft's statemachine. In my code design, lastApplied is updated only in the change apply goroutine and likewise snapshot's statemachine only push to KVRaft in the same goroutine. In this way, we can ensure that the change is applied in a serialize way

2. Due to the same above reason, lastApplied is kinda lag in the update, and should not reply on lastApplied index to do any business logic checking outside of that apply-change-goroutine. So lastApplied can be updated serially without holding locks

3. Once KVRaft received the snapshot reset request from raft instance, it should reply any pending client requests if those requests have been applied in the snapshot. Otherwise those clients may wait forever if this server keeps being the leader.

4. Always keep statemachine and snapshot's lastIncludedIdx and lastInlucdedTerm sync, and also lastIncludedIdx can never decrease, always check for the new snapshot's lastIncludedIdx and only set statemachine if found current lastIncludedIdx stale. This way, we can ensure we will never apply stale statemachine.

5. In order to execute TestSnapshotSize3B under 20s, when leader receives client requests, it should send out appendEntries requests immediaty instead of waiting for next heartbeat cycle. This can make client's requests replicated, committed,  applied and replied back much faster.

6. The linearizability test uses WGL depth-first search algorithm, which cannot points out which is going wrong, so better log the subhistory and check linearization manually to spot which request(s) are wrong, then search out that request(s) in the server log

## Lab4 - Sharded Key/Value Service

The same code structure of 2015's lab4 (https://github.com/pinkfloyda/6.824-Labs-2015) but instead of leader-less, all reconfiguration operations (bump configNum, receiving incoming shards, sending out shards) are initiated by leader and replicate to followers accordingly. During leader changes, new leader need to pickup any pending outgoing shards and re-send them to other shards' leaders. Like in 2015's lab, the configNum is very important, all servers need to ignore if incoming configNum is lower than current and signal the other side to retry if incoming configNum is bigger than current. Only wire reconfiguration operation into Raft log if configNum match with current and it is in configuration mode.

The challenges tasks are already handled in 2015's lab4 also, whenever receive the incoming shards, those shards will be immediately ready to server clients' requests.

And also because of the way RaftKV manage the client-reply map, all reconfiguration operations' log ClientName needs to be differentiated with each other, otherwise one reconfiguration may steal the reply channel of the other and cause the other waiting indefinitely.
