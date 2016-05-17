#  内容简介
本案例演示了如何通过datafoundry平台提供的mysql后端服务来部署wordpress应用，部署流程如下：  
##  部署wordpress镜像  
为了能够让wordpress自动适配后端服务提供的环境变量，我们对dockerhub官方wordpress镜像进行了微调，先来部署wordpress
  ```
  oc new-app https://github.com/datafoundry/wordpress.git --context-dir=apache
--> Found Docker image 10e778c (12 days old) from Docker Hub for "library/php:5.6-apache"
    * An image stream will be created as "php:5.6-apache" that will track the source image
    * A Docker build using source code from https://github.com/datafoundry/wordpress.git will be created
      * The resulting image will be pushed to image stream "wordpress:latest"
      * Every time "php:5.6-apache" changes a new build will be triggered
    * This image will be deployed in deployment config "wordpress"
    * Port 80/tcp will be load balanced by service "wordpress"
      * Other containers can access this service through the hostname "wordpress"
    * WARNING: Image "wordpress" runs as the 'root' user which may not be permitted by your cluster administrator
--> Creating resources with label app=wordpress ...
    imagestream "php" created
    imagestream "wordpress" created
    buildconfig "wordpress" created
    deploymentconfig "wordpress" created
    service "wordpress" created
--> Success
    Build scheduled, use 'oc logs -f bc/wordpress' to track its progress.
    Run 'oc status' to view your app.
  ```  
查看部署结果  
  ```
  oc get pods
  NAME                 READY     STATUS             RESTARTS   AGE
  wordpress-1-build    0/1       Completed          0          3m
  wordpress-1-deploy   1/1       Running            0          1m
  wordpress-1-hfzhs    0/1       CrashLoopBackOff   3          1m
  ``` 
  因为没有提供mysql后端服务，wordpress一直在重启，我们现在来创建mysql backingservice     
##  查看datafoundry后端服务列表  
  　　在通过datafoundry平台生成一个MySQL的后端服务之前我们可以先查看一下目前datafoundry平台已经集成的后端服务  
  ```   
  oc get bs -n openshift  
  ```
  　　注意：   
    * 后端服务是datafoundry特有功能，所以必须要使用datafoundry客户端连接datafoundry服务端后查看;   
    * 在查看datafoundry平台已集成的后端时要添加后端服务默认的集成命名空间openshift;    
　　以上命令输出结果为：  
  ```   
  NAME         LABELS                           BINDABLE   STATUS
  Cassandra    asiainfo.io/servicebroker=etcd   true       Active
  ETCD         asiainfo.io/servicebroker=etcd   true       Active
  Greenplum    asiainfo.io/servicebroker=rdb    true       Active
  Kafka        asiainfo.io/servicebroker=etcd   true       Active
  MongoDB      asiainfo.io/servicebroker=rdb    true       Active
  Mysql        asiainfo.io/servicebroker=rdb    true       Active
  PostgreSQL   asiainfo.io/servicebroker=rdb    true       Active
  RabbitMQ     asiainfo.io/servicebroker=etcd   true       Active
  Redis        asiainfo.io/servicebroker=etcd   true       Active
  Spark        asiainfo.io/servicebroker=etcd   true       Active
  Storm        asiainfo.io/servicebroker=etcd   true       Active
  ZooKeeper    asiainfo.io/servicebroker=etcd   true       Active
  ```   
　　可以看到datafoundry平台已经集成非常丰富的后端服务组件，下面我们创建一个Mysql后端服务实例  
  
##  创建后端服务实例  
　　在创建实例之前，我们要先通过`oc describe bs <backingservcie-name> `获取应用所需的后端服务计划（plan），例如我们获取mysql后端服务的服务计划为：
  ```   
  oc describe bs Mysql -n openshift
  Name:			Mysql
Created:		21 hours ago
Labels:			asiainfo.io/servicebroker=rdb
Annotations:		<none>
Description:		A MYSQL DataBase
Status:			Active
Bindable:		true
Updateable:		false
documentationUrl:	http://docs.mysql.com
longDescription:	OpenSoure RDMBS Mysql
providerDisplayName:	asiainfoLDP
supportUrl:		http://www.mysql.com
displayName:		Mysql
────────────────────
Plan:		Experimental
PlanID:		56660431-6032-43D0-A114-FFA3BF521B66
PlanDesc:	share a mysql database in one instance
PlanFree:	true
Bullets:
  20 GB of Disk
  20 connections
PlanCosts:
  CostUnit:	MONTHLY
  Amount:
    eur: 49
    usd: 99
  CostUnit:	1GB of messages over 20GB
  Amount:
    eur: 0.49
    usd: 0.99

  ```
 　　我们选取Experimental计划创建mysql后端实例
   ```   
  oc new-backingserviceinstance mysql-inst1 \
  --backingservice_name=Mysql \
  --planid=56660431-6032-43D0-A114-FFA3BF521B66
  Backing Service Instance has been created.
  ```
  　　查看后端服务列表
  ```   
  oc get bsi
NAME          SERVICE   PLAN           BOUND     STATUS
mysql-inst1   Mysql     Experimental   0         Unbound 
  ```
  　　查看后端服务详细信息
  ```
  oc describe bsi mysql-inst1
Name:			mysql-inst1
Created:		5 minutes ago
Labels:			<none>
Annotations:		<none>
Status:			Unbound
DashboardUrl:		http://e412377c9b5f3db:1394e5f077d1519@phpmyadmin-service-broker-db.app.dataos.io?db=8c60f229ef4dac0
BackingServiceName:	Mysql
BackingServicePlanName:	Experimental
BackingServicePlanGuid:	56660431-6032-43D0-A114-FFA3BF521B66
Parameters:
instance_id:	340082e4-1734-11e6-a653-fa163edcfb45
Bound:		0
Events:
  FirstSeen	LastSeen	Count	From	SubobjectPath	Type		Reason		Message
  ---------	--------	-----	----	-------------	--------	------		-------
  5m		5m		1	{bsi }			Normal		Provisioning	bsi provisioning done, instanceid: 340082e4-1734-11e6-a653-fa163edcfb45
  ```
  　　以上服务实例创建完成，我们继续把mysql backingservice绑定到workdpress应用中
  ```
   oc bind mysql-inst1 wordpress
   oc env dc/wordpress MYSQLBSI=MYSQLINST1
  ```  
  　　查看部署结果,POD已正常启动，给wordpress生成一个route地址后就可以访问了
  ```
  oc get pods
  NAME                READY     STATUS      RESTARTS   AGE
  wordpress-1-build   0/1       Completed   0          7m 
  wordpress-2-55q5a   1/1       Running     0          41s
  ```  

