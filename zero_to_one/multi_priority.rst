.. vim: syntax=rst

支持多优先级
================

在本章之前，OS还没有到优先级，只支持两个任务互相切换，从本章开始，任务中我们开始加入优先级的功能。在μC/OS-III中，数字优先级越小，逻辑优先级越高。

定义优先级相关全局变量
~~~~~~~~~~~

在支持任务多优先级的时候，需要在os.h头文件添加两个优先级相关的全局变量，具体定义见代码清单9‑1。

代码清单9‑1定义优先级相关全局变量

1 /\* 在os.h中定义 \*/

2 /\* 当前优先级 \*/

3 OS_EXT OS_PRIO OSPrioCur;

4 /\* 最高优先级 \*/

5 OS_EXT OS_PRIO OSPrioHighRdy;

修改OSInit()函数
~~~~~~~~~~~~

刚刚新添加的优先级相关的全部变量，需要在OSInit()函数中进行初始化，具体见代码清单9‑2中的加粗部分代码，其实OS中定义的所有的全局变量都是在OSInit()中初始化的。

代码清单9‑2OSInit()函数

1 void OSInit (OS_ERR \*p_err)

2 {

3 /\* 配置OS初始状态为停止态 \*/

4 OSRunning = OS_STATE_OS_STOPPED;

5

6 /\* 初始化两个全局TCB，这两个TCB用于任务切换 \*/

7 OSTCBCurPtr = (OS_TCB \*)0;

8 OSTCBHighRdyPtr = (OS_TCB \*)0;

9

**10 /\* 初始化优先级变量 \*/**

**11 OSPrioCur = (OS_PRIO)0;**

**12 OSPrioHighRdy = (OS_PRIO)0;**

13

14 /\* 初始化优先级表 \*/

15 OS_PrioInit();

16

17 /\* 初始化就绪列表 \*/

18 OS_RdyListInit();

19

20 /\* 初始化空闲任务 \*/

21 OS_IdleTaskInit(p_err);

22 if (*p_err != OS_ERR_NONE) {

23 return;

24 }

25 }

修改任务控制块TCB
~~~~~~~~~~

在任务控制块中，加入优先级字段Prio，具体见代码清单9‑3中的加粗代码。优先级Prio的数据类型为OS_PRIO，宏展开后是8位的整型，所以只支持255个优先级。

代码清单9‑3在TCB中加入优先级

1 struct os_tcb {

2 CPU_STK \*StkPtr;

3 CPU_STK_SIZE StkSize;

4

5 /\* 任务延时周期个数 \*/

6 OS_TICK TaskDelayTicks;

7

**8 /\* 任务优先级 \*/**

**9 OS_PRIO Prio;**

10

11 /\* 就绪列表双向链表的下一个指针 \*/

12 OS_TCB \*NextPtr;

13 /\* 就绪列表双向链表的前一个指针 \*/

14 OS_TCB \*PrevPtr;

15 };

修改OSTaskCreate()函数
~~~~~~~~~~~~~~~~~~

修改OSTaskCreate()函数，在里面加入优先级相关的处理，具体见代码清单9‑4的加粗部分。

代码清单9‑4OSTaskCreate()函数加入优先级处理

1 void OSTaskCreate (OS_TCB \*p_tcb,

2 OS_TASK_PTR p_task,

3 void \*p_arg,

**4 OS_PRIO prio,**\ (1)

5 CPU_STK \*p_stk_base,

6 CPU_STK_SIZE stk_size,

7 OS_ERR \*p_err)

8 {

9 CPU_STK \*p_sp;

**10 CPU_SR_ALLOC();**\ (2)

11

**12 /\* 初始化TCB为默认值 \*/**

**13 OS_TaskInitTCB(p_tcb);**\ (3)

14

15 /\* 初始化栈 \*/

16 p_sp = OSTaskStkInit( p_task,

17 p_arg,

18 p_stk_base,

19 stk_size );

20

**21 p_tcb->Prio = prio;**\ (4)

22

23 p_tcb->StkPtr = p_sp;

24 p_tcb->StkSize = stk_size;

25

**26 /\* 进入临界段 \*/**

**27 OS_CRITICAL_ENTER();**\ (5)

28

**29 /\* 将任务添加到就绪列表 \*/**\ (6)

**30 OS_PrioInsert(p_tcb->Prio);**

**31 OS_RdyListInsertTail(p_tcb);**

**32**

**33 /\* 退出临界段 \*/**

**34 OS_CRITICAL_EXIT();**\ (7)

35

36 \*p_err = OS_ERR_NONE;

37 }

代码清单9‑4（1）：在函数形参中，加入优先级字段。任务的优先级由用户在创建任务的时候通过形参Prio传进来。

代码清单9‑4（2）：定义一个局部变量，用来存CPU关中断前的中断状态，因为接下来将任务添加到就绪列表这段代码属于临界短代码，需要关中断。

代码清单9‑4（3）：初始化TCB为默认值，其实就是全部初始化为0，OS_TaskInitTCB()函数在os_task.c的开头定义，具体见代码清单9‑5。

代码清单9‑5OS_TaskInitTCB()函数

1 void OS_TaskInitTCB (OS_TCB \*p_tcb)

2 {

3 p_tcb->StkPtr = (CPU_STK \*)0;

4 p_tcb->StkSize = (CPU_STK_SIZE )0u;

5

6 p_tcb->TaskDelayTicks = (OS_TICK )0u;

7

8 p_tcb->Prio = (OS_PRIO )OS_PRIO_INIT;(1)

9

10 p_tcb->NextPtr = (OS_TCB \*)0;

11 p_tcb->PrevPtr = (OS_TCB \*)0;

12 }

代码清单9‑5（1）：OS_PRIO_INIT是任务TCB初始化的时候给的默认的一个优先级，宏展开等于OS_CFG_PRIO_MAX，这是一个不会被OS使用到的优先级。OS_PRIO_INIT具体在os.h中定义。

代码清单9‑4（4）：将形参传进来的优先级存到任务控制块TCB的优先级字段。

代码清单9‑4（5）：进入临界段。

代码清单9‑4（6）：将任务插入就绪列表，这里需要分成两步来实现：1、根据优先级置位优先级表中的相应位置；2、将任务TCB放到OSRdyList[优先级]中，如果同一个优先级有多个任务，那么这些任务的TCB就会被放到OSRdyList[优先级]串成一个双向链表。

代码清单9‑4（7）：退出临界段。

修改OS_IdleTaskInit()函数
~~~~~~~~~~~~~~~~~~~~~

修改OS_IdleTaskInit()函数，是因为该函数调用了任务创建函数OSTaskCreate()，OSTaskCreate()我们刚刚加入了优先级，所以这里我们要跟空闲任务分配一个优先级，具体见。代码清单9‑6的加粗部分。

代码清单9‑6OS_IdleTaskInit()函数

1 /\* 空闲任务初始化 \*/

2 void OS_IdleTaskInit(OS_ERR \*p_err)

3 {

4 /\* 初始化空闲任务计数器 \*/

5 OSIdleTaskCtr = (OS_IDLE_CTR)0;

6

7 /\* 创建空闲任务 \*/

8 OSTaskCreate( (OS_TCB \*)&OSIdleTaskTCB,

9 (OS_TASK_PTR )OS_IdleTask,

10 (void \*)0,

**11 (OS_PRIO)(OS_CFG_PRIO_MAX - 1u),**\ (1)

12 (CPU_STK \*)OSCfg_IdleTaskStkBasePtr,

13 (CPU_STK_SIZE)OSCfg_IdleTaskStkSize,

14 (OS_ERR \*)p_err );

15 }

代码清单9‑6（1）：空闲任务是μC/OS-III的内部任务，在OSInit()中被创建，在系统没有任何用户任务运行的情况下，空闲任务就会被运行，优先级最低，即等于OS_CFG_PRIO_MAX
- 1u。

修改OSStart()函数
~~~~~~~~~~~~~

加入优先级之后，OSStart()函数需要修改，具体哪一个任务最先运行，由优先级决定，新加入的代码具体见代码清单9‑7的加粗部分。

代码清单9‑7OSStart()函数

1 /\* 启动RTOS，将不再返回 \*/

2 void OSStart (OS_ERR \*p_err)

3 {

4 if ( OSRunning == OS_STATE_OS_STOPPED ) {

5 #if 0

6 /\* 手动配置任务1先运行 \*/

7 OSTCBHighRdyPtr = OSRdyList[0].HeadPtr;

8 #endif

**9 /\* 寻找最高的优先级 \*/**

**10 OSPrioHighRdy = OS_PrioGetHighest();**\ (1)

**11 OSPrioCur = OSPrioHighRdy;**

12

**13 /\* 找到最高优先级的TCB \*/**

**14 OSTCBHighRdyPtr = OSRdyList[OSPrioHighRdy].HeadPtr;**\ (2)

**15 OSTCBCurPtr = OSTCBHighRdyPtr;**

16

17 /\* 标记OS开始运行 \*/

18 OSRunning = OS_STATE_OS_RUNNING;

19

20 /\* 启动任务切换，不会返回 \*/

21 OSStartHighRdy();

22

23 /\* 不会运行到这里，运行到这里表示发生了致命的错误 \*/

24 \*p_err = OS_ERR_FATAL_RETURN;

25 } else {

26 \*p_err = OS_STATE_OS_RUNNING;

27 }

28 }

代码清单9‑7（1）：调取OS_PrioGetHighest()函数从全局变量优先级表OSPrioTbl[]获取最高的优先级，放到OSPrioHighRdy这个全局变量中，然后把OSPrioHighRdy的值再赋给当前优先级OSPrioCur这个全局变量。在任务切换的时候需要用到OSPrioHigh
Rdy和OSPrioCur这两个全局变量。

代码清单9‑7（2）：根据OSPrioHighRdy的值，作为全局变量OSRdyList[]的下标索引找到最高优先级任务的TCB，传给全局变量OSTCBHighRdyPtr，然后再将OSTCBHighRdyPtr赋值给OSTCBCurPtr。在任务切换的时候需要使用到OSTCBHighRdyPtr和
OSTCBCurPtr这两个全局变量。

修改PendSV_Handler()函数
~~~~~~~~~~~~~~~~~~~~

PendSV_Handler()函数中添加了优先级相关的代码，具体见代码清单9‑8中加粗部分。有关PendSV_Handler()这个函数的具体讲解要参考《任务的定义与任务切换的实现》这个章节，这里不再赘述。

代码清单9‑8PendSV_Handler()函数

1 ;\*

2 ; PendSVHandler异常

3 ;\*

4

5 OS_CPU_PendSVHandler_nosave

6

**7 ; OSPrioCur = OSPrioHighRdy**

**8 LDR R0, =OSPrioCur**

**9 LDR R1, =OSPrioHighRdy**

**10 LDRB R2, [R1]**

**11 STRB R2, [R0]**

12

13 ; OSTCBCurPtr = OSTCBHighRdyPtr

14 LDR R0, = OSTCBCurPtr

15 LDR R1, = OSTCBHighRdyPtr

16 LDR R2, [R1]

17 STR R2, [R0]

18

19 LDR R0, [R2]

20 LDMIA R0!, {R4-R11}

21

22 MSR PSP, R0

23 ORR LR, LR, #0x04

24 CPSIE I

25 BX LR

26

27

28 NOP

29

30 ENDP

修改OSTimeDly()函数
~~~~~~~~~~~~~~~

任务调用OSTimeDly()函数之后，任务就处于阻塞态，需要将任务从就绪列表中移除，具体修改的代码见代码清单9‑9的加粗部分。

代码清单9‑9OSTimeDly()函数

1 /\* 阻塞延时 \*/

2 void OSTimeDly(OS_TICK dly)

3 {

4 #if 0

5 /\* 设置延时时间 \*/

6 OSTCBCurPtr->TaskDelayTicks = dly;

7

8 /\* 进行任务调度 \*/

9 OSSched();

10 #endif

11

**12 CPU_SR_ALLOC();**\ (1)

**13**

**14 /\* 进入临界区 \*/**

**15 OS_CRITICAL_ENTER();**\ (2)

16

17 /\* 设置延时时间 \*/

18 OSTCBCurPtr->TaskDelayTicks = dly;

19

**20 /\* 从就绪列表中移除 \*/**

**21 //OS_RdyListRemove(OSTCBCurPtr);**

**22 OS_PrioRemove(OSTCBCurPtr->Prio);**\ (3)

**23**

**24 /\* 退出临界区 \*/**

**25 OS_CRITICAL_EXIT();**\ (4)

26

27 /\* 任务调度 \*/

28 OSSched();

29 }

代码清单9‑9（1）：定义一个局部变量，用来存CPU关中断前的中断状态，因为接下来将任务从就绪列表移除这段代码属于临界短代码，需要关中断。

代码清单9‑9（2）：进入临界段

代码清单9‑9（3）：将任务从就绪列表移除，这里只需将任务在优先级表中对应的位清除即可，暂时不需要把任务TCB从OSRdyList[]中移除，因为接下来OSTimeTick()函数还是通过扫描OSRdyList[]来判断任务的延时时间是否到期。当我们加入了时基列表之后，当任务调用OSTimeDly(
)函数进行延时，就可以把任务的TCB从就绪列表删除，然后把任务TCB插入时基列表，OSTimeTick()函数判断任务的延时是否到期只需通过扫描时基列表即可，时基列表在下一个章节实现。所以这里暂时不能把TCB从就绪列表中删除，只是将任务优先级在优先级表中对应的位清除来达到任务不处于就绪态的目的。

代码清单9‑9（4）：退出临界段。

修改OSSched()函数
~~~~~~~~~~~~~

任务调度函数OSSched()不再是之前的两个任务轮流切换，需要根据优先级来调度，具体修改部分见代码清单9‑10的加粗部分，被迭代的代码已经通过条件编译屏蔽。

代码清单9‑10OSSched()函数

1 void OSSched(void)

2 {

3 #if 0

4 /\* 如果当前任务是空闲任务，那么就去尝试执行任务1或者任务2，

5 看看他们的延时时间是否结束，如果任务的延时时间均没有到期，

6 那就返回继续执行空闲任务 \*/

7 if ( OSTCBCurPtr == &OSIdleTaskTCB ) {

8 if (OSRdyList[0].HeadPtr->TaskDelayTicks == 0) {

9 OSTCBHighRdyPtr = OSRdyList[0].HeadPtr;

10 } else if (OSRdyList[1].HeadPtr->TaskDelayTicks == 0) {

11 OSTCBHighRdyPtr = OSRdyList[1].HeadPtr;

12 } else {

13 return; /\* 任务延时均没有到期则返回，继续执行空闲任务 \*/

14 }

15 } else {

16 /*如果是task1或者task2的话，检查下另外一个任务,

17 如果另外的任务不在延时中，就切换到该任务，

18 否则，判断下当前任务是否应该进入延时状态，

19 如果是的话，就切换到空闲任务。否则就不进行任何切换 \*/

20 if (OSTCBCurPtr == OSRdyList[0].HeadPtr) {

21 if (OSRdyList[1].HeadPtr->TaskDelayTicks == 0) {

22 OSTCBHighRdyPtr = OSRdyList[1].HeadPtr;

23 } else if (OSTCBCurPtr->TaskDelayTicks != 0) {

24 OSTCBHighRdyPtr = &OSIdleTaskTCB;

25 } else {

26 /\* 返回，不进行切换，因为两个任务都处于延时中 \*/

27 return;

28 }

29 } else if (OSTCBCurPtr == OSRdyList[1].HeadPtr) {

30 if (OSRdyList[0].HeadPtr->TaskDelayTicks == 0) {

31 OSTCBHighRdyPtr = OSRdyList[0].HeadPtr;

32 } else if (OSTCBCurPtr->TaskDelayTicks != 0) {

33 OSTCBHighRdyPtr = &OSIdleTaskTCB;

34 } else {

35 /\* 返回，不进行切换，因为两个任务都处于延时中 \*/

36 return;

37 }

38 }

39 }

40

41 /\* 任务切换 \*/

42 OS_TASK_SW();

43 #endif

44

**45 CPU_SR_ALLOC();**\ (1)

**46**

**47 /\* 进入临界区 \*/**

**48 OS_CRITICAL_ENTER();**\ (2)

**49**

**50 /\* 查找最高优先级的任务 \*/**\ (3)

**51 OSPrioHighRdy = OS_PrioGetHighest();**

**52 OSTCBHighRdyPtr = OSRdyList[OSPrioHighRdy].HeadPtr;**

**53**

**54 /\* 如果最高优先级的任务是当前任务则直接返回，不进行任务切换 \*/**\ (4)

**55 if (OSTCBHighRdyPtr == OSTCBCurPtr) {**

**56 /\* 退出临界区 \*/**

**57 OS_CRITICAL_EXIT();**

**58**

**59 return;**

**60 }**

**61 /\* 退出临界区 \*/**

**62 OS_CRITICAL_EXIT();**\ (5)

63

64 /\* 任务切换 \*/

65 OS_TASK_SW();(6)

66 }

代码清单9‑10（1）：定义一个局部变量，用来存CPU关中断前的中断状态，因为接下来查找最高优先级这段代码属于临界短代码，需要关中断。

代码清单9‑10（2）：进入临界段。

代码清单9‑10（3）：查找最高优先级任务。

代码清单9‑10（4）：判断最高优先级任务是不是当前任务，如果是则直接返回，否则将继续往下执行，最后执行任务切换。

代码清单9‑10（5）：退出临界段。

代码清单9‑10（6）：任务切换。

修改OSTimeTick()函数
~~~~~~~~~~~~~~~~

OSTimeTick()函数在SysTick中断服务函数中被调用，是一个周期函数，具体用于扫描就绪列表OSRdyList[]，判断任务的延时时间是否到期，如果到期则将任务在优先级表中对应的位置位，修改部分的代码见代码清单9‑11的加粗部分，被迭代的代码则通过条件编译屏蔽。

代码清单9‑11OSTimeTick()函数

1 void OSTimeTick (void)

2 {

3 unsigned int i;

**4 CPU_SR_ALLOC();**\ (1)

5

**6 /\* 进入临界区 \*/**

**7 OS_CRITICAL_ENTER();**\ (2)

8

9 /\* 扫描就绪列表中所有任务的TaskDelayTicks，如果不为0，则减1 \*/

10 #if 0

11 for (i=0; i<OS_CFG_PRIO_MAX; i++) {

12 if (OSRdyList[i].HeadPtr->TaskDelayTicks > 0) {

13 OSRdyList[i].HeadPtr->TaskDelayTicks --;

14 }

15 }

16 #endif

17

**18 for (i=0; i<OS_CFG_PRIO_MAX; i++) {**\ (3)

**19 if (OSRdyList[i].HeadPtr->TaskDelayTicks > 0) {**

**20 OSRdyList[i].HeadPtr->TaskDelayTicks --;**

**21 if (OSRdyList[i].HeadPtr->TaskDelayTicks == 0) {**

**22 /\* 为0则表示延时时间到，让任务就绪 \*/**

**23 //OS_RdyListInsert (OSRdyList[i].HeadPtr);**

**24 OS_PrioInsert(i);**

**25 }**

**26 }**

**27 }**

28

**29 /\* 退出临界区 \*/**

**30 OS_CRITICAL_EXIT();**\ (4)

31

32 /\* 任务调度 \*/

33 OSSched();

34 }

代码清单9‑11（1）：定义一个局部变量，用来存CPU关中断前的中断状态，因为接下来扫描就绪列表OSRdyList[]这段代码属于临界短代码，需要关中断。

代码清单9‑11（2）：进入临界段。

代码清单9‑11（3）：扫描就绪列表OSRdyList[]，判断任务的延时时间是否到期，如果到期则将任务在优先级表中对应的位置位。

代码清单9‑11（4）：退出临界段。

main()函数
~~~~~~~~

main()函数具体见代码清单9‑12，修改部分代码已经加粗显示。

代码清单9‑12 main()函数

1 /\*

2 \\*

3 \* 全局变量

4 \\*

5 \*/

6

7 uint32_t flag1;

8 uint32_t flag2;

9 uint32_t flag3;

10

11 /\*

12 \\*

13 \* TCB & STACK &任务声明

14 \\*

15 \*/

16 #define TASK1_STK_SIZE 128

17 #define TASK2_STK_SIZE 128

18 #define TASK3_STK_SIZE 128

19

20

21 static OS_TCB Task1TCB;

22 static OS_TCB Task2TCB;

23 static OS_TCB Task3TCB;

24

25

26 static CPU_STK Task1Stk[TASK1_STK_SIZE];

27 static CPU_STK Task2Stk[TASK2_STK_SIZE];

28 static CPU_STK Task3Stk[TASK2_STK_SIZE];

29

30

31 void Task1( void \*p_arg );

32 void Task2( void \*p_arg );

33 void Task3( void \*p_arg );

34

35

36 /\*

37 \\*

38 \* 函数声明

39 \\*

40 \*/

41 void delay(uint32_t count);

42

43 /\*

44 \\*

45 \* main()函数

46 \\*

47 \*/

48 /\*

49 \* 注意事项：1、该工程使用软件仿真，debug需选择 Ude Simulator

50 \* 2、在Target选项卡里面把晶振Xtal(Mhz)的值改为25，默认是12，

51 \* 改成25是为了跟system_ARMCM3.c中定义的__SYSTEM_CLOCK相同，

52 \* 确保仿真的时候时钟一致

53 \*/

54 int main(void)

55 {

56 OS_ERR err;

57

58

59 /\* CPU初始化：1、初始化时间戳 \*/

60 CPU_Init();

61

62 /\* 关闭中断 \*/

63 CPU_IntDis();

64

65 /\* 配置SysTick 10ms 中断一次 \*/

66 OS_CPU_SysTickInit (10);

67

**68 /\* 初始化相关的全局变量 \*/**

**69 OSInit(&err);**\ (1)

70

71 /\* 创建任务 \*/

72 OSTaskCreate( (OS_TCB*)&Task1TCB,

73 (OS_TASK_PTR )Task1,

74 (void \*)0,

**75 (OS_PRIO)1,**\ (2)

76 (CPU_STK*)&Task1Stk[0],

77 (CPU_STK_SIZE) TASK1_STK_SIZE,

78 (OS_ERR \*)&err );

79

80 OSTaskCreate( (OS_TCB*)&Task2TCB,

81 (OS_TASK_PTR )Task2,

82 (void \*)0,

**83 (OS_PRIO)2,**\ (3)

84 (CPU_STK*)&Task2Stk[0],

85 (CPU_STK_SIZE) TASK2_STK_SIZE,

86 (OS_ERR \*)&err );

87

88 OSTaskCreate( (OS_TCB*)&Task3TCB,

89 (OS_TASK_PTR )Task3,

90 (void \*)0,

**91 (OS_PRIO)3,**\ (4)

92 (CPU_STK*)&Task3Stk[0],

93 (CPU_STK_SIZE) TASK3_STK_SIZE,

94 (OS_ERR \*)&err );

**95 #if 0**

**96 /\* 将任务加入到就绪列表 \*/**\ (5)

**97 OSRdyList[0].HeadPtr = &Task1TCB;**

**98 OSRdyList[1].HeadPtr = &Task2TCB;**

**99 #endif**

100

101 /\* 启动OS，将不再返回 \*/

102 OSStart(&err);

103 }

104

105 /\*

106 \\*

107 \* 函数实现

108 \\*

109 \*/

110 /\* 软件延时 \*/

111 void delay (uint32_t count)

112 {

113 for (; count!=0; count--);

114 }

115

116

117

118 void Task1( void \*p_arg )

119 {

120 for ( ;; ) {

121 flag1 = 1;

122 OSTimeDly(2);

123 flag1 = 0;

124 OSTimeDly(2);

125 }

126 }

127

128 void Task2( void \*p_arg )

129 {

130 for ( ;; ) {

131 flag2 = 1;

132 OSTimeDly(2);

133 flag2 = 0;

134 OSTimeDly(2);

135 }

136 }

137

138 void Task3( void \*p_arg )

139 {

140 for ( ;; ) {

141 flag3 = 1;

142 OSTimeDly(2);

143 flag3 = 0;

144 OSTimeDly(2);

145 }

146 }

代码清单9‑12（1）：加入了优先级相关的全局变量OSPrioCur和OSPrioHighRdy的初始化。

代码清单9‑12（2）、（3）和（4）：为每个任务分配了优先级，任务1的优先级为1，任务2的优先级为2，任务3的优先级为3。

代码清单9‑12（5）：将任务插入就绪列表这部分功能由OSTaskCreate()实现，这里通过条件编译屏蔽掉。

实验现象
~~~~

进入软件调试，全速运行程序，从逻辑分析仪中可以看到三个任务的波形是完全同步，就好像CPU在同时干三件事情，具体仿真的波形图见图9‑1。任务开始的启动过程具体见图9‑2，这个启动过程要认真的理解下。

|multip002|

图9‑1实验现象（宏观）

|multip003|

图9‑2任务的启动过程（微观）

图9‑2是任务1、2和3刚开始启动时的软件仿真波形图，系统从启动到任务1开始运行前花的时间为TIME1，等于0.26MS。任务1开始运行，然后调用OSTimeDly(1)进入延时，随后进行任务切换，切换到任务2开始运行，从任务1切换到任务2花费的时间等于TIME2-TIME1，等于0.01MS。任务
2开始运行，然后调用OSTimeDly(1)进入延时，随后进行任务切换，切换到任务3开始运行，从任务2切换到任务3花费的时间等于TIME3-TIME1，等于0.01MS。任务3开始运行，然后调用OSTimeDly(1)进入延时，随后进行任务切换，这个时候我们创建的3个任务都处于延时状态，那么系统就切
换到空闲任务，在三个任务延时未到期之前，系统一直都是在运行空闲任务。当第一个SysTick中断产生，中断服务函数会调用OSTimeTick()函数扫描每个任务的延时是否到期，因为是延时1个SysTick周期，所以第一个SysTick中断产生就意味着延时都到期，任务1、2和3依次进入就绪态，再次回到任
务本身接着运行，将自身的Flag清零，然后任务1、2和3又依次调用OSTimeDly(1)进入延时状态，直到下一个SysTick中断产生前，系统都处在空闲任务中，一直这样循环下去。

但是，有些同学肯定就会问图9‑1中任务1、2和3的波形图是同步的，而图9‑2中任务的波形就不同步，有先后顺序？答案是图9‑2是将两个任务切换花费的时间0.01ms进行放大后观察的波形，就好像我们用放大镜看微小的东西一样，如果不用放大镜，在宏观层面观察就是图9‑1的实验现象。

.. |multip002| image:: media\multip002.png
   :width: 5.76806in
   :height: 2.19931in
.. |multip003| image:: media\multip003.png
   :width: 4.56944in
   :height: 2.45972in
