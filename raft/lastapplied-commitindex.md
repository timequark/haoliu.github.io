# Raft源码分析（三） - commit提交状态跟踪

##### storage.py

```python
class Log:
  """Persistent Raft Log on a disk
  Log entries:
​    {term: <term>, command: <command>}
​    {term: <term>, command: <command>}
​    ...
​    {term: <term>, command: <command>}
  Entry index is a corresponding line number
  """
  
  UPDATE_CACHE_EVERY = 5

  def __init__(self, node_id, serializer=None):
​    self.filename = os.path.join(config.log_path, '{}.log'.format(node_id.replace(':', '_')))
​    os.makedirs(os.path.dirname(self.filename), exist_ok=True)
​    open(self.filename, 'a').close()

​    self.serializer = serializer or config.serializer

​    self.cache = self.read()

​    # All States

    # commit_index、last_applied在所有Role角色中均会使用到，分析后得知，目前只有Leader/Follower才会使用到，其实 Candidate 只是用来election时才会用到，是一个临时跳板角色
​    """Volatile state on all servers: index of highest log entry known to be committed
​    (initialized to 0, increases monotonically)"""
​    self.commit_index = 0

​    """Volatile state on all servers: index of highest log entry applied to state machine
​    (initialized to 0, increases monotonically)"""
​    self.last_applied = 0

​    # Leaders



​    """Volatile state on Leaders: for each server, index of the next log entry to send to that server

​    (initialized to leader last log index + 1)

​      {<follower>: index, ...}

​    """

​    self.next_index = None



​    """Volatile state on Leaders: for each server, index of highest log entry known to be replicated on server

​    (initialized to 0, increases monotonically)

​      {<follower>: index, ...}

​    """

​    self.match_index = None
```



## lastApplied & commitIndex & latLogIndex 示意图

## Leader 中lastApplied & commitIndex源码分析

## Follower 中commitIndex源码分析
