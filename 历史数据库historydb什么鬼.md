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
key：命名空间ns + compositeKeySep + 写值key + compositeKeySep + block序列号 + 交易ID
value：[]byte{}

命名空间ns 作用指定链码
```

##### 作用
```
本质上是辅助作用
比如你想知道哪些交易动过我账户的钱，就可以通过历史库相关函数进行检索（存储key中有写值key），就可有获取相应区块、交易ID，进一步去账本中查询。
```

