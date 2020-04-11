# raft4j

基于开源的 [wenweihu86/raft-java](https://github.com/wenweihu86/raft-java)，对raft原理方法进行源代码解读。

在源码分析前，强烈建议小伙伴先去[raft.github.io](https://raft.github.io/)上了解raft的工作原理，特别是要阅读[raft白皮书](https://raft.github.io/raft.pdf)。

了解raft原理的好处、目的是，更好的理解raft源码，不至于非常吃力~

## raft4j源码分析（一） - [RPC通信](https://timequark.github.io/raft4j/rpc)

