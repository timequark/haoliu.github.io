# RPC通信

 [wenweihu86/raft-java](https://github.com/wenweihu86/raft-java) 的 RPC 通信层使用了 gPRC + Netty 方式，服务端与客户端使用 gRPC proto 定义消息格式，发送、接收消息时，对消息使用自定义 Netty  的 encoder、decoder 来完成 消息体 与 字节流 之间的编码、解码工作，使用自定义的 handler 来完成业务逻辑的处理。

先上 UML 图，了解一下大体的流程逻辑过程：

## RPC 模型概览

![](https://timequark.github.io/raft4j/img/rpc-architecture.jpg)

## RPCServer

![](https://timequark.github.io/raft4j/img/rpc-server-uml.jpg)

## RPCClient

![](https://timequark.github.io/raft4j/img/rpc-client-uml.jpg)