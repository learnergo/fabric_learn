### peer start 
 是peer节点启动命令，源码文件：
 
```
GOPATH\src\github.com\hyperledger\fabric\peer\node\start.go
```


### startCmd 命令


```
func startCmd() *cobra.Command {
    flags := nodeStartCmd.Flags()
    flags.BoolVarP(&chaincodeDevMode, "peer-chaincodedev", "", false,
    	"Whether peer in chaincode development mode")
    flags.BoolVarP(&peerDefaultChain, "peer-defaultchain", "", false,
    	"Whether to start peer with chain testchainid")
    flags.StringVarP(&orderingEndpoint, "orderer", "o", "orderer:7050", "Ordering service endpoint")
    
    //以上配置环境变量，start真正操作是返回nodeStartCmd
    return nodeStartCmd
}
```


看下面nodeStartCmd具体实现

### nodeStartCmd 实现


```
var nodeStartCmd = &cobra.Command{
	Use:   "start",
	Short: "Starts the node.",
	Long:  `Starts a node that interacts with the network.`,
	RunE: func(cmd *cobra.Command, args []string) error {
		return serve(args)
	},
}
```

nodeStartCmd 命令行主要实现为serve ，看下面serve具体实现：

### serve 实现

serve 是peer start的真正实现，包括账本、链码、背书等等功能，接下来会以注释的形式说明
该篇主要介绍peer start大体实现，具体分支箱账本初始化过程会在后续详细分解
英文是原注释，中文是本人注释


```
func serve(args []string) error {
	logger.Infof("Starting %s", version.GetInfo())
	
	//初始化账本管理，peer 节点可以加入多个通道管理多个账本，比如账本的创建、打开、列表和关闭等等
	ledgermgmt.Initialize()
	
	// Parameter overrides must be processed before any parameters are
	// cached. Failures to cache cause the server to terminate immediately.
	if chaincodeDevMode {
		logger.Info("Running in chaincode development mode")
		logger.Info("Disable loading validity system chaincode")

		viper.Set("chaincode.mode", chaincode.DevModeUserRunsChaincode)

	}
	//缓存常用配置，主要：localAddress 和 peerEndpoint 也就是peer节点的地址和名称。缓存失败则返回
	if err := peer.CacheConfiguration(); err != nil {
		return err
	}
	
	//前面已经缓存成功，这里直接取
	peerEndpoint, err := peer.GetPeerEndpoint()
	if err != nil {
		err = fmt.Errorf("Failed to get Peer Endpoint: %s", err)
		return err
	}

	listenAddr := viper.GetString("peer.listenAddress")

	//peer 通讯配置，有无使用tls以及相关文件路径
	secureConfig, err := peer.GetSecureConfig()
	if err != nil {
		logger.Fatalf("Error loading secure config for peer (%s)", err)
	}
	
	//peer start是启动一个peer服务（严格说是grpc服务），根据监听地址和通讯配置创建一grpc服务
	peerServer, err := peer.CreatePeerServer(listenAddr, secureConfig)
	if err != nil {
		logger.Fatalf("Failed to create peer server (%s)", err)
	}

	if secureConfig.UseTLS {
		logger.Info("Starting peer with TLS enabled")
		// set up CA support
		
		//GetCASupport 创建了全局单例，也就是说这个peer服务所有需要的root ca支持是一致的
		
		caSupport := comm.GetCASupport()
		caSupport.ServerRootCAs = secureConfig.ServerRootCAs
	}

	//TODO - do we need different SSL material for events ?
	//创建并返回peer节点事件服务
	ehubGrpcServer, err := createEventHubServer(secureConfig)
	if err != nil {
		grpclog.Fatalf("Failed to create ehub server: %v", err)
	}

	// enable the cache of chaincode info
	//开启链码缓存默认不开启，将安装的链码保存到内存从内存中读取，否则从 /var/hyperledger/production/chaincodes 目录下读取文件 
	ccprovider.EnableCCInfoCache()

	//ccSrv 主要包含grpc服务端，ccEpFun 是一个函数返回节点ID 和地址
	ccSrv, ccEpFunc := createChaincodeServer(peerServer, listenAddr)
	//注册链码服务，主要包括系统链码的注册和链码提供者实例
	registerChaincodeSupport(ccSrv.Server(), ccEpFunc)
	//服务启动
	go ccSrv.Start()

	logger.Debugf("Running peer")

	// Register the Admin server
	//注册Admin服务
	pb.RegisterAdminServer(peerServer.Server(), core.NewAdminServer())

	// Register the Endorser server
	
	//注册Endorser服务
	serverEndorser := endorser.NewEndorserServer()
	pb.RegisterEndorserServer(peerServer.Server(), serverEndorser)

	// Initialize gossip component
	//获取bootstrap url
	bootstrap := viper.GetStringSlice("peer.gossip.bootstrap")

	//获取签名证书
	serializedIdentity, err := mgmt.GetLocalSigningIdentityOrPanic().Serialize()
	if err != nil {
		logger.Panicf("Failed serializing self identity: %v", err)
	}

	messageCryptoService := peergossip.NewMCS(
		peer.NewChannelPolicyManagerGetter(),
		localmsp.NewSigner(),
		mgmt.NewDeserializersManager())
	secAdv := peergossip.NewSecurityAdvisor(mgmt.NewDeserializersManager())

	// callback function for secure dial options for gossip service
	secureDialOpts := func() []grpc.DialOption {
		var dialOpts []grpc.DialOption
		// set max send/recv msg sizes
		dialOpts = append(dialOpts, grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(comm.MaxRecvMsgSize()),
			grpc.MaxCallSendMsgSize(comm.MaxSendMsgSize())))
		// set the keepalive options
		dialOpts = append(dialOpts, comm.ClientKeepaliveOptions()...)

		if comm.TLSEnabled() {
			tlsCert := peerServer.ServerCertificate()
			dialOpts = append(dialOpts, grpc.WithTransportCredentials(comm.GetCASupport().GetPeerCredentials(tlsCert)))
		} else {
			dialOpts = append(dialOpts, grpc.WithInsecure())
		}
		return dialOpts
	}
	err = service.InitGossipService(serializedIdentity, peerEndpoint.Address, peerServer.Server(),
		messageCryptoService, secAdv, secureDialOpts, bootstrap...)
	if err != nil {
		return err
	}
	defer service.GetGossipService().Stop()

	//initialize system chaincodes
	//前面介绍注册了系统链码，这里是部署系统链码
	initSysCCs()

	//this brings up all the chains (including testchainid)
	//拉起该节点所有通道链
	peer.Initialize(func(cid string) {
		logger.Debugf("Deploying system CC, for chain <%s>", cid)
		scc.DeploySysCCs(cid)
	})

	logger.Infof("Starting peer with ID=[%s], network ID=[%s], address=[%s]",
		peerEndpoint.Id, viper.GetString("peer.networkId"), peerEndpoint.Address)

	// Start the grpc server. Done in a goroutine so we can deploy the
	// genesis block if needed.
	serve := make(chan error)

	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		sig := <-sigs
		logger.Debugf("sig: %s", sig)
		serve <- nil
	}()

	go func() {
		var grpcErr error
		if grpcErr = peerServer.Start(); grpcErr != nil {
			grpcErr = fmt.Errorf("grpc server exited with error: %s", grpcErr)
		} else {
			logger.Info("peer server exited")
		}
		serve <- grpcErr
	}()

	if err := writePid(config.GetPath("peer.fileSystemPath")+"/peer.pid", os.Getpid()); err != nil {
		return err
	}

	// Start the event hub server
	if ehubGrpcServer != nil {
		go ehubGrpcServer.Start()
	}

	// Start profiling http endpoint if enabled
	if viper.GetBool("peer.profile.enabled") {
		go func() {
			profileListenAddress := viper.GetString("peer.profile.listenAddress")
			logger.Infof("Starting profiling server with listenAddress = %s", profileListenAddress)
			if profileErr := http.ListenAndServe(profileListenAddress, nil); profileErr != nil {
				logger.Errorf("Error starting profiler: %s", profileErr)
			}
		}()
	}

	logger.Infof("Started peer with ID=[%s], network ID=[%s], address=[%s]",
		peerEndpoint.Id, viper.GetString("peer.networkId"), peerEndpoint.Address)

	// set the logging level for specific modules defined via environment
	// variables or core.yaml
	overrideLogModules := []string{"msp", "gossip", "ledger", "cauthdsl", "policies", "grpc"}
	for _, module := range overrideLogModules {
		err = common.SetLogLevelFromViper(module)
		if err != nil {
			logger.Warningf("Error setting log level for module '%s': %s", module, err.Error())
		}
	}

	flogging.SetPeerStartupModulesMap()

	// Block until grpc server exits
	return <-serve
}
```


### start 主要工作

```
从前面我们可以看出start中主要做了几件工作
- 初始化账本
- 服务创建
- 注册链码服务
- 注册Admin服务
- 注册Endorser服务
```

---------------------

代码版本：v1.0.0