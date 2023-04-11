---
title:  misfire执行策略
tags: 
 - Quartz
---

# misfire执行策略

默认策略为：**MISFIRE_INSTRUCTION_SMART_POLICY** = 0

**repeatCount**: 根据定时任务类型判断出的重复次数

0. **MISFIRE_INSTRUCTION_SMART_POLICY**
   智能根据trigger属性选择策略：
   repeatCount为0，则策略同**MISFIRE_INSTRUCTION_FIRE_NOW**
   repeatCount为**REPEAT_INDEFINITELY**(无限期重复)，则策略同**MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT**
   否则策略同**MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT**

1. **MISFIRE_INSTRUCTION_FIRE_NOW**

   以当前时间为触发频率立即触发执行

2. **MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT**

​		以当前时间为触发频率立即触发执行，以总次数-已执行次数作为剩余周期次数，重新计算FinalTime
​		调整后的FinalTime会略大于根据starttime计算的到的FinalTime值

3. **MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT**

   不触发立即执行
   等待下次触发频率周期时刻，执行至FinalTime的剩余周期次数
   保持FinalTime不变，重新计算剩余周期次数(相当于错过的当做已执行)

