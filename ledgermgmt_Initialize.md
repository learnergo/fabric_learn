### 账本源码目录
- common/ledger
- core/ledger

### 数据库操作
- goleveldb common/ledger/util/leveldbhelper
- couchDB   core/ledger/util/couchdb

### ledgermgmt.Initialize
fabric\core\ledger\ledgermgmt\ledger_mgmt.go

函数内部直接once.Do了initialize()函数。在initialize()函数中，除了日志操作和锁保护外，所做的只是初始化了三个对象：
openedLedgers，
ledgerProvider，
initialized

该三个对象均为文件中的全局变量：

initialized = true
openedLedgers = make(map[string]ledger.PeerLedger)
provider, err := kvledger.NewProvider()
ledgerProvider = provider

操作结果：
  initialized和openedLedgers分别赋了初值和分配了内存，initialized则是是否初始化的标识，openedLedgers存放peer的账本映射的，两者并未有进一步操作，可暂时搁置一旁。
  而ledgerProvider则被赋于由kvledger.NewProvider()函数返回的值，从其名字我们就可以知道该对象是一个账本服务提供者，接下来主要看ledger.PeerLedgerProvider接口。

### PeerLedgerProvider
文件目录：
core/ledger/ledger_interface.go

type PeerLedgerProvider interface {
    Create(genesisBlock *common.Block) (PeerLedger, error)
    Open(ledgerID string) (PeerLedger, error)
    Exists(ledgerID string) (bool, error)
    List() ([]string, error)
    Close()
}

根据Fabric的惯例，有Provider字样的对象，或大或小，都是某一主题模块服务的提供者，提供该主题模块的一系列操作服务。
看上面接口，大概可以知道该provider主要负责“提供”账本对象（PeerLedger），如：创建、打开、关闭、是否存在、列表等操作。

### Provider 

接口PeerLedgerProvider，在具体实现上则会分为多种具体的Provider以供使用（同时也留下了扩展空间）。ledger的Provider亦然如此，kvledger.NewProvider()函数返回的就是PeerLedgerProvider接口的一个具体实现Provider，在core/ledger/kvledger/kv_ledger_provider.go中定义：

type Provider struct {
    idStore            *idStore                      //ledgerID数据库
    blockStoreProvider blkstorage.BlockStoreProvider //block数据库存储服务对象
    vdbProvider        statedb.VersionedDBProvider   //状态数据库存储服务对象
    historydbProvider  historydb.HistoryDBProvider   //历史数据库存储服务对象
}

Provider 实现了PeerLedgerProvider接口可以“提供”账本对象，那Provider中四个对象是什么鬼？！
这就要介绍下peer中存储目录结构：

- /var/hyperledger/production
   - ledgersData 
      - ledgerProvider 
      - chains  
        - index //block索引数据库目录
        - chains //block块存储数据库目录
            账本ID1
            账本ID2
            …
      - stateLeveldb 
      - historyLeveldb 

可以看到ledgersData（账本目录）目录下有四个子目录：
- ledgerProvider ledgerID数据库目录
- chains 真正账本
- stateLeveldb 状态数据库目录
- historyLeveldb 历史数据库目录

这下可以理解Provider下为啥有四个字段，每个字段的含义也相当明显了，定义了Provider对象相应的“提供”各自对象，
idStore 记录ledgerProvider数据库信息， stateLeveldb和historyLeveldb 相当简单，提供了状态和历史数据库。看上面结构chains下还有一层结构，就知道blkstorage.BlockStoreProvider最复杂。 结构讲到这里，代码也就不难了。

### 代码目录介绍

core/ledger/ledgerconfig  账本配置比如各个数据库存储位置，账本存储大小
core/ledger/ledgermgmt 账目管理类，主要负责账本的初始化入口，提供创建和打开账本的方法
core/ledger/util 最重要的提供了couchdb的操作类

core/ledger/kvledger/kv_ledger.go  ledger.PeerLedger的实现
core/ledger/kvledger/kv_ledger_provider.go 上面一直在说的Provider的实现文件
core/ledger/kvledger/history history库实现
core/ledger/kvledger/txmgmt 世界状态库实现，但其中有重要的交易校验有机会再讲
