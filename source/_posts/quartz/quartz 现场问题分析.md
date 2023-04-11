---
title:  Quartz 现场问题分析
tags: 
 - Quartz
---

# Quartz 现场问题分析

# 背景：

李宁项目使用了quartz集群，支持高可用服务。

# 问题：

artemis系统在出库单同步接口请求了多次iwms，导致iwms系统报错，日志表现为一个定时任务本应该一分钟执行一次，但是存在一秒内执行了2次的现象，由于两个服务是交替执行所以也存在每个服务执行一次的情况。

![img](.\f\media\现场日志.png)

# 问题定位：

1. 一开始怀疑是数据库不一致导致，两个服务一起执行定时任务，最后排除了数据库不一致的问题

   ![img](.\f\media\数据库连接.png)

2. 在偶然翻看定时任务时发现了问题，存在两条trigger_name一样的定时任务

   ![image-20220708162904441](.\f\media\存在两条定时任务.png)

3. 在本地采用此配置成功复现了此问题

   ![img](.\f\media\问题复现.png)

# 源码分析

1. artemis采用trigger_name作为triggerCode

   <img src="f\media\Artemis赋值triggercode.png" alt="image-20220708164509385" style="zoom:50%;" />

2. 线程池中先获取到需要触发的trigger

![image-20220708160842046](.\f\media\数据库拉取trigger.png)

2. 等待2ms，直到到达触发时间

   ![image-20220708161049886](.\f\media\trigger job触发前等待.png)

   3. 触发执行job

      ![image-20220708161120719](.\f\media\执行.png)

# 原因分析：

1. quartz是通过timerConfigTask定时执行，刷新数据库中定时任务的状态配置，corn （0 0/1 * * * ?）

![image-20220708163741488](.\f\media\刷新定时任务.png)

2. 在指定出库定时任务为10 0/1 * * * ?时，现象消失

![image-20220708164020634](.\f\media\现象消失.png)

# 验证

日志可见，删除前的任务第一次执行，删除后立马执行第二次

![image-20220708172158549](.\f\media\两次执行日志.png)

# 结论

​	quartz执行定时任务原理是通过quartz是通过调度线程QuartzSchedulerThread不断的扫描数据库中的数据来获取到那些已经到点要触发的任务，然后调度执行。

​	数据库配置两条一样的trigger_name时，会导致qrtz_cron_triggers表这条定时任务不停的删除，新增。当出库单定时任务在一分钟执行时，已经生成jobBean放到线程池中执行，与此同时，timerConfigTask也执行，将此条定时任务删除并新增，且这条新的数据，会被调度线程再次拉取出来，生成jobBean丢到线程池中执行，导致一个定时任务出现两次。

负责任务调度的几个线程：
 （1）任务执行线程：通常使用一个线程池(SimpleThreadPool)维护一组线程，负责实际每个job的执行。
 （2）Scheduler调度线程QuartzSchedulerThread ：轮询存储的所有 trigger，如果有需要触发的 trigger，即到达了下一次触发的时间，则从任务执行线程池获取一个空闲线程，执行与该 trigger 关联的任务。
 （3）处理misfire job的线程MisfireHandler：轮训所有misfire的trigger，原理就是从数据库中查询所有下次触发时间小于当前时间的trigger，按照每个trigger设定的misfire策略处理这些trigger。

# 解决办法

删除多余配置