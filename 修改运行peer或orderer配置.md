##### 场景
----------
本文章以修改peer日志级别为例，从DEBUG 改为 WARNING

你可能会疑问，我去这也用的着写文章？！容器内直接 CORE_LOGGING_LEVEL=WARNING 不就行了

如果有这种想法，那就有两个误区：
1、peer 在start的时候读取配置，而此时正在运行中，任何修改都不会生效
2、容器内修改环境变量不可永久化

这里深入理解下docker：
   容器不应该是长久性的东西，要保持容器的可抛弃性，有问题就应该rm掉；但要做好数据备份，然后直接run新的容器

##### 实现方案
-------------------
我们的实现方案：
  备份数据-》重建容器（修改环境变量）-》数据还原

##### 具体步骤
---------------------
具体步骤：

1、进入操作目录,停止peer容器
	docker stop peer0.org1.example.com
	
2、备份数据
	docker cp peer0.org1.example.com:/var/hyperledger/production /home
	

3、修改文件
	vi base/peer-base.yaml
	CORE_LOGGING_LEVEL 字段值改为 WARNING
4、重启节点
	docker-compose -f docker-cli.yaml up -d  --no-deps peer0.org1.example.com
	
5、停止peer节点
	docker stop peer0.org1.example.com
	
6、删除链码容器
	docker ps -a
	docker rm ...
	
7、数据恢复
	docker cp /home/production peer0.org1.example.com:/var/hyperledger/
	
8、重启节点
	docker restart peer0.org1.example.com
	
9、校验
   教研方式一：
   docker exec -it peer0.org1.example.com bash
   env
   
   查看CORE_LOGGING_LEVEL 是否为 WARNING
   
   校验方式二：
   docker logs -f peer0.org1.example.com
   观察日志，一些警告如：PKIid 可以忽略