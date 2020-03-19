# Raft源码分析（一） - State

State 是 raft 的核心类，封装了框架性的性重要操作，持有Node (server.py)、Log、FileStorage、StateMachine对象；以及  Role 	角色，完成 Leader/Follower/Candidate 之间的转换。

BaseRole的构造方法，将传入State对象。

State 是上下文，将网络、Role转换、Log、FileStorage、StateMachine串联起来。

以下是**State类图**:

![](https://timequark.github.io/raft/img/state.jpg)

（可以下载保存图片看大图~~~）



可以看出，State 是整个系统中的核心引擎类，衔接了Network、Node、Role（包括Leader/Follower/Candidate）、Log、StateMachine、FileStorage 等各个对象之间的联系。



## State源码分析

##### state.py

```python
class State:
    """Abstraction layer between Server & Raft State and Storage/Log & Raft State"""

    # <Leader object> if state is leader
    # <state_id> if state is follower
    # <None> if leader is not chosen yet
    # Roel为 Leader 时，leader 即 Leader 自身
    # Role为 Follower/Candidate 时，leader 为 Leader 的 id
    # 注意：
    # Leader 选举成功后，如果还有 Candidate （即可能出现的 split vote 所导致的多个 Candidate 共存），
    # Candidate 收到 Leader 的 append_entries 后，自动转成 Follower 角色
    leader = None

    # Await this future for election ending
    # 异步通知：阻塞在Leader上的动作
    leader_future = None

    # Node id that's waiting until it becomes leader and corresponding future
    # 异步通知：等待指定节点变为 Leader
    wait_until_leader_id = None
    wait_until_leader_future = None

    def __init__(self, server):
        self.server = server
        self.id = self._get_id(server.host, server.port)
        self.__class__.loop = self.server.loop

        # storage, log, state_machine, role
        # 均在 State 中实例化
        self.storage = FileStorage(self.id)
        self.log = Log(self.id)
        self.state_machine = StateMachine(self.id)

        # Leader/Follower/Candidate构造方法传入 State
        self.role = Follower(self)

    # Role启动
    def start(self):
        self.role.start()

    # Role停止
    def stop(self):
        self.role.stop()

    # 见@leader_required的注解实现
    @classmethod
    @leader_required
    async def get_value(cls, name):
        return cls.leader.state_machine[name]

    # 见@leader_required的注解实现
    @classmethod
    @leader_required
    async def set_value(cls, name, value):
        await cls.leader.execute_command({name: value})

    # 发送消息
    def send(self, data, destination):
        return self.server.send(data, destination)

    # 广播消息
    def broadcast(self, data):
        """Sends request to all cluster excluding itself"""
        return self.server.broadcast(data)

    # 数据传递流程：
    # UDPProtocl.datagram_received -> Node.request_handler -> State.request_handler
    # 根据type，调用相应的 on_reveive_*** 方法（on_receive_*** 方法在Leader/Follower/Candidate各自有不同的实现）
    def request_handler(self, data):
        getattr(self.role, 'on_receive_{}'.format(data['type']))(data)

    @staticmethod
    def _get_id(host, port):
        return '{}:{}'.format(host, port)

    def get_sender_id(self, sender):
        return self._get_id(*sender)

    @property
    def cluster(self):
        return [self._get_id(*address) for address in self.server.cluster]

    # 过半判定
    def is_majority(self, count):
        return count > (self.server.cluster_count // 2)

    # Role 转成 Candidate
    def to_candidate(self):
        self._change_role(Candidate)
        self.set_leader(None)

    # Role 转成 Leader
    def to_leader(self):
        self._change_role(Leader)
        self.set_leader(self.role)
        # 回调on_leader监听方法
        if asyncio.iscoroutinefunction(config.on_leader):
            asyncio.ensure_future(config.on_leader())
        else:
            config.on_leader()

    # Role 转成 Follower
    def to_follower(self):
        self._change_role(Follower)
        self.set_leader(None)
        # 回调on_follower监听方法
        if asyncio.iscoroutinefunction(config.on_follower):
            asyncio.ensure_future(config.on_follower())
        else:
            config.on_follower()

    def set_leader(self, leader):
        '''
        We can have a look at description in Class State. Like the following part:
        
        # <Leader object> if state is leader
        # <state_id> if state is follower
        # <None> if leader is not chosen yet
        leader = None
        '''
        cls = self.__class__
        cls.leader = leader

        # 异步通知 cls.leader_future
        if cls.leader and cls.leader_future and not cls.leader_future.done():
            # We release the future when leader is elected
            cls.leader_future.set_result(cls.leader)

        # 异步通知 cls.wait_until_leader_future
        if cls.wait_until_leader_id and (
            cls.wait_until_leader_future and not cls.wait_until_leader_future.done()
        ) and cls.get_leader() == cls.wait_until_leader_id:
            # We release the future when specific node becomes a leader
            cls.wait_until_leader_future.set_result(cls.leader)

    def _change_role(self, new_role):
        self.role.stop()
        self.role = new_role(self)
        self.role.start()

    # 获取 Leader id
    @classmethod
    def get_leader(cls):
        if isinstance(cls.leader, Leader):
            return cls.leader.id

        return cls.leader

    # 等待选举成功
    @classmethod
    async def wait_for_election_success(cls):
        """Await this function if your cluster must have a leader"""
        if cls.leader is None:
            cls.leader_future = asyncio.Future(loop=cls.loop)
            await cls.leader_future

    # 等待指定节点成为 Leader
    @classmethod
    async def wait_until_leader(cls, node_id):
        """Await this function if you want to do nothing until node_id becomes a leader"""
        if node_id is None:
            raise ValueError('Node id can not be None!')

        if cls.get_leader() != node_id:
            cls.wait_until_leader_id = node_id
            cls.wait_until_leader_future = asyncio.Future(loop=cls.loop)
            await cls.wait_until_leader_future

            cls.wait_until_leader_id = None
            cls.wait_until_leader_future = None
```



[下一篇](https://timequark.github.io/raft/role )

