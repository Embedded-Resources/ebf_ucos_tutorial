.. vim: syntax=rst

μC/OS-III的启动流程
====================

在目前的RTOS中，主要有两种比较流行的启动方式，暂时还没有看到第三种，接下来我将通过伪代码的方式来讲解下这两种启动方式的区别，然后再具体分析下μC/OS的启动流程。

万事俱备，只欠东风
~~~~~~~~~

第一种我称之为万事俱备，只欠东风法。这种方法是在main()函数中将硬件初始化，RTOS系统初始化，所有任务的创建这些都弄好，这个我称之为万事都已经准备好。最后只欠一道东风，即启动RTOS的调度器，开始多任务的调度，具体的伪代码实现见代码清单16‑1。

代码清单16‑1万事俱备，只欠东风法伪代码实现

1 int main (void)

2 {

3 /\* 硬件初始化 \*/

4 HardWare_Init();\ **(1)**

5

6 /\* RTOS 系统初始化 \*/

7 RTOS_Init();\ **(2)**

8

9 /\* 创建任务1，但任务1不会执行，因为调度器还没有开启 \*/**(3)**

10 RTOS_TaskCreate(Task1);

11 /\* 创建任务2，但任务2不会执行，因为调度器还没有开启 \*/

12 RTOS_TaskCreate(Task2);

13

14 /\* ......继续创建各种任务 \*/

15

16 /\* 启动RTOS，开始调度 \*/

17 RTOS_Start();\ **(4)**

18 }

19

20 voidTask1( void \*arg )\ **(5)**

21 {

22 while (1)

23 {

24 /\* 任务实体，必须有阻塞的情况出现 \*/

25 }

26 }

27

28 voidTask1( void \*arg )\ **(6)**

29 {

30 while (1)

31 {

32 /\* 任务实体，必须有阻塞的情况出现 \*/

33 }

34 }

代码清单16‑1\ **(1)**\ ：硬件初始化。硬件初始化这一步还属于裸机的范畴，我们可以把需要使用到的硬件都初始化好而且测试好，确保无误。

代码清单16‑1\ **(2)**\ ：RTOS系统初始化。比如RTOS里面的全局变量的初始化，空闲任务的创建等。不同的RTOS，它们的初始化有细微的差别。

代码清单16‑1\ **(3)**\ ：创建各种任务。这里把所有要用到的任务都创建好，但还不会进入调度，因为这个时候RTOS的调度器还没有开启。

代码清单16‑1\ **(4)**\ ：启动RTOS调度器，开始任务调度。这个时候调度器就从刚刚创建好的任务中选择一个优先级最高的任务开始运行。

代码清单16‑1\ **(5)(6)**\ ：任务实体通常是一个不带返回值的无限循环的C函数，函数体必须有阻塞的情况出现，不然任务（如果优先权恰好是最高）会一直在while循环里面执行，导致其他任务没有执行的机会。

小心翼翼，十分谨慎
~~~~~~~~~

第二种我称之为小心翼翼，十分谨慎法。这种方法是在main()函数中将硬件和RTOS系统先初始化好，然后创建一个启动任务后就启动调度器，然后在启动任务里面创建各种应用任务，当所有任务都创建成功后，启动任务把自己删除，具体的伪代码实现见代码清单16‑2。

代码清单16‑2小心翼翼，十分谨慎法伪代码实现

1 int main (void)

2 {

3 /\* 硬件初始化 \*/

4 HardWare_Init();\ **(1)**

5

6 /\* RTOS 系统初始化 \*/

7 RTOS_Init();\ **(2)**

8

9 /\* 创建一个任务 \*/

10 RTOS_TaskCreate(AppTaskCreate);\ **(3)**

11

12 /\* 启动RTOS，开始调度 \*/

13 RTOS_Start();\ **(4)**

14 }

15

16 /\* 起始任务，在里面创建任务 \*/

17 voidAppTaskCreate( void \*arg )\ **(5)**

18 {

19 /\* 创建任务1，然后执行 \*/

20 RTOS_TaskCreate(Task1);\ **(6)**

21

22 /\* 当任务1阻塞时，继续创建任务2，然后执行 \*/

23 RTOS_TaskCreate(Task2);

24

25 /\* ......继续创建各种任务 \*/

26

27 /\* 当任务创建完成，删除起始任务 \*/

28 RTOS_TaskDelete(AppTaskCreate);\ **(7)**

29 }

30

31 void Task1( void \*arg )\ **(8)**

32 {

33 while (1)

34 {

35 /\* 任务实体，必须有阻塞的情况出现 \*/

36 }

37 }

38

39 void Task2( void \*arg )\ **(9)**

40 {

41 while (1)

42 {

43 /\* 任务实体，必须有阻塞的情况出现 \*/

44 }

45 }

代码清单16‑2 **(1)**\ ：硬件初始化。来到硬件初始化这一步还属于裸机的范畴，我们可以把需要使用到的硬件都初始化好而且测试好，确保无误。

代码清单16‑2 **(2)**\ ：RTOS系统初始化。比如RTOS里面的全局变量的初始化，空闲任务的创建等。不同的RTOS，它们的初始化有细微的差别。

代码清单16‑2 **(3)**\ ：创建一个开始任务。然后在这个初始任务里面创建各种应用任务。

代码清单16‑2 **(4)**\ ：启动RTOS调度器，开始任务调度。这个时候调度器就去执行刚刚创建好的初始任务。

代码清单16‑2 **(5)**\ ：我们通常说任务是一个不带返回值的无限循环的C函数，但是因为初始任务的特殊性，它不能是无限循环的，只执行一次后就关闭。在初始任务里面我们创建我们需要的各种任务。

代码清单16‑2 **(6)**\ ：创建任务。每创建一个任务后它都将进入就绪态，系统会进行一次调度，如果新创建的任务的优先级比初始任务的优先级高的话，那将去执行新创建的任务，当新的任务阻塞时再回到初始任务被打断的地方继续执行。反之，则继续往下创建新的任务，直到所有任务创建完成。

代码清单16‑2 **(7)**\ ：各种应用任务创建完成后，初始任务自己关闭自己，使命完成。

代码清单16‑2 **(8)(9)**\ ：任务实体通常是一个不带返回值的无限循环的C函数，函数体必须有阻塞的情况出现，不然任务（如果优先权恰好是最高）会一直在while循环里面执行，其他任务没有执行的机会。

孰优孰劣
~~~~

那有关这两种方法孰优孰劣？我暂时没发现，我个人还是比较喜欢使用第二种。COS和LiteOS第一种和第二种都可以使用，由用户选择，RT-Thread和FreeRTOS则默认使用第二种。接下来我们详细讲解下μC/OS的启动流程。

系统的启动
~~~~~

我们知道，在系统上电的时候第一个执行的是启动文件里面由汇编编写的复位函数Reset_Handler，具体见代码清单16‑3。复位函数的最后会调用C库函数__main，具体见代码清单16‑3的加粗部分。__main()函数的主要工作是初始化系统的堆和栈，最后调用C中的main()函数，从而去到C的世界
。

代码清单16‑3Reset_Handler函数

1 Reset_Handler PROC

2 EXPORT Reset_Handler [WEAK]

3 IMPORT \__main

4 IMPORT SystemInit

5 LDRR0, =SystemInit

6 BLX R0

7 LDRR0, =__main

8 BX R0

9 ENDP

系统初始化
^^^^^

在调用创建任务函数之前，我们必须要对系统进行一次初始化，而系统的初始化是根据我们配置宏定义进行初始化的，有一些则是系统必要的初始化，如空闲任务，时钟节拍任务等，下面我们来看看系统初始化的源码，具体见代码清单16‑4。

代码清单16‑4系统初始化（已删减）

1 void OSInit (OS_ERR \*p_err)

2 {

3 CPU_STK \*p_stk;

4 CPU_STK_SIZE size;

5

6 if (p_err == (OS_ERR \*)0) {

7 OS_SAFETY_CRITICAL_EXCEPTION();

8 return;

9 }

10 #endi

11 OSInitHook(); /*初始化钩子函数相关的代码*/

12

13 OSIntNestingCtr= (OS_NESTING_CTR)0; /*清除中断嵌套计数器*/

14

15 OSRunning = OS_STATE_OS_STOPPED; /*未启动多任务处理*/

16

17 OSSchedLockNestingCtr = (OS_NESTING_CTR)0;/*清除锁定计数器*/

18

19 OSTCBCurPtr= (OS_TCB \*)0; /*将OS_TCB指针初始化为已知状态 \*/

20 OSTCBHighRdyPtr = (OS_TCB \*)0;

21

22 OSPrioCur = (OS_PRIO)0; /*将优先级变量初始化为已知状态*/

23 OSPrioHighRdy = (OS_PRIO)0;

24 OSPrioSaved = (OS_PRIO)0;

25

26

27 if (OSCfg_ISRStkSize > (CPU_STK_SIZE)0) {

28 p_stk = OSCfg_ISRStkBasePtr; /*清除异常栈以进行栈检查*/

29 if (p_stk != (CPU_STK \*)0) {

30 size = OSCfg_ISRStkSize;

31 while (size > (CPU_STK_SIZE)0) {

32 size--;

33 \*p_stk = (CPU_STK)0;

34 p_stk++;

35 }

36 }

37 }

38

39 OS_PrioInit(); /*初始化优先级位图表*/

40

41 OS_RdyListInit(); /*初始化就绪列表*/

42

43 OS_TaskInit(p_err); /*初始化任务管理器*/

44 if (*p_err != OS_ERR_NONE) {

45 return;

46 }

47

48 OS_IdleTaskInit(p_err); /\* 初始化空闲任务 \*/ **(1)**

49 if (*p_err != OS_ERR_NONE) {

50 return;

51 }

52

53 OS_TickTaskInit(p_err); /\* 初始化时钟节拍任务*/ **(2)**

54 if (*p_err != OS_ERR_NONE) {

55 return;

56 }

57

58 OSCfg_Init();

59 }

在这个系统初始化中，我们主要看两个地方，一个是空闲任务的初始化，一个是时钟节拍任务的初始化，这两个任务是必须存在的任务，否则系统无法正常运行。

空闲任务
''''

代码清单16‑4\ **(1)**\ ：其实初始化就是创建一个空闲任务，空闲任务的相关信息由系统默认指定，用户不能修改，OS_IdleTaskInit()源码具体见代码清单16‑5。

代码清单16‑5 OS_IdleTaskInit()源码

1 void OS_IdleTaskInit (OS_ERR \*p_err)

2 {

3 #ifdef OS_SAFETY_CRITICAL

4 if (p_err == (OS_ERR \*)0) {

5 OS_SAFETY_CRITICAL_EXCEPTION();

6 return;

7 }

8 #endif

9

10 OSIdleTaskCtr = (OS_IDLE_CTR)0; **(1)**

11 /\* ---------------- CREATE THE IDLE TASK ---------------- \*/

12 OSTaskCreate((OS_TCB \*)&OSIdleTaskTCB,

13 (CPU_CHAR \*)((void \*)"μC/OS-III Idle Task"),

14 (OS_TASK_PTR)OS_IdleTask,

15 (void \*)0,

16 (OS_PRIO )(OS_CFG_PRIO_MAX - 1u),

17 (CPU_STK \*)OSCfg_IdleTaskStkBasePtr,

18 (CPU_STK_SIZE)OSCfg_IdleTaskStkLimit,

19 (CPU_STK_SIZE)OSCfg_IdleTaskStkSize,

20 (OS_MSG_QTY )0u,

21 (OS_TICK )0u,

22 (void \*)0,

23 (OS_OPT)(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR \|OS_OPT_TASK_NO_TLS),

24 (OS_ERR \*)p_err); **(2)**

25 }

代码清单16‑5\ **(1)**\ ：OSIdleTaskCtr在os.h头文件中定义，是一个32位无符号整型变量，该变量的作用是用于统计空闲任务的运行的，怎么统计呢，我们在空闲任务中讲解。现在初始化空闲任务，系统就将OSIdleTaskCtr清零。

代码清单16‑5\ **(2)**\ ：我们可以很容易看到系统只是调用了OSTaskCreate()函数来创建一个任务，这个任务就是空闲任务，任务优先级为OS_CFG_PRIO_MAX-1，OS_CFG_PRIO_MAX是一个宏，该宏定义表示μC/OS的任务优先级数值的最大值，我们知道，在μC/OS
系统中，任务的优先级数值越大，表示任务的优先级越低，所以空闲任务的优先级是最低的。空闲任务栈大小为OSCfg_IdleTaskStkSize，它也是一个宏，在os_cfg_app.c文件中定义，默认为128，则空闲任务栈默认为128*4=512字节。

空闲任务其实就是一个函数，其函数入口是OS_IdleTask，源码具体见代码清单16‑6。

代码清单16‑6 OS_IdleTask()源码

1 void OS_IdleTask (void \*p_arg)

2 {

3 CPU_SR_ALLOC();

4

5

6 /\* Prevent compiler warning for not using 'p_arg'*/

7 p_arg = p_arg;

8

9 while (DEF_ON) {

10 CPU_CRITICAL_ENTER();

11 OSIdleTaskCtr++;

12 #if OS_CFG_STAT_TASK_EN > 0u

13 OSStatTaskCtr++;

14 #endif

15 CPU_CRITICAL_EXIT();

16 /\* Call user definable HOOK \*/

17 OSIdleTaskHook();

18 }

19 }

空闲任务的作用还是很大的，它是一个无限的死循环，因为其优先级是最低的，所以任何优先级比它高的任务都能抢占它从而取得CPU的使用权，为什么系统要空闲任务呢？因为CPU是不会停下来的，即使啥也不干，CPU也不会停下来，此时系统就必须保证有一个随时处于就绪态的任务，而且这个任务不会抢占其他任务，当且仅当系
统的其他任务处于阻塞中，系统才会运行空闲任务，这个任务可以做很多事情，任务统计，钩入用户自定义的钩子函数实现用户自定义的功能等，但是需要注意的是，在钩子函数中用户不允许调用任何可以使空闲任务阻塞的函数接口，空闲任务是不允许被阻塞的。

代码清单16‑4\ **(2)**\ ：同样的，OS_TickTaskInit()函数也是创建一个时钟节拍任务，具体见代码清单16‑7。

代码清单16‑7 OS_TickTaskInit()源码

1 void OS_TickTaskInit (OS_ERR \*p_err)

2 {

3 #ifdef OS_SAFETY_CRITICAL

4 if (p_err == (OS_ERR \*)0) {

5 OS_SAFETY_CRITICAL_EXCEPTION();

6 return;

7 }

8 #endif

9

10 OSTickCtr = (OS_TICK)0u; /\* Clear the tick counter \*/

11

12 OSTickTaskTimeMax = (CPU_TS)0u;

13

14

15 OS_TickListInit();/\* Initialize the tick list data structures \*/

16

17 /\* ---------------- CREATE THE TICK TASK ---------------- \*/

18 if (OSCfg_TickTaskStkBasePtr == (CPU_STK \*)0) {

19 \*p_err = OS_ERR_TICK_STK_INVALID;

20 return;

21 }

22

23 if (OSCfg_TickTaskStkSize < OSCfg_StkSizeMin) {

24 \*p_err = OS_ERR_TICK_STK_SIZE_INVALID;

25 return;

26 }

27 /\* Only one task at the 'Idle Task' priority \*/

28 if (OSCfg_TickTaskPrio >= (OS_CFG_PRIO_MAX - 1u)) {

29 \*p_err = OS_ERR_TICK_PRIO_INVALID;

30 return;

31 }

32

33 OSTaskCreate((OS_TCB \*)&OSTickTaskTCB,

34 (CPU_CHAR \*)((void \*)"μC/OS-III Tick Task"),

35 (OS_TASK_PTR )OS_TickTask,

36 (void \*)0,

37 (OS_PRIO )OSCfg_TickTaskPrio,

38 (CPU_STK \*)OSCfg_TickTaskStkBasePtr,

39 (CPU_STK_SIZE)OSCfg_TickTaskStkLimit,

40 (CPU_STK_SIZE)OSCfg_TickTaskStkSize,

41 (OS_MSG_QTY )0u,

42 (OS_TICK )0u,

43 (void \*)0,

44 (OS_OPT)(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR \| OS_OPT_TASK_NO_TLS),

45 (OS_ERR \*)p_err);

46 }

当然啦，系统的初始化远远不止这两个任务，系统的其他资源也是会进行初始化的，我们在这里就暂时不讲解，有兴趣的图像可以自行查看系统初始化的源码。

CPU初始化
^^^^^^

在main()函数中，我们除了需要对板级硬件进行初始化，还需要进行一些系统相关的初始化，如CPU的初始化，在μC/OS 中，有一个很重要的功能就是时间戳，它的精度高达ns级别，是CPU内核的一个资源，所以使用的时候要对CPU进行相关的初始化，具体见代码清单16‑8。

代码清单16‑8CPU初始化源码

1 void CPU_Init (void)

2 {

3 /\* --------------------- INIT TS ---------------------- \*/

4 #if ((CPU_CFG_TS_EN == DEF_ENABLED) \|\| \\

5 (CPU_CFG_TS_TMR_EN == DEF_ENABLED))

6 CPU_TS_Init(); /\* 时间戳测量的初始化 \*/

7

8 #endif

9 /\* -------------- INIT INT DIS TIME MEAS -------------- \*/

10 #ifdef CPU_CFG_INT_DIS_MEAS_EN

11 CPU_IntDisMeasInit(); /\* 最大关中断时间测量初始化 \*/

12

13 #endif

14

15 /\* ------------------ INIT CPU NAME ------------------- \*/

16 #if (CPU_CFG_NAME_EN == DEF_ENABLED)

17 CPU_NameInit(); //CPU 名字初始化

18 #endif

19 }

我们重点来介绍一下μC/OS的时间戳。

在Cortex-M（注意：M0内核不可用）内核中有一个外设叫DWT(Data Watchpoint and Trace)，是用于系统调试及跟踪，它有一个32位的寄存器叫CYCCNT，它是一个向上的计数器，记录的是内核时钟运行的个数，内核时钟跳动一次，该计数器就加1，当CYCCNT溢出之后，会清零重新
开始向上计数。CYCCNT的精度非常高，其精度取决于内核的频率是多少，如果是STM32F1系列，内核时钟是72M，那精度就是1/72M = 14ns，而程序的运行时间都是微秒级别的，所以14ns的精度是远远够的。最长能记录的时间为：60s=2的32次方/72000000(假设内核频率为72M，内核跳
一次的时间大概为1/72M=14ns)，而如果是STM32H7系列这种400M主频的芯片，那它的计时精度高达2.5ns（1/400000000 = 2.5），而如果是 i.MX RT1052这种比较厉害的处理器，最长能记录的时间为： 8.13s=2的32次方/528000000
(假设内核频率为528M，内核跳一次的时间大概为1/528M=1.9ns) 。

想要启用DWT外设，需要由另外的内核调试寄存器DEMCR的位24控制，写1启用，DEMCR的地址是0xE000 EDFC。

|startu002|

图16‑1启用DWT

启用DWT_CYCCNT寄存器之前，先清零。让我们看看DWT_CYCCNT的基地址，从ARM-Cortex-M手册中可以看到其基地址是0xE000 1004，复位默认值是0，而且它的类型是可读可写的，我们往0xE000 1004这个地址写0就将DWT_CYCCNT清零了。

|startu003|

图16‑2DWT_CYCCNT

关于CYCCNTENA，它是DWT控制寄存器的第一位，写1启用，则启用CYCCNT计数器，否则CYCCNT计数器将不会工作，它的地址是0xE000EDFC。

|startu004|

图16‑3 CYCCNTENA

所以想要使用DWT的CYCCNT步骤：

1. 先启用DWT外设，这个由另外内核调试寄存器DEMCR的位24控制，写1启用

2. 在启用CYCCNT寄存器之前，先清零。

3. 启用CYCCNT寄存器，这个由DWT的CYCCNTENA 控制，也就是DWT控制寄存器的位0控制，写1启用

这样子，我们就能去看看μC/OS的时间戳的初始化了，具体见代码清单16‑9

代码清单16‑9 CPU_TS_TmrInit()源码

1 #define DWT_CR \*(CPU_REG32 \*)0xE0001000

2 #define DWT_CYCCNT \*(CPU_REG32 \*)0xE0001004

3 #define DEM_CR \*(CPU_REG32 \*)0xE000EDFC

4

5 #define DEM_CR_TRCENA (1 << 24)

6

7 #define DWT_CR_CYCCNTENA (1 << 0)

8

9 #if (CPU_CFG_TS_TMR_EN == DEF_ENABLED)

10 void CPU_TS_TmrInit (void)

11 {

12 CPU_INT32U cpu_clk_freq_hz;

13

14 /\* Enable Cortex-M3's DWT CYCCNT reg.
\*/

15 DEM_CR \|= (CPU_INT32U)DEM_CR_TRCENA;

16

17 DWT_CYCCNT = (CPU_INT32U)0u;

18 DWT_CR \|= (CPU_INT32U)DWT_CR_CYCCNTENA;

19

20 cpu_clk_freq_hz = BSP_CPU_ClkFreq();

21 CPU_TS_TmrFreqSet(cpu_clk_freq_hz);

22 }

23 #endif

SysTick初始化
^^^^^^^^^^

时钟节拍的频率表示操作系统每1秒钟产生多少个tick，tick即是操作系统节拍的时钟周期，时钟节拍就是系统以固定的频率产生中断（时基中断），并在中断中处理与时间相关的事件，推动所有任务向前运行。时钟节拍需要依赖于硬件定时器，在STM32 裸机程序中经常使用的SysTick
时钟是MCU的内核定时器，通常都使用该定时器产生操作系统的时钟节拍。用户需要先在“ os_cfg_app.h”中设定时钟节拍的频率，该频率越高，操作系统检测事件就越频繁，可以增强任务的实时性，但太频繁也会增加操作系统内核的负担加重，所以用户需要权衡该频率的设置。我们在这里采用默认的 1000
Hz（本书之后若无特别声明，均采用 1000 Hz），也就是时钟节拍的周期为 1 ms。

函数OS_CPU_SysTickInit()用于初始化时钟节拍中断，初始化中断的优先级，SysTick中断的启用等等，这个函数要跟不同的CPU
进行编写，并且在系统任务的第一个任务开始的时候进行调用，如果在此之前进行调用，可能会造成系统奔溃，因为系统还没有初始化好就进入中断，可能在进入和退出中断的时候会调用系统未初始化好的一些模块，具体见代码清单16‑10。

代码清单16‑10SysTick初始化

1 cpu_clk_freq = BSP_CPU_ClkFreq();/\* Determine SysTick reference freq.
\*/

2 cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;

3 OS_CPU_SysTickInit(cnts); /*Init μC/OS periodic time src (SysTick).*/

内存初始化
^^^^^

我们都知道，内存在嵌入式中是很珍贵的存在，而一个系统它是软件，则必须要有一块内存属于系统所管理的，所以在系统创建任务之前，就必须将系统必要的东西进行初始化，μC/OS采用一块连续的大数组作为系统管理的内存，CPU_INT08U
Mem_Heap[LIB_MEM_CFG_HEAP_SIZE]，在使用之前就需要先将管理的内存进行初始化，具体见代码清单16‑11。

代码清单16‑11内存初始化

1 Mem_Init(); /\* Initialize Memory Management Module \*/

OSStart()
^^^^^^^^^

在创建完任务的时候，我们需要开启调度器，因为创建仅仅是把任务添加到系统中，还没真正调度，那怎么才能让系统支持运行呢，μC/OS为我们提供一个系统启动的函数接口——OSStart()，我们使用OSStart()函数就能让系统开始运行，具体见代码清单16‑12。

代码清单16‑12vTaskStartScheduler()函数

1 void OSStart (OS_ERR \*p_err)

2 {

3 #ifdef OS_SAFETY_CRITICAL

4 if (p_err == (OS_ERR \*)0) {

5 OS_SAFETY_CRITICAL_EXCEPTION();

6 return;

7 }

8 #endif

9

10 if (OSRunning == OS_STATE_OS_STOPPED) {

11 OSPrioHighRdy = OS_PrioGetHighest();/\* Find the highest priority \*/

12 OSPrioCur = OSPrioHighRdy;

13 OSTCBHighRdyPtr = OSRdyList[OSPrioHighRdy].HeadPtr;

14 OSTCBCurPtr = OSTCBHighRdyPtr;

15 OSRunning = OS_STATE_OS_RUNNING;

16 OSStartHighRdy();/\* Execute target specific code to start task \*/

17 \*p_err = OS_ERR_FATAL_RETURN;

18 /\* OSStart() is not supposed to return \*/

19 } else {

20 \*p_err = OS_ERR_OS_RUNNING; /\* OS is already running \*/

21

22 }

23 }

关于任务切换的详细过程在第一部分已经讲解完毕，此处就不再重复赘述。

app.c
^^^^^

当我们拿到一个移植好μC/OS的例程的时候，不出意外，你首先看到的是main()函数，当你认真一看main()函数里面只是让系统初始化和硬件初始化，然后创建并启动一些任务，具体见代码清单16‑13。因为这样子高度封装的函数让我们使用起来非常方便，防止用户一不小心忘了初始化系统的某些必要资源，造成系统
启动失败，而作为用户，如果只是单纯使用μC/OS的话，无需太过于关注μC/OS 接口函数里面的实现过程，但是我们还是建议需要深入了解μC/OS然后再去使用，避免出现问题。

代码清单16‑13 main()函数

1 int main (void)

2 {

3 OS_ERR err;

4

5

6 OSInit(&err); /\* Init μC/OS-III.
\*/

7

8

9 OSTaskCreate((OS_TCB \*)&AppTaskStartTCB,/*Create the start task*/

10

11 (CPU_CHAR \*)"App Task Start",

12 (OS_TASK_PTR ) AppTaskStart, **(1)**

13 (void \*) 0,

14 (OS_PRIO ) APP_TASK_START_PRIO,

15 (CPU_STK \*)&AppTaskStartStk[0],

16 (CPU_STK_SIZE) APP_TASK_START_STK_SIZE / 10,

17 (CPU_STK_SIZE) APP_TASK_START_STK_SIZE,

18 (OS_MSG_QTY ) 5u,

19 (OS_TICK ) 0u,

20 (void \*) 0,

21 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

22 (OS_ERR \*)&err);

23 /*Start multitasking (i.e.
give control to μC/OS-III)*/

24 OSStart(&err); **(2)**

25

26 }

代码清单16‑13\ **(1)**\ ：系统初始化完成，就创建一个AppTaskStart任务，在AppTaskStart任务中创建各种应用任务，具体见代码清单16‑14。

代码清单16‑14 AppTaskCreate函数

1 static void AppTaskStart (void \*p_arg)

2 {

3 CPU_INT32U cpu_clk_freq;

4 CPU_INT32U cnts;

5 OS_ERR err;

6

7

8 (void)p_arg;

9

10 BSP_Init(); /\* Initialize BSP functions \*/

11

12 CPU_Init();

13

14 cpu_clk_freq = BSP_CPU_ClkFreq();/*Determine SysTick reference freq*/

15 /\* Determine nbr SysTick increments \*/

16 cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;

17

18 OS_CPU_SysTickInit(cnts); /*Init μC/OS periodic time src (SysTick) \*/

19

20

21 Mem_Init(); /\* Initialize Memory Management Module \*/

22

23

24 #if OS_CFG_STAT_TASK_EN > 0u

25 /\* Compute CPU capacity with no task running*/

26 OSStatTaskCPUUsageInit(&err);

27 #endif

28

29 CPU_IntDisMeasMaxCurReset();

30

31

32 OSTaskCreate((OS_TCB \*)&AppTaskLed1TCB, /*Create the Led1 task*/

33 (CPU_CHAR \*)"App Task Led1",

34 (OS_TASK_PTR ) AppTaskLed1,

35 (void \*) 0,

36 (OS_PRIO ) APP_TASK_LED1_PRIO,

37 (CPU_STK \*)&AppTaskLed1Stk[0],

38 (CPU_STK_SIZE) APP_TASK_LED1_STK_SIZE / 10,

39 (CPU_STK_SIZE) APP_TASK_LED1_STK_SIZE,

40 (OS_MSG_QTY ) 5u,

41 (OS_TICK ) 0u,

42 (void \*) 0,

43 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

44 (OS_ERR \*)&err);

45

46 OSTaskCreate((OS_TCB \*)&AppTaskLed2TCB, /\* Create the Led2 task*/

47 (CPU_CHAR \*)"App Task Led2",

48 (OS_TASK_PTR ) AppTaskLed2,

49 (void \*) 0,

50 (OS_PRIO ) APP_TASK_LED2_PRIO,

51 (CPU_STK \*)&AppTaskLed2Stk[0],

52 (CPU_STK_SIZE) APP_TASK_LED2_STK_SIZE / 10,

53 (CPU_STK_SIZE) APP_TASK_LED2_STK_SIZE,

54 (OS_MSG_QTY ) 5u,

55 (OS_TICK ) 0u,

56 (void \*) 0,

57 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

58 (OS_ERR \*)&err);

59

60 OSTaskCreate((OS_TCB \*)&AppTaskLed3TCB, /\* Create the Led3 task*/

61 (CPU_CHAR \*)"App Task Led3",

62 (OS_TASK_PTR ) AppTaskLed3,

63 (void \*) 0,

64 (OS_PRIO ) APP_TASK_LED3_PRIO,

65 (CPU_STK \*)&AppTaskLed3Stk[0],

66 (CPU_STK_SIZE) APP_TASK_LED3_STK_SIZE / 10,

67 (CPU_STK_SIZE) APP_TASK_LED3_STK_SIZE,

68 (OS_MSG_QTY ) 5u,

69 (OS_TICK ) 0u,

70 (void \*) 0,

71 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

72 (OS_ERR \*)&err);

73

74

75 OSTaskDel ( & AppTaskStartTCB, & err );

76 }

当在AppTaskStart中创建的应用任务的优先级比AppTaskStart任务的优先级高、低或者相等时候，程序是如何执行的？假如像我们代码一样在临界区创建任务，任务只能在退出临界区的时候才执行最高优先级任务。假如没使用临界区的话，就会分三种情况：

1. 应用任务的优先级比初始任务的优先级高，那创建完后立马去执行刚刚创建的应用任务，当应用任务被阻塞时，继续回到初始任务被打断的地方继续往下执行，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命；

2. 应用任务的优先级与初始任务的优先级一样，那创建完后根据任务的时间片来执行，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命；

3. 应用任务的优先级比初始任务的优先级低，那创建完后任务不会被执行，如果还有应用任务紧接着创建应用任务，如果应用任务的优先级出现了比初始任务高或者相等的情况，请参考1和2的处理方式，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命。

代码清单16‑13\ **(2)**\
：在启动任务调度器的时候，假如启动成功的话，任务就不会有返回了，假如启动没成功，则通过LR寄存器指定的地址退出，在创建AppTaskStart任务的时候，任务栈对应LR寄存器指向是任务退出函数OS_TaskReturn()，当系统启动没能成功的话，系统就不会运行。

.. |startu002| image:: media\startu002.png
   :width: 5.76806in
   :height: 4.82292in
.. |startu003| image:: media\startu003.png
   :width: 5.76806in
   :height: 3.55417in
.. |startu004| image:: media\startu004.png
   :width: 5.40069in
   :height: 3.28472in
