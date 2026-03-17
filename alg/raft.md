# Raft 

## Q: Raft算法是什么？

A: Raft算法是一种共识算法，用于处理数据一致性问题。Raft采用一主多从的架构。

Raft将一致性问题分为三个问题：

1. 领导者选举
   - 集群中有且只有一个leader
   - 每个节点都有一个角色 leader, follower， candidate
   - 每个节点存在一个定时器，节点根据定时器执行对应的操作
     - 当follower的定时器超时，会转化为candidate
     - candidate会向其他的follower发送request vote
     - 超过半数的vote的candidate会变为leader
     - leader定期发送heatbeat更新follower的定时器
2. 日志复制
   - leader接收客户端请求后会追加为entry
   - leader会通过heartbeat将日志发送给follower进行追加
   - 当大部分都follower收到ACK后，leader标记为commit
   - leader通知follower提交
