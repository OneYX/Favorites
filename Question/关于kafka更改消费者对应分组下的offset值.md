# 关于kafka更改消费者对应分组下的offset值

[其他](https://www.itread01.com/infolist/其他/1/) ·发表2018-07-04

解析验证zkcli 四种参数解析fse 其他bootstra test

kafka的offset保存位置分为两种情况0.9.0.0版本之前默认保存在zookeeper当中
0.9.0.0版本之后保存在broker对应的topic当中

1. 如何辨别你启用的consumer的offset保存位置
进入zookeeper的命令行当中zkCli.sh localhost:2181用ls /查看目录
如果你在代码中定义的group id没有在/consumers这个文件夹中，代表offset保存在broker的topic中
前提是consumer确实已经创建并启动
如果group ID在/consumers目录下存在则offset的保存位置是/consumers/{group}/offsets/{topic}/{partition}
反之offset的保存位置则是/broker/topic/__consumer_offsets当中

2. 如何查看自己消费者分组对应的topic的offset
第一种方式
bin/kafka-run-class.sh kafka.tools. ConsumerOffsetChecker --topic test-topic --zookeeper zookeeper:2181 --group group名字
第二种方式
kafka-consumer-offset-checker --zookeeper localhost :2181/kafka --group test-consumer-group --topic test
第三种方式
bin/kafka-run-class.sh kafka.tools. ConsumerOffsetChecker --topic test --zookeeper 10.0.10.10:2181 --group test-consumer-group
第四种方式
bin/kafka-consumer-groups.sh --bootstrap-server dbnode4:9092, dbnode5:9092, dbnode6:9092 --group test -consumer-group --describe

3. 更改指定消费者分组对应topic的offset
第一种情况offset信息保存在topic中
bin/kafka-consumer-groups.sh --bootstrap-server dbnode4:9092, dbnode5:9092, dbnode6:9092 --group test -consumer-group --topic test --execute --reset-offsets --to-offset 10000
参数解析： --bootstrap-server代表你的offset保存在topic中
--group代表你的消费者分组
--topic代表你消费的主题
--execute代表支持复位偏移
--reset-offsets代表要进行偏移操作
--to-offset代表你要偏移到哪个位置是long类型数值，只能比前面查询出来的小
还有其他的--to- **方式可以自己验证本人验证过--to-datetime没有成功

第二种方式offset信息保存在zookeeper当中
bin/kafka-consumer-groups.sh --zookeeper z1:2181, z2:2181, z3:2181 --group test-consumer-group --topic test --execute -- reset-offsets --to-offset 10000
--zookeeper和--bootstrap-server只能选一种方式
