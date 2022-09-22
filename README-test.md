# 启动一个clickhouse-server容器，获取clickhouse-server的配置文件，操作完可以删除该容器
docker run -d --name clickhouse-server --ulimit nofile=262144:262144 yandex/clickhouse-server
#复制容器内的配置文件到宿主机，这里拷贝到/etc目录下
docker cp clickhouse-server:/etc/clickhouse-server/ .

# 测试
```
-- 查询cluster集群名称 比如cluster_3shard_1replica
select * from system.clusters;
-- 只在当前节点创建本地数据库
create database test;
drop  database test;
--company_cluster 是集群名称
drop  database test on clustercompany_cluster ;
-- on cluster 会在指定集群中的所有节点下创建同样的数据库,执行后会反馈各节点的执行结果
create database test on clustercompany_cluster ;

-- 建本地表(on cluster 会在集群的各个节点上建表, 但是insert数据只会在当前节点)
drop table test.cmtest on clustercompany_cluster;
CREATE TABLE test.cmtest on clustercompany_cluster
(
	`id` String COMMENT 'id',
	`nginxTime` DateTime COMMENT 'nginxTime'
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(nginxTime)
ORDER BY (nginxTime);
 
insert into test.cmtest  values ('1',now());
insert into test.cmtest  values ('10',now());
insert into test.cmtest  values ('100',now()+3600*24);
-- insert数据只会在当前节点查到
select * from test.cmtest ;
-- 分布式引擎本身不存储数据, 但可以在多个服务器上进行分布式查询。
-- 分布式表(对分布式表的查询会查询到所有节点上的数据)
create TABLE test.cmtest_dist on clustercompany_cluster as test.cmtest
ENGINE = Distributed("cluster_3shard_1replica", "test", "cmtest", rand());

-- 对本地insert只插入到本地
insert into test.cmtest  values ('100108',now()+3600*24);
-- 对分布式表插入会根据规则路由到某个节点
insert into test.cmtest_dist  values ('1004000',now()+3600*24);
-- 对分布式表的查询会查询到所有节点上的数据
select * from test.cmtest_dist;
-- 本地表只查询到当前节点上的数据
select * from test.cmtest;

-- 删除分布式表不会删除数据,重新创建分布式表后仍然可以查询到全量数据
drop table test.cmtest_dist on clustercompany_cluster;
-- 删除本地表会删除数据
drop table test.cmtest on clustercompany_cluster;
```
