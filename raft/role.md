# Raft源码分析（二） - Role转换

raft 中的 Role 角色共有三类

- **Leader**

  Leader的职能有：

  （1）处理read/write请求

  （2）存储 Log 数据

  （3）向集群其它节点发送 heartbeat 心跳请求，确保集群通信正常

  （4）向Follower发送Log Entry数据，完成 Replication 冗余

  （5）跟踪Follower的数据复制状态

  （6）Log Compation（raftos目前不完备）

  （7）snapshot（raftos目前不完备）

  Leader 会不停的向集群其它节点发送 heartbeat 心跳，且每个心跳请求都有一个 ID （int类型递增），如果收到过半节点的 append_entries_response，则重置 step_down_timer 定时器；如果没有收到过半节点的回应，累计次数超过 step_down_missed_heartbeats 次，step_down_timer 会被触发，Leader 退化为 Follower 。

- **Candidate**

  只用来做 election 选举。

  

- **Follower**