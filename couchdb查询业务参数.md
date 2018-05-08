
注意：fabric现在支持leveldb和couchdb两种存储方式，由于账本和历史库只支持增和读所以默认为leveldb不可变。只有世界状态（state，业务参数）才可以选择存储方式。

couchdb提供了非常丰富的restful api 以example02为例（存储了a、b两个值），查询b的数值：

```
curl http://192.168.5.65:5984/mychannel/mycc%00b?attachments=true -H 'Accept:text/plain'
```


```
- 192.168.5.65:5984 路径端口
- mychannel 所属通道
- mycc%00b 查询key

- attachments=true  返回附近实体（默认false）
- -H 'Accept:text/plain' 以明文返回（默认Gzip）
```

需要注意的是mycc%00b",b是我们的业务参数key，但是fabric存储的时候，会将该参数与所属链码联系在一起即：


```
mycc + \u0000 + b
```

但我们请求的时候需要将\u0000转为%00

结果：


```
{"_id":"mycc\u0000b","_rev":"5-331e4a0122c63cfe553f161134610712","chaincodeid":"mycc","version":"8:0","_attachments":{"valueBytes":{"content_type":"application/octet-stream","revpos":5,"digest":"md5-TcPMX+YbJ0SWm26KCA62uQ==","data":"MjQw"}}}
```

"data":"MjQw"  的data值就是我们要的结果了，base64解码就可以了

[couchdb 官网命令详解](http://docs.couchdb.org/en/2.1.1/api/document/common.html)