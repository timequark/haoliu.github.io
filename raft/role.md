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

  首先，term + 1，voted_for 置为自身的 ID，给自己投1票，然后广播 request_vote 请求。收到过半 vote_granted 为 True 的 response 后，升级为 Leader。如果定时器触发前，没有赢得过半的投票，则直接转变成  Follower 角色。

  下面小节会具体分析 request_vote  请求携带的参数。

- **Follower**

  接收来自 Leader  的 append_entries 请求、来自 Candidate 的 request_vote 请求。这里要注意以下几点：

  （1）Follower.start 时，  init_storage 方法只能第一次加载时才对 term 置 0，但每次都会重置 voted_for。

  （2）on_receive_append_entries 只有在顺利通过 @validate_term、@validate_commit_index 验证时，才会重置 election_timer，否则就有退化为 Candidate 进行重新选举的可能。

  （3）on_receive_request_vote 只有在没有投过票，并且来自 Candidate  的 last_log_term、last_log_index  有效时，才会回应 vote_granted 为 True。

  （4）***on_receive_request_vote 没有重置 election_timer 动作***。因为作为 Follower 自身，并不知道此次选举是否会有新的 Leader 生成，只能通过有效的 on_receive_request_vote  才能感知  Leader  的存在。

## Leader 分析

## Candidate 分析

## Follower 分析

