.. vim: syntax=rst

任务的挂起和恢复
===================

本章开始，我们让OS的任务支持挂起和恢复的功能，挂起就相当于暂停，暂停后任务从就绪列表中移除，恢复即重新将任务插入就绪列表。一个任务挂起多少次就要被恢复多少次才能重新运行。

实现任务的挂起和恢复
~~~~~~~~~~

定义任务的状态
^^^^^^^

在任务实现挂起和恢复的时候，要根据任务的状态来操作，任务的状态不同，操作也不同，有关任务状态的宏定义在os.h中实现，总共有9种状态，具体定义见代码清单12‑1。

代码清单12‑1定义任务的状态

1 /\* ---------- 任务的状态 -------*/

2 #define OS_TASK_STATE_BIT_DLY (OS_STATE)(0x01u)/\* /-------- 挂起位 \*/

3 /\* \| \*/

4 #define OS_TASK_STATE_BIT_PEND (OS_STATE)(0x02u)/\* \| /----- 等待位 \*/

5/\* \| \| \*/

6 #define OS_TASK_STATE_BIT_SUSPENDED (OS_STATE)(0x04u)/\* \| \| /--- 延时/超时位 \*/

7 /\* \| \| \| \*/

8 /\* V V V \*/

9

10 #define OS_TASK_STATE_RDY (OS_STATE)( 0u)/\* 0 0 0 就绪 \*/

11 #define OS_TASK_STATE_DLY (OS_STATE)( 1u)/\* 0 0 1 延时或者超时 \*/

12 #define OS_TASK_STATE_PEND (OS_STATE)( 2u)/\* 0 1 0 等待 \*/

13 #define OS_TASK_STATE_PEND_TIMEOUT (OS_STATE)( 3u)/\* 0 1 1 等待+超时*/

14 #define OS_TASK_STATE_SUSPENDED (OS_STATE)( 4u)/\* 1 0 0 挂起 \*/

15 #define OS_TASK_STATE_DLY_SUSPENDED (OS_STATE)( 5u)/\* 1 0 1 挂起 + 延时或者超时*/

16 #define OS_TASK_STATE_PEND_SUSPENDED (OS_STATE)( 6u)/\* 1 1 0 挂起 + 等待 \*/

17 #define OS_TASK_STATE_PEND_TIMEOUT_SUSPENDED (OS_STATE)( 7u)/\* 1 1 1 挂起 + 等待 + 超时*/

18 #define OS_TASK_STATE_DEL (OS_STATE)(255u)

修改任务控制块TCB
^^^^^^^^^^

为了实现任务的挂起和恢复，需要先在任务控制中TCB中添加任务的状态TaskState和任务挂起计数器SusPendCtr这两个成员，具体见代码清单12‑2的加粗部分。

代码清单12‑2任务TCB

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

25 OS_TICK TimeQuanta;

26 OS_TICK TimeQuantaCtr;

27

**28 OS_STATE TaskState;(1)**

**29**

**30 #if OS_CFG_TASK_SUSPEND_EN > 0u(2)**

**31 /\* 任务挂起函数OSTaskSuspend()计数器 \*/**

**32 OS_NESTING_CTR SuspendCtr;(3)**

**33 #endif**

34

35 };

代码清单12‑2（1）：TaskState用来表示任务的状态，在本章之前，任务出现了两种状态，一是任务刚刚创建好的时候，处于就绪态，调用阻塞延时函数的时候处于延时态。本章要实现的是任务的挂起态，再往后的章节中还会有等待态，超时态，删除态等。TaskState能够取的值具体见代码清单12‑1。

代码清单12‑2（2）：任务挂起功能是可选的，通过宏OS_CFG_TASK_SUSPEND_EN来控制，该宏在os_cfg.h文件中定义。

代码清单12‑2（3）：任务挂起计数器，任务每被挂起一次，SuspendCtr递增一次，一个任务挂起多少次就要被恢复多少次才能重新运行。

编写任务挂起和恢复函数
^^^^^^^^^^^

OSTaskSuspend()函数
'''''''''''''''''

OSTaskSuspend()函数

代码清单12‑3OSTaskSuspend()函数

1 #if OS_CFG_TASK_SUSPEND_EN > 0u

2 void OSTaskSuspend (OS_TCB \*p_tcb,

3 OS_ERR \*p_err)

4 {

5 CPU_SR_ALLOC();

6

7

8 #if 0/\* 屏蔽开始 \*/ **(1)**

9 #ifdef OS_SAFETY_CRITICAL

10 /\* 安全检查，OS_SAFETY_CRITICAL_EXCEPTION()函数需要用户自行编写 \*/

11 if (p_err == (OS_ERR \*)0) {

12 OS_SAFETY_CRITICAL_EXCEPTION();

13 return;

14 }

15 #endif

16

17 #if OS_CFG_CALLED_FROM_ISR_CHK_EN > 0u

18 /\* 不能在ISR程序中调用该函数 \*/

19 if (OSIntNestingCtr > (OS_NESTING_CTR)0) {

20 \*p_err = OS_ERR_TASK_SUSPEND_ISR;

21 return;

22 }

23 #endif

24

25 /\* 不能挂起空闲任务 \*/

26 if (p_tcb == &OSIdleTaskTCB) {

27 \*p_err = OS_ERR_TASK_SUSPEND_IDLE;

28 return;

29 }

30

31 #if OS_CFG_ISR_POST_DEFERRED_EN > 0u

32 /\* 不能挂起中断处理任务 \*/

33 if (p_tcb == &OSIntQTaskTCB) {

34 \*p_err = OS_ERR_TASK_SUSPEND_INT_HANDLER;

35 return;

36 }

37 #endif

38

39 #endif/\* 屏蔽结束 \*/ **(2)**

40

41 CPU_CRITICAL_ENTER();

42

43 /\* 是否挂起自己 \*/ **(3)**

44 if (p_tcb == (OS_TCB \*)0) {

45 p_tcb = OSTCBCurPtr;

46 }

47

48 if (p_tcb == OSTCBCurPtr) {

49 /\* 如果调度器锁住则不能挂起自己 \*/

50 if (OSSchedLockNestingCtr > (OS_NESTING_CTR)0) {

51 CPU_CRITICAL_EXIT();

52 \*p_err = OS_ERR_SCHED_LOCKED;

53 return;

54 }

55 }

56

57 \*p_err = OS_ERR_NONE;

58

59 /\* 根据任务的状态来决定挂起的动作 \*/(4)

60 switch (p_tcb->TaskState) {

61 case OS_TASK_STATE_RDY:(5)

62 OS_CRITICAL_ENTER_CPU_CRITICAL_EXIT();

63 p_tcb->TaskState = OS_TASK_STATE_SUSPENDED;

64 p_tcb->SuspendCtr = (OS_NESTING_CTR)1;

65 OS_RdyListRemove(p_tcb);

66 OS_CRITICAL_EXIT_NO_SCHED();

67 break;

68

69 case OS_TASK_STATE_DLY:(6)

70 p_tcb->TaskState = OS_TASK_STATE_DLY_SUSPENDED;

71 p_tcb->SuspendCtr = (OS_NESTING_CTR)1;

72 CPU_CRITICAL_EXIT();

73 break;

74

75 case OS_TASK_STATE_PEND:(7)

76 p_tcb->TaskState = OS_TASK_STATE_PEND_SUSPENDED;

77 p_tcb->SuspendCtr = (OS_NESTING_CTR)1;

78 CPU_CRITICAL_EXIT();

79 break;

80

81 case OS_TASK_STATE_PEND_TIMEOUT:(8)

82 p_tcb->TaskState = OS_TASK_STATE_PEND_TIMEOUT_SUSPENDED;

83 p_tcb->SuspendCtr = (OS_NESTING_CTR)1;

84 CPU_CRITICAL_EXIT();

85 break;

86

87 case OS_TASK_STATE_SUSPENDED:(9)

88 case OS_TASK_STATE_DLY_SUSPENDED:

89 case OS_TASK_STATE_PEND_SUSPENDED:

90 case OS_TASK_STATE_PEND_TIMEOUT_SUSPENDED:

91 p_tcb->SuspendCtr++;

92 CPU_CRITICAL_EXIT();

93 break;

94

95 default:(10)

96 CPU_CRITICAL_EXIT();

97 \*p_err = OS_ERR_STATE_INVALID;

98 return;

99 }

100

101 /\* 任务切换 \*/

102 OSSched();(11)

103 }

104 #endif

代码清单12‑3（1）和（2）：这部分代码是为了程序的健壮性写的代码，即是加了各种判断，避免用户的误操作。在μC/OS-
III中，这段代码随处可见，但为了讲解方便，我们把这部分代码注释掉，里面涉及的一些宏和函数我们均不实现，只需要了解即可，在后面的讲解中，要是出现这段代码，我们直接删除掉，删除掉也不会影响核心功能。

代码清单12‑3（3）：如果任务挂起的是自己，则判断下调度器是否锁住，如果锁住则退出返回错误码，没有锁则继续往下执行。

代码清单12‑3（4）：根据任务的状态来决定挂起操作。

代码清单12‑3（5）：任务在就绪状态，则将任务的状态改为挂起态，挂起计数器置1，然后从就绪列表删除。

代码清单12‑3（6）：任务在延时状态，则将任务的状态改为延时加挂起态，挂起计数器置1，不用改变TCB的位置，即还是在延时的时基列表。

代码清单12‑3（7）：任务在等待状态，则将任务的状态改为等待加挂起态，挂起计数器置1，不用改变TCB的位置，即还是在等待列表等待。等待列表暂时还没有实现，将会在后面的章节实现。

代码清单12‑3（8）：任务在等待加超时态，则将任务的状态改为等待加超时加挂起态，挂起计数器置1，不用改变TCB的位置，即还在等待和时基这两个列表中。

代码清单12‑3（9）：只要有一个是挂起状态，则将挂起计数器加一操作，不用改变TCB的位置。

代码清单12‑3（10）：其他状态则无效，退出返回状态无效错误码。

代码清单12‑3（11）：任务切换。凡是涉及改变任务状态的地方，都需要进行任务切换。

OSTaskResume()函数
''''''''''''''''

OSTaskResume()函数用于恢复被挂起的函数，但是不能恢复自己，挂起倒是可以挂起自己，具体实现见代码清单12‑4。

代码清单12‑4OSTaskResume()函数

1 #if OS_CFG_TASK_SUSPEND_EN > 0u

2 void OSTaskResume (OS_TCB \*p_tcb,

3 OS_ERR \*p_err)

4 {

5 CPU_SR_ALLOC();

6

7

8 #if 0/\* 屏蔽开始 \*/(1)

9 #ifdef OS_SAFETY_CRITICAL

10 /\* 安全检查，OS_SAFETY_CRITICAL_EXCEPTION()函数需要用户自行编写 \*/

11 if (p_err == (OS_ERR \*)0) {

12 OS_SAFETY_CRITICAL_EXCEPTION();

13 return;

14 }

15 #endif

16

17 #if OS_CFG_CALLED_FROM_ISR_CHK_EN > 0u

18 /\* 不能在ISR程序中调用该函数 \*/

19 if (OSIntNestingCtr > (OS_NESTING_CTR)0) {

20 \*p_err = OS_ERR_TASK_RESUME_ISR;

21 return;

22 }

23 #endif

24

25

26 CPU_CRITICAL_ENTER();

27 #if OS_CFG_ARG_CHK_EN > 0u

28 /\* 不能自己恢复自己 \*/

29 if ((p_tcb == (OS_TCB \*)0) \|\|

30 (p_tcb == OSTCBCurPtr)) {

31 CPU_CRITICAL_EXIT();

32 \*p_err = OS_ERR_TASK_RESUME_SELF;

33 return;

34 }

35 #endif

36

37 #endif/\* 屏蔽结束 \*/(2)

38

39 \*p_err = OS_ERR_NONE;

40 /\* 根据任务的状态来决定挂起的动作 \*/

41 switch (p_tcb->TaskState) {(3)

42 case OS_TASK_STATE_RDY:(4)

43 case OS_TASK_STATE_DLY:

44 case OS_TASK_STATE_PEND:

45 case OS_TASK_STATE_PEND_TIMEOUT:

46 CPU_CRITICAL_EXIT();

47 \*p_err = OS_ERR_TASK_NOT_SUSPENDED;

48 break;

49

50 case OS_TASK_STATE_SUSPENDED:(5)

51 OS_CRITICAL_ENTER_CPU_CRITICAL_EXIT();

52 p_tcb->SuspendCtr--;

53 if (p_tcb->SuspendCtr == (OS_NESTING_CTR)0) {

54 p_tcb->TaskState = OS_TASK_STATE_RDY;

55 OS_TaskRdy(p_tcb);

56 }

57 OS_CRITICAL_EXIT_NO_SCHED();

58 break;

59

60 case OS_TASK_STATE_DLY_SUSPENDED:(6)

61 p_tcb->SuspendCtr--;

62 if (p_tcb->SuspendCtr == (OS_NESTING_CTR)0) {

63 p_tcb->TaskState = OS_TASK_STATE_DLY;

64 }

65 CPU_CRITICAL_EXIT();

66 break;

67

68 case OS_TASK_STATE_PEND_SUSPENDED:(7)

69 p_tcb->SuspendCtr--;

70 if (p_tcb->SuspendCtr == (OS_NESTING_CTR)0) {

71 p_tcb->TaskState = OS_TASK_STATE_PEND;

72 }

73 CPU_CRITICAL_EXIT();

74 break;

75

76 case OS_TASK_STATE_PEND_TIMEOUT_SUSPENDED:(8)

77 p_tcb->SuspendCtr--;

78 if (p_tcb->SuspendCtr == (OS_NESTING_CTR)0) {

79 p_tcb->TaskState = OS_TASK_STATE_PEND_TIMEOUT;

80 }

81 CPU_CRITICAL_EXIT();

82 break;

83

84 default:(9)

85 CPU_CRITICAL_EXIT();

86 \*p_err = OS_ERR_STATE_INVALID;

87 return;

88 }

89

90 /\* 任务切换 \*/

91 OSSched();(10)

92 }

93 #endif

代码清单12‑4（1）和（2）：这部分代码是为了程序的健壮性写的代码，即是加了各种判断，避免用户的误操作。在μC/OS-
III中，这段代码随处可见，但为了讲解方便，我们把这部分代码注释掉，里面涉及的一些宏和函数我们均不实现，只需要了解即可，在后面的讲解中，要是出现这段代码，我们直接删除掉，删除掉也不会影响核心功能。

代码清单12‑4（3）：根据任务的状态来决定恢复操作。

代码清单12‑4（4）：只要任务没有被挂起，则退出返回任务没有被挂起的错误码。

代码清单12‑4（5）：任务只在挂起态，则递减挂起计数器SuspendCtr，如果SuspendCtr等于0，则将任务的状态改为就绪态，并让任务就绪。

代码清单12‑4（6）：任务在延时加挂起态，则递减挂起计数器SuspendCtr，如果SuspendCtr等于0，则将任务的状态改为延时态。

代码清单12‑4（7）：任务在延时加等待态，则递减挂起计数器SuspendCtr，如果SuspendCtr等于0，则将任务的状态改为等待态。

代码清单12‑4（8）：任务在等待加超时加挂起态，则递减挂起计数器SuspendCtr，如果SuspendCtr等于0，则将任务的状态改为等待加超时态

代码清单12‑4（9）：其他状态则无效，退出返回状态无效错误码。

代码清单12‑4（10）：任务切换。凡是涉及改变任务状态的地方，都需要进行任务切换。

main()函数
~~~~~~~~

这里，我们创建任务1、2和3，其中任务的优先级为1，任务2的优先级为2，任务3的优先级为3。任务1将自身的flag每翻转一次后均将自己挂起，任务2在经过两个时钟周期后将任务1恢复，任务3每隔一个时钟周期翻转一次。具体代码见代码清单12‑5。

代码清单12‑5 main()函数

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

22 (OS_PRIO )1,

23 (CPU_STK \*)&Task1Stk[0],

24 (CPU_STK_SIZE )TASK1_STK_SIZE,

25 (OS_TICK )0,

26 (OS_ERR \*)&err );

27

28 OSTaskCreate( (OS_TCB \*)&Task2TCB,

29 (OS_TASK_PTR )Task2,

30 (void \*)0,

31 (OS_PRIO )2,

32 (CPU_STK \*)&Task2Stk[0],

33 (CPU_STK_SIZE )TASK2_STK_SIZE,

34 (OS_TICK )0,

35 (OS_ERR \*)&err );

36

37 OSTaskCreate( (OS_TCB \*)&Task3TCB,

38 (OS_TASK_PTR )Task3,

39 (void \*)0,

40 (OS_PRIO )3,

41 (CPU_STK \*)&Task3Stk[0],

42 (CPU_STK_SIZE )TASK3_STK_SIZE,

43 (OS_TICK )0,

44 (OS_ERR \*)&err );

45

46 /\* 启动OS，将不再返回 \*/

47 OSStart(&err);

48 }

49

50 void Task1( void \*p_arg )

51 {

52 OS_ERR err;

53

54 for ( ;; ) {

55 flag1 = 1;

56 OSTaskSuspend(&Task1TCB,&err);

57 flag1 = 0;

58 OSTaskSuspend(&Task1TCB,&err);

59 }

60 }

61

62 void Task2( void \*p_arg )

63 {

64 OS_ERR err;

65

66 for ( ;; ) {

67 flag2 = 1;

68 OSTimeDly(1);

69 //OSTaskResume(&Task1TCB,&err);

70 flag2 = 0;

71 OSTimeDly(1);;

72 OSTaskResume(&Task1TCB,&err);

73 }

74 }

75

76 void Task3( void \*p_arg )

77 {

78 for ( ;; ) {

79 flag3 = 1;

80 OSTimeDly(1);

81 flag3 = 0;

82 OSTimeDly(1);

83 }

84 }

实验现象
~~~~

进入软件调试，单击全速运行按钮就可看到实验波形，具体见图12‑1。在图12‑1中，可以看到任务2和任务3的波形图是一样的，任务1的波形周期是任务2的两倍，与代码实现相符。如果想实现其他效果可自行修改代码实现。

|Suspen002|

图12‑1实验现象

.. |Suspen002| image:: media\Suspen002.png
   :width: 5in
   :height: 1.92986in
