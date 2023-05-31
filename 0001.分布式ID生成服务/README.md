# 分布式ID生成

## 1. 分布式ID服务的要求

全局唯一：不能出现重复的ID
高性能：生成ID的速度要快，ID生成响应要快，否则反而成为系统的瓶颈
高可用：无限接近100%的可用性
高可扩展：支持集群部署，支持动态扩容
好接入：提供简单易用的接口，在系统设计和实现上要尽量简单
趋势递增：生成的ID要尽量趋势递增，这样生成的索引树才能有序，提高检索效率(也看具体的业务需求)

## 2. 常见的分布式ID生成方案

- UUID
- 数据库自增ID
- 数据库多主模式
- 号段模式
- Redis生成ID
- SnowFlake（Twitter）
- TinyId（滴滴）
- Leaf（美团）
- UidGenerator（百度）

## 3. UUID

<details>
  <summary>UUID</summary>

UUID是最容易想到的分布式唯一ID的实现方式，它具有全球唯一的特性。

```java
public class UUIDTest {
    public static void main(String[] args) {
        String uuid = UUID.randomUUID().toString().replaceAll("-", "");
        System.out.println(uuid);
    }
}
```

但是他的缺点十分明显：

- 无序：UUID是无序的，这对于数据库索引来说是一个灾难，因为它会导致索引树频繁分裂，从而影响检索性能。而且是个字符串，存储性能差，查询很耗时。
- 无意义：UUID是无意义的，如果用它作为订单号，看不出任何和订单相关的信息。

</details>

## 4. 数据库自增ID

<details>
  <summary>数据库自增ID</summary>

基于数据库的自增ID也可以做一个分布式ID生成器。

具体的实现就是使用一个单独的MySQL实例作为ID生成器，然后在这个实例上创建一个表，表中只有一个字段，就是自增ID。

```sql
CREATE TABLE `id_generator`
(
    `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;
```

然后在应用程序中，每次获取ID的时候，都去访问这个MySQL实例，获取一个自增ID，然后返回给应用程序。

```sql
INSERT INTO id_generator
VALUES ();
```

这种方式的优点是简单易用，而且可以保证ID的有序性。

但是缺点也很明显，访问量大的时候，这个MySQL实例会成为系统的瓶颈，单点DB实例不可靠，不可扩展。

</details>

## 5. 数据库多主模式

<details>
  <summary>数据库多主模式</summary>

单点MySQL实例不可靠，我们可以做一些优化。
单点模式不行，我们就换成主从模式，这样就可以提高可用性，而且还可以提高读性能。
一个主节点挂掉导致不可用，我们就换成双主模式集群，这样每个实例都能够单独生产自增ID。

但是这样还有个问题，就是两个节点的实例自增都会从1开始，这样就会导致ID冲突。

解决这个问题的方式：

- 让两个实例的自增ID的起始值不一样，比如一个从1开始，一个从2开始；
- 设置两个实例的自增步长为2，这样一个实例生成的ID都是奇数，另一个实例生成的ID都是偶数，就不会冲突了。

MySQL_1:

```sql
set @@auto_increment_offset = 1; -- 设置自增起始值为1
set @@auto_increment_increment = 2; -- 设置自增步长为2
```

MySQL_2:

```sql
set @@auto_increment_offset = 2; -- 设置自增起始值为2
set @@auto_increment_increment = 2; -- 设置自增步长为2
```

这样这两个MySQL的实例的自增ID就分别为：

```
MySQL_1: 1, 3, 5, 7, 9, 11, 13, 15, 17, 19, ...
MySQL_2: 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, ...
```

如果之后的集群的性能还是不够，我们可以继续扩展，变成多主模式，这样就可以无限扩展了。

但是扩展起来就比较麻烦了。
例如扩容到3台机器：

```
设置 步长 = 3
MySQL_1: 1, 4, 7, 10, 13, 16, 19, 22, 25, 28, ...
MySQL_2: 2, 5, 8, 11, 14, 17, 20, 23, 26, 29, ...
MySQL_3: 3, 6, 9, 12, 15, 18, 21, 24, 27, 30, ...
```

而且在增加新的实例的时候，

- 要么修改已有的两台机器的起始值和步长，并且把第三台机器的起始值设置的尽量大于现有的最大ID，确保不会冲突；
- 要么把所有的实例都停掉，然后重新设置自增ID的起始值和步长，然后再启动

</details>

## 6. 基于数据库的号段模式

我们之前的基于数据库的自增ID实现方式都是每一次从数据库中获取一个自增ID，然后返回给应用程序。所以才会有数据库性能成为瓶颈的问题。

那我们换个思路，我们可以一次从数据库中获取一批ID，然后缓存在应用程序中，然后应用程序需要的时候，就从缓存中获取一个ID，而不是每次都去数据库中获取。当缓存中的ID用完了，再去数据库中获取一批ID，然后缓存起来...

这种方式就是号段模式，也叫做Segment模式。

我们可以创建一个表，用来存储当前的ID号段：

```sql
CREATE TABLE `id_generator`
(
    `id`      bigint(20) unsigned NOT NULL AUTO_INCREMENT,
    `service` varchar(128)        NOT NULL DEFAULT 'default',
    `max_id`  bigint(20) unsigned NOT NULL DEFAULT 0,
    `step`    int(11) unsigned    NOT NULL DEFAULT 1000,
    `version` int(11) unsigned    NOT NULL DEFAULT 0,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;
```

其中，

- service: 表示当前号段是给哪个服务使用的
- max_id: 表示当前号段的最大ID
- step: 表示每次获取的号段的大小
- version: 表示当前号段的版本号，作为一个乐观锁，用来实现并发控制

**示例：**

<details>
  <summary>请点击查看示例</summary>


如果我们的表中没有任何记录:

| id | service | max_id | step | version |
|----|---------|--------|------|---------|

那么我们可以插入一条记录：

```sql
INSERT INTO id_generator (service, max_id, step, version)
VALUES ('Backend', 0, 1000, 0);
```

这个时候，我们的表中就有一条记录了：

| id | service | max_id | step | version |
|----|---------|--------|------|---------|
| 1  | Backend | 0      | 1000 | 0       |

我们也获取了一批ID，从0到1000，然后缓存起来，然后应用程序就可以从缓存中获取ID了。

当一批ID用完了，我们就去数据库中获取下一批ID， 获取的方式：

- 先查询当前的记录，例如我们得到了id=1的记录，max_id=1000，step=1000，version=0；
- 然后更新这条记录，把max_id加上step，version加1，同时把当前的version作为条件，确保并发安全；

例如，应用程序A和应用程序B同时去获取ID，那么就会同时查询到id=1的记录：

```
应用程序A: max_id=1000, step=1000, version=0
应用程序B: max_id=1000, step=1000, version=0
```

这时候，两个应用程序都会认为当前的max_id是1000，然后都会去获取1000个ID。

但是总会有一个应用程序先更新这条记录，例如应用程序A先更新了这条记录，以获取到新的号段：

```sql
UPDATE id_generator
SET max_id  = max_id + step,
    version = version + 1
WHERE version = 0
  AND service = 'Backend';
```

这时候，数据库中的记录就变成了：

| id | service | max_id | step | version |
|----|---------|--------|------|---------|
| 1  | Backend | 1000   | 1000 | 1       |

然后应用程序B再去更新这条记录的时候，就会发现version不是0了，就会更新失败，然后重试。

这样就可以保证并发安全了。

</details>

这种方式的好处是，可以减少数据库的访问次数，提高性能。

## 7. 基于Redis的自增ID

我们也可以使用Redis来实现自增ID，利用redis的incr命令，可以实现自增ID的功能。

这个命令的特点是，每次自增1，而且是原子的，所以可以保证并发安全。

首先，我们可以创建一个key，用来存储当前的ID：

```
127.0.0.1:6379> set id 0 // 初始化ID为0
OK
```

每次获取ID的时候，就可以使用incr命令，获取一个自增ID：

```
127.0.0.1:6379> incr id // 获取一个自增ID
(integer) 1
```

这样就可以实现自增ID的功能了。

但是，这种方式有一个问题，就是如果Redis重启了，那么ID就会从0开始，这样就会导致ID重复。

为了解决这个问题，有两种方式：

- 第一种方式，就是在Redis重启的时候，从数据库中获取当前的最大ID，然后设置到Redis中，这样就可以避免ID重复的问题了；
- 第二种方式，就是考虑redis的持久化机制。Redis有两种持久化机制，一种是RDB，一种是AOF。
    - 其中，RDB是指定时间间隔内，把内存中的数据快照保存到磁盘中。但是如果ID一直在生成，但是RDB的时间间隔比较长，重启的时候就会导致ID重复；
    - 而AOF是把每一条写命令都保存到磁盘中。这样就可以保证重启的时候，不会导致ID重复。但是AOF的性能会比较差，Redis重启的时候需要的时间也会比较长。

甚至我们可以扩展一下思路，将Redis和号段模式结合起来，这样就可以实现高性能的ID生成器了。

我们需要使用Redis的incrby命令，每次获取一批ID。这个就不再赘述了。