.. vim: syntax=rst

实现时间片
============

本章开始，我们让OS支持同一个优先级下可以有多个任务的功能，这些任务可以分配不同的时间片，当任务时间片用完的时候，任务会从链表的头部移动到尾部，让下一个任务共享时间片，以此循环。

.. _实现时间片-1:

实现时间片
~~~~~

修改任务TCB
^^^^^^^

为了实现时间片功能，我们需要先在任务控制块TCB中添加两个时间片相关的变量，具体见代码清单11‑1的加粗部分。

代码清单11‑1在 TCB中添加时间片相关的变量

1 struct os_tcb {

2 CPU_STK \*StkPtr;

3 CPU_STK_SIZE StkSize;

4

5 /\* 任务延时周期个数 \*/

6 OS_TICK TaskDelayTicks;

7

8 /\* 任务优先级 \*/

9 OS_PRIO Prio;

10

11 /\* 就绪列表双向链表的下一个指针 \*/

12 OS_TCB \*NextPtr;

13 /\* 就绪列表双向链表的前一个指针 \*/

14 OS_TCB \*PrevPtr;

15

16 /*时基列表相关字段*/

17 OS_TCB \*TickNextPtr;

18 OS_TCB \*TickPrevPtr;

19 OS_TICK_SPOKE \*TickSpokePtr;

20

21 OS_TICK TickCtrMatch;

22 OS_TICK TickRemain;

23

24 /\* 时间片相关字段 \*/

**25 OS_TICK TimeQuanta;**\ (1)

**26 OS_TICK TimeQuantaCtr;**\ (2)

27 };

代码清单11‑1（1）：TimeQuanta表示任务需要多少个时间片，单位为系统时钟周期Tick。

代码清单11‑1（2）：TimeQuantaCtr表示任务还剩下多少个时间片，每到来一个系统时钟周期，TimeQuantaCtr会减一，当TimeQuantaCtr等于零的时候，表示时间片用完，任务的TCB会从就绪列表链表的头部移动到尾部，好让下一个任务共享时间片。\ |slidin002|

实现时间片调度函数
^^^^^^^^^

OS_SchedRoundRobin()函数
''''''''''''''''''''''

时间片调度函数OS_SchedRoundRobin()在os_core.c中实现，在OSTimeTick()调用，具体见代码清单11‑2。在阅读代码清单11‑2的时候，可配图11‑1一起理解，该图画的是在一个就绪链表中，有三个任务就绪，其中在优先级2下面有两个任务，均分配了两个时间片，其中任务3的时
间片已用完，则位于链表的末尾，任务2的时间片还剩一个，则位于链表的头部。当下一个时钟周期到来的时候，任务2的时间片将耗完，相应的TimeQuantaCtr会递减为0，任务2的TCB会被移动到链表的末尾，任务3则被成为链表的头部，然后重置任务3的时间片计数器TimeQuantaCtr的值为2，重新享有
时间片。

|slidin003|

图11‑1时间片调度函数讲解配套

代码清单11‑2时间片调度函数

1 #if OS_CFG_SCHED_ROUND_ROBIN_EN > 0u(1)

2 void OS_SchedRoundRobin(OS_RDY_LIST \*p_rdy_list)

3 {

4 OS_TCB \*p_tcb;

5 CPU_SR_ALLOC();

6

7 /\* 进入临界段 \*/

8 CPU_CRITICAL_ENTER();

9

10 p_tcb = p_rdy_list->HeadPtr;(2)

11

12 /\* 如果TCB节点为空，则退出 \*/

13 if (p_tcb == (OS_TCB \*)0) {(3)

14 CPU_CRITICAL_EXIT();

15 return;

16 }

17

18 /\* 如果是空闲任务，也退出 \*/

19 if (p_tcb == &OSIdleTaskTCB) {(4)

20 CPU_CRITICAL_EXIT();

21 return;

22 }

23

24 /\* 时间片自减 \*/

25 if (p_tcb->TimeQuantaCtr > (OS_TICK)0) {(5)

26 p_tcb->TimeQuantaCtr--;

27 }

28

29 /\* 时间片没有用完，则退出 \*/

30 if (p_tcb->TimeQuantaCtr > (OS_TICK)0) {(6)

31 CPU_CRITICAL_EXIT();

32 return;

33 }

34

35 /\* 如果当前优先级只有一个任务，则退出 \*/

36 if (p_rdy_list->NbrEntries < (OS_OBJ_QTY)2) {(7)

37 CPU_CRITICAL_EXIT();

38 return;

39 }

40

41 /\* 时间片耗完，将任务放到链表的最后一个节点 \*/

42 OS_RdyListMoveHeadToTail(p_rdy_list);(8)

43

44 /\* 重新获取任务节点 \*/

45 p_tcb = p_rdy_list->HeadPtr;(9)

46 /\* 重载默认的时间片计数值 \*/

47 p_tcb->TimeQuantaCtr = p_tcb->TimeQuanta;

48

49 /\* 退出临界段 \*/

50 CPU_CRITICAL_EXIT();

51 }

52 #endif/\* OS_CFG_SCHED_ROUND_ROBIN_EN > 0u \*/

代码清单11‑2（1）：时间片是一个可选的功能，是否选择由OS_CFG_SCHED_ROUND_ROBIN_EN控制，该宏在os_cfg.h定义。

代码清单11‑2（2）：获取链表的第一个节点。

代码清单11‑2（3）：如果节点为空，则退出。

代码清单11‑2（4）：如果节点不为空，看看是否是空闲任务，如果是则退出。

代码清单11‑2（5）：如果不是空闲任务，则时间片计数器TimeQuantaCtr减一操作。

代码清单11‑2（6）：时间片计数器TimeQuantaCtr递减之后，则判断下时间片是否用完，如果没有用完，则退出。

代码清单11‑2（7）：如果时间片用完，则判断性该优先级下有多少个任务，如果是一个，就退出。

代码清单11‑2（8）：时间片用完，如果该优先级下有两个以上任务，则将刚刚耗完时间片的节点移到链表的末尾，此时位于末尾的任务的TCB字段中的TimeQuantaCtr是等于0的，只有等它下一次运行的时候值才会重置为TimeQuanta。

代码清单11‑2（9）：重新获取链表的第一个节点，重置时间片计数器TimeQuantaCtr的值等于TimeQuanta，任务重新享有时间片。

修改OSTimeTick()函数
~~~~~~~~~~~~~~~~

任务的时间片的单位在每个系统时钟周期到来的时候被更新，时间片调度函数则由时基周期处理函数OSTimeTick()调用，只需要在更新时基列表之后调用时间片调度函数即可，具体修改见代码清单11‑3的加粗部分。

代码清单11‑3OSTimeTick()函数

1 void OSTimeTick (void)

2 {

3 /\* 更新时基列表 \*/

4 OS_TickListUpdate();

5

**6 #if OS_CFG_SCHED_ROUND_ROBIN_EN > 0u**

**7 /\* 时间片调度 \*/**

**8 OS_SchedRoundRobin(&OSRdyList[OSPrioCur]);**

**9 #endif**

10

11 /\* 任务调度 \*/

12 OSSched();

13 }

修改OSTaskCreate()函数
~~~~~~~~~~~~~~~~~~

任务的时间片在函数创建的时候被指定，具体修改见代码清单11‑4中的加粗部分。

代码清单11‑4OSTaskCreate()函数

1 void OSTaskCreate (OS_TCB \*p_tcb,

2 OS_TASK_PTR p_task,

3 void \*p_arg,

4 OS_PRIO prio,

5 CPU_STK \*p_stk_base,

6 CPU_STK_SIZE stk_size,

7 **OS_TICK time_quanta,(1)**

8 OS_ERR \*p_err)

9 {

10 CPU_STK \*p_sp;

11 CPU_SR_ALLOC();

12

13 /\* 初始化TCB为默认值 \*/

14 OS_TaskInitTCB(p_tcb);

15

16 /\* 初始化栈 \*/

17 p_sp = OSTaskStkInit( p_task,

18 p_arg,

19 p_stk_base,

20 stk_size );

21

22 p_tcb->Prio = prio;

23

24 p_tcb->StkPtr = p_sp;

25 p_tcb->StkSize = stk_size;

26

27 /\* 时间片相关初始化 \*/

**28 p_tcb->TimeQuanta = time_quanta;(2)**

**29 #if OS_CFG_SCHED_ROUND_ROBIN_EN > 0u**

**30 p_tcb->TimeQuantaCtr = time_quanta;(3)**

**31 #endif**

32

33 /\* 进入临界段 \*/

34 OS_CRITICAL_ENTER();

35

36 /\* 将任务添加到就绪列表 \*/

37 OS_PrioInsert(p_tcb->Prio);

38 OS_RdyListInsertTail(p_tcb);

39

40 /\* 退出临界段 \*/

41 OS_CRITICAL_EXIT();

42

43 \*p_err = OS_ERR_NONE;

44 }

代码清单11‑4（1）：时间片在任务创建的时候由函数形参time_quanta指定。

代码清单11‑4（2）：初始化任务TCB字段的时间片变量TimeQuanta，该变量表示任务能享有的最大的时间片是多少，该值一旦初始化后就不会变，除非认为修改。

代码清单11‑4（3）：初始化时间片计数器TimeQuantaCtr的值等于TimeQuanta，每经过一个系统时钟周期，该值会递减，如果该值为0，则表示时间片耗完。

修改OS_IdleTaskInit()函数
~~~~~~~~~~~~~~~~~~~~~

因为在OS_IdleTaskInit()函数中创建了空闲任务，所以该函数也需要修改，只需在空闲任务创建函数中，添加一个时间片的形参就可，时间片我们分配为0，因为在空闲任务优先级下只有空闲任务一个任务，没有其他的任务，具体修改见代码清单11‑5的加粗部分。

代码清单11‑5OS_IdleTaskInit()函数

1 void OS_IdleTaskInit(OS_ERR \*p_err)

2 {

3 /\* 初始化空闲任务计数器 \*/

4 OSIdleTaskCtr = (OS_IDLE_CTR)0;

5

6 /\* 创建空闲任务 \*/

7 OSTaskCreate( (OS_TCB \*)&OSIdleTaskTCB,

8 (OS_TASK_PTR )OS_IdleTask,

9 (void \*)0,

10 (OS_PRIO)(OS_CFG_PRIO_MAX - 1u),

11 (CPU_STK \*)OSCfg_IdleTaskStkBasePtr,

12 (CPU_STK_SIZE)OSCfg_IdleTaskStkSize,

13 **(OS_TICK )0,**

14 (OS_ERR \*)p_err );

15 }

main()函数
~~~~~~~~

这里，我们创建任务1、2和3，其中任务1的优先级为1，时间片为0，任务2和任务3的优先级相同，均为2，均分配两个两个时间片，当任务创建完毕后，就绪列表的分布图具体见图11‑2。

|slidin002|

图11‑2main()函数代码讲解配图

代码清单11‑6 main()函数

1 int main(void)

2 {

3 OS_ERR err;

4

5

6 /\* CPU初始化：1、初始化时间戳 \*/

7 CPU_Init();

8

9 /\* 关闭中断 \*/

10 CPU_IntDis();

11

12 /\* 配置SysTick 10ms 中断一次 \*/

13 OS_CPU_SysTickInit (10);

14

15 /\* 初始化相关的全局变量 \*/

16 OSInit(&err);

17

18 /\* 创建任务 \*/

19 OSTaskCreate( (OS_TCB \*)&Task1TCB,

20 (OS_TASK_PTR )Task1,

21 (void \*)0,

22 **(OS_PRIO )1,(1)**

23 (CPU_STK \*)&Task1Stk[0],

24 (CPU_STK_SIZE )TASK1_STK_SIZE,

25 **(OS_TICK )0,(1)**

26 (OS_ERR \*)&err );

27

28 OSTaskCreate( (OS_TCB \*)&Task2TCB,

29 (OS_TASK_PTR )Task2,

30 (void \*)0,

31 **(OS_PRIO )2,(2)**

32 (CPU_STK \*)&Task2Stk[0],

33 (CPU_STK_SIZE )TASK2_STK_SIZE,

34 **(OS_TICK )1,(2)**

35 (OS_ERR \*)&err );

36

37 OSTaskCreate( (OS_TCB \*)&Task3TCB,

38 (OS_TASK_PTR )Task3,

39 (void \*)0,

40 **(OS_PRIO )2,(2)**

41 (CPU_STK \*)&Task3Stk[0],

42 (CPU_STK_SIZE )TASK3_STK_SIZE,

43 **(OS_TICK )1,(2)**

44 (OS_ERR \*)&err );

45

46 /\* 启动OS，将不再返回 \*/

47 OSStart(&err);

48 }

49

50 void Task1( void \*p_arg )

51 {

52 for ( ;; ) {

53 flag1 = 1;

54 OSTimeDly(2);

55 flag1 = 0;

56 OSTimeDly(2);

57 }

58 }

59

60 void Task2( void \*p_arg )

61 {

62 for ( ;; ) {

63 flag2 = 1;

**64 //OSTimeDly(1);(3)**

**65 delay(0xff);**

66 flag2 = 0;

**67 //OSTimeDly(1);**

**68 delay(0xff);**

69 }

70 }

71

72 void Task3( void \*p_arg )

73 {

74 for ( ;; ) {

75 flag3 = 1;

**76 //OSTimeDly(1);(3)**

**77 delay(0xff);**

78 flag3 = 0;

**79 //OSTimeDly(1);**

**80 delay(0xff);**

81 }

82 }

代码清单11‑6（1）：任务1的优先级为1，时间片为0。当同一个优先级下有多个任务的时候才需要时间片功能。

代码清单11‑6（2）：任务2和任务3的优先级相同，均为2，且分配相同的时间片，时间片也可以不同。

代码清单11‑6（3）：因为任务2和3的优先级相同，分配了相同的时间片，也可以分配不同的时间片，并把阻塞延时换成软件延时，不管是阻塞延时还是软件延时，延时的时间都必须小于时间片，因为相同优先级的任务在运行的时候最大不能超过时间片的时间。

实验现象
~~~~

进入软件调试，单击全速运行按钮就可看到实验波形，具体见图11‑3。在图中我们可以看到，在任务1的flag1置1和置0的两个时间片内，任务2和3都各运行了一次，运行的时间均为1个时间片，在这1个时间片内任务2和3的flag变量翻转了好多次，即任务运行了好多次。

|slidin004|

图11‑3实验现象

.. |slidin002| image:: media\slidin002.png
   :width: 4.55903in
   :height: 3.44167in
.. |slidin003| image:: media\slidin003.png
   :width: 4.43889in
   :height: 3.20069in
.. |slidin002| image:: media\slidin002.png
   :width: 4.55903in
   :height: 3.44167in
.. |slidin004| image:: media\slidin004.png
   :width: 5.76806in
   :height: 1.20278in
