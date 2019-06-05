# The Raft Consensus Algorithm
_____
#### What is consensus algorithm?
Consensus is a fundamental problem in fault-tolerant distributed systems. (zookeeper, etcd)

![](https://img.alicdn.com/tfs/TB1fmKeiWmWBuNjy1XaXXXCbXXa-879-306.jpg)

1. **safety ensure**: (never returning an incorrect result) under all non-Byzantine conditions, including
network delays, partitions, and packet loss, duplication, and reordering.
2. **fully functional (available)**: as long as any majority of the servers are operational and can communicate with each other and with clients. Thus, a typical cluster of five servers can tolerate the failure of any two servers. Servers are assumed to fail by stopping; they may later recover from state on stable storage and rejoin the cluster.
3. **do not depend on timing**

#### Breif of Paxos
##### Who?
Lamport, working in Microsoft, winner of the 2013 **Turing Award** by theory of consensus.
Is best known for his seminal work in distributed systems and as the initial developer of the document preparation system LaTeX.
pls click me to read his blog ---> [blog](http://www.lamport.org/)
##### What is Paxos
Paxos is not a **single algorithm**, but **a family of protocols**. (lots of papers)
The best one before raft, lots of implementations. (ex. Zookeeper)  
But it is truely complex and obscure, even zookeepr is not implemented exactly according to Paxos.

#### What is raft?
Raft is a consensus algorithm that is designed to be **easy to understand** (compare with Paxos). It's equivalent to Paxos in fault-tolerance and performance. The difference is that it's decomposed into relatively independent subproblems, and it cleanly addresses all major pieces needed for practical systems. We hope Raft will make consensus available to a wider audience, and that this wider audience will be able to develop a variety of higher quality consensus-based systems.
**In another word, Paxos is too complex, also too difficult to implement**
[The implements of raft](https://raft.github.io/#implementations)

## The detail of Raft
* Leader election
* Log replication
* Safety

#### Leader election
##### Three kind of roles in Raft cluster
1. Leader: There is only one leader in a distributed cluster that can issue instructions to other followers
2. Candidate: Leader candidates in the election process. Only appear in election process.
3. Followers: Followers of leader. They do not send any request, only accept heartbeat from leader, except forwarding client requests to leader.

##### Changing of role
1. Follower: When meets the heartbeat timeout, convert to a candidate. 
2. Candidate: Converted by follower, ask for vote from other nodes. If the election succeed(get vote over half nodes), convert to a leader, if fail returns to follower.
3. Leader: When it recive a message with the highter leader term than itself (means there are other leaders, cause by partition), it turns to follower.

![](https://images2018.cnblogs.com/blog/471426/201804/471426-20180421111136094-922352391.png)
![](https://images2018.cnblogs.com/blog/471426/201804/471426-20180421111255916-1287292350.png)

*Question: How to make sure there is no deadlock in election procesing? 
(for example, few candidates ask for vote at same time, but no one get over half response?)*

#### Log replication
##### How logs look like
![](https://raw.githubusercontent.com/maemual/raft-zh_cn/master/images/raft-%E5%9B%BE6.png)
Logs are composed of entries, which are numbered sequentially. 
Each entry contains three important info:
1. The leader term (the number in each box, different collor means different term) 
2. The command for the state machine (two status, commited or not). 
3. Log index.

##### Append-Entries processing
1. client send a request to leader or any follower(follower would forward this to leader)
2. The leader append this command(set x=4) to its log, then send request to followers. Followers will add this cmd to their logs(uncommit), then reply to leader.
3. if leader gets syncs from **over half followers**, then commit this cmd to state machine, reply success to client and notify followers to commit as well. 

##### How to make logs correct in any case?
Raft maintains the following properties:
* If two entries in different logs have the same index and term, then they store the same command.
* If two entries in different logs have the same index and term, then the logs are identical in all preceding entries.

Let's see why:
1. A leader only create one entry with given index in given term.
2. In an Append-Entries RPC, a leader sends not only current index log and term but also previous log and index.  If the follower does not find an entry in its log with the same index and term, then it refuses the new entries. 

##### Inconsistencies handling
During normal operation, the logs of the leader and followers stay consistent, so the AppendEntries consistency check never fails. However, leader crashes can leave the logs inconsistent (the old leader may not have fully replicated all of the entries in its log)
![](https://raw.githubusercontent.com/maemual/raft-zh_cn/master/images/raft-%E5%9B%BE7.png)
When the leader at the top comes to power, it is possible that any of scenarios (a–f) could occur in follower
logs. Each box represents one log entry; the number in the box is its term. A follower may be missing entries (a–b), may have extra uncommitted entries (c–d), or both (e–f). For example, scenario (f) could occur if that server was the leader for term 2, added several entries to its log, then crashed before committing any of them; it restarted quickly, became leader for term 3, and added a few more entries to its log; before any of the entries in either term 2 or term 3 were committed, the server crashed again and remained down for several terms
###### How to handle?
In Raft, the leader handles inconsistencies by forcing the followers’ logs to duplicate its own. This means that conflicting entries in follower logs will be overwritten with entries from the leader’s log.
To bring a follower’s log into consistency with its own, the leader must find the latest log entry where the two logs agree, delete any entries in the follower’s log after that point, and send the follower all of the leader’s entries after that point

#### Safty (fancy part!)
The previous sections described how Raft elects leaders and replicates log entries. However, the mechanisms described so far are not enough to ensure that log commited to state machine cannot lost. 
Let's see a example. (check the picture above, what if b become a leader?)

##### election safety rules
**solution:**
Do not let b become leader! (wtf, it is true)
If we make sure a candidate must store whole commited logs so it can become a leader, the problem is resolved.
How? 
During the election procesing, followers only vote for candidate who has more commited logs than themselves.(Nice!)

Question: In this case, is there any possible that no one can become leader?




