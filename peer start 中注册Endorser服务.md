### peer start 引用

```
文件：/fabric/peer/node/start.go
函数：serve
代码：serverEndorser := endorser.NewEndorserServer()
	 pb.RegisterEndorserServer(peerServer.Server(), serverEndorser)
```

### Endorser 原型和实现

```
原型：
/fabric/protos/peer/peer.proto
/fabric/protos/peer/peer.pb.go
核心实现：
/fabric/core/endorser/endorser.go
```

### Endorser 原型定义

```
service Endorser {
	rpc ProcessProposal(SignedProposal) returns (ProposalResponse) {}
}

从服务定义可以看出，Endorser处理背书请求。
```

### 核心代码实现
```
type Endorser struct {
    policyChecker policy.PolicyChecker
}

//Endorser专用初始化函数
func NewEndorserServer() pb.EndorserServer {
    e := new(Endorser)
    e.policyChecker = policy.NewPolicyChecker(
        peer.NewChannelPolicyManagerGetter(),
        mgmt.GetLocalMSP(),
        mgmt.NewLocalMSPPrincipalGetter(),
    )
    return e
}

//实现ProcessProposal服务
func (e *Endorser) ProcessProposal(...){...}
func (e *Endorser) endorseProposal(...){...}


Endorser服务的核心实现中，只有一个核心函数ProcessProposal，endorser.go中其余的函数都是相互配合供ProcessProposal调用，处理客户端发来的SignedProposal数据，返回ProposalResponse数据，完成最终的任务
```

### 实现过程
这里暂时赋上过程

参数校验->模拟执行->背书

-------
代码版本：v1.0.0