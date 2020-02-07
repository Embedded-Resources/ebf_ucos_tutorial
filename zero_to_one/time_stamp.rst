.. vim: syntax=rst

时间戳
==============

本章实现时间戳用的是ARM Cortex-M系列内核中的DWT这个外设的功能，有关这个外设的功能和寄存器说明具体见手册“STM32F10xxx Cortex-M3 programming manual”

时间戳简介
~~~~~

在μC/OS-III中，很多地方的代码都加入了时间测量的功能，比如任务关中断的时间，关调度器的时间等。知道了某段代码的运行时间，就明显地知道该代码的执行效率，如果时间过长就可以优化或者调整代码策略。如果要测量一段代码A的时间，那么可以在代码段A运行前记录一个时间点TimeStart，在代码段A运行完
记录一个时间点TimeEnd，那么代码段A的运行时间TimeUse就等于TimeEnd减去TimeStart。这里面的两个时间点TimeEnd和TimeStart，就叫作时间戳，时间戳实际上就是一个时间点。

时间戳的实现
~~~~~~

通常执行一条代码是需要多个时钟周期的，即是ns级别。要想准确测量代码的运行时间，时间戳的精度就很重要。通常单片机中的硬件定时器的精度都是us级别，远达不到测量几条代码运行时间的精度。

在ARM Cortex-M系列内核中，有一个DWT的外设，该外设有一个32位的寄存器叫CYCCNT，它是一个向上的计数器，记录的是内核时钟HCLK运行的个数，当CYCCNT溢出之后，会清零重新开始向上计数。该计数器在μC/OS-III中正好被用来实现时间戳的功能。

在STM32F103系列的单片机中，HCLK时钟最高为72M，单个时钟的周期为1/72us = 0.0139us = 14ns，CYCCNT总共能记录的时间为2\ :sup:`32`\ \*14=60S。在μC/OS-
III中，要测量的时间都是很短的，都是ms级别，根本不需要考虑定时器溢出的问题。如果内核代码执行的时间超过s的级别，那就背离了实时操作系统实时的设计初衷了，没有意义。

时间戳代码讲解
~~~~~~~

CPU_Init()函数
^^^^^^^^^^^^

CPU_Init()函数在cpu_core.c（cpu_core.c文件第一次使用需要自行在文件夹uC-CPU中新建并添加到工程的uC-
CPU组）中实现，主要做三件事：1、初始化时间戳，2、初始化中断禁用时间测量，3、初始化CPU名字。第2和3个功能目前还没有使用到，只实现了第1个初始化时间戳的代码，具体见代码清单6‑1。

代码清单6‑1 CPU_Init()函数

1 /\* CPU初始化函数 \*/

2 void CPU_Init (void)

3 {

4 /\* CPU初始化函数中总共做了三件事

5 1、初始化时间戳

6 2、初始化中断禁用时间测量

7 3、初始化CPU名字

8 这里只讲时间戳功能，剩下两个的初始化代码则删除不讲 \*/

9

10 #if ((CPU_CFG_TS_EN == DEF_ENABLED) \|\| \\(1)

11 (CPU_CFG_TS_TMR_EN == DEF_ENABLED))

12 CPU_TS_Init();(2)

13 #endif

14

15 }

代码清单6‑1（1）：CPU_CFG_TS_EN和CPU_CFG_TS_TMR_EN这两个宏在cpu_core.h中定义，用于控制时间戳相关的功能代码，具体定义见代码清单6‑2。

代码清单6‑2CPU_CFG_TS_EN和CPU_CFG_TS_TMR_EN宏定义

1 #if ((CPU_CFG_TS_32_EN == DEF_ENABLED) \|\| \\(1)

2 (CPU_CFG_TS_64_EN == DEF_ENABLED))

3 #define CPU_CFG_TS_EN DEF_ENABLED

4 #else

5 #define CPU_CFG_TS_EN DEF_DISABLED

6 #endif

7

8 #if ((CPU_CFG_TS_EN == DEF_ENABLED) \|\| \\

9 (defined(CPU_CFG_INT_DIS_MEAS_EN)))

10 #define CPU_CFG_TS_TMR_EN DEF_ENABLED

11 #else

12 #define CPU_CFG_TS_TMR_EN DEF_DISABLED

13 #endif

代码清单6‑2（1）：CPU_CFG_TS_32_EN和CPU_CFG_TS_64_EN这两个宏在cpu_cfg.h（cpu_cfg.h文件第一次使用需要自行在文件夹uC-CPU中新建并添加到工程的uC-CPU组）文件中定义，用于控制时间戳是32位还是64位的，默认启用32位，具体见代码清单6‑3。

代码清单6‑3CPU_CFG_TS_32_EN和CPU_CFG_TS_64_EN宏定义

1 #ifndef CPU_CFG_MODULE_PRESENT

2 #define CPU_CFG_MODULE_PRESENT

3

4

5 #define CPU_CFG_TS_32_EN DEF_ENABLED

6 #define CPU_CFG_TS_64_EN DEF_DISABLED

7

8 #define CPU_CFG_TS_TMR_SIZE CPU_WORD_SIZE_32

9

10

11 #endif/\* CPU_CFG_MODULE_PRESENT \*/

CPU_TS_Init()函数
^^^^^^^^^^^^^^^

代码清单6‑1（2）：CPU_TS_Init()是时间戳初始化函数，在cpu_core.c中实现，具体见代码清单6‑4。

代码清单6‑4CPU_TS_Init()函数

1 #if ((CPU_CFG_TS_EN == DEF_ENABLED) \|\| \\

2 (CPU_CFG_TS_TMR_EN == DEF_ENABLED))

3 static void CPU_TS_Init (void)

4 {

5

6 #if (CPU_CFG_TS_TMR_EN == DEF_ENABLED)

7 CPU_TS_TmrFreq_Hz = 0u;(1)

8 CPU_TS_TmrInit();(2)

9 #endif

10

11 }

12 #endif

代码清单6‑4（1）：CPU_TS_TmrFreq_Hz是一个在cpu_core.h中定义的全局变量，表示CPU的系统时钟，具体大小跟硬件相关，如果使用STM32F103系列，那就等于72000000HZ。CPU_TS_TmrFreq_Hz变量的定义和时间戳相关的数据类型的定义具体见代码清单6‑5。

代码清单6‑5CPU_TS_TmrFreq_Hz和时间戳相关的数据类型定义

1 /\*

2 \\*

3 \* EXTERNS

4 \* 在cpu_core.h开头定义

5 \\*

6 \*/

7

8 #ifdef CPU_CORE_MODULE/\* CPU_CORE_MODULE 只在cpu_core.c文件的开头定义 \*/

9 #define CPU_CORE_EXT

10 #else

11 #define CPU_CORE_EXT extern

12 #endif

13

14 /\*

15 \\*

16 \* 时间戳数据类型

17 \* 在cpu_core.h文件定义

18 \\*

19 \*/

20

21 typedef CPU_INT32U CPU_TS32;

22

23 typedef CPU_INT32U CPU_TS_TMR_FREQ;

24 typedef CPU_TS32 CPU_TS;

25 typedef CPU_INT32U CPU_TS_TMR;

26

27

28 /\*

29 \\*

30 \* 全局变量

31 \* 在cpu_core.h文件定义

32 \\*

33 \*/

34

35 #if (CPU_CFG_TS_TMR_EN == DEF_ENABLED)

36 CPU_CORE_EXT CPU_TS_TMR_FREQ CPU_TS_TmrFreq_Hz;

37 #endif

CPU_TS_TmrInit()函数
^^^^^^^^^^^^^^^^^^

代码清单6‑4（2）：时间戳定时器初始化函数CPU_TS_TmrInit()在cpu_core.c实现，具体见代码清单6‑6。

代码清单6‑6CPU_TS_TmrInit()函数

1 /\* 时间戳定时器初始化 \*/

2 #if (CPU_CFG_TS_TMR_EN == DEF_ENABLED)

3 void CPU_TS_TmrInit (void)

4 {

5 CPU_INT32U fclk_freq;

6

7

8 fclk_freq = BSP_CPU_ClkFreq();(2)

9

10 /\* 启用DWT外设 \*/

11 BSP_REG_DEM_CR \|= (CPU_INT32U)BSP_BIT_DEM_CR_TRCENA;(1)

12 /\* DWT CYCCNT寄存器计数清零 \*/

13 BSP_REG_DWT_CYCCNT = (CPU_INT32U)0u;

14 /\* 注意：当使用软件仿真全速运行的时候，会先停在这里，

15 就好像在这里设置了一个断点一样，需要手动运行才能跳过，

16 当使用硬件仿真的时候却不会 \*/

17 /\* 启用Cortex-M3 DWT CYCCNT寄存器 \*/

18 BSP_REG_DWT_CR \|= (CPU_INT32U)BSP_BIT_DWT_CR_CYCCNTENA;

19

20 CPU_TS_TmrFreqSet((CPU_TS_TMR_FREQ)fclk_freq);(3)

21 }

22 #endif

代码清单6‑6（1）：初始化时间戳计数器CYCCNT，启用CYCCNT计数的操作步骤：

1、先启用DWT外设，这个由另外内核调试寄存器DEMCR的位24控制，写1启用。

2、启用CYCCNT寄存器之前，先清零。

3、启用CYCCNT寄存器，这个由DWT_CTRL(代码上宏定义为DWT_CR)的位0控制，写1启用。这三个步骤里面涉及的寄存器定义在cpu_core.c文件的开头，具体见代码清单6‑7。

代码清单6‑7 DWT外设相关寄存器定义

1 /\*

2 \\*

3 \* 寄存器定义

4 \\*

5 \*/

6 #define BSP_REG_DEM_CR (*(CPU_REG32 \*)0xE000EDFC)

7 #define BSP_REG_DWT_CR (*(CPU_REG32 \*)0xE0001000)

8 #define BSP_REG_DWT_CYCCNT (*(CPU_REG32 \*)0xE0001004)

9 #define BSP_REG_DBGMCU_CR (*(CPU_REG32 \*)0xE0042004)

10

11 /\*

12 \\*

13 \* 寄存器位定义

14 \\*

15 \*/

16

17 #define BSP_DBGMCU_CR_TRACE_IOEN_MASK 0x10

18 #define BSP_DBGMCU_CR_TRACE_MODE_ASYNC 0x00

19 #define BSP_DBGMCU_CR_TRACE_MODE_SYNC_01 0x40

20 #define BSP_DBGMCU_CR_TRACE_MODE_SYNC_02 0x80

21 #define BSP_DBGMCU_CR_TRACE_MODE_SYNC_04 0xC0

22 #define BSP_DBGMCU_CR_TRACE_MODE_MASK 0xC0

23

24 #define BSP_BIT_DEM_CR_TRCENA (1<<24)

25

26 #define BSP_BIT_DWT_CR_CYCCNTENA (1<<0)

BSP_CPU_ClkFreq()函数
^^^^^^^^^^^^^^^^^^^

代码清单6‑6（2）：BSP_CPU_ClkFreq()是一个用于获取CPU的HCLK时钟的BSP函数，具体跟硬件相关，目前只是使用软件仿真，则把硬件相关的代码注释掉，直接手动设置CPU的HCLK的时钟等于软件仿真的时钟25000000HZ。BSP_CPU_ClkFreq()在cpu_core.c实
现，具体定义见代码清单6‑8。

代码清单6‑8BSP_CPU_ClkFreq()函数

1 /\* 获取CPU的HCLK时钟

2 这个是跟硬件相关的，目前我们是软件仿真，我们暂时把跟硬件相关的代码屏蔽掉，

3 直接手动设置CPU的HCLK时钟*/

4 CPU_INT32U BSP_CPU_ClkFreq (void)

5 {

6 #if 0

7 RCC_ClocksTypeDef rcc_clocks;

8

9

10 RCC_GetClocksFreq(&rcc_clocks);

11 return ((CPU_INT32U)rcc_clocks.HCLK_Frequency);

12 #else

13 CPU_INT32U CPU_HCLK;

14

15

16 /\* 目前软件仿真我们使用25M的系统时钟 \*/

17 CPU_HCLK = 25000000;

18

19 return CPU_HCLK;

20 #endif

21 }

CPU_TS_TmrFreqSet()函数
^^^^^^^^^^^^^^^^^^^^^

代码清单6‑6（3）：CPU_TS_TmrFreqSet()函数在cpu_core.c定义，具体的作用是把函数BSP_CPU_ClkFreq()获取到的CPU的HCLK时钟赋值给全局变量CPU_TS_TmrFreq_Hz，具体实现见代码清单6‑9。

代码清单6‑9CPU_TS_TmrFreqSet()函数

1 /\* 初始化CPU_TS_TmrFreq_Hz，这个就是系统的时钟，单位为HZ \*/

2 #if (CPU_CFG_TS_TMR_EN == DEF_ENABLED)

3 void CPU_TS_TmrFreqSet (CPU_TS_TMR_FREQ freq_hz)

4 {

5 CPU_TS_TmrFreq_Hz = freq_hz;

6 }

7 #endif

CPU_TS_TmrRd()函数
^^^^^^^^^^^^^^^^

CPU_TS_TmrRd()函数用于获取CYCNNT计数器的值，在cpu_core.c中定义，具体实现见代码清单6‑10。

代码清单6‑10CPU_TS_TmrRd()函数

1 #if (CPU_CFG_TS_TMR_EN == DEF_ENABLED)

2 CPU_TS_TMR CPU_TS_TmrRd (void)

3 {

4 CPU_TS_TMR ts_tmr_cnts;

5

6

7 ts_tmr_cnts = (CPU_TS_TMR)BSP_REG_DWT_CYCCNT;

8

9 return (ts_tmr_cnts);

10 }

11 #endif

OS_TS_GET()函数
^^^^^^^^^^^^^

OS_TS_GET()函数用于获取CYCNNT计数器的值，实际上是一个宏定义，将CPU底层的函数CPU_TS_TmrRd()重新取个名字封装，供内核和用户函数使用，在os_cpu.h头文件定义，具体实现见代码清单6‑11。

代码清单6‑11OS_TS_GET()函数

1 /\*

2 \\*

3 \* 时间戳配置

4 \\*

5 \*/

6 /\* 启用时间戳，在os_cfg.h头文件中启用 \*/

7 #define OS_CFG_TS_EN 1u

8

9 #if OS_CFG_TS_EN == 1u

10 #define OS_TS_GET() (CPU_TS)CPU_TS_TmrRd()

11 #else

12 #define OS_TS_GET() (CPU_TS)0u

13 #endif

main()函数
~~~~~~~~

主函数与上一章区别不大，首先在main()函数开头加入CPU_Init()函数，然后在任务1中对延时函数的执行时间进行测量。新加入的代码做了加粗显示，具体见代码清单6‑12。

代码清单6‑12主函数

**1 uint32_t TimeStart;/\* 定义三个全局变量 \*/**

**2 uint32_t TimeEnd;**

**3 uint32_t TimeUse;**

4

5

6 /\*

7 \\*

8 \* main()函数

9 \\*

10 \*/

11

12 int main(void)

13 {

14 OS_ERR err;

15

16

**17 /\* CPU初始化：1、初始化时间戳 \*/**

**18 CPU_Init();**

19

20 /\* 关闭中断 \*/

21 CPU_IntDis();

22

23 /\* 配置SysTick 10ms 中断一次 \*/

24 OS_CPU_SysTickInit (10);

25

26 /\* 初始化相关的全局变量 \*/

27 OSInit(&err);

28

29 /\* 创建任务 \*/

30 OSTaskCreate ((OS_TCB*) &Task1TCB,

31 (OS_TASK_PTR ) Task1,

32 (void \*) 0,

33 (CPU_STK*) &Task1Stk[0],

34 (CPU_STK_SIZE) TASK1_STK_SIZE,

35 (OS_ERR \*) &err);

36

37 OSTaskCreate ((OS_TCB*) &Task2TCB,

38 (OS_TASK_PTR ) Task2,

39 (void \*) 0,

40 (CPU_STK*) &Task2Stk[0],

41 (CPU_STK_SIZE) TASK2_STK_SIZE,

42 (OS_ERR \*) &err);

43

44 /\* 将任务加入到就绪列表 \*/

45 OSRdyList[0].HeadPtr = &Task1TCB;

46 OSRdyList[1].HeadPtr = &Task2TCB;

47

48 /\* 启动OS，将不再返回 \*/

49 OSStart(&err);

50 }

51

52 /\* 任务1 \*/

53 void Task1( void \*p_arg )

54 {

55 for ( ;; ) {

56 flag1 = 1;

57

**58 TimeStart = OS_TS_GET();**

59 OSTimeDly(20);

**60 TimeEnd = OS_TS_GET();**

**61 TimeUse = TimeEnd - TimeStart;**

62

63 flag1 = 0;

64 OSTimeDly(2);

65 }

66 }

实验现象
~~~~

时间戳时间测量功能在软件仿真的时候使用不了，只能硬件仿真，这里仅能够讲解代码功能。有关硬件仿真，本书有提供一个测量SysTick定时时间的例程，名称叫“7-SysTick—系统定时器 STM32 时间戳【硬件仿真】”，在配套的程序源码里面可以找到。
