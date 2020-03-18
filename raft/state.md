# Raft源码分析（一） - State

State 是 raft 的核心类，封装了框架性的性重要操作，持有Node (server.py)、Log、FileStorage、StateMachine对象；以及  Role 	角色，完成 Leader/Follower/Candidate 之间的转换。

BaseRole的构造方法，将传入State对象。

State 是上下文，将网络、Role转换、Log、FileStorage、StateMachine串联起来。

以下是**State类图**:





##### State源码分析

state.py





[下一篇](https://timequark.github.io/raft/role )

