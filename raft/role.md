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

## Leader

**state.py**

```python
class Leader(BaseRole):
    """Raft Leader
    Upon election: send initial empty AppendEntries RPCs (heartbeat) to each server;
    repeat during idle periods to prevent election timeouts

    — If command received from client: append entry to local log, respond after entry applied to state machine
    - If last log index ≥ next_index for a follower: send AppendEntries RPC with log entries starting at next_index
    — If successful: update next_index and match_index for follower
    — If AppendEntries fails because of log inconsistency: decrement next_index and retry
    — If there exists an N such that N > commit_index, a majority of match_index[i] ≥ N,
    and log[N].term == self term: set commit_index = N
    """

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        self.heartbeat_timer = Timer(config.heartbeat_interval, self.heartbeat)
        self.step_down_timer = Timer(
            config.step_down_missed_heartbeats * config.heartbeat_interval,
            self.state.to_follower
        )

        # Heartbeat 时自加1
        self.request_id = 0
        # 收到 append_entries_response 时，根据 request_id ，判定是否有过半 Follower 回应
        self.response_map = {}

    def start(self):
        self.init_log()

        # LIUHAO: Trigger leader call 'append_entries' automatically
        self.heartbeat()
        self.heartbeat_timer.start()

        self.step_down_timer.start()

    def stop(self):
        self.heartbeat_timer.stop()
        self.step_down_timer.stop()

    # init_log 是在 start() 方法而不是 __init__ 方法中调用，
    # Candidate 升级为 Leader 时，只有 next_index、match_index 会重新初始化，其它数据保持不变
    def init_log(self):
        # LIUHAO
        # - Initiate next_index of each follower to leader's last_log_index+1. Leader will try to broadcast 'append_entries' command to each follower with lastest log data.
        #         If follower reply not 'success', next_index will descrease automatically.
        #         If follower reply 'success', leader will update 'match_index' to 'last_log_index' of follower.
        # - 'self.state.cluster' doesn't include this node refer to register.py:register
        self.log.next_index = {
            follower: self.log.last_log_index + 1 for follower in self.state.cluster
        }

        # LIUHAO
        # - Initiate match_index to 0. match_index will catch up to the 'next_index' of each server after leader broadcasting 'append_entries' commands and receives 'success' response
        self.log.match_index = {
            follower: 0 for follower in self.state.cluster
        }

    async def append_entries(self, destination=None):
        """AppendEntries RPC — replicate log entries / heartbeat
        Args:
            destination — destination id

        Request params:
            term — leader’s term
            leader_id — so follower can redirect clients
            prev_log_index — index of log entry immediately preceding new ones
            prev_log_term — term of prev_log_index entry
            commit_index — leader’s commit_index

            entries[] — log entries to store (empty for heartbeat)
        """

        # Send AppendEntries RPC to destination if specified or broadcast to everyone
        # 支持 send 单点或 broadcast 广播消息
        destination_list = [destination] if destination else self.state.cluster
        for destination in destination_list:
            data = {
                'type': 'append_entries',

                'term': self.storage.term,
                'leader_id': self.id, # LIUHAO: It's just a leader_id. When a Follower receives 'append_entries' message, the Follower will update its Leader property.
                'commit_index': self.log.commit_index,

                'request_id': self.request_id
            }

            next_index = self.log.next_index[destination]
            prev_index = next_index - 1

            if self.log.last_log_index >= next_index:
                # Follower 节点数据未同步时，这里仅仅只同步 1 个 entry
                data['entries'] = [self.log[next_index]]

            else:
                # heartbeat 心跳，不携带数据
                data['entries'] = []

            # Follower 需要检查上一个 Log Entry 的 index、term 是否与 Leader 匹配，确保 Follower 数据的一致性
            data.update({
                'prev_log_index': prev_index,
                'prev_log_term': self.log[prev_index]['term'] if self.log and prev_index else 0
            })

            asyncio.ensure_future(self.state.send(data, destination), loop=self.loop)

    @validate_commit_index
    @validate_term
    def on_receive_append_entries_response(self, data):
        sender_id = self.state.get_sender_id(data['sender'])

        # Count all unqiue responses per particular heartbeat interval
        # and step down via <step_down_timer> if leader doesn't get majority of responses for
        # <step_down_missed_heartbeats> heartbeats

        if data['request_id'] in self.response_map:
            self.response_map[data['request_id']].add(sender_id)

            if self.state.is_majority(len(self.response_map[data['request_id']]) + 1):
                # 回应过半，重置 step_down_timer，删除 response_map 中 request_id 的请求记录
                self.step_down_timer.reset()
                del self.response_map[data['request_id']]

        if not data['success']:
            # LIUHAO: next_index is descreasing. Maybe in order to tolerant the follower to recover log data and catch up Leader
            # next_index[follower] 自减 1，供下一次 append_entries 使用
            self.log.next_index[sender_id] = max(self.log.next_index[sender_id] - 1, 1)

        else:
            # LIUHAO: Trace next_index, match_index for follower inside Leader.
            # append_entries 成功时，
            # next_index[follower_id] 更新为Follower的last_log_index+1,
            # match_index[follower_id]更新为Follower的last_log_index
            self.log.next_index[sender_id] = data['last_log_index'] + 1
            self.log.match_index[sender_id] = data['last_log_index']
            # 更新commit_index
            self.update_commit_index()

        # Send AppendEntries RPC to continue updating fast-forward log (data['success'] == False)
        # or in case there are new entries to sync (data['success'] == data['updated'] == True)
        if self.log.last_log_index >= self.log.next_index[sender_id]:
            # LIUHAO: Continue to send data to the follower
            # 继续向 Follower 同步数据
            asyncio.ensure_future(self.append_entries(destination=sender_id), loop=self.loop)

    def update_commit_index(self):
        commited_on_majority = 0

        # 在当前[commit_index+1, last_log_index+1)范围内遍历，Leader中的 index 已得到 match_index 半数以
        # 上 Follower 回应，并且，log[index]['term'] 与最新 storage.term 相同时，更新 commit_index
        for index in range(self.log.commit_index + 1, self.log.last_log_index + 1):
            commited_count = len([
                1 for follower in self.log.match_index
                if self.log.match_index[follower] >= index
            ])

            # If index is matched on at least half + self for current term — commit
            # That may cause commit fails upon restart with stale logs
            is_current_term = self.log[index]['term'] == self.storage.term
            if self.state.is_majority(commited_count + 1) and is_current_term:
                commited_on_majority = index

            else:
                break

        if commited_on_majority > self.log.commit_index:
            self.log.commit_index = commited_on_majority

    # Write 接口
    async def execute_command(self, command):
        """Write to log & send AppendEntries RPC"""
        self.apply_future = asyncio.Future(loop=self.loop)

        entry = self.log.write(self.storage.term, command)
        asyncio.ensure_future(self.append_entries(), loop=self.loop)
```



## Candidate

## Follower









[上一篇](https://timequark.github.io/raft/state) 

[下一篇](https://timequark.github.io/raft/lastapplied-commitindex )

