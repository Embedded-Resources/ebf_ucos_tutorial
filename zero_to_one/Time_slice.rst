.. vim: syntax=rst

任务时间片运行
==================
本章在上一章的基础上，加入SysTick中断，在SysTick中断服务函数里面进行任务切换，从而实现双任务的时间片运行，即每个任务运行的时间都是一样的。

SysTick简介
~~~~~~~~~

RTOS需要一个时基来驱动，系统任务调度的频率等于该时基的频率。通常该时基由一个定时器来提供，也可以从其他周期性的信号源获得。刚好Cortex-
M内核中有一个系统定时器SysTick，它内嵌在NVIC中，是一个24位的递减的计数器，计数器每计数一次的时间为1/SYSCLK。当重装载数值寄存器的值递减到0的时候，系统定时器就产生一次中断，以此循环往复。因为SysTick是嵌套在内核中的，所以使得OS在Cortex-
M器件中编写的定时器代码不必修改，使移植工作一下子变得简单很多。所以SysTick是最适合给操作系统提供时基，用于维护系统心跳的定时器。有关SysTick的寄存器汇总见表4‑1，各个寄存器的用法见表4‑2、表4‑3、表4‑4和表4‑5。

表4‑1SysTick寄存器汇总

================= =================================
寄存器名称        寄存器描述
================= =================================
CTRL              SysTick控制及状态寄存器
LOAD              SysTick重装载数值寄存器
VAL               SysTick当前数值寄存器

================= =================================

表4‑2SysTick控制及状态寄存器

.. list-table::
   :widths: 20 20 20 20 20
   :header-rows: 0


   * - 位段 |
     - 称      | 类型
     - | 复位值
     - 描述
     - |

   * - 16
     - COUNTFLAG
     - R/W
     - 0
     - 如果在上次读取本寄存器后，       | SysTick 已经计到                 | 了 0，则该位为 1。               |

   * - 2
     - CLKSOURCE
     - R/W
     - 0
     - 时钟源                           | 选择位，0=AHB/8，1=处理器时钟AHB |

   * - 1
     - TICKINT
     - R/W
     - 0
     - 1=SysTick倒数计数到 0时产生      | SysTick异常请                    | 求，0=数到 0                     | 时无动作。也可以通过读取COUNTF   | LAG标志位来确定计数器是否递减到0 |

   * - 0
     - ENABLE
     - R/W
     - 0
     - SysTick 定时器的启用位           |


表4‑3SysTick 重装载数值寄存器

==== ====== ==== ====== ================================
位段 名称   类型 复位值 描述
==== ====== ==== ====== ================================
23:0 RELOAD R/W  0      当倒数计数至零时，将被重装载的值
==== ====== ==== ====== ================================

表4‑4SysTick当前数值寄存器

.. list-table::
   :widths: 20 20 20 20 20
   :header-rows: 0


   * - 位段 |
     - 称    | 类型
     - | 复位值
     - 描述
     - |

   * - 23:0
     - CURRENT
     - R/W
     - 0
     - 读取时返回当前倒计数的值           | ，写它则使之清零，同时还会清除在Sy | sTick控制及状态寄存器中的COUNTFLAG | 标志                               |


.. list-table::
   :widths: 20 20 20 20 20
   :header-rows: 0


   * -  |
     - [STRI EOUT:名称] | KE
     - [STRI T:类型] | OUT:
     - [STRIKE 值] | KEOUT:描
     - [STRI |

   * -
     - [STRIK EOUT:NOREF]
     - [S TRIKEOUT:R]
     - [S TRIKEOUT:0]
     - [STRI KEOUT:NOREF flag.
       Reads as zero.
       Indicates that a separate reference clock is provided.
       The frequency of this clock is HCLK/8]

   * -
     - [STRI KEOUT:SKEW]
     - [S TRIKEOUT:R]
     - [S TRIKEOUT:1]
     - [STRI KEOUT:Reads as one.
       Calibration value for the 1 ms inexact timing is not known because TENMS is not known.
       This can affect the suitability of SysTick as a software real time clock]

   * -
     - [STRIK EOUT:TENMS]
     - [S TRIKEOUT:R]
     - [S TRIKEOUT:0]
     - [STRIKEOU T:Indicates the calibration value when the SysTick counter runs on HCLK max/8 as external clock.
       The value is product dependent, please refer to the Product Reference Manual, SysTick Calibration Value section.
       When HCLK is programmed at the maximum frequency, the SysTick period is 1ms.
       If calibration information is not known, calculate the calibration value required from the frequency of the processor clock or external clock.]


初始化SysTick
~~~~~~~~~~

使用SysTick非常简单，只需一个初始化函数搞定，OS_CPU_SysTickInit函数在os_cpu_c.c中定义，具体实现见代码清单4‑1。在这里，SysTick初始化函数我们没有使用μC/OS-III官方的，我们是自己另外编写了一个。区别是uC/OS-
III官方的OS_CPU_SysTickInit函数里面涉及SysTick寄存器都是重新在cpu.h中定义，而我们自己编写的则是使用ARMCM3.h（记得在os_cpu_c.c的开头包含ARMCM3.h这个头文件）这个固件库文件里面定义的寄存器，仅此区别而已。

代码清单4‑1SysTick初始化

1 #if 0/\* 不用μC/OS-III自带的 \*/

2 void OS_CPU_SysTickInit (CPU_INT32U cnts)

3 {

4 CPU_INT32U prio;

5

6 /\* 填写 SysTick 的重载计数值 \*/

7 CPU_REG_NVIC_ST_RELOAD = cnts - 1u;

8

9 /\* 设置 SysTick 中断优先级 \*/

10 prio = CPU_REG_NVIC_SHPRI3;

11 prio &= DEF_BIT_FIELD(24, 0);

12 prio \|= DEF_BIT_MASK(OS_CPU_CFG_SYSTICK_PRIO, 24);

13

14 CPU_REG_NVIC_SHPRI3 = prio;

15

16 /\* 启用 SysTick 的时钟源和启动计数器 \*/

17 CPU_REG_NVIC_ST_CTRL \|= CPU_REG_NVIC_ST_CTRL_CLKSOURCE \|

18 CPU_REG_NVIC_ST_CTRL_ENABLE;

19 /\* 启用 SysTick 的定时中断 \*/

20 CPU_REG_NVIC_ST_CTRL \|= CPU_REG_NVIC_ST_CTRL_TICKINT;

21 }

22

23 #else/\* 直接使用头文件ARMCM3.h里面现有的寄存器定义和函数来实现 \*/

24 void OS_CPU_SysTickInit (CPU_INT32U ms)

25 {

26 /\* 设置重装载寄存器的值 \*/

27 SysTick->LOAD = ms \* SystemCoreClock / 1000 - 1;(1)

28

29 /\* 配置中断优先级为最低 \*/

30 NVIC_SetPriority (SysTick_IRQn, (1<<__NVIC_PRIO_BITS) - 1);(2)

31

32 /\* 复位当前计数器的值 \*/

33 SysTick->VAL = 0;(3)

34

35 /\* 选择时钟源、启用中断、启用计数器 \*/

36 SysTick->CTRL = SysTick_CTRL_CLKSOURCE_Msk \|(4)

37 SysTick_CTRL_TICKINT_Msk \|(5)

38 SysTick_CTRL_ENABLE_Msk;(6)

39 }

40 #endif

代码清单4‑1（1）：配置重装载寄存器的值，我们配合函数形参ms来配置，如果需要配置为10ms产生一次中断，形参设置为10即可。

代码清单4‑1（2）：配置SysTick的优先级，这里配置为15，即最低。

代码清单4‑1（3）：复位当前计数器的值。

代码清单4‑1（4）：选择时钟源，这里选择SystemCoreClock。

代码清单4‑1（5）：启用中断。

代码清单4‑1（6）：启用计数器开始计数。

编写SysTick中断服务函数
~~~~~~~~~~~~~~~

SysTick中断服务函数也是在os_cpu_c.c中定义，具体实现见代码清单4‑2。

代码清单4‑2SysTick中断服务函数

1 /\* SysTick 中断服务函数 \*/

2 void SysTick_Handler(void)

3 {

4 OSTimeTick();

5 }

SysTick中断服务函数很简单，里面仅调用了函数OSTimeTick()。OSTimeTick()是与时间相关的函数，在os_time.c（os_time.c第一次使用需要自行在文件夹μC/OS-III\Source中新建并添加到工程的μC/OS-III
Source组）文件中定义，具体实现见代码清单4‑3。

代码清单4‑3OSTimeTick()函数

1 void OSTimeTick (void)

2 {

3 /\* 任务调度 \*/

4 OSSched();

5 }

OSTimeTick()很简单，里面仅调用了函数OSSched，OSSched函数暂时没有修改，与上一章一样，具体见代码清单4‑4。

代码清单4‑4 OSSched函数

1 void OSSched (void)

2 {

3 if ( OSTCBCurPtr == OSRdyList[0].HeadPtr ) {

4 OSTCBHighRdyPtr = OSRdyList[1].HeadPtr;

5 } else {

6 OSTCBHighRdyPtr = OSRdyList[0].HeadPtr;

7 }

8

9 OS_TASK_SW();

10 }

main()函数
~~~~~~~~

main()函数与上一章区别不大，仅仅是加入了SysTick相关的内容，具体见代码清单4‑5。

代码清单4‑5 main()函数和任务代码

1 int main(void)

2 {

3 OS_ERR err;

4

5 **/\* 关闭中断 \*/**

6 **CPU_IntDis();(1)**

7

8 **/\* 配置SysTick 10ms 中断一次 \*/**

9 **OS_CPU_SysTickInit (10);(2)**

10

11 /\* 初始化相关的全局变量 \*/

12 OSInit(&err);

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

37

38

39 /\* 任务1 \*/

40 void Task1( void \*p_arg )

41 {

42 for ( ;; ) {

43 flag1 = 1;

44 delay( 100 );

45 flag1 = 0;

46 delay( 100 );

47

48 /\* 任务切换，这里是手动切换 \*/

49 **//OSSched();(3)**

50 }

51 }

52

53 /\* 任务2 \*/

54 void Task2( void \*p_arg )

55 {

56 for ( ;; ) {

57 flag2 = 1;

58 delay( 100 );

59 flag2 = 0;

60 delay( 100 );

61

62 /\* 任务切换，这里是手动切换 \*/

63 **//OSSched();(4)**

64 }

65 }

代码清单4‑5（1）：关闭中断。因为在OS系统初始化之前我们启用了SysTick定时器产生10ms的中断，在中断里面触发任务调度，如果一开始我们不关闭中断，就会在OS还有启动之前就进入SysTick中断，然后发生任务调度，既然OS都还没启动，那调度是不允许发生的，所以先关闭中断。系统启动后，中断由O
SStart()函数里面的OSStartHighRdy()重新开启。

代码清单4‑5（2）：配置SysTick为10ms中断一次。任务的调度是在SysTick的中断服务函数中完成的，中断的频率越高就意味着OS的调度越高，系统的负荷就越重，一直在不断的进入中断，则执行任务的时间就减小。选择合适的SysTick中断频率会提供系统的运行效率，μC/OS-
III官方推荐为10ms，或者高点也行。

代码清单4‑5（3）、（4）：任务调度将不再在各自的任务里面实现，而是放到了SysTick中断服务函数中。从而实现每个任务都运行相同的时间片，平等的享有CPU。

实验现象
~~~~

进入软件调试，单击全速运行按钮就可看到实验波形，具体见图4‑1。

|Timesl002|

图4‑1实验现象

从图4‑1我们可以看到，两个任务轮流的占有CPU，享有相同的时间片。其实目前的实验现象与上一章的实验现象还没有本质上的区别，加入SysTick只是为了后续章节做准备。上一章两个任务也是轮流的占有CPU，也是享有相同的时间片，该时间片是任务单次运行的时间。不同的是本章任务的时间片等于SysTick定时
器的时基，是很多个任务单次运行时间的综合。即在这个时间片里面任务运行了非常多次，如果我们把波形放大，就会发现大波形里面包含了很多小波形，具体见图4‑2。

|Timesl003|

图4‑2实验现象2

.. |Timesl002| image:: media\Timesl002.png
   :width: 4.73125in
   :height: 2.3375in
.. |Timesl003| image:: media\Timesl003.png
   :width: 3.86458in
   :height: 2.39514in
