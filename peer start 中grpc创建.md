请先看“grpc 服务”文章

### peer node start启动的grpc服务

在start.go中serve函数中，创建的GRPCServer服务对象有两个：


```
- peerServer
- globalEventsServer
```

创建peerServer，在/fabric/core/peer/peer.go中定义。事件服务器globalEventsServer，是一个全局单例，在/fabric/events/producer/producer.go中定义。

##### peerServer

创建peerServer

追溯serve函数中peerServer对象的创建，==代码最终都使用了/fabric/core/comm/server.go中的NewGRPCServerFromListener函数创建了一个grpcServerImpl实例对象==，对象中的。


```
//在serve函数中
peerServer, err := peer.CreatePeerServer(listenAddr, secureConfig)

//在CreatePeerServer函数中
peerServer, err = comm.NewGRPCServer(listenAddress, secureConfig)

//在NewGRPCServer函数中
lis, err := net.Listen("tcp", address)
NewGRPCServerFromListener(lis, secureConfig)

//在NewGRPCServerFromListener函数中，最终建立grpc标准服务器并返回grpcServerImpl
grpcServer.server = grpc.NewServer(serverOpts...)
return grpcServer
```

##### globalEventsServer


```
//在/fabric/events/producer/producer.go中定义
//全局单例
var globalEventsServer *EventsServer
//定义和Chat实现
type EventsServer struct {
}
func (p *EventsServer) Chat(stream pb.Events_ChatServer) error {...}
//初始化函数
func NewEventsServer(bufferSize uint, timeout int) *EventsServer {...}
```

事件服务器这个全局单例没有任何成员，只有一个专用初始化函数NewEventsServer，一个Chat实现。这一切看上去非常简单，只是在专用初始化函数中的一句initializeEvents(bufferSize, timeout)又牵扯出了一段文字。原来globalEventsServer自己只是一个事件服务器的代表，实际做事情的是initializeEvents(bufferSize, timeout)初始化并运行的eventProcessor对象，以后再说。

在serve函数中，使用ehubGrpcServer, err := createEventHubServer(secureConfig)完成了对事件服务器的创建和注册，ehubGrpcServer承接的就是globalEventsServer这个全局单例。

创建globalEventsServer


```
//在createEventHubServer中
//创建grpcServerImpl对象，其中包含了grpc标准服务器
lis, err = net.Listen("tcp", viper.GetString("peer.events.address"))
grpcServer, err := comm.NewGRPCServerFromListener(lis, secureConfig)

//创建事件服务器，NewEventsServer返回的就是globalEventsServer
ehServer := producer.NewEventsServer(
        uint(viper.GetInt("peer.events.buffersize")),
        viper.GetInt("peer.events.timeout"))
```

##### 结论

本文主要介绍peer中grpc server的创建，都是“grpc 服务”文章中定义的对象。并没有提及服务注册，会在后面依次介绍

------------------------------------
代码版本：v1.0.0