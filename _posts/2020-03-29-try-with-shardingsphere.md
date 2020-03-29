---
layout:     post
title:      "初识ShardingSphere"
subtitle:   "尝试目前最流行的数据库中间件之一"
date:       2020-03-29 12:00:00
author:     "Mrproliu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 数据库
---

## 初次了解

在几年前刚开始看开始了解到分库分表的时候，就了解到了当当网的Sharding-JDBC，专注于数据库层访问。当时主要解决的数据库分片，现在已经变成了ShardingSphere生态圈，并且已经加入了apache，再不认真学习一下就追随不了时代的潮流了。

去年年末，有幸参与了一次开源社区组织活动，更加加深了对ShardingSphere的了解。并且我之前的一位同事还参与到了他们社区，成为Committer。正好最近也有一些自己的时间，准备好好研究一下这个框架，也是我首次写Blog，就当做记录吧。

目前ShardingSphere总共分为三大组件，各个组件之间相互独立。这里也欢迎大家去[官网查看说明](https://shardingsphere.apache.org/document/current/cn/overview/)，我这里只是简单带过。

* Sharding-JDBC: 我感觉也是使用最多的，以嵌入应用的形式的操作集群数据库的框架。
* Sharding-Proxy: 我个人感觉和MyCat类似，提供专门的应用程序，统一对分布式数据库中操作，对应用程序是完全透明的。操作这个应用就像在操作单机一样简单。
* Sharding-Sidecar: 我对其的了解不是很多，但是从一些资料上来看，类似于基于云原声的组件。通过每个节点中的数据层组件去进行代理操作真实的数据库。

## 快速启动

说了那么多，还是快速上手体验一下吧。demo搭建的是基于当下的最新RELEASE版本: 4.0.1。体验的项目为Sharding-JDBC。

### 申请MYSQL机器

为了进行测试，我选择在阿里云上申请两台可以公网访问的MYSQL，用于当做测试环境。

![MYSQL配置](/img/in-post/try-with-shardingsphere/108857709208782.png)

### 编写项目

这里主要是创建了一个订单表，并对其进行插入和查询操作。利用两台MYSQL实例来实现分库逻辑。

|字段|类型|说明|
|---|---|----|
|order_id|bitint|订单ID，用作主键|
|user_id|int|用户编号，这里默认按照用户来进行分库操作|
|amount|int|金额(元)|

```sql
CREATE TABLE `t_order` (
  `order_id` bigint(20) NOT NULL COMMENT '主键',
  `user_id` int(11) DEFAULT NULL COMMENT '用户id',
  `amount` int(11) DEFAULT NULL COMMENT '金额(元)',
  PRIMARY KEY (`order_id`)
) ENGINE=InnoDB DEFAULT COMMENT='用户订单';
```

这里我们通过`YAML`文件的方式来进行配置，通过这种方式比java代码配置的方式更直观，而且不用引入第三方框架更快运行。

我们在配置中将上面创建的节点信息进行录入，并且按照用户id进行分库，规则是取余2，如果为0则入第一个库的`t_order`表，为1则入第二个库中的`t_order`表。

```yaml
# 两台主机的数据源信息,两个数据节点
dataSources:
  ds0: !!com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://vm1-host:3306/shardingtest?serverTimezone=UTC&useSSL=false&useUnicode=true&characterEncoding=UTF-8
    username: shardingtest
    password: xxxxxx
  ds1: !!com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://v2-host:3306/shardingtest?serverTimezone=UTC&useSSL=false&useUnicode=true&characterEncoding=UTF-8
    username: shardingtest
    password: xxxxxx

# 路由配置
shardingRule:
  # 表配置
  tables:
    t_order:
      # 在进行处理时的数据节点选择
      actualDataNodes: ds${0..1}.t_order
      # 数据节点的选择
      databaseStrategy:
        inline:
          # 按照用户id进行选择数据库
          shardingColumn: user_id
          algorithmExpression: ds${user_id % 2}
      # order_id主键通过SNOWFLAKE算法进行生成
      keyGenerator:
        type: SNOWFLAKE
        column: order_id

  # 默认数据源
  defaultDataSourceName: ds0
```

在了解了这个配置后，我们再启动程序进行真正的数据库操作。

这里面我们读取该配置文件，根据用户id创建多个订单信息，让其创建数据至这两个数据节点中。并且最后通过用户id来对数据进行查询。

```java
// 构建数据源
final URL resources = ClusterShardingSimpleDemo.class.getResource("/cluster-sharding.yaml");
final DataSource dataSource = YamlShardingDataSourceFactory.createDataSource(new File(resources.getPath()));

// 插入数据
try (final Connection connection = dataSource.getConnection()) {
    for (int i = 1; i <= 10; i++) {
        final PreparedStatement statement = connection.prepareStatement("insert into t_order(user_id, amount) values (?, ?)");
        statement.setInt(1, i);
        statement.setInt(2, new Random().nextInt(100));

        System.out.println("插入第" + i + "条数据，返回结果:" + statement.executeUpdate());
    }
}

// 查询数据
try (final Connection connection = dataSource.getConnection()) {
    for (int i = 1; i <= 10; i++) {
        final PreparedStatement statement = connection.prepareStatement("select * from t_order where user_id=?");

        statement.setInt(1, i);
        try (ResultSet resultSet = statement.executeQuery()) {
            while (resultSet.next()) {
                final long orderId = resultSet.getLong("order_id");
                final int userId = resultSet.getInt("user_id");
                final int amount = resultSet.getInt("amount");
                System.out.println("获取到数据: 订单编号:" + orderId + ", 用户编号:" + userId + ", 价格:" + amount);
            }
        }
    }
}
```

通过运行这段程序，我们便完成了简单的demo。下面我们再来分别看看数据库。

![](/img/in-post/try-with-shardingsphere/131784658964864.png)
![](/img/in-post/try-with-shardingsphere/131825739988370.png)

通过这两张截图就可以看出数据是真实保存在两个数据库中，并且是根据用户id进行余2处理的。

## 总结

目前从基本的demo中可以看出，使用ShardingSphere还是非常容易的，他是基于DataSource级别的，你可以把它理解为在应用内部的一个独立数据源，所以也有很好的通用性。也可以很好的和现有的Spring等框架结合。

这里只是基本介绍了基本的分库功能，当然他还有很多很好用的功能，比如读写分离，配置中心集成，分布式事务等高级功能。欢迎大家去使用。
