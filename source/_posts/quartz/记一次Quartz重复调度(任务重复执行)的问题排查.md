---
title:  记一次Quartz重复调度(任务重复执行)的问题排查
tags: 
 - Quartz
---
# 记一次Quartz重复调度(任务重复执行)的问题排查

 

## 1. 引子

公司前期改用quartz做任务调度，一日的调度量均在两百万次以上。随着调度量的增加，突然开始出现job重复调度的情况，且没有规律可循。网上也没有说得较为清楚的解决办法，于是我们开始调试Quartz源码，并最终找到了问题所在。 如果没有耐性看完源码解析，可以直接拉到文章最末，有直接简单的解决办法。
注：本文中使用的quartz版本为2.3.0，且使用JDBC模式存储Job。

## 2. 准备

首先，因为本文是代码级别的分析文章，因而需要提前了解Quartz的用途和用法，网上还是有很多不错的文章，可以提前自行了解。

其次，在用法之外，我们还需要了解一些Quartz框架的基础概念：

\1) Quartz把触发job，叫做**fire**。**TRIGGER_STATE**是当前trigger的状态，**PREV_FIRE_TIME**是上一次触发时间，**NEXT_FIRE_TIME**是下一次触发时间，**misfire**是指这个job在某一时刻要触发，却因为某些原因没有触发的情况。

\2) Quartz在运行时，会起两类线程（不止两类），一类用于调度job的调度线程（单线程），一类是用于执行job具体业务的工作池。

\3) Quartz自带的表里面，本文主要涉及以下3张表：

- triggers表。triggers表里记录了，某个trigger的PREV_FIRE_TIME（上次触发时间），NEXT_FIRE_TIME（下一次触发时间），TRIGGER_STATE（当前状态）。虽未尽述，但是本文用到的只有这些。
- locks表。Quartz支持分布式，也就是会存在多个线程同时抢占相同资源的情况，而Quartz正是依赖这张表，处理这种状况，至于如何做到，参见3.1。
- fired_triggers表，记录正在触发的triggers信息。

\4) TRIGGER_STATE，也就是trigger的状态，主要有以下几类：
![图片描述](media/记一次Quartz重复调度(任务重复执行)的问题排查/bVbdx7u.png)

图2-1 trigger状态变化图

trigger的初始状态是**WAITING**，处于**WAITING**状态的trigger等待被触发。调度线程会不停地扫triggers表，根据NEXT_FIRE_TIME提前拉取即将触发的trigger，如果这个trigger被该调度线程拉取到，它的状态就会变为**ACQUIRED**。因为是提前拉取trigger，并未到达trigger真正的触发时刻，所以调度线程会等到真正触发的时刻，再将trigger状态由**ACQUIRED**改为**EXECUTING**。如果这个trigger不再执行，就将状态改为**COMPLETE**,否则为**WAITING**，开始新的周期。如果这个周期中的任何环节抛出异常，trigger的状态会变成**ERROR**。如果手动暂停这个trigger，状态会变成**PAUSED**。

## 3. 开始排查

### 3.1分布式状态下的数据访问

前文提到，trigger的状态储存在数据库，Quartz支持分布式，所以如果起了多个quartz服务，会有多个调度线程来抢夺触发同一个trigger。mysql在默认情况下执行select 语句，是不上锁的，那么如果同时有1个以上的调度线程抢到同一个trigger，是否会导致这个trigger重复调度呢？我们来看看，Quartz是如何解决这个问题的。

首先，我们先来看下`JobStoreSupport`类的`executeInNonManagedTXLock()`方法：

![图片描述](media/记一次Quartz重复调度(任务重复执行)的问题排查/bVbdaoM.png)

图3-1 executeInNonManagedTXLock方法的具体实现

这个方法的官方介绍：

```applescript
/**

*Execute the given callback having acquired the given lock.

*Depending on the JobStore,the surrounding transaction maybe

*assumed to be already present(managed).

*

*@param lockName The name of the lock to acquire,for example

*"TRIGGER_ACCESS".If null, then no lock is acquired ,but the

*lockCallback is still executed in a transaction.

*/
```

也就是说，传入的callback方法在执行的过程中是携带了指定的锁，并开启了事务，注释也提到，lockName就是指定的锁的名字，如果lockName是空的，那么callback方法的执行不在锁的保护下，但依然在事务中。

这意味着，我们使用这个方法，不仅可以保证事务，还可以选择保证，callback方法的线程安全。

接下来，我们来看一下`executeInNonManagedTXLock（…）`中的`obtainLock(conn,lockName)`方法，即抢锁的过程。这个方法是在`Semaphore`接口中定义的，`Semaphore`接口通过锁住线程或者资源，来保护资源不被其他线程修改，由于我们的调度信息是存在数据库的，所以现在查看`DBSemaphore.java`中`obtainLock`方法的具体实现：

![图片描述](media/记一次Quartz重复调度(任务重复执行)的问题排查/bVbdaoW.png)

图3-2 obtainLock方法具体实现

我们通过调试查看`expandedSQL`和`expandedInsertSQL`这两个变量：

![图片描述](media/记一次Quartz重复调度(任务重复执行)的问题排查/bVbdaoZ.png)

图3-3 expandedSQL和expandedInsertSQL的具体内容

图3-3可以看出，`obtainLock`方法通过locks表的一个行锁（lockName确定）来保证callback方法的事务和线程安全。拿到锁后，`obtainLock`方法将`lockName`写入`threadlocal`。当然在`releaseLock`的时候，会将`lockName`从`threadlocal`中删除。

总而言之，`executeInNonManagedTXLock()`方法，保证了在分布式的情况，同一时刻，只有一个线程可以执行这个方法。

### 3.2 quartz的调度过程

![图片描述](media/记一次Quartz重复调度(任务重复执行)的问题排查/bVbdao5.png)

图3-4 Quartz的调度时序图

`QuartzSchedulerThread`是调度线程的具体实现，图3-4 是这个线程`run()`方法的主要内容，图中只提到了正常的情况下，也就是流程中没有出现异常的情况下的处理过程。由图可以看出，调度流程主要分为以下三步：

#### 1）拉取待触发trigger:

调度线程会一次性拉取距离现在，一定时间窗口内的，一定数量内的，即将触发的trigger信息。那么，时间窗口和数量信息如何确定呢，我们先来看一下，以下几个参数：

- `idleWaitTime`： 默认30s，可通过配置属性`org.quartz.scheduler.idleWaitTime`设置。
- `availThreadCount`：获取可用（空闲）的工作线程数量，总会大于1，因为该方法会一直阻塞，直到有工作线程空闲下来。
- `maxBatchSize`：一次拉取trigger的最大数量，默认是1，可通过`org.quartz.scheduler.batchTriggerAcquisitionMaxCount`改写
- `batchTimeWindow`：时间窗口调节参数，默认是0，可通过`org.quartz.scheduler.batchTriggerAcquisitionFireAheadTimeWindow`改写
- `misfireThreshold`： 超过这个时间还未触发的trigger,被认为发生了misfire,默认60s，可通过`org.quartz.jobStore.misfireThreshold`设置。

调度线程一次会拉取**NEXT_FIRE_TIME**小于（`now + idleWaitTime +batchTimeWindow`）,大于（`now - misfireThreshold`）的，`min(availThreadCount,maxBatchSize)`个triggers，默认情况下，会拉取未来30s，过去60s之间还未fire的1个trigger。随后将这些triggers的状态由**WAITING**改为**ACQUIRED**，并插入fired_triggers表。

#### 2）触发trigger：

首先，我们会检查每个trigger的状态是不是**ACQUIRED**，如果是，则将状态改为**EXECUTING**，然后更新trigger的**NEXT_FIRE_TIME**，如果这个trigger的**NEXT_FIRE_TIME**为空，也就是未来不再触发，就将其状态改为**COMPLETE**。如果trigger不允许并发执行（即Job的实现类标注了`@DisallowConcurrentExecution`），则将状态变为**BLOCKED**，否则就将状态改为**WAITING**。

#### 3）包装trigger，丢给工作线程池：

遍历triggers，如果其中某个trigger在第二步出错，即返回值里面有exception或者为null，就会做一些triggers表，fired_triggers表的内容修正，跳过这个trigger，继续检查下一个。否则，则根据trigger信息实例化`JobRunShell`（实现了Thread接口），同时依据`JOB_CLASS_NAME`实例化`Job`，随后我们将`JobRunShell`实例丢入工作线。

在`JobRunShell`的`run()`方法，Quartz会在执行`job.execute()`的前后通知之前绑定的监听器，如果`job.execute()`执行的过程中有异常抛出，则执行结果`jobExEx`会保存异常信息，反之如果没有异常抛出，则`jobExEx`为null。然后根据`jobExEx`的不同，得到不同的执行指令`instCode`。

`JobRunShell`将trigger信息，job信息和执行指令传给`triggeredJobComplete()`方法来完成最后的数据表更新操作。例如如果job执行过程有异常抛出，就将这个trigger状态变为**ERROR**，如果是**BLOCKED**状态，就将其变为**WAITING**等等，最后从fired_triggers表中删除这个已经执行完成的trigger。注意，这些是在工作线程池异步完成。

### 3.3 排查问题

在前文，我们可以看到，Quartz的调度过程中有3次（可选的）上锁行为，为什么称为可选？因为这三个步骤虽然在`executeInNonManagedTXLock`方法的保护下，但`executeInNonManagedTXLock`方法可以通过设置传入参数lockName为空，取消上锁。在翻阅代码时，我们看到第一步拉取待触发的trigger时：

```java
public List<OperableTrigger> acquireNextTriggers(final long noLaterThan, final int maxCount, final long timeWindow) JobPersistenceException {
    String lockName;
    //判断是否需要上锁
    if (isAcquireTriggersWithinLock() || maxCount > 1) {
        lockName = LOCK_TRIGGER_ACCESS;
    } else {
        lockName = null;
    }
    return executeInNonManagedTXLock(lockName, 
                                     new TransactionCallback<List<OperableTrigger>>(){
        public List<OperableTrigger> execute(Connection conn) throws JobPersistenceException {
            return acquireNextTrigger(conn, noLaterThan, maxCount, timeWindow);
        }
    }, new TransactionValidator<List<OperableTrigger>>() {
         //省略
    });
}
```

在加锁之前对lockName做了一次判断，而非像其他加锁方法一样，默认传入的就是**LOCK_TRIGGER_ACCESS**：

```java
public List<TriggerFiredResult> triggersFired(final List<OperableTrigger> triggers) throws JobPersistenceException {
    //默认上锁
    return executeInNonManagedTXLock(LOCK_TRIGGER_ACCESS,
        new TransactionCallback<List<TriggerFiredResult>>() {
        //省略
        },new TransactionValidator<List<TriggerFiredResult>>() {
            //省略
           });
}
```

通过调试发现`isAcquireTriggersWithinLock()`的值是`false`，因而导致传入的lockName是null。我在代码中加入日志，可以更清楚的看到这个过程。
![图片描述](media/记一次Quartz重复调度(任务重复执行)的问题排查/bVbdapa.png)
图3-5 调度日志

由图3-5可以清楚看到，在拉取待触发的trigger时，默认是不上锁。如果这种默认配置有问题，岂不是会频繁发生重复调度的问题？而事实上并没有，原因在于Quartz默认采取乐观锁，也就是允许多个线程同时拉取同一个trigger。我们看一下Quartz在调度流程的第二步fire trigger的时候做了什么，注意此时是上锁状态：

```java
protected TriggerFiredBundle triggerFired(Connection conn, OperableTrigger trigger)
    throws JobPersistenceException {
    JobDetail job;
    Calendar cal = null;
    // Make sure trigger wasn't deleted, paused, or completed...
    try { // if trigger was deleted, state will be STATE_DELETED
        String state = getDelegate().selectTriggerState(conn,trigger.getKey());
         if (!state.equals(STATE_ACQUIRED)) {
            return null;
        }
    } catch (SQLException e) {
            throw new JobPersistenceException("Couldn't select trigger state: "
                    + e.getMessage(), e);
    }
```

调度线程如果发现当前trigger的状态不是**ACQUIRED**，也就是说，这个trigger被其他线程fire了，就会返回null。在3.2，我们提到，在调度流程的第三步，如果发现某个trigger第二步的返回值是null，就会跳过第三步，取消fire。在通常的情况下，乐观锁能保证不发生重复调度，但是难免发生ABA问题，我们看一下这是发生重复调度时的日志：

![图片描述](media/记一次Quartz重复调度(任务重复执行)的问题排查/bVbdapd.png)

图3-5 重复调度的日志

在第一步时，也就是quartz在拉取到符合条件的triggers 到将他们的状态由**WAITING**改为**ACQUIRED**之间停顿了有超过9ms的时间，而另一台服务器正是趁着这9ms的空档完成了**WAITING-->ACQUIRED-->EXECUTING-->WAITING**（也就是一个完整的状态变化周期）的全部过程，图示参见图3-6。

![图片描述](media/记一次Quartz重复调度(任务重复执行)的问题排查/bVbdapg.png)

图3-6 重复调度原因示意图

### 3.4 解决办法

如何去解决这个问题呢？在配置文件加上`org.quartz.jobStore.acquireTriggersWithinLock=true`，这样，在调度流程的第一步，也就是拉取待即将触发的triggers时，是上锁的状态，即不会同时存在多个线程拉取到相同的trigger的情况，也就避免的重复调度的危险。

### 3.5 心得

此次排查过程并非一帆风顺，走过一些坑，也有一些非技术相关的体会：

1）学习是一个需要不断打磨，修正的能力。就我个人而言，为了学Quartz，刚开始去翻一个2.4MB大小的源码是毫无头绪，并且效率低下的，所以立刻转换方向，先了解这个框架的运行模式，在做什么，有哪些模块，是怎么做的，再找主线，翻相关的源码。之后在一次次使用中，碰到问题再翻之前没看的源码，就越来越顺利。

之前也听过其他同事的学习方法，感觉并不完全适合自己，可能每个人状态经验不同，学习方法也稍有不同。在平时的学习中，需要去感受自己的学习效率，参考建议，尝试，感受效果，改进，会越来越清晰自己适合什么。这里很感谢我的师父，用简短的话先帮我捋顺了调度流程，这样我再看源码就不那么吃力了。

2）要质疑“经验”和“理所应当”，惯性思维会蒙住你的双眼。在大规模的代码中很容易被习惯迷惑，一开始，我们看到上锁的那个方法的时候，认为这个上锁技巧很棒，这个方法就是为了解决并发的问题，“应该”都上锁了，上锁了就不会有并发的问题了，怎么可能几次与数据库的交互都上锁，突然某一次不上锁呢？直到看到拉取待触发的trigger方法时，觉得有丝丝不对劲，打下日志，才发现实际上是没上锁的。

3）日志很重要。虽然我们可以调试，但是没有日志，我们是无法发现并证明，程序发生了ABA问题。

4）最重要的是，不要害怕问题，即使是Quartz这样大型的框架，解决问题也不一定需要把2.4MB的源码通通读懂。只要有时间，问题都能解决，只是好的技巧能缩短这个时间，而我们需要在一次次实战中磨练技巧。