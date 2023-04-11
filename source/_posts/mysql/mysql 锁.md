---
title: mysql 锁
tag: mysql
---

# 锁

# 1. 什么是锁

​	锁是数据库系统区别于文件系统的一个关键特性。**锁机制用于管理对共享资源的并发访问。**InnoDB存储引擎会在行级别上对表数据上锁，这固然不错。不过InnoDB存诸引擎也会在数据库内部其他多个地方使用锁，从而允许对多种不同资源提供并发访问。例如，操作缓冲池中的LRU列表，删除、添加、移动LRU列表中的元素，为了保正一致性，必须有锁的介人。**数据库系统使用锁是为了支持对共享资源进行并发访问，提供数据的完整性和一致性。**

# 2. MySQL的锁分类

> 相对其他数据库而言，MySQL的锁机制比较简单，其最显著的特点是不同的存储引擎支持不同的锁机制。

MySQL大致可归纳为以下3种锁：

- **行级锁**：**共享锁（S Lock）、排他锁（X Lock）**
  - 开销大，加锁慢；
  - 会出现死锁；
  - 锁定粒度最小，发生锁冲突的概率最低，并发度也最高。
- **表级锁**：**意向共享锁（IS Lock）、意向排他锁（IX Lock）**
  - 开销小，加锁快；
  - 不会出现死锁；
  - 锁定粒度大，发生锁冲突的概率最高，并发度最低。
- 页面锁：
  - 开销和加锁时间界于表锁和行锁之间；
  - 会出现死锁；
  - 锁定粒度界于表锁和行锁之间，并发度一般

## 表锁 VS 行锁

> 表锁和行锁的概念很容易理解**一个是锁定整张表****一个是锁定一行记录**，那么两者有什么区别呢。

**锁的粒度**：表锁 > 行锁 -- 这是因为表锁会锁定更多的记录以及资源因此粒度比较大

**加锁效率**：表锁 > 行锁 -- 这是因为表锁直接锁定了整个表资源而不需要向行锁一样一行行锁

**冲突概论**：表锁 > 行锁 -- 锁整张表数据所有写的操作都需要阻塞因此冲突更多

**并发性能**：表锁 < 行锁 -- 行锁的冲突概率小自然并发高

## 2.1 行级锁

**InnoDB存储引擎实现了如下两种标准的行级锁**

| 锁之间的兼容性         | 读锁S    | 写锁X  |
| ---------------------- | -------- | ------ |
| 共享锁（读锁、S Lock） | **兼容** | 不兼容 |
| 排他锁（写锁、X Lock） | 不兼容   | 不兼容 |

> - 可以发现**X锁与任何的锁都不兼容**，而**S锁仅和S锁兼容**。
> - 需要特别注意的是，S和X锁都是行锁，兼容是指对同一记录（row）锁的兼容性情况。

### 共享锁——S Lock

> 允许事务**读一行**数据

### 排他锁——X Lock

> 允许事务**删除或者更新一行**数据

## 2.2 表级锁

​	InnoDB存储引擎支持多粒度（(granular〉锁定，这种锁定允许事务在行级上的锁和表级上的锁同时存在。**为了支持在不同粒度上进行加锁操作，InnoDB存储引擎支持一种额外的锁方式**，称之为**意向锁(Intention Lock)**。意向锁是将锁定的对象分为多个层次，意向锁意味着事务希望在更细粒度(fine granularity)上进行加锁，如下图所示：![image-20220601105306954](.\media\7.jpg)

- 若将上锁的对象看成一棵树，那么对最下层的对象上锁，也就是**对最细粒度的对象进行上锁**
- 那么首先需要对粗粒度的对象上锁，如果需要对页上的记录r进行上X锁，那么分别需要对**数据库A、表、页**上，最后对**记录r**上。若`意向锁IX``X锁`

**由于InnoDB存储引擎支持的是行级别的锁，因此意向锁其实不会阻塞除全表扫意外的任何请求**

| 锁之间的兼容性 | X    | IX   | S    | IS   |
| -------------- | ---- | ---- | ---- | ---- |
| IS             |      | 兼容 | 兼容 | 兼容 |
| IX             |      | 兼容 |      | 兼容 |
| S              |      |      | 兼容 | 兼容 |
| X              |      |      |      |      |

- 如果一个事务请求的锁模式与当前的锁兼容，InnoDB就请求的锁授予该事务；
- 反之，**如果两者两者不兼容，该事务就要等待锁释放。**

### 意向共享锁——IS Lock

> 事务想要获得**一张表中的某几行**的共享锁

### 意向排他锁——IX Lock

> 事务想要获得**一张表中某几行**的排他锁

## 多版本并发控制——MVCC

快照数据其实就是当前行数据之前的历史版本，每行记录可能有多个版本。一个行记录可能有不止一个快照数据，一般称这种技术为行多版本技术。由此带来的并发控制，称之为**多版本并发控制(Multi VersionConcurrency Control，MVCC)。**

文章可以看：[深入理解MySQL的MVCC原理](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fhuyuyang6688%2Farticle%2Fdetails%2F123028254)

## 2.3 一致性非锁定读

​	一致性的非锁定读(consistent nonlocking read）是指InnoDB存储引擎通过的方式来读取当前执行时间数据库中行的数据。`行多版本控制（multi versioning）`

> ​	如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此去等待行上锁的释放。相反地，InnoDB存储引擎会去读取行的一个快照数据。**这样就不需要等待行上的X锁释放了，极大的提高了数据库的并发性**
>
> ![image-20220601110046377](.\media\8.jpg)
>
> **说明：快照数据是指该行的之前版本的数据，该实现是通过undo段来完成。而undo用来在事务中回滚数据，因此快照数据本身是没有额外的开销。此外，读取快照数据是不需要上锁的，因为没有事务需要对历史的数据进行修改操作。**

**注意：**

​	在事务隔离级别和 (InnoDB存储引擎的默认事务隔离级别)下，InnoDB存储引擎使用非锁定的一致性读。然而，对于快照数据的**定义却不相同**。在READ COMMITTED事务隔离级别下，对于快照数据，非一致性读总是读取被锁定行的最新一份快照数据。而在**REPEATABLE READ事务隔离级别下，对于快照数据，非一致性读总是读取事务开始时的行数据版本。**来看下面的一个例：`READ COMMITTED``REPEATABLE READ`

### 演示

一样是我们的tb_user表，可以看这篇[Mysql高级——索引篇](https://juejin.cn/post/7103493390318190606#heading-13)的环境准备获取

| 时间 | 事务A                                   | 事务B                                             |
| ---- | --------------------------------------- | ------------------------------------------------- |
| 1    |                                         | begin;                                            |
| 2    |                                         | select * from tb_user where id = 1;（首次读-值1） |
| 3    | begin;                                  |                                                   |
| 4    | update tb_user set age=50 where id = 1; |                                                   |
| 5    |                                         | select * from tb_user where id = 1;（值1）        |
| 6    | commit;                                 |                                                   |
| 7    |                                         | select * from tb_user where id = 1;（值2）        |
| 8    |                                         | commit;                                           |

#### 测试——REPEATABLE READ(可重复读事务隔离级别)

```mysql
-- 由于mysql数据库的更新，在旧版本中tx_isolation是作为transaction_isolation的别名被应用的，新版本已经弃用了，所以输入会显示未知变量.
-- 新版mysql查看事务隔离级别
SELECT @@transaction_isolation

-- 旧版mysql
SELECT @@tx_isolation;

-- 设置事务隔离级别
set session transaction isolation level  REPEATABLE READ
复制代码
```

![image-20220601140935100](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/578cc416f7d042c4a1c966c44cd3acc4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![image-20220601134237107](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/036a2a80b86e446b87464aec05a85247~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 开启事务B，进行首次读取

![image-20220601134457169](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0690b0c4ab1747cf98b953844258308b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 开启事务A，对该条数据进行修改，**将age修改为50**

![image-20220601134556694](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf28e92e497a4a559e36eabc1b4a9ebe~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 此时，再回到事务B再次对该行数据进行读取，发现该行数据**还是我们事务B首次读取到的数据**，**并没有因为事务A的更新操作而受到影响**

![image-20220601134724021](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa39b8c3db8c4952b24ffc7c762e015c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 此时对**事务A进行提交操作**，再次对该行数据进行读取操作，还是一样的结果（如果此时是READ COMMITTED事务隔离级别，也是一样的结果，因为事务A提交事务后，只生成了一份数据快照，该**快照的age值也是23**，则此时事务B读取到的是最新的一份快照数据，也是一样的）

![image-20220601134843841](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1a52445dd464da9b9cc3f28711ab995~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

![image-20220601135000970](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1d07ac80fbb4d39b3ef47535e5ab76d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- **提交事务B**后再次进行读取操作，可以看到此时读到的age值已经改变了

![image-20220601135211255](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/107c6b302cb343da83d9d6c0adc0a8b8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

#### 测试——READ COMMITTED

> 修改事务的隔离级别为——`read committed`

```mysql
-- 修改事务的隔离级别为——read committed
set session transaction isolation level read committed
select @@transaction_isolation
复制代码
```

![image-20220601141324664](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b71a19e8ddd4289938bf951f5810426~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> 注意：此时对于id=1这一行的数据，已经有一个数据快照了，是age=23的数据快照

| 时间 | 事务A                                   | 事务B                               |
| ---- | --------------------------------------- | ----------------------------------- |
| 1    |                                         | begin;                              |
| 2    |                                         | select * from tb_user where id = 1; |
| 3    | begin;                                  |                                     |
| 4    | update tb_user set age=50 where id = 1; |                                     |
| 5    |                                         | select * from tb_user where id = 1; |
| 6    | commit;                                 |                                     |
| 7    |                                         | select * from tb_user where id = 1; |
| 8    |                                         | commit;                             |

> -- 开启事务B，进行首次读取

![image-20220601151924209](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f28173bee0e148f69ba986c7979d3656~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- **开启事务A，进行一次更新操作，修改age为49**

![image-20220601152046666](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f293169dc2314c2a8f4e4d84f7821f21~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 在**事务B**中读取一次数据，**依旧是50**

![image-20220601152116193](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88b3cbb9170b4234acbea07b927b78a8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- **在事务A中提交事务后**，此时数据快照多了一份是49的

![image-20220601152316735](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2917f27105bf46b1932c1926e972b886~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 再次去到事务B中读取id=1的该行数据，会发现读取的是最新的数据快照，**此时事务B还没进行事务的提交，但是事务A进行事务的提交了**

![image-20220601152407088](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e416ea0ea5bf4ff89adf608fb225cd14~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### **总结**

> **可见**
>
> - 在**REPEATABLE READ事务隔离级别下，对于快照数据，非一致性读总是读取事务开始时的行数据版本。**
> - 在**READ COMMITTED事务隔离级别下，对于快照数据，非一致性读总是读取被锁定行的最新一份快照数据。**

## 2.4 一致性锁定读

​	在默认配置下，即事务的隔离级别为REPEATABLE READ模式下，InnoDB存储引擎的SELECT操作使用一致性非锁定读。但是在某些情况下，用户需要显式地对数据库读取操作进行加锁以保证数据逻辑的一致性。而这要求数据库支持加锁语句，即使是对于SELECT的只读操作。InnoDB存储引擎对于SELECT语句支持两种一致性的锁定读操作：

- **共享锁（Ｓ）：SELECT \* FROM table_name WHERE ... LOCK IN SHARE MODE**
- **排他锁（X）：SELECT \* FROM table_name WHERE ... FOR UPDATE**

> 1. 意向锁是InnoDB自动加的，不需用户干预。
> 2. 对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及的数据集加排他锁Ｘ；
> 3. 对于普通SELECT语句，InnoDB不会加任何锁；
> 4. 事务可以通过上面的语句显示给记录集加共享锁或排他锁X。
>
> ​    用SELECT .. IN SHARE MODE获得共享锁，主要用在需要数据依存关系时确认某行记录是否存在，并确保没有人对这个记录进行UPDATE或者DELETE操作。
>
> ​    但是如果当前事务也需要对该记录进行更新操作，则很有可能造成死锁，对于锁定行记录后需要进行更新操作的应用，应该使用SELECT ... FOR UPDATE方式获取排他锁。

### 演示

```mysql
-- 事务隔离级别设置为默认的
set session transaction isolation level  REPEATABLE READ
复制代码
```

#### for update测试

| 时间 | 事务A                                             | 事务B                                        |
| ---- | ------------------------------------------------- | -------------------------------------------- |
| 1    |                                                   | begin;                                       |
| 2    |                                                   | select * from tb_user where id=1 for update; |
| 3    | begin;                                            |                                              |
| 4    | update tb_user set age=40 where id = 2;（被阻塞） |                                              |
| 5    |                                                   | commit；（释放X锁）                          |
| 6    | update tb_user set age=40 where id = 2;（成功）   |                                              |
| 7    | commit；                                          |                                              |

> -- 事务A开启事务后使用一致性的锁定读操作给id=1的行上了X锁

```mysql
begin;
select * from tb_user where id = 1 for update;
复制代码
```

![image-20220601165621942](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/886e1cc7e5134e5980dbbe42b6f7d47e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 事务B此时开启事务，对id=1的行进行更新操作，**出现被阻塞情况**

```mysql
begin;
update tb_user set age=40 where id = 1;
复制代码
```

![image-20220601170306504](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcf9cd9b682d42cdb72f3b273fa847e1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 同样的使用for update操作对表上X锁（锁不兼容），**一样会被阻塞**

```mysql
select * from tb_user where id = 1 for update;
复制代码
```

![image-20220601170251538](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed32d91abb1144a39984572814eeede7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

#### lock in share mode测试

> -- **事务A**开启事务后使用一致性的锁定读操作给id=1的行上了S锁

```mysql
begin ;
select * from tb_user where id=1 lock in share mode;
复制代码
```

![image-20220601170949968](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02afdfff8c5b4c41b1f59c4f38e48acc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- **事务B**开启事务后也请求对id=1的行数据上S锁，因为S锁之间互相兼容，并**没有出现阻塞**

```mysql
begin;
select * from tb_user where id=1 lock in share mode;
复制代码
```

![image-20220601171112299](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88713e1a1516426d8d4f5b368aba2431~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 事务B也请求对id=1的行数据上X锁，会**出现阻塞的情况**

```mysql
select * from tb_user where id = 1 for update
复制代码
```

![image-20220601171250234](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd8d49ae2ffa475baece156a2e4cce0f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### 总结

> `SELECT…FOR UPDATE`对读取的行记录**加一个X锁**，其他事务不能对已锁定的行加上任何锁。
>
> `SELECT…LOCK IN SHARE MODE`对读取的行记录**加一个S锁**，其他事务可以向被锁定的行加S锁，但是如果加X锁，则会被阻塞。

## 非锁定读与锁定读注意点

> 1. **对于一致性非锁定读**，即使读取的行已被执行了，也是可以进行读取的，这和之前讨论的情况一样。`SELECT…FOR UPDATE`
> 2. 此外，，必须在一个事务中，当事务提交了，锁也就释放了。**因此在使用上述两句SELECT锁定语句时，务必加上BEGIN，START TRANSACTION或SETAUTOCOMMIT=0。**`SELBCT…FOR UPDATE``SELECT…LOCK IN SHARE MODE`

## 2.5 自增长与锁——AUTO-INC Locking

自增长在数据库中是非常常见的一种属性，也是很多DBA或开发人员首选的主键方式。在InnoDB存储引擎的内存结构中，对每个含有自增长值的表都有一个自增长计数器（ auto-increment counter)。当对含有自增长的计数器的表进行插人操作时，这个计数器会被初始化，**插人操作会依据这个自增长的计数器值加1赋予自增长列。这个实现方式称做AUTO-INC Locking。\**这种锁其实是\**采用一种特殊的表锁机制**，为了提高插入的性能，锁不是在一个事务完成后才释放，而是在完成对自增长值插入的SQL语句后**立即释放**。

**注意点**：

- 在InnoDB存储引擎中，自增长值的列必须是索引，同时必须是索引的第个列。
- 如果不是第一个列，则 MySQL数据库会抛出异常，而MyISAM存储引擎没有这个问题。

![image-20220601201848034](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/222108cc4c214b7cade6f2a04aaa0369~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

**存在的问题**：

虽然AUTO-INC Locking 从一定程度上提高了并发插入的效率，但还是存在一些性能上的问题。

1. 首先，对于有自增长值的列的并发插入性能较差，事务必须等待前一个插入的完成（虽然不用等待事务的完成)。
2. 其次，对于 INSERT…SELECT 的大数据量的插人会影响插入的性能，因为另一个事务中的插入会被阻塞。

## 2.6 外键和锁

​    外键主要用于引用完整性的约束检查。**在InnoDB存储引擎中，对于一个外键列，如果没有显式地对这个列加索引，InnoDB存储引擎自动对其加一个索引，因为这样可以避免表锁**（这比Oracle数据库做得好，Oracle数据库不会自动添加索引，用户必须自己手动添加，这也导致了Oracle数据库中可能产生死锁。）

**注意点**：

对于外键值的插入或更新，首先需要查询父表中的记录，即SELECT父表。但是**对于父表的SELECT操作**，不是使用一致性非锁定读的方式，因为这样会发生数据不一致的问题，**因此这时使用的是SELECT…LOCK IN SHARE MODE方式**，即**主动对父表加一个S锁。**

**解释**：

> 设想如果这时父表上已经这样加X锁，子表上的操作会被阻塞，如下图：
>
> **parent是父表，child是子表**

![image-20220601202657505](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7278f865339942888c2dafb5bce8f5e4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> 设想，如果访问父表时，使用的是一致性的非锁定读，这时的Session B会读到父表有id=3的记录，可以进行插入操作。但是如果会话A对事务提交了，则父表中就不存在id为3的记录。数据在父、子表就会存在不一致的情况。若这时用户查询，会看到如下结果:`INNODB_LOCKS表`
>
> ![image-20220601203302777](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ed84fb74ba64fe7b468a398c92e39f6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)
>
> 参数说明：
>
> ![image-20220601203339844](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25b7e071621e42d493e91291ba505ca2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

# 3. 锁的算法

## 3.1 行锁的三种算法

InnoDB存储引擎有三种行锁的算法：

1. **Record Lock**：单个行记录上的锁
2. **Gap Lock**：间隙锁，锁定一个范围，但不包含记录本身
3. **Next-Key Lock**：Gap Lock+Record Lock，锁定一个范围，并且锁定记录本身

**锁类型锁定的大致范围：**

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e8683dbabcf4b2ca98439e1459ace52~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> Record Lock：10
>
> Gap Lock：（-∞，10），（10，20），（20，+∞）
>
> Next-Key Lock：（-∞，10]，（10，20]，（20，+∞）

### 1️⃣Record Lock

**Record Lock总是会去锁住索引记录，如果InnoDB存储引擎表在建立的时候没有设置任何一个索引，那么这时InnoDB存储引擎会使用隐式的主键来进行锁定。**

> 行锁是加在索引上的，**如果当你的查询语句不走索引的话，那么它就会升级到表锁**，最终造成效率低下，所以在写SQL语句时需要特别注意。

### 2️⃣Gap Lock

当我们使用范围条件而不是相等条件去检索，并请求锁时，InnoDB就会给符合条件的记录的索引项加上锁；而对于键值在条件范围内但并不存在（参考上面所说的空闲块）的记录，就叫做间隙，InnoDB在此时也会对间隙加锁

> - 间隙锁只有一个目的就是在RR、SERIALIZABLE隔离级别下为了防止其他事务插入数据。
> - 假如一个索引有2、4、5、9、12 五个值，那该索引可能被间隙锁锁的范围为(-∞ , 2),(2 , 4),(4 , 5),(5 , 9),(9 , 12),(12 , +∞)。

### 3️⃣Next-Key Lock

**Next-Key Lock是结合了Gap Lock和 Record Lock的一种锁定算法，在Next-KeyLock算法下，InnoDB对于行的查询都是采用这种锁定算法。**其目的就是为了解决Phantom Problem（幻象问题）。

> - 对【某一个行记录】和【这条记录与它前一条记录之间的范围/间隙】都上锁，这里我们称它为邻键锁。
> - 假如一个索引有2、4、5、9、12 五个值，那该索引可能被邻键锁锁的范围为(-∞ , 2],(2 , 4],(4 , 5],(5 , 9],(9 , 12],(12 , +∞)。在InnoDB中，加锁的基本单位是Next-Key Lock，只不过在某些特殊情况下会退化为 Record Lock 或者 Gap Lock。

> **特别说明：**
>
> 然而，**当查询的索引含有唯一属性时，InnoDB存储引擎会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身，而不是范围。**（可在下面测试代码中看到）

### 测试环境

```mysql
-- 创建a作为主键（即唯一索引），b为辅助索引
CREATE TABLE lock_test ( a INT,b INT,PRIMARY KEY(a),KEY(b));
INSERT INTO lock_test SELECT 1,1;
INSERT INTO lock_test SELECT 3,1;
INSERT INTO lock_test SELECT 5,3;
INSERT INTO lock_test SELECT 7,6;
INSERT INTO lock_test SELECT 10,8;
复制代码
```

![image-20220601223051155](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63179baa8b8540d6994e222042b787e9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### 唯一索引的锁定示例

> **当查询的索引含有唯一属性时，InnoDB存储引擎会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身，而不是范围。**

| 时间 | 事务A                                          | 事务B                                 |
| ---- | ---------------------------------------------- | ------------------------------------- |
| 1    | begin；                                        |                                       |
| 2    | select * from lock_test where a=5 for update； |                                       |
| 3    |                                                | begin；                               |
| 4    |                                                | **insert into lock_test select 4,5;** |
| 5    |                                                | commit； #成功，不需要等待            |
| 6    | commit；                                       |                                       |

![image-20220601223019835](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d61e6c7c08841b88ec19f011f811d7f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 事务A开启事务后，先给记录a=5上一个X锁，但是因为此时a索引是主键即唯一索引，Next-Key Lock会降级为Record Lock，**仅仅锁住了a=5这一行的记录**

```mysql
-- 事务A先给记录a=5上一个X锁，但是因为此时a索引是主键即唯一索引，Next-Key Lock会降级为Record Lock
begin;
select * from lock_test where a=5 for update
复制代码
```

> -- 事务B插入一条a=4，b=5的记录，**因为事务A的X锁仅仅加在了a=5这一行，所以事务B在执行插入操作时候可以正常获取X锁，并不会被阻塞**

```mysql
-- 事务B插入一条a=4，b=5的记录，因为事务A的X锁仅仅加在了a=5这一行，所以事务B在执行插入操作时候可以正常获取X锁
begin;
insert into lock_test select 4,5;
复制代码
```

![image-20220601223526341](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/879b058e828c48e1b5be3e8b77d824df~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

### 辅助索引的锁定示例

> **对于辅助索引，其加上的是Next-Key Lock，锁定的是一个范围值（即Record Lock和Gap Lock的混合锁）**

| 时间 | 事务A                                          | 事务B                                                   |
| ---- | ---------------------------------------------- | ------------------------------------------------------- |
| 1    | begin；                                        |                                                         |
| 2    | select * from lock_test where a=5 for update； |                                                         |
| 3    |                                                | begin；                                                 |
| 4    |                                                | **select \* from lock_test where b=3 for update** #阻塞 |

> 在上个操作的事务B中进行一致性锁定读操作，请求对辅助索引b=3加上一个X锁，**会有阻塞的出现**

```mysql
-- 在上个操作的事务B中进行一致性锁定读操作，请求对辅助索引b=3加上一个X锁
select * from lock_test where b=3 for update
复制代码
```

![image-20220601224329511](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edb40207d579499c87f603da34fe0698~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> ![image-20220601224653635](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ee5e80e1b384568b324fe7a12e26bf5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)
>
> **说明：**
>
> 这是因为，针对辅助索引，其上锁上的是Next-Key Lock，这个邻键锁会对辅助索引的上一个键值和下一个键值上一个范围锁，示例中**加X锁的辅助索引是b=3，其上一个键值是1，后一个键值是6**，故Next-Key Lock锁定的范围是 **(1, 3],(3,6)**，**这区间内的都被加上了X锁，其他事务无法获取，若获取会被阻塞**
>
> **所以，此时，因为`select \* from lock_test where b=3 for update`无法获取a=5这一行记录的X锁而导致了事务B发生了阻塞**

**此时重新进行操作**

| 时间 | 事务A                                        | 事务B                                                       |
| ---- | -------------------------------------------- | ----------------------------------------------------------- |
| 1    | begin;                                       |                                                             |
| 2    | select * from lock_test where b=3 for update |                                                             |
| 3    |                                              | begin;                                                      |
| 4    |                                              | select * from lock_test where a=5 lock in share mode; #阻塞 |
| 5    |                                              | insert into lock_test select 4,2；#阻塞                     |
| 6    |                                              | insert into lock_test select 6,5;  #阻塞                    |
| 7    |                                              | insert into lock_test select 8,6;  #正常执行                |

> -- 事务A开启事务，对b=3这条记录进行一致性锁读（X锁），**此时被加锁的范围：(1, 3],(3,6)**
>
> ![image-20220601232131174](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0ea27e922dc497db8b5ce755df39dac~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

```mysql
-- 事务A开启事务，对b=3这条记录上锁
begin;
select * from lock_test where b=3 for update;
复制代码
```

![image-20220601230805400](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d0fe386b01f4e55818c4e85f992ace5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 开启事务B，对a=5的记录上X锁，会出现阻塞

```mysql
-- 开启事务B，对a=5的记录上X锁，会出现阻塞
begin;
select * from lock_test where a=5 lock in share mode;
复制代码
```

![image-20220601231731724](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0201fcd3da1d49439cd723a1bfcbd0ea~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 事务B插入一条a=4，b=2的数据（X锁）

```mysql
insert into lock_test select 4,2
复制代码
```

![image-20220601233714262](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed4bc5359f014b2594a376cf332d51d0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> -- 事务B插入一条a=6，b=5的数据（X锁），会出现阻塞

```mysql
insert into lock_test select 6,5;
复制代码
```

![image-20220601234025094](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/82bf1d4e1a0243f58ac89ae342ebef24~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> 解释：
>
> **第一个SQL语句不能执行**，因为在会话A中执行的SQL语句已经对聚集索引中列a =5的值加上X锁，因此执行会被阻塞。
>
> 第二个SQL语句，主键插入4，没有问题，但是插人的辅助索引值2在锁定的范围(1，3)中，因此执行同样会被阻塞。
>
> 第三个SQL语句，插入的主键6没有被锁定，5也不在范围(1，3)之间。但插入的值5在另一个锁定的范围(3，6)中，故同样需要等待。

而**下面的SQL语句，不会被阻塞**，可以立即执行:

> -- 事务B插入一条a=8，b=6的数据（X锁），不会出现阻塞

```mysql
-- 下面sql
insert into lock_test select 8,6;
复制代码
```

![image-20220602005321000](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edecf561fe394c6f857dcdd83780f30b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

## 3.2 深入分析MySQL行锁加锁规则

具体详细的可以看——》[深入分析MySQL行锁加锁规则](https://link.juejin.cn?target=https%3A%2F%2Fhuyuyang.blog.csdn.net%2Farticle%2Fdetails%2F123508245%3Fspm%3D1001.2014.3001.5502)

## 3.3 幻象问题（幻读）

> Phantom Problem是指在同一事务下，**连续执行两次同样的SQL语句可能导致不同的结果**，第二次的SQL语句可能会返回之前不存在的行。

​	**在默认的事务隔离级别下，即REPEATABLE READ 下，InnoDB存储引擎采用Next-Key Locking机制来避免Phantom Problem（幻像问题)**。（这点可能不同于与其他的数据库，如Oracle数据库，因为其可能需要在SERIALIZABLE的事务隔离级别下才能解决Phantom Problem。）

> 此外，用户可以通过InnoDB存储引擎的Next-Key Locking机制在应用层面实现唯一性的检查。

​	如果用户通过索引查询一个值，并对该行加上一个SLock，那么即使查询的值不在，其锁定的也是一个范围，因此若没有返回任何行，那么新插人的值一定是唯一的。也许有读者会有疑问，如果在进行第一步SELECT…LOCK IN SHARE MODE操作时，有多个事务并发操作，那么这种唯一性检查机制是否存在问题。**其实并不会，因为这时会导致死锁，只有一个事务的插人操作会成功，而其余的事务会抛出死锁的错误**，如表6-14所示。

![image-20220602010206860](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78d9f8301d124eb580f99113a870abca~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

# 4. 锁问题

## 4.1 脏读

> 脏读指的就是在不同的事务下，当前事务可以读到另外事务未提交的数据，简单来说就是可以读到脏数据。

![image-20220601234330312](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/186bf15a970b47b1aace2e5dc5138836~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

事务的隔离级别进行了更换，由默认的 ——>。`REPEATABLE READ``READ UNCOMMITTED`

- 可以看到，在会话A中，在事务并没有提交的前提下，会话B中的两次SELECT操作取得了不同的结果。
- 并且2这条记录是在会话A中并未提交的数据，即产生了脏读，违反了事务的隔离性。

## 4.2 不可重复读

> 不可重复读是指在一个事务内多次读取同一数据集合。

![image-20220601234514701](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eea6a46d9c0144e888221781aa647d07~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

- 在事务开始前，会话A和会话B的事务隔离级别都调整为。`READ COMMITTED`
- 在**会话A**中开始一个事务，第一次读取到的记录是1。
- 在另一个**会话B**中开始了另一个事务，插入一条为2的记录。
- 在没有提交之前，对**会话A**中的事务进行再次读取时，读到的记录还是1，没有发生脏读的现象。
- **但会话B中的事务提交后**，在对**会话A**中的事务进行读取时，这时**读到是1和2两条记录**。

## 4.3 丢失更新

> 丢失更新是另一个锁导致的问题，简单来说其就是一个事务的更新操作会被另一个事务的更新操作所覆盖，从而导致数据的不一致。

# 5. 死锁

## 5.1 概念

> 死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象。若无外力作用，事务都将无法推进下去。

## 5.2 死锁的检测

死锁的检查方法：

1. 超时机制——**被动**检测死锁
2. wait-for group(等待图)——主动检测死锁

### 超时机制

​	解决死锁问题最简单的一种方法是超时，**即当两个事务互相等待时，当一个等待时间超过设置的某一阈值时，其中一个事务进行回滚，另一个等待的事务就能继续进行。**在 InnoDB存储引擎中，参数innodb_lock_wait_timeout用来设置超时的时间。

> **注意：**超时机制虽然简单，但是其仅通过超时后对事务进行回滚的方式来处理，或者说其是根据FIFO的顺序选择回滚对象。但若超时的事务所占权重比较大，如事务操作更新了很多行，占用了较多的undo log，这时采用FIFO的方式，就显得不合适了，因为回滚这个事务的时间相对另一个事务所占用的时间可能会很多。

### wait-for group(等待图)

wait-for graph要求数据库保存以下两种信息：

1. 锁的信息链表
2. 事务等待链表

**通过上述链表可以构造出一张图，而在这个图中若存在回路，就代表存在死锁，因此资源间相互发生等待。**

- 在 wait-for graph 中，事务为图中的节点。
- 事务T1指向T2边的定义为：
  - 事务T1等待事务T2所占用的资源
  - 事务T1最终等待T2所占用的资源，也就是事务之间在等待相同的资源，而事务T1发生在事务T2的后面

**根据下图**

![image-20220601214238586](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/500228d94f854938b19a52968eb70808~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

**解释：**

- 在Transaction Wait Lists中可以看到共有4个事务t1、t2、t3、t4，故在wait-forgraph中应有4个节点。
- 而事务t2对row1占用x锁，事务t1对row2占用s锁。
- 事务t1需要等待事务t2中row1的资源，因此在wait-for graph中有条边从节点t1指向节点t2。
- 事务t2需要等待事务t1、t4所占用的row2对象，故而存在节点t2到节点t1、t4的边。
- 同样，存在节点t3到节点t1、t2、t4的边，因此最终的wait-for graph，如图6-6所示。

**产生的wait-for group(等待图)**

![image-20220601214259726](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2df0b5dfec904dc4821fce32e01fbb6f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> **通过图6-6可以发现存在回路(t1，t2)，因此存在死锁。**
>
> 通过上述的介绍，可以发现wait-for graph是一种较为主动的死锁检测机制，**在每个事务请求锁并发生等待时都会判断是否存在回路，若存在则有死锁，通常来说InnoDB存储引擎选择回滚undo量最小的事务。**

## 5.3 死锁的示例

**表结构**

```mysql
CREATE TABLE t (
	a INT PRIMARY KEY
)ENGINE=InnoDB;

INSERT INTO t VALUES (1), (2), (4), (5);
复制代码
```

**经典的AB-BA死锁**

![image-20220601214725244](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b424e5c1442413cba047b6cc631feda~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> **解释**：
>
> 在上述操作中，会话B中的事务抛出了1213这个错误提示，即表示事务发生了死锁。
>
> 死锁的原因是**会话A和B的资源在互相等待**。大多数的死锁InnoDB存储引擎本身可以侦测到，不需要人为进行干预。
>
> 但是在上面的例子中，在**会话B**中的事务抛出死锁异常后，**会话A**中马上得到了记录为2的这个资源，这其实是因为会话B中的事务发生了回滚，否则会话A中的事务是不可能得到该资源的。
>
> **因为InnoDB存储引擎并不会回滚大部分的错误异常，但是死锁除外。发现死锁后，InnoDB存储引擎会马上回滚一个事务，这点是需要注意的。因此如果在应用程序中捕获了1213这个错误，其实并不需要对其进行回滚。**

![image-20220601215650438](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfea99ffcf234e94816198857400dab0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

> 可以看到，**会话A**中已经对持有了X锁，但是**会话A**中插人时会导致死锁发生。`记录4``记录3`
>
> 这个问题的产生是由于**会话B**中请求的S锁而发生等待，但之前请求的锁对于主键值记录1、2都已经成功，若在事件点5能插入记录，那么**会话B**在获得持有的S锁后，还需要向后获得的记录，这样就显得有点不合理。`记录4``记录4``记录3`
>
> 因此InnoDB存储引擎在这里主动选择了死锁，而回滚的是undo log记录大的事务，这与AB-BA死锁的处理方式又有所不同。





> 资料来源——>《MySQL技术内幕》
>
> [深入分析MySQL行锁加锁规则](https://link.juejin.cn?target=https%3A%2F%2Fhuyuyang.blog.csdn.net%2Farticle%2Fdetails%2F123508245%3Fspm%3D1001.2014.3001.5502)

