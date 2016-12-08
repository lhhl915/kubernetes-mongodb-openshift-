# openshift kubernetes mongodb cluster(sharding replica-set)


==如果是openshift平台可直接使用，kubernetes平台的话，针对编排做相应的修改==

### 一、replica-set模式：
##### 1.build镜像

##### 1.1 使用源镜像build，可能因为网络原因build失败

```
cd replica-set
docker build -t registry.dataos.io/liuliu/mongo-replica-set:latest .
docker push registry.dataos.io/liuliu/mongo-replica-set:latest  #push到私有镜像库
```
##### 1.2 直接使用以下镜像，如果build失败

```
# 构建好的镜像
registry.dataos.io/liuliu/mongo-replica-set:latest
```
##### 1.3 再或者使用以下修改过镜像源的Dockerfile

```
cd replica-set
docker build -f Dockerfile-repair -t registry.dataos.io/liuliu/mongo-replica-set:latest .
docker push registry.dataos.io/liuliu/mongo-replica-set:latest  #push到私有镜像库

```
 
##### 2.创建持久化卷

```
mongo-test1 # 例子
mongo-test2 # 例子 
mongo-test3 # 例子
```

##### 3.修改mongo-replica-rs1.yaml中镜像地址(改成dockerfile的镜像地址)和持久化卷名称

##### 4.创建mongodb replica-set编排，以下选其一即可。

```
oc create -f mongo-replica-rs-passwd.yaml  #持久化卷+密码验证

oc create -f mongo-replica-rs1.yaml  #持久化卷

oc create -f mongo-replica-not-storage.yaml  #未持久化
```
#### 以下是三种测试方案：

##### 4-1.直接每个pod登录测试
```
oc rsh <podID1> bash
oc rsh <podID2> bash
oc rsh <podID3> bash
```

##### 4-2.创新一个mongo客户端进行测试
```
oc create -f mongo-client.yaml

oc rsh <podID> bash

mongo -u<mongodb_user> -p<mongodb_passwd> --host my_replica_set/mongo-replica-node-1:27017,mongo-replica-node-2:27017,mongo-replica-node-3:27017 admin     #连接测试replica-set
```

##### 4-3. 下面我就以node.js利用rrestjs框架 和 node-mongodb-native 模块进行mongodb副本集的操作（未实验）

https://github.com/christkv/node-mongodb-native/blob/master/docs/replicaset.md


##### 5.会使用到的测试命令：

```
#先进入设置账号密码的库
use admin;
#创建账号密码
db.createUser({user:'mongo',pwd:'mongodbpass',roles:['userAdminAnyDatabase','dbAdminAnyDatabase']})
#查看创建的用户：
show users;

#创建测试库及其账号：
use test
db.createUser(
   {
       user: "test1",
       pwd: "12345678",
       roles: [ { role: "readWrite", db: "test" } ]
     }
 )

#插入数据
db.test.insert({Name: "test"})
db.test.find();

# 连接从节点时，需要先
rs.slaveOk();

#如果设置了密码，需要验证一下
db.auth('mongo','mongodbpass')

```

### 二、mongodb sharding模式
##### 1.编排描述：
1.1 按照以下步骤将会起11个pod，其中shard部分两个replica-set副本集占6个，config部分3个pod（也是replica-set模式），route部分2个pod

1.2 架构图（以下操作流程只起shard1和shard2）
![image](https://github.com/asiainfoLDP/mongodb-cluster/blob/master/mongodb-sharding图示.png)

1.3 新建三个dockerfile并替换相应文件的镜像源，例子：

```
关键镜像：
registry.dataos.io/liuliu/mongod-replica-set:3.2.10    #replica-set副本集部分
registry.dataos.io/liuliu/mongos:3.2.10          #mongos route部分
registry.dataos.io/liuliu/mongodb-configsvr:3.2.10   #config部分

```

1.4 持久化卷占用(GlustFS),以下为卷名（创建略）:

```
mongo-conf-storage-1 --    
mongo-conf-storage-2   |   ----> config节点 /data/configdb 配置文件目录            
mongo-conf-storage-3 --


mongo-config-db-1 --    
mongo-config-db-2   |   ----> config节点 /data/db 数据库目录            
mongo-config-db-3 --


mongo-storage1-1 --       
mongo-storage1-2   |    ----> shard1 /data/db 目录
mongo-storage1-3 --


mongo-storage2-1 --       
mongo-storage2-2   |    ----> shard2 /data/db 目录  
mongo-storage2-3 --
```

##### 2.创建config配置节点（replica-set模式）

```
oc create -f mongo-configsvr.yaml  #执行此步需要等待1分钟左右，等待三个pod同时启动、初始化mongodb数据库、设置replica-set和设置configsvr，之后再做下一步操作。
```

##### 3. 创建路由节点，用于对mongodb群集的访问（负载均衡模式）

```
oc create -f mongo-route-with-SLB.yaml 
```
##### 4. 创建两个shard分片群集（replica-set模式）
```
oc create -f mongo-replica-rs1.yaml   #shard1
oc create -f mongo-replica-rs2.yaml   #shard2

# 等待以上6个pod正常启动后，等待1分钟初始化mongodb群集，之后进行以下操作
```

##### 5. 添加shard1 和 shard2 到 sharding群集
```
oc rsh <MONGO_ROUTE_POD:ID> bash   # 登录任意一个route节点

mongo   #登录mongo数据库

mongos> sh.addShard("my_replica_set1/mongo-replica-nodea-0:27017"); #添加shard1

mongos> sh.addShard("my_replica_set2/mongo-replica-nodeb-1:27017"); #添加shard2

```

##### 6.将test库设置成分片方式

```
use admin

db.runCommand({enablesharding:"test"});    #使能test库分片
sh.enableSharding("test");   #同上，执行其一即可

sh.shardCollection("test.users", { "_id": "hashed" })    #test.users表进行分片处理
db.runCommand({shardcollection:"test.users",key:{_id:1}})    #同上，执行其一即可

sh.status()    #分片群集状态查询

```

##### 7.验证Sharding 正常工作

7.1 数据插入测试

```
use test
for(var i=1;i<2000;i++)db.users.insert({id:i,addr_1:"Beijing",addr_2:"Shanghai"});
db.users.stats()  #查看是否分片
db.users.find()  #查看所有数据
it   #显示更多

test.test1插入100条数据
sh.shardCollection("test.test1", { "_id": "hashed" })
for(var i=1;i<100;i++)db.test1.insert({id:i,addr_1:"Beijing",addr_2:"Shanghai"});
db.test1.stats()  #查看是否分片
db.test1.find()  #查看所有数据
it   #显示更多

```

7.2 其他验证方法：
分别登录shard1和shard2测试数据是否分片存储
```
mongo

use test
db.test1.find() 
it
```

8.3 持久化存储验证：

```
oc delete pod `oc get pod | grep 'mongo-config' | awk '{print $1}'` #删除配置节点


```

##### 8.问题处理：
1 如果出现添加shard1 和shard2 无法添加到群集的现象：
```
oc delete pod `oc get pod | grep 'mongo-replica-rc'| awk '{print $1}' `    # 删除shard 的pod 重新生成即可
```

2 configsvr 部分：

```
config节点需要将以下两个路径持久化
/data/configdb
/data/db
不要直接持久化/data/，因为属主属组等问题，会导致数据不能持久化。
```

3 如果涉及的服务节点中所有的pod全部删除，服务无法正常启动，将两个shard1和shard2重启即可。

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

