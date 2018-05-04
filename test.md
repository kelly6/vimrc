# 架构图
## 任务处理架构
```
graph TD
frontend-->redis
frontend-->mysql
redis-->manager
mysql-->manager
manager-->zk
zk-->worker
zk-->filters
zk-->router
worker-->w1
worker-->w...
worker-->wn
filters-->f1
filters-->f...
filters-->fn
router-->r1
router-->r...
router-->rn
```

---
## 入库服务
```
graph TD
storage-->s1
storage-->s...
storage-->sn

```

---
## 插件微服务
```
graph TD
plugins_service-->plugin-m
plugins_service-->plugin-*
plugins_service-->plugin-n

```
# 服务器地址
## mysql
    账号:zhxg 密码: ZHxg2018!
    vip 192.168.32.53 rip1: 192.168.32.42 rip2:192.168.32.43
## nginx  
    http://60.29.201.145:8280/
    vip:192.168.32.82 rip1: 192.168.32.44 rip2:192.168.32.45
## redis
    vip:192.168.32.54 主:192.168.32.44 从:192.168.32.45
## docker nodes
   leader: 192.168.32.42 nodes:192.168.32.43  192.168.32.44 192.168.32.45

# 服务名词
## worker
    负责查询数据，需要部署在大硬盘机器(docker global模式)
    日志路径: /home/work/worker/worker/log
## filters
    数据处理模块，需要部署在大内存机器
## router
    数据路由模块，如对外网提供服务需要部署在能访问公网的机器
## storage
    入库模块，负责将数据写入数据平台，需要跟worker配对部署(docker global 模式)
## frontend
    前后台WEB服务，需要暴露端口，对外提供服务
## acquisition
    数据源收集程序，用来收集不同数据源配置的数据到数据平台
## file
    提供文件下载服务
## manager
    任务管理模块，每个节点(NODE_ID)只起一个
    

---

# 镜像列表:

## filters:
    docker service create --name filters_n1 --limit-cpu=4 -e NODE_ID=1 --mount type=bind,source=/mfs,target=/mfs --replicas 12 -t docker-registry.istarshine.net.cn:5000/xgsj3/alpha/filters:V3.0.63
## router:
    docker service create --name router_n1 --limit-cpu=2 -e NODE_ID=1 --mount type=bind,source=/mfs,target=/mfs --replicas 12 -t docker-registry.istarshine.net.cn:5000/xgsj3/alpha/router:V3.0.76
    #启动之后会在zookeeper  /router创建临时节点
## worker:
    docker service create --name worker_n1 --limit-cpu=30 -e NODE_ID=1 --mount type=bind,source=/data1,target=/data1 --mount type=bind,source/mfs,target=/mfs --mode global -t docker-registry.istarshine.net.cn:5000/xgsj3/alpha/worker:V3.0.61
    #启动之后会在zookeeper  /task创建临时节点
## storage:
    docker service create --name storage -e NODE_ID=1 --mount type=volume,target=/data1,source=/data1 --mount type=volume,source=/data3,target=/data3 --mount type=bind,source=/mfs,target=/mfs --mode global -t docker-registry.istarshine.net.cn:5000/xgsj3/alpha/storage:V3.0.77
## frontend:
    docker service create --name frontend -e NODE_ID=1 -e ZK_SEV_LIST=192.168.224.13:2181,192.168.224.14:2181,192.168.224.15:2181 -p 80:8080 -v /mfs:/mfs -t docker-registry.istarshine.net.cn:5000/xgsj3/alpha/frontend:V3.0.176
## acquisition:
    docker service create --name acquisition --mount type=bind,source=/mfs,target=/mfs --replicas 4 -t docker-registry.istarshine.net.cn:5000/xgsj3/alpha/acquisition:V3.0.27
## file:
    docker service create --name file -replicas 2 -p 8050:8080 -v /mfs:/mfs docker-registry.istarshine.net.cn:5000/xgsj3/alpha/file:V3.0.29
## manager:
    docker service create --name manager_n1 -e NODE_ID=1 --mount type=bind,source=/mfs,target=/mfs --replicas 1 -t docker-registry.istarshine.net.cn:5000/xgsj3/alpha/manager:V3.0.61

## docker镜像启动参数(worker, filter, router):
    -e NODE_ID=1 //集群编号
    -e SELF_ID=0 //worker编号,如果没有rocksdb服务,或者没有设置值,必须手动指定
    -e ZK_SEV_LIST=192.168.224.7:2181,192.168.224.8:2181,192.168.224.9:2181

---

# 配置
## gaea_cfg_zk.py
    设置mysql zookeeper redis等地址 其余配置不用改动
    设置环境变量ZK_SEV_LIST=192.168.224.1:2181,192.168.224.2:2181,192.168.224.3:2181
    python gaea_cfg_zk.py set
    
    data_que_url:"redis://192.168.1.1/0"  #数据流地址(worker==>filters==>router)
    worker_id_list=["192.168.1.1", "192.168.1.2", "192.168.1.3"] #worker机地址列表
    job_redis_url="redis://192.168.1.1/0"  #任务信息redis地址
    mysql_host = "mysql-xgsj-node.istarshine.net.cn"  #mysql地址
    mysql_port = 3306
    mysql_user = "root"
    dbname = "gaea"
    task_redis_url = "redis://192.168.1.1/0"
    
## config.properties
    redis.host=192.168.1.1
    redis.port=6379
    dayRedis.host=192.168.1.1
    dayRedis.port=6379
    dataRedis.host=192.168.1.1
    dataRedis.port=6379
    zookeeper.cluster=192.168.1.1:2181,192.168.1.2:2181,192.168.1.3:2181
    kafka.cluster=192.168.1.1:9092,192.168.1.2:9092,192.168.1.3:9092
    zmq.num=3
    zmq.host=192.168.1.4,192.168.1.5,192.168.1.6

## application.properties
    jdbc.url=jdbc\:mysql\://mysql-xgsj-node1.istarshine.net.cn\:3306/gaea?useUnicode\=true&characterEncoding\=utf-8&zeroDateTimeBehavior\=convertToNull&allowMultiQueries\=true
    jdbc.username=root
    jdbc.password=123456
    
## config.properties配置文本
	login_type=true
	is_send_validate_code=true
	is.dingding.notify=true
	dingding.notify.url=https://oapi.dingtalk.com/robot/send?access_token=5e8489b79a1213b5bd65c03d209030246872b84add0dc7bcd9062a929e27b416
	forbidEmail=@qq.com,163.com,@yahoo.com,@sina.com,@sina.cn,@sohu.com,@sundns.com,@sogou com,@gmail.com

	redis.pool.maxActive=500
	redis.pool.maxIdle=100
	redis.pool.minIdle=10
	redis.pool.maxWaitMillis=20000
	redis.pool.maxWait=300
	redis.timeout=100000000
	redis.host=192.168.32.54
	redis.port=6379
	redis.session.db=9
	redis.machinestatuskey = gaea_java_machine_status
	redis.redisTaskKeyLow = gaea_java_task_list
	redis.redisTaskKeyMiddle = gaea_java_task_list_middle
	redis.redisTaskKeyHigh = gaea_java_task_list_high
	redis.ipCountLimit = gaea_java_ipcount_limit
	redis.key.commentData=comment_data_all
	ipCountLimitNum = 10
	ipCountLimitExpireTime = 60
	#realtime
	redis_real_time_host = 192.168.32.54
	#redis_real_time_host =redis://alpha-redis-xgsj3cache-1.istarshine.net.cn
	#putongzhuanti
	redis_real_time_port = 6379
	redis_real_time_db_putong = 10
	redis_real_time_key_publish2 = xgsj3_uw_update
	redis_real_time_key_zset2 = xgsj3_uw_update_time2subjectid
	redis_real_time_key_hset2 = xgsj3_uw_update_subject

	#gaojizhuanti
	redis_real_time_db_gaoji = 11
	redis_real_time_key_publish = xgsj3_ul_update
	redis_real_time_key_zset = xgsj3_ul_update_time2subjectid
	redis_real_time_key_hset = xgsj3_ul_update_subject

	#zhuantifilter mark plugin 
	redis_real_time_db_filter = 0
	redis_real_time_key_publish_filter = rt_filter_cfg_notify
	redis_real_time_key_zset_filter = rt_filter_cfg_zset
	redis_real_time_key_hset_filter = rt_filter_cfg_hash

	#redis.password=zhxg2016!

	##type 1 compress by size,2 compress by percent
	img.compress.type=1
	img.compress.width=88
	img.compress.hight=88
	img.compress.percent=0.25f

	#dayRedis.host = 192.168.184.214
	dayRedis.host = 192.168.32.54
	dayRedis.port = 6379
	dayRedis.dbSize = 0
	dayRedis.keystr = gaea_stat_hash_infoflag_
	dataRedis.host = 192.168.32.54
	dataRedis.port = 6379

	#zookeeper.cluster=192.168.141.31:2181,192.168.141.32:2181,192.168.141.33:2181
	#kafka.cluster=192.168.141.31:9092,192.168.141.32:9092,192.168.141.33:9092
	zookeeper.cluster=192.168.32.42:2181,192.168.32.43:2181,192.168.32.44:2181
	kafka.cluster=192.168.224.13:9092,192.168.224.14:9092,192.168.224.15:9092,192.168.224.16:9092,192.168.224.17:9092,192.168.224.18:9092,192.168.224.19:9092,192.168.224.20:9092,192.168.224.21:9092,192.168.224.22:9092,192.168.224.23:9092
	kafka.logPath=gaea_log

	##zmq info
	zmq.num=12
	zmq.host=192.168.32.43,192.168.32.44,192.168.32.45
	zmq.heartbeat=60

	#result.url.excel=http://192.168.141.31:90/file
	result.url.excel=http://xgsj.istarshine.com/file
	trun.url.excel=http://192.168.224.1:8080/db_2_xlsx

	mail.host=smtp.exmail.qq.com
	mail.user=mailcenter@yqzbw.com
	mail.uname=mailcenter@yqzbw.com
	mail.pwd=qybmail001

	spider.comments=http://ispider.istarshine.com/comments/api/weibo/comments/
	spider.datas=http://ispider.istarshine.com/comments/api/weibo/datas/
	spider.comments.common=http://ispider.istarshine.com/comments/api/now/spider/
	spider.wechat.read_comments=http://ispider.istarshine.com/wechat/api/read_comments/

	search.userId=221935599093760_306788307044352_258990052804608
	search.isEmail=true
	#total station crawler redis info
	redis.crawler.confpath=1
	redis.crawler.host=192.168.187.12
	redis.crawler.key=list_session_task_key
	redis.crawler.port=6379
	redis.crawler.db=4

	url.crawler.visual=http://webspider.istarshine.com/webSpider

	# redis or rocketmq sendMessage
	log.type=redis
	redis.log.key=gaea_log
	redis.log.host=192.168.32.54
	redis.log.port=6379
	redis.log.db=0
	rocketmq.log.addr=192.168.141.10:9876;192.168.141.12:9876;192.168.141.14:9876;192.168.141.16:9876
	rocketmq.log.group=gaea_log_group
	rocketmq.log.producer.instance=producer
	rocketmq.log.consumer.instance=consumer
	rocketmq.log.topic=gaea_log

	weibo.userinfo.schema=-3
	ecom.comments.schema=359802973046784
	weibo.comments.schema=349742418413568
	common.comments.schema=270070116829184

	#ecom comments v1 api redis
	redis.comments.ecom.host=192.168.187.24
	redis.comments.ecom.db=7
	redis.comments.ecom.key.list=comments_ecom_list
	redis.ecom.key.list=ecom_list

	##in storage redis
	storage.redis.cluster=false
	storage.redis.host=192.168.32.45
	storage.redis.port=6379
	storage.redis.db=15
	storage.redis.timeout=10000

	## secretary  config
	## db
	secretary.db.ip=192.168.19.199
	secretary.db.port =3306
	secretary.db.username =ruser
	secretary.db.password =jbymydyc3r
	secretary.db =yqms2
	## redis
	secretary.redis.ip=192.168.184.124
	secretary.redis.port=6379
	secretary.redis.db=5
	secretary.redis.key=common_data
	## url
	secretary.url.open_db=http://192.168.224.1:8888/open_db
	secretary.url.fetch=http://192.168.224.1:8888/fetch

	##search schema Redis
	search.schema.redis=redis://192.168.184.124/5/common_data

## application.properties
	#mysql database setting
	jdbc.driver=com.mysql.jdbc.Driver
	jdbc.url=jdbc\:mysql\://192.168.32.53\:3306/gaea?useUnicode\=true&characterEncoding\=utf-8&zeroDateTimeBehavior\=convertToNull&allowMultiQueries\=true
	jdbc.username=zhxg
	jdbc.password=ZHxg2018!
	validationQuery=select 1

	#EhCache  

	#Druid
	druid.initialSize=10
	druid.minIdle=5
	druid.maxActive=150
	druid.maxWait=60000
	druid.timeBetweenEvictionRunsMillis=60000

	#C3P0
	c3p0.acquireIncrement=10
	c3p0.initialPoolSize=15
	c3p0.minPoolSize=30
	c3p0.maxPoolSize=120
	c3p0.maxIdleTime=30
	c3p0.idleConnectionTestPeriod=30
	c3p0.maxStatements=0
	c3p0.numHelperThreads=100
	c3p0.checkoutTimeout=0
	c3p0.validate=true

	#BoneCP  
	bonecp.minPoolSize=5
	bonecp.maxPoolSize=20
	bonecp.maxIdleTime=1800
	bonecp.idleConnectionTestPeriodInMinutes=240
	bonecp.maxStatements=0
	bonecp.idleMaxAgeInMinutes=240
	bonecp.maxConnectionsPerPartition=30
	bonecp.minConnectionsPerPartition=5
	bonecp.partitionCount=3
	bonecp.acquireIncrement=5
	bonecp.statementsCacheSize=50
	bonecp.releaseHelperThreads=2
	bonecp.disableConnectionTracking=true

	#dbcp settings
	dbcp.maxIdle=5
	dbcp.maxActive=40


	history.driver=com.mysql.jdbc.Driver
	history.url=jdbc:mysql://192.168.32.53:3306/gaea
	#history.url=jdbc:mysql://192.168.149.31:3306/bigdata
	history.database=jdbc:mysql://192.168.32.53:3306/gaea
	#history.database=jdbc:mysql://192.168.149.31:3306/bigdata
	history.username=root
	history.password=ZHxg2018!


	config.driver=com.mysql.jdbc.Driver
	config.url=jdbc:mysql://192.168.16.199:3306/yqht
	config.database=jdbc:mysql://192.168.16.199:3306/yqht
	config.username=root
	config.password=a1d2g3j4l5
	
## 部署后检查
    登录后台： http://127.0.0.1/admin/login.html
    查看"工作机状态" 绿色为空闲 黄色为忙碌 红色为宕机
    
## 更新
    worker|filter|router更新
    找到配置的redis 执行：
    set job_stop_n1 1
    这个命令使节点1停止接收新任务（其余节点以此类推），然后可以安全地更新docker
    运行完之后观察后台"工作机状态"，全为绿色即为停止了， 否则是还在执行当前任务，需要等待全变绿色之后再升级

