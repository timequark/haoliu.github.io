# Raft源码分析（四） - next_index&match_index



## next_index、match_index 的定义

##### storage.py

```python
class Log:
    ...
    
    def __init__(self, node_id, serializer=None):
        ...
        
        # Leaders
        # next_index仅在Leader中使用，记录Follower中的下一个entry的index位置，
        # Leader启动时将其初始化为 Leader.last_log_index + 1，后续的更新逻辑具体参见 Leader.on_receive_append_entries_response 方法
        """Volatile state on Leaders: for each server, index of the next log entry to send to that server
        (initialized to leader last log index + 1)
            {<follower>:  index, ...}
        """
        self.next_index = None

        """Volatile state on Leaders: for each server, index of highest log entry known to be replicated on server
        (initialized to 0, increases monotonically)
            {<follower>:  index, ...}
        """
        # match_index仅在Leader中使用，记录Follower中已经replication完成的entry index
        # Leader启动时将其初始化为0，后续的更新逻辑具体参见 Leader.on_receive_append_entries_response 方法
        self.match_index = None
```

## Leader.on_receive_append_entries_response

##### state.py

```python
class Leader(BaseRole):
    ...
    
    @validate_commit_index
    @validate_term
    def on_receive_append_entries_response(self, data):
        ...
        
        if not data['success']:
            # LIUHAO: next_index is descreasing. Maybe in order to tolerant the follower to recover log data and catch up Leader
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
```





[上一篇](https://timequark.github.io/raft/lastapplied-commitindex.md) 

[下一篇](https://timequark.github.io/raft/) 



