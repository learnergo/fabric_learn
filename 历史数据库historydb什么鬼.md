##### history数据库开启

```
core.yaml 中enableHistoryDatabase字段，默认true
```

##### 存储
```
由于historydb是辅助记录数据库，只支持增读不支持修改，所以用leveldb存储
```

##### 存储过程
```
提交块后，对块中有效交易进行检索，只保存交易写集中的key值
```

##### 存储格式


```
因为leveldb存储，必定有key和value：
key：命名空间ns + compositeKeySep + 写值key + compositeKeySep + block序列号 + 交易ID(区块中的序列)
value：[]byte{}

命名空间ns 作用指定链码
```

##### 作用
```
本质上是辅助作用
比如你想知道哪些交易动过我账户的钱，就可以通过历史库相关函数进行检索（存储key中有写值key），就可有获取相应区块、交易ID，进一步去账本中查询。

历史查询器

对HistoryDB的检索主要通过HistoryDB提供的一个HistoryDBQueryExecutor对象来实现。HistoryQueryExecutor在historyleveldb/historyleveldb_query_executer.go中实现，只提供GetHistoryForKey(...)一个接口，该接口根据提供的命名空间和写值key，返回一个迭代器historyScanner（内部封装了levedb数据库的Iterator）。是迭代器必定涉及起点和止点，historyScanner的起点是命名空间ns + compositeKeySep + 写值key + compositeKeySep的组合键，止点是命名空间ns + compositeKeySep + 写值key + compositeKeySep + 0xff的组合键。对比起点和止点，止点多了一个0xff，相当于一个字符的极限值，也因此这个范围查询的是所有以命名空间ns + compositeKeySep + 写值key + compositeKeySep为开头的key值。比如起点是[]byte("A")，止点是append([]byte("A"),[]byte{0xff}...)，则这个范围查询的是所有以字符A开头的key。又因为有HistoryDB存储的格式为前提，假设这里的ns为“chaincode_example02”，写值key为“A账户”，则这个起止范围相当于在查询所有chaincode_example02链码上改动过A账户的“ blockID+有效交易ID ”的信息。而有了blockID+有效交易ID这两个信息，自然可以通过BlockStore定位查询出原交易的所有信息，如改动时间，改动值，是否是删除操作等。
```



