## 介绍存储卷以及容器之间如何管理数据的方法。
就像在容器中运行一个数据库程序一样，你可以将软件打包在镜像中，当启动容器时，它将初始化空的DB，当其他程序接入该DB时并存入数据，这些数据要如何才能保存下来？暂停一个容器或删除一个容器，这些数据要怎么办？升级数据库程序，如何迁移？

### 存储卷简介
一个主机或容器的目录树是由一组挂载点创建而成，这些挂载点描述了如何能构建出一个或多个文件系统。![](/assets/Snip20190305_1.png)

#### 存储卷提供容器无关的数据管理方式
存储卷是一个数据分割和共享的工具，有一个与容器无关的范围或生命周期。这使得存储卷成为容器化系统设计中关于文件共享或写入最重要的一部分。数据示例根据其范围或者接入容器方式的不同分为以下几种：
- 数据库软件与数据库中的数据
- web应用程序与日志数据
- web数据处理应用程序的输入和输出数据
- web服务器与静态内容
- 产品与支持工具

镜像适合打包和分发相对静态的文件，如程序；存储卷则持有动态或专门数据。这种区别使得镜像可重用，数据可简单分享。

更为基本的是：存储卷可以隔离应用程序和主机的关系。镜像被装载到主机，创建出一个容器。Docker不知道主机在哪里运行，只能判断哪些文件在容器中可用。也就是Docker本身就没有办法利用主机上的设施，如装载的网络存储、混合光纤、固态硬盘。但有主机知识的用户可以使用存储卷，在容器中将这些目录映射到主机的存储上。

#### NoSQL数据库使用存储卷
Apache Cassandra项目提供了一个具有内置集群，最终一致性和线性写入可伸缩的列数据库。下面使用官方的Cassandra镜像创建一个节点集群，并创建一个键空间，删除容器，然后再另一个容器中恢复这个新节点的键空间。

创建已定义存储卷的单个容器，这也被称为存储卷容器（高级模式）
```sh
docker run -d --volume /var/lib/cassandra/data --name cass-shared alpine echo Data Container
```
存储卷容器将立即停止。不要删除他。会在后面创建运行Cassandra新容器时，使用这个存储卷：
```sh
docker run -d --volumes-from cass-shared --name cass1 cassandra:2.2
```
Docker下载完镜像，创建一个新容器，并复制存储卷容器的卷定义。然后容器的存储卷挂载在/var/lib/cassandra/data。指向主机目录树相同的位置。接下来，从cassandra:2.2镜像启动容器，运行Cassandra客户端工具，并连接到正在运行的服务器：
```sh
docker run -it --rm --link cass1:cass cassandra:2.2 cqlsh cass
```
接下来就可以从CQLSH命令行检查或修改Cassandra数据库。首先查找一个`docker_hello_world`的键空间
```sql
 select * from system.schema_keyspaces where keyspace_name = 'docker_hello_world';
```

Cassandra应该返回的是`(0 rows)`的空列表，下面使用命令进行创建
```sql
create keyspace docker_hello_world with replication = { 'class':'SimpleStrategy', 'replication_factor':1 };
```



















