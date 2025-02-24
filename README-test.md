# 测试

## 查询cluster
集群名称company_cluster
```
select * from system.clusters;
```
## 只在当前节点创建本地数据库
```
create database test;
drop  database test;
```
## 在集群上创建本地数据库
```
drop  database test on cluster company_cluster ;
-- on cluster 会在指定集群中的所有节点下创建同样的数据库,执行后会反馈各节点的执行结果
create database test on cluster company_cluster ;
```
## 建本地表,在本地插入数据(on cluster 会在集群的各个节点上建表, 但是insert数据只会在当前节点)、
```
drop table test.cmtest on cluster company_cluster;
CREATE TABLE test.cmtest on cluster company_cluster
(
	`id` String COMMENT 'id',
	`nginxTime` DateTime COMMENT 'nginxTime'
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(nginxTime)
ORDER BY (nginxTime);
 ```
 ```
insert into test.cmtest  values ('1',now());
insert into test.cmtest  values ('10',now());
insert into test.cmtest  values ('100',now()+3600*24);
```
-- insert数据只会在当前节点查到
```
select * from test.cmtest ;
```
## 分布式表
-- 分布式引擎本身不存储数据, 但可以在多个服务器上进行分布式查询。
-- 分布式表(对分布式表的查询会查询到所有节点上的数据)
-- 建一个分布式表，与本地表关联
```
create TABLE test.cmtest_dist on cluster company_cluster as test.cmtest
ENGINE = Distributed("company_cluster", "test", "cmtest", rand());
```
-- 对本地insert只插入到本地
```
insert into test.cmtest  values ('100108',now()+3600*24);
```
-- 对分布式表插入会根据规则路由到某个节点
```
insert into test.cmtest_dist  values ('1004000',now()+3600*24);
```
-- 对分布式表的查询会查询到所有节点上的数据
```
select * from test.cmtest_dist;
```
-- 本地表只查询到当前节点上的数据
```
select * from test.cmtest;
```
-- 删除分布式表不会删除数据,重新创建分布式表后仍然可以查询到全量数据
```
drop table test.cmtest_dist on cluster company_cluster;
```
-- 删除本地表会删除数据
```
drop table test.cmtest on cluster company_cluster;
```
