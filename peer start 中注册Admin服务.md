### peer start 引用

```
文件：/fabric/peer/node/start.go
函数：serve
代码：pb.RegisterAdminServer(peerServer.Server(), core.NewAdminServer())
```

### Admin 原型和实现

```
原型：
/fabric/protos/peer/admin.proto
/fabric/protos/peer/admin.pb.go
核心实现：
/fabric/core/admin.go
```

### Admin 原型定义

```
service Admin {
    // Return the serve status.
    rpc GetStatus(google.protobuf.Empty) returns (ServerStatus) {}
    rpc StartServer(google.protobuf.Empty) returns (ServerStatus) {}
    rpc GetModuleLogLevel(LogLevelRequest) returns (LogLevelResponse) {}
    rpc SetModuleLogLevel(LogLevelRequest) returns (LogLevelResponse) {}
    rpc RevertLogLevels(google.protobuf.Empty) returns (google.protobuf.Empty) {}
}

从服务定义可以看出，Admin服务实现对服务器、模块日志级别的获取和控制。
```

### TODO

可以看/fabric/core/admin.go代码比较简练，简单的实现了Admin定义的服务


------------------------------------
代码版本：v1.0.0

