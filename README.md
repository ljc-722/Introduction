# Introduction

1.通过docker将两个tar文件（clickhouse.tar/rocketmq.tar）上传到ACR上。

# 接下来部署的所有SAE实例容量都设置成最少1个，不然可能出现报错。

2.基于SAE先部署<微服务应用>clickhouse，和nacos一样步骤，就换了一个镜像。实例启动成功的话日志会显示
  processing ...
  Merging ...
  Logging ...
  Logging ...
  
3.打开实例终端（点击webshell），先进入客户端终端创建数据库
  $clickhouse-client
  
# 注意一下，如果进入客户端终端后，自己手打代码会有一些不太美观的表现，会影响书写体验，最好还是复制粘贴。:)代表进入客户端终端了

  :)create database if not exists db_log;
  :)use db_log;
  :)create table tbl_devices_log (
    timestamp DateTime, 
    log_level String, 
    message String, 
  )ENGINE=MergeTree()
  ORDER BY timestamp;

# 下一步测试，可选，注意是tbl的l是字母l不是数字1
  :)insert into db_log.tbl_devices_log (timestamp, log_level, message) values
  ('2024-10-19 14:00:00', 'WARNING', 'Abnormal device[001] temperature');

  :)select * from db_log.tb1_devices_log;

4.基于SAE依次部署<微服务应用>三个rocketmq组件，即RocketMQ-NameServer、RocketMQ-Broker、RocketMQ-Proxy
  
# 依RocketMQ-NameServer -> RocketMQ-Broker -> RocketMQ-Proxy顺序部署，实例启动完成且成功后再部署下一个
# 注意部署RocketMQ-Broker的时候，要提高规格，我弄的是2核4GB的，不然可能出现内存不够的情况

其他和clickhouse无二，三个组件注意换成同一个rocketmq镜像，以下是每个组件在高级设置中 <启动命令> 和 <环境变量> 的区别————
  ·RocketMQ-NameServer:sh mqnamesrv 无环境变量，先启动，获得实例的临时ip地址 <ip_addr>
    -> The Name Server boot success...（日志）
  ·RocketMQ-Broker:sh mqbroker 环境变量：NAMESRV_ADDR == <ip_addr>:9876
    -> The broker[xxx] boot success...（日志）
  ·RocketMQ-Proxy:sh mqproxy 环境变量：NAMESRV_ADDR == <ip_addr>:9876
    -> <xxx> startup successfully...（日志）

# 如果出现'DefaultHeartBeatSyncerTopic'的error报错，就检查一下实例是不是只部署了一个，以及ip地址是否对应的上

5.在RocketMQ-Proxy的实例终端执行以下命令创建Topic（主题）和ConsumerGroup（消费者组）
  $./mqadmin updateTopic -n <ip_addr>:9876 -t TAH -c DefaultCluster
    create...success
  $./mqadmin updateSubGroup -n <ip_addr>:9876 -c DefaultCluster -g TAH_GROUP
    SubscriptionGroupConfig{...}

6.下载我github上的两个文件夹 <log-producer> 和 <log-consumer> ，在log-producer中的application.yml文件的地址改成
