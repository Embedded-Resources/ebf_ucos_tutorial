.. vim: syntax=rst

任务的删除
============

本章开始，我们让OS的任务支持删除操作，一个任务被删除后就进入休眠态，要想继续运行必须创新创建。

实现任务删除
~~~~~~

编写任务删除函数
^^^^^^^^

OSTaskDel()函数
'''''''''''''

任务删除函数OSTaskDel()用于删除一个指定的任务，也可以删除自身，在os_task.c中定义，具体实现见代码清单13‑1。

代码清单13‑1OSTaskDel()函数

1 #if OS_CFG_TASK_DEL_EN > 0u(1)

2 void OSTaskDel (OS_TCB \*p_tcb,

3 OS_ERR \*p_err)

4 {

5 CPU_SR_ALLOC();

6

7 /\* 不允许删除空闲任务 \*/(2)

8 if (p_tcb == &OSIdleTaskTCB) {

9 \*p_err = OS_ERR_TASK_DEL_IDLE;

10 return;

11 }

12

13 /\* 删除自己 \*/

14 if (p_tcb == (OS_TCB \*)0) {(3)

15 CPU_CRITICAL_ENTER();

16 p_tcb = OSTCBCurPtr;

17 CPU_CRITICAL_EXIT();

18 }

19

20 OS_CRITICAL_ENTER();

21

22 /\* 根据任务的状态来决定删除的动作 \*/

23 switch (p_tcb->TaskState) {

24 case OS_TASK_STATE_RDY:(4)

25 OS_RdyListRemove(p_tcb);

26 break;

27

28 case OS_TASK_STATE_SUSPENDED:(5)

29 break;

30

31 /\* 任务只是在延时，并没有在任何等待列表*/

32 case OS_TASK_STATE_DLY:(6)

33 case OS_TASK_STATE_DLY_SUSPENDED:

34 OS_TickListRemove(p_tcb);

35 break;

36

37 case OS_TASK_STATE_PEND:(7)

38 case OS_TASK_STATE_PEND_SUSPENDED:

39 case OS_TASK_STATE_PEND_TIMEOUT:

40 case OS_TASK_STATE_PEND_TIMEOUT_SUSPENDED:

41 OS_TickListRemove(p_tcb);

42

43 #if 0/\* 目前我们还没有实现等待列表，暂时先把这部分代码注释 \*/

44 /\* 看看在等待什么 \*/

45 switch (p_tcb->PendOn) {

46 case OS_TASK_PEND_ON_NOTHING:

47 /\* 任务信号量和队列没有等待队列，直接退出 \*/

48 case OS_TASK_PEND_ON_TASK_Q:

49 case OS_TASK_PEND_ON_TASK_SEM:

50 break;

51

52 /\* 从等待列表移除 \*/

53 case OS_TASK_PEND_ON_FLAG:

54 case OS_TASK_PEND_ON_MULTI:

55 case OS_TASK_PEND_ON_MUTEX:

56 case OS_TASK_PEND_ON_Q:

57 case OS_TASK_PEND_ON_SEM:

58 OS_PendListRemove(p_tcb);

59 break;

60

61 default:

62 break;

63 }

64 break;

65 #endif

66 default:

67 OS_CRITICAL_EXIT();

68 \*p_err = OS_ERR_STATE_INVALID;

69 return;

70 }

71

72 /\* 初始化TCB为默认值 \*/

73 OS_TaskInitTCB(p_tcb);(8)

74 /\* 修改任务的状态为删除态，即处于休眠 \*/

75 p_tcb->TaskState = (OS_STATE)OS_TASK_STATE_DEL;(9)

76

77 OS_CRITICAL_EXIT_NO_SCHED();

78 /\* 任务切换，寻找最高优先级的任务 \*/

79 OSSched();(10)

80

81 \*p_err = OS_ERR_NONE;

82 }

83 #endif/\* OS_CFG_TASK_DEL_EN > 0u \*/

代码清单13‑1（1）：任务删除是一个可选功能，由OS_CFG_TASK_DEL_EN控制，该宏在os_cfg.h中定义。

代码清单13‑1（2）：空闲任务不能被删除。系统必须至少有一个任务在运行，当没有其他用户任务运行的时候，系统就会运行空闲任务。

代码清单13‑1（3）：删除自己。

代码清单13‑1（4）：任务只在就绪态，则从就绪列表移除。

代码清单13‑1（5）：任务只是被挂起，则退出返回，不用做什么。

代码清单13‑1（6）：任务在延时或者是延时加挂起，则从时基列表移除。

代码清单13‑1（7）：任务在多种状态，但只要有一种是等待状态，就需要从等待列表移除。如果任务等待是任务自身的信号量和消息，则直接退出返回，因为任务信号量和消息是没有等待列表的。等待列表我们暂时还没实现，所以暂时将等待部分相关的代码用条件编译屏蔽掉。

代码清单13‑1（8）：初始化TCB为默认值。

代码清单13‑1（9）：修改任务的状态为删除态，即处于休眠。

代码清单13‑1（10）：任务调度，寻找优先级最高的任务来运行。

main()函数
~~~~~~~~

本章main()函数没有添加新的测试代码，只需理解章节内容即可。

实验现象
~~~~

本章没有实验，只需理解章节内容即可。
