---
layout: post
title: HDP2.2でResourceManagerが両系standbyになった事象
date: 2015-01-27 17:45:06 +0900
comments: true
categories: [Hadoop]
published: true
---

HDP2.2で、daemonやクラスタの再起動を繰り返していた所、ResourceManagerが両系standbyになってしまった.

ResourceManagerのログには以下のように出力されている.

```
2015-01-16 16:29:19,495 WARN  resourcemanager.RMAuditLogger
(RMAuditLogger.java:logFailure(285)) - USER=yarn
OPERATION=transitionToActive    TARGET=RMHAProtocolService
RESULT=FAILURE  DESCRIPTION=Exception transitioning to active
PERMISSIONS=All users are allowed
2015-01-16 16:29:19,495 WARN  ha.ActiveStandbyElector
(ActiveStandbyElector.java:becomeActive(809)) - Exception handling the
winning of election
org.apache.hadoop.ha.ServiceFailedException: RM could not transition to Active
        at org.apache.hadoop.yarn.server.resourcemanager.EmbeddedElectorService.becomeActive(EmbeddedElectorService.java:128)
        at org.apache.hadoop.ha.ActiveStandbyElector.becomeActive(ActiveStandbyElector.java:805)
        at org.apache.hadoop.ha.ActiveStandbyElector.processResult(ActiveStandbyElector.java:416)
        at org.apache.zookeeper.ClientCnxn$EventThread.processEvent(ClientCnxn.java:599)
        at org.apache.zookeeper.ClientCnxn$EventThread.run(ClientCnxn.java:498)
Caused by: org.apache.hadoop.ha.ServiceFailedException: Error when
transitioning to Active mode
        at org.apache.hadoop.yarn.server.resourcemanager.AdminService.transitionToActive(AdminService.java:304)
        at org.apache.hadoop.yarn.server.resourcemanager.EmbeddedElectorService.becomeActive(EmbeddedElectorService.java:126)
        ... 4 more
Caused by: org.apache.hadoop.service.ServiceStateException:
org.apache.hadoop.yarn.exceptions.YarnException: Application with id
application_1421115867116_0001 is already present! Cannot add a
duplicate!
        at org.apache.hadoop.service.ServiceStateException.convert(ServiceStateException.java:59)
        at org.apache.hadoop.service.AbstractService.start(AbstractService.java:204)
        at org.apache.hadoop.yarn.server.resourcemanager.ResourceManager.startActiveServices(ResourceManager.java:1014)
        at org.apache.hadoop.yarn.server.resourcemanager.ResourceManager$1.run(ResourceManager.java:1051)
        at org.apache.hadoop.yarn.server.resourcemanager.ResourceManager$1.run(ResourceManager.java:1047)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:415)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1628)
        at org.apache.hadoop.yarn.server.resourcemanager.ResourceManager.transitionToActive(ResourceManager.java:1047)
        at org.apache.hadoop.yarn.server.resourcemanager.AdminService.transitionToActive(AdminService.java:295)
        ... 5 more
Caused by: org.apache.hadoop.yarn.exceptions.YarnException:
Application with id application_1421115867116_0001 is already present!
Cannot add a duplicate!
        at org.apache.hadoop.yarn.ipc.RPCUtil.getRemoteException(RPCUtil.java:45)
        at org.apache.hadoop.yarn.server.resourcemanager.RMAppManager.createAndPopulateNewRMApp(RMAppManager.java:338)
        at org.apache.hadoop.yarn.server.resourcemanager.RMAppManager.recoverApplication(RMAppManager.java:309)
        at org.apache.hadoop.yarn.server.resourcemanager.RMAppManager.recover(RMAppManager.java:413)
        at org.apache.hadoop.yarn.server.resourcemanager.ResourceManager.recover(ResourceManager.java:1207)
        at org.apache.hadoop.yarn.server.resourcemanager.ResourceManager$RMActiveServices.serviceStart(ResourceManager.java:590)
        at org.apache.hadoop.service.AbstractService.start(AbstractService.java:193)

```

Zookeeperに変なエントリが残ってる？と思われたのでRMを停止した上で`/rmstore/ZKRMStateRoot/RMAppRoot/`以下のApplicatoinIDのエントリを全て削除.

```
[zk: localhost:2181(CONNECTED) 2] rmr /rmstore/ZKRMStateRoot/RMAppRoot/application_1421387925857_0002
[zk: localhost:2181(CONNECTED) 3] rmr /rmstore/ZKRMStateRoot/RMAppRoot/application_1421115867116_0002
[zk: localhost:2181(CONNECTED) 4] rmr /rmstore/ZKRMStateRoot/RMAppRoot/application_1421115867116_0003
[zk: localhost:2181(CONNECTED) 5] rmr /rmstore/ZKRMStateRoot/RMAppRoot/application_1421115867116_0004
[zk: localhost:2181(CONNECTED) 6] rmr /rmstore/ZKRMStateRoot/RMAppRoot/application_1421320519530_0002
[zk: localhost:2181(CONNECTED) 7] rmr /rmstore/ZKRMStateRoot/RMAppRoot/application_1421115867116_0005
[zk: localhost:2181(CONNECTED) 8] rmr /rmstore/ZKRMStateRoot/RMAppRoot/application_1421320519530_0001
[zk: localhost:2181(CONNECTED) 11] rmr /rmstore/ZKRMStateRoot/RMAppRoot/application_1421387925857_0001

```

で、RMを起動したら復旧した.

どうやら[YARN-2865](https://issues.apache.org/jira/browse/YARN-2865)の事象っぽい. Hadoop 2.7.0でFIX. ZK上のエントリを消してしまったので、その分のジョブについてはクライアントから再投入する必要がある.
