MVCC_READ_CONFLICT 顾名思义是交易读集冲突，并发导致的结果

到现在我发现两种情况：

##### 情况一：
日志

```
{"log":"\u001b[36m2018-05-04 08:43:58.760 UTC [statebasedval] validateKVRead -\u003e DEBU 10e0ae7\u001b[0m Version mismatch for key [ccdasset:DA_hc1PC3PkhsUeLh1wKfkV7G176taFWhvmf1Ye]. Committed version = [\u0026version.Height{BlockNum:0x51e0, TxNum:0x0}], Version in readSet [\u0026kvrwset.Version{BlockNum:0x51df, TxNum:0x0}]\n","stream":"stderr","time":"2018-05-04T08:43:58.762209964Z"}
{"log":"\u001b[33m2018-05-04 08:43:58.760 UTC [statebasedval] ValidateAndPrepareBatch -\u003e WARN 10e0ae8\u001b[0m Block [20961] Transaction index [0] TxId [313e8a9ea20eda08711f7a62f4b2cc1c16860a33861401aa6fb0b891f2924044] marked as invalid by state validator. Reason code [MVCC_READ_CONFLICT]\n","stream":"stderr","time":"2018-05-04T08:43:58.762232563Z"}
```


从下面摘选可以看出，新块已经是0x51e0，但是读集版本为0x51df

```
Committed version = [\u0026version.Height{BlockNum:0x51e0, TxNum:0x0}], Version in readSet [\u0026kvrwset.Version{BlockNum:0x51df, TxNum:0x0}]\n"
```


情况二：
日志

```
{"log":"\u001b[36m2018-05-04 08:45:57.670 UTC [statebasedval] ValidateAndPrepareBatch -\u003e DEBU 10e5122\u001b[0m Block [20989] Transaction index [0] TxId [cad9bc415e92b55b650f5af051d9e998b65711f3033b2d0ce760b137f31221d4] marked as valid by state validator\n","stream":"stderr","time":"2018-05-04T08:45:57.671659556Z"}
{"log":"\u001b[33m2018-05-04 08:45:57.671 UTC [statebasedval] ValidateAndPrepareBatch -\u003e WARN 10e5125\u001b[0m Block [20989] Transaction index [1] TxId [950946b7da4d526470a244292a3a16a3ee84952a183367a0bd1b68e04d4b7bf5] marked as invalid by state validator. Reason code [MVCC_READ_CONFLICT]\n","stream":"stderr","time":"2018-05-04T08:45:57.671833852Z"}
```


该块有两笔交易，交易一成功交易而读冲突，这是因为这两笔交易都读同一个key，第一个交易可能会影响key值，所以第二笔失败

