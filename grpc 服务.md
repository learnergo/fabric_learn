对grpc入门请看grpc目录中helloworld示例

### fabric中的grpc服务接口和实例
注意是grpc的服务接口，这里是对grpc.server 的封装


```
/fabric/core/comm/server.go
```


##### TLS安全配置项


```
type SecureServerConfig struct {
    //Whether or not to use TLS for communication
    UseTLS bool
    //PEM-encoded X509 public key to be used by the server for TLS communication
    //在core.yaml中指定，读取的tls目录下server.cert文件数据存储于此
    ServerCertificate []byte
    //PEM-encoded private key to be used by the server for TLS communication
    //在core.yaml中指定，读取的tls目录下server.key文件数据存储于此
    ServerKey []byte
    //Set of PEM-encoded X509 certificate authorities to optionally send
    //as part of the server handshake
    //在core.yaml中指定，读取的tls目录下ca.crt文件数据存储于此
    ServerRootCAs [][]byte
    //Whether or not TLS client must present certificates for authentication
    RequireClientCert bool
    //Set of PEM-encoded X509 certificate authorities to use when verifying
    //client certificates
    ClientRootCAs [][]byte
}
```


##### GRPCServer接口


```
type GRPCServer interface {
    //返回GRPCServer监听的地址
    Address() string
    //启动下层的grpc.Server
    Start() error
    //停止下层的grpc.Server
    Stop()
    //返回GRPCServer实例对象
    Server() *grpc.Server
    //返回GRPCServer使用的网络监听实例对象
    Listener() net.Listener
    //返回grpc.Server使用的Certificate
    ServerCertificate() tls.Certificate
    //标识GRPCServer实例是否使用TLS
    TLSEnabled() bool
    //增加PEM-encoded X509 certificate authorities到
    //用于验证客户端certificates的authorities列表
    AppendClientRootCAs(clientRoots [][]byte) error
    //用于验证客户端certificates的authorities列表中
    //删除PEM-encoded X509 certificate authorities
    RemoveClientRootCAs(clientRoots [][]byte) error
    //基于一个PEM-encoded X509 certificate authorities列表
    //设置用于验证客户端certificates的authorities列表
    SetClientRootCAs(clientRoots [][]byte) error
}
```


##### GRPCServer实现实例


```
type grpcServerImpl struct {
    //server指定的监听地址，地址格式：hostname:port
    address string
    //监听address的监听对象，用于处理网络请求
    listener net.Listener
    //标准的grpc服务器，通过此对象进行各种grpc服务操作
    server *grpc.Server
    //Certificate presented by the server for TLS communication
    serverCertificate tls.Certificate
    //Key used by the server for TLS communication
    serverKeyPEM []byte
    //List of certificate authorities to optionally pass to the client during
    //the TLS handshake
    serverRootCAs []tls.Certificate
    //lock to protect concurrent access to append / remove
    lock *sync.Mutex
    //Set of PEM-encoded X509 certificate authorities used to populate
    //the tlsConfig.ClientCAs indexed by subject
    clientRootCAs map[string]*x509.Certificate
    //TLS configuration used by the grpc server
    tlsConfig *tls.Config
    //Is TLS enabled?
    tlsEnabled bool
}
```

--------------------------
代码版本: v1.0.0