.. vim: syntax=rst

阻塞延时与空闲任务
===================

在上一章节中，任务体内的延时使用的是软件延时，即还是让CPU空等来达到延时的效果。使用RTOS的很大优势就是榨干CPU的性能，永远不能让它闲着，任务如果需要延时也就不能再让CPU空等来实现延时的效果。RTOS中的延时叫阻塞延时，即任务需要延时的时候，任务会放弃CPU的使用权，CPU可以去干其他的事情
，当任务延时时间到，重新获取CPU使用权，任务继续运行，这样就充分地利用了CPU的资源，而不是干等着。

当任务需要延时，进入阻塞状态，那CPU又去干什么事情了？如果没有其他任务可以运行，RTOS都会为CPU创建一个空闲任务，这个时候CPU就运行空闲任务。在μC/OS-
III中，空闲任务是系统在初始化的时候创建的优先级最低的任务，空闲任务主体很简单，只是对一个全局变量进行计数。鉴于空闲任务的这种特性，在实际应用中，当系统进入空闲任务的时候，可在空闲任务中让单片机进入休眠或者低功耗等操作。

实现空闲任务
~~~~~~

定义空闲任务栈
^^^^^^^

空闲任务栈在os_cfg_app.c（os_cfg_app.c第一次使用需要自行在文件夹μC/OS-III\Source中新建并添加到工程的μC/OS-III Source组）文件中定义，具体见代码清单5‑1。

代码清单5‑1os_cfg_app.c文件代码

1 /\*

2 \\*

3 \* 数据域

4 \\*

5 \*/

6

7 CPU_STK OSCfg_IdleTaskStk[OS_CFG_IDLE_TASK_STK_SIZE];(1)

8

9

10

11 /\*

12 \\*

13 \* 常量

14 \\*

15 \*/

16

17 /\* 空闲任务栈起始地址 \*/

18 CPU_STK \* const OSCfg_IdleTaskStkBasePtr = \\(2)

19 (CPU_STK \*)&OSCfg_IdleTaskStk[0];

20 /\* 空闲任务栈大小 \*/

21 CPU_STK_SIZE const OSCfg_IdleTaskStkSize = \\

22 (CPU_STK_SIZE)OS_CFG_IDLE_TASK_STK_SIZE;

代码清单5‑1（1）：空闲任务的栈是一个定义好的数组，大小由OS_CFG_IDLE_TASK_STK_SIZE这个宏控制。OS_CFG_IDLE_TASK_STK_SIZE在os_cfg_app.h这个头文件定义，大小为128，具体见代码清单5‑2。

代码清单5‑2os_cfg_app.h文件代码

1 #ifndef OS_CFG_APP_H

2 #define OS_CFG_APP_H

3

4 /\*

5 \\*

6 \* 常量

7 \\*

8 \*/

9

10 /\* 空闲任务栈大小 \*/

11 #define OS_CFG_IDLE_TASK_STK_SIZE 128u

12

13 #endif/\* OS_CFG_APP_H \*/

代码清单5‑1（2）：空闲任务的栈的起始地址和大小均被定义成一个常量，不能被修改。变量OSCfg_IdleTaskStkBasePtr和OSCfg_IdleTaskStkSize同时还在os.h中声明，这样就具有全局属性，可以在其他文件里面被使用，具体声明见代码清单5‑3。

代码清单5‑3OSCfg_IdleTaskStkBasePtr和OSCfg_IdleTaskStkSize声明

1 /\* 空闲任务栈起始地址 \*/

2 extern CPU_STK \* const OSCfg_IdleTaskStkBasePtr;

3 /\* 空闲任务栈大小 \*/

4 extern CPU_STK_SIZE const OSCfg_IdleTaskStkSize;

定义空闲任务TCB
^^^^^^^^^

任务控制块TCB是每一个任务必须的，空闲任务的TCB在os.h中定义，是一个全局变量，具体见代码清单5‑4。

代码清单5‑4定义空闲任务TCB

/\* 空闲任务TCB \*/

1 OS_EXT OS_TCB OSIdleTaskTCB;

定义空闲任务函数
^^^^^^^^

空闲任务正如其名，空闲，任务体里面只是对全局变量OSIdleTaskCtr ++ 操作，具体实现见代码清单5‑5。

代码清单5‑5空闲任务函数

1 /\* 空闲任务 \*/

2 void OS_IdleTask (void \*p_arg)

3 {

4 p_arg = p_arg;

5

6 /\* 空闲任务什么都不做，只对全局变量OSIdleTaskCtr ++ 操作 \*/

7 for (;;) {

8 OSIdleTaskCtr++;

9 }

10 }

代码清单5‑5中的全局变量OSIdleTaskCtr在os.h中定义，具体见代码清单5‑6。

代码清单5‑6OSIdleTaskCtr定义

/\* 空闲任务计数变量 \*/

1 OS_EXT OS_IDLE_CTR OSIdleTaskCtr;

代码清单5‑6中的OS_IDLE_CTR是在os_type.h中重新定义的数据类型，具体见代码清单5‑7。

代码清单5‑7OS_IDLE_CTR定义

/\* 空闲任务计数变量定义 \*/

1 typedef CPU_INT32U OS_IDLE_CTR;

空闲任务初始化
^^^^^^^

空闲任务的初始化在OSInit()在完成，意味着在系统还没有启动之前空闲任务就已经创建好，具体在os_core.c定义，具体代码见代码清单5‑8。

代码清单5‑8空闲任务初始化函数

1 void OSInit (OS_ERR \*p_err)

2 {

3 /\* 配置OS初始状态为停止态 \*/

4 OSRunning = OS_STATE_OS_STOPPED;

5

6 /\* 初始化两个全局TCB，这两个TCB用于任务切换 \*/

7 OSTCBCurPtr = (OS_TCB \*)0;

8 OSTCBHighRdyPtr = (OS_TCB \*)0;

9

10 /\* 初始化就绪列表 \*/

11 OS_RdyListInit();

12

13 **/\* 初始化空闲任务 \*/**

14 **OS_IdleTaskInit(p_err);(1)**

15 if (*p_err != OS_ERR_NONE) {

16 return;

17 }

18 }

19

20 /\* 空闲任务初始化 \*/

21 void OS_IdleTaskInit(OS_ERR \*p_err)

22 {

23 /\* 初始化空闲任务计数器 \*/

24 OSIdleTaskCtr = (OS_IDLE_CTR)0;(2)

25

26 /\* 创建空闲任务 \*/

27 OSTaskCreate( (OS_TCB \*)&OSIdleTaskTCB,(3)

28 (OS_TASK_PTR )OS_IdleTask,

29 (void \*)0,

30 (CPU_STK \*)OSCfg_IdleTaskStkBasePtr,

31 (CPU_STK_SIZE)OSCfg_IdleTaskStkSize,

32 (OS_ERR \*)p_err );

33 }

代码清单5‑8（1）：空闲任务初始化函数在OSInit中调用，在系统还没有启动之前就被创建。

代码清单5‑8（2）：初始化空闲任务计数器，我们知道，这个是预先在os.h中定义好的全局变量。

代码清单5‑8（3）：创建空闲任务，把栈，TCB，任务函数联系在一起。

实现阻塞延时
~~~~~~

阻塞延时的阻塞是指任务调用该延时函数后，任务会被剥离CPU使用权，然后进入阻塞状态，直到延时结束，任务重新获取CPU使用权才可以继续运行。在任务阻塞的这段时间，CPU可以去执行其他的任务，如果其他的任务也在延时状态，那么CPU就将运行空闲任务。阻塞延时函数在os_time.c中定义，具体代码实现见代
码清单5‑9。

代码清单5‑9阻塞延时代码

1 /\* 阻塞延时 \*/

2 void OSTimeDly(OS_TICK dly)

3 {

4 /\* 设置延时时间 \*/

5 OSTCBCurPtr->TaskDelayTicks = dly;(1)

6

7 /\* 进行任务调度 \*/

8 OSSched();(2)

9 }

代码清单5‑9（1）：TaskDelayTicks是任务控制块的一个成员，用于记录任务需要延时的时间，单位为SysTick的中断周期。比如我们本书当中SysTick的中断周期为10ms，调用OSTimeDly(2)则完成2*10ms的延时。TaskDelayTicks的定义具体见代码清单5‑10。

代码清单5‑10TaskDelayTicks定义

1 struct os_tcb {

2 CPU_STK \*StkPtr;

3 CPU_STK_SIZE StkSize;

4

5 **/\* 任务延时周期个数 \*/**

6 **OS_TICK TaskDelayTicks;**

7 };

代码清单5‑9（2）：任务调度。这个时候的任务调度与上一章节的不一样，具体见代码清单5‑11，其中加粗部分为上一章节的代码，现已用条件编译屏蔽掉。

代码清单5‑11任务调度

1 void OSSched(void)

2 {

**3 #if 0/\* 非常简单的任务调度：两个任务轮流执行 \*/**

**4 if ( OSTCBCurPtr == OSRdyList[0].HeadPtr ) {**

**5 OSTCBHighRdyPtr = OSRdyList[1].HeadPtr;**

**6 } else {**

**7 OSTCBHighRdyPtr = OSRdyList[0].HeadPtr;**

**8 }**

**9 #endif**

10

11 /\* 如果当前任务是空闲任务，那么就去尝试执行任务1或者任务2，

12 看看他们的延时时间是否结束，如果任务的延时时间均没有到期，

13 那就返回继续执行空闲任务 \*/

14 if ( OSTCBCurPtr == &OSIdleTaskTCB ) {(1)

15 if (OSRdyList[0].HeadPtr->TaskDelayTicks == 0) {

16 OSTCBHighRdyPtr = OSRdyList[0].HeadPtr;

17 } else if (OSRdyList[1].HeadPtr->TaskDelayTicks == 0) {

18 OSTCBHighRdyPtr = OSRdyList[1].HeadPtr;

19 } else {

20 /\* 任务延时均没有到期则返回，继续执行空闲任务 \*/

21 return;

22 }

23 } else {(2)

24 /*如果是task1或者task2的话，检查下另外一个任务,

25 如果另外的任务不在延时中，就切换到该任务

26 否则，判断下当前任务是否应该进入延时状态，

27 如果是的话，就切换到空闲任务。否则就不进行任何切换 \*/

28 if (OSTCBCurPtr == OSRdyList[0].HeadPtr) {

29 if (OSRdyList[1].HeadPtr->TaskDelayTicks == 0) {

30 OSTCBHighRdyPtr = OSRdyList[1].HeadPtr;

31 } else if (OSTCBCurPtr->TaskDelayTicks != 0) {

32 OSTCBHighRdyPtr = &OSIdleTaskTCB;

33 } else {

34 /\* 返回，不进行切换，因为两个任务都处于延时中 \*/

35 return;

36 }

37 } else if (OSTCBCurPtr == OSRdyList[1].HeadPtr) {

38 if (OSRdyList[0].HeadPtr->TaskDelayTicks == 0) {

39 OSTCBHighRdyPtr = OSRdyList[0].HeadPtr;

40 } else if (OSTCBCurPtr->TaskDelayTicks != 0) {

41 OSTCBHighRdyPtr = &OSIdleTaskTCB;

42 } else {

43 /\* 返回，不进行切换，因为两个任务都处于延时中 \*/

44 return;

45 }

46 }

47 }

48

49 /\* 任务切换 \*/

50 OS_TASK_SW();(3)

51 }

代码清单5‑11（1）：如果当前任务是空闲任务，那么就去尝试执行任务1或者任务2，看看他们的延时时间是否结束，如果任务的延时时间均没有到期，那就返回继续执行空闲任务。

代码清单5‑11（2）：如果当前任务不是空闲任务则会执行到此，那就看看当前任务是哪个任务。无论是哪个任务，都要检查下另外一个任务是否在延时中，如果没有在延时，那就切换到该任务，如果有在延时，那就判断下当前任务是否应该进入延时状态，如果是的话，就切换到空闲任务。否则就不进行任务切换。

代码清单5‑11（3）：任务切换，实际就是触发PendSV异常。

main()函数
~~~~~~~~

main()函数和任务代码变动不大，具体见代码清单5‑12，有变动部分代码已加粗。

代码清单5‑12 main()函数

1 int main(void)

2 {

3 OS_ERR err;

4

5 /\* 关闭中断 \*/

6 CPU_IntDis();

7

8 /\* 配置SysTick 10ms 中断一次 \*/

9 OS_CPU_SysTickInit (10);

10

**11 /\* 初始化相关的全局变量 \*/**

**12 OSInit(&err);(1)**

13

14 /\* 创建任务 \*/

15 OSTaskCreate ((OS_TCB*) &Task1TCB,

16 (OS_TASK_PTR ) Task1,

17 (void \*) 0,

18 (CPU_STK*) &Task1Stk[0],

19 (CPU_STK_SIZE) TASK1_STK_SIZE,

20 (OS_ERR \*) &err);

21

22 OSTaskCreate ((OS_TCB*) &Task2TCB,

23 (OS_TASK_PTR ) Task2,

24 (void \*) 0,

25 (CPU_STK*) &Task2Stk[0],

26 (CPU_STK_SIZE) TASK2_STK_SIZE,

27 (OS_ERR \*) &err);

28

29 /\* 将任务加入到就绪列表 \*/

30 OSRdyList[0].HeadPtr = &Task1TCB;

31 OSRdyList[1].HeadPtr = &Task2TCB;

32

33 /\* 启动OS，将不再返回 \*/

34 OSStart(&err);

35 }

36

37 /\* 任务1 \*/

38 void Task1( void \*p_arg )

39 {

40 for ( ;; ) {

41 flag1 = 1;

**42 //delay( 100 );**

**43 OSTimeDly(2);(2)**

44 flag1 = 0;

**45 //delay( 100 );**

**46 OSTimeDly(2);**

47

48 /\* 任务切换，这里是手动切换 \*/

49 //OSSched();

50 }

51 }

52

53 /\* 任务2 \*/

54 void Task2( void \*p_arg )

55 {

56 for ( ;; ) {

57 flag2 = 1;

**58 //delay( 100 );**

**59 OSTimeDly(2);(3)**

60 flag2 = 0;

**61 //delay( 100 );**

**62 OSTimeDly(2);**

63

64 /\* 任务切换，这里是手动切换 \*/

65 //OSSched();

66 }

67 }

代码清单5‑12（1）：空闲任务初始化函数在OSInint中调用，在系统启动之前创建好空闲任务。

代码清单5‑12（2）和（3）：延时函数均替代为阻塞延时，延时时间均为2个SysTick中断周期，即20ms。

实验现象
~~~~

进入软件调试，全速运行程序，从逻辑分析仪中可以看到两个任务的波形是完全同步，就好像CPU在同时干两件事情，具体仿真的波形图见图5‑1和图5‑2。

|idleta002|

图5‑1实验现象1

|idleta003|

图5‑2实验现象2

从图5‑1和图5‑2可以看出，flag1和flag2的高电平的时间为(0.1802-0.1602)s，刚好等于阻塞延时的20ms，所以实验现象跟代码要实现的功能是一致的。

.. |idleta002| image:: media\idleta002.png
   :width: 4.53472in
   :height: 2.02431in
.. |idleta003| image:: media\idleta003.png
   :width: 4.48611in
   :height: 2.32708in
