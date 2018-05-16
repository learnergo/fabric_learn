### peer start 引用

```
文件：/fabric/peer/node/start.go
函数：serve
代码：registerChaincodeSupport(peerServer.Server())
```


### ChaincodeSupport 原型和实现

```
原型：
/fabric/protos/peer/chaincode_shim.proto
/fabric/protos/peer/chaincode_shim.pb.go
核心实现：
/fabric/core/chaincode/chaincode_support.go
```


chaincode_shim.proto中定义服务

```
rpc Register(stream ChaincodeMessage) returns (stream ChaincodeMessage) {}
```

该服务实现客户端和服务器端ChaincodeMessage类型流数据的交换。

用于服务端流数据交换的grpc流服务接口象为ChaincodeSupport_RegisterServer，定义于

```
/fabric/protos/peer/chaincode_shim.pb.go
```

### ChaincodeSupport 作用

```
代码中注释：
ChaincodeSupport responsible for providing interfacing with chaincodes from the Peer.
即提供与链码的交互
```
前面介绍过Provider是提供对象，而这个Support是提供操作，主要有Launch，Register，Execute，HandleChaincodeStream，Stop。而在grpc中公开的只有Register（这个Register不是提供给客户端，而是提供给链码容器）

### ChaincodeSupport 怎么管理多容器
一个peer有一个ChaincodeSupport单例，那它是怎么管理多容器呢？

ChaincodeSupport中chaincodeMap 字典结构维护所有关联链码容器的处理句柄。

### ChaincodeSupport 方法说明


```
代码中注释
Launch：Launch will launch the chaincode if not running (if running return nil) and will wait for handler of the chaincode to get into FSM ready state.
Register：Register the bidi stream entry point called by chaincode to register with the Peer.
Execute：Execute executes a transaction and waits for it to complete until a timeout value.
HandleChaincodeStream：
Stop：Stop stops a chaincode if running

中文理解：
Launch：启动链码容器，如果正运行返回nil，这个方法也常被用作校验，在取值和存值前都会执行
Register：链码容器启动后会向peer提交注册，注册的目的是让peer记录该容器处理句柄（注：ChaincodeSupport 怎么管理多容器）
Execute：执行链码交易
HandleChaincodeStream：处理容器返回流
Stop：停止容器
```

