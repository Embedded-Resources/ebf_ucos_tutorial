.. vim: syntax=rst

移植μC/OS-III到STM32
=========================

本章开始，先新建一个基于野火STM32全系列（包含M3/4/7）开发板的的μC/OS-III的工程模板，让μC/OS-III先跑起来。
以后所有的μC/OS-III相关的例程我们都在此模板上修改和添加代码，不用再反反复复地新建。在本书配套的例程中，
每一章的例程对野火STM32的每一个板子都会有一个对应的例程，但是区别都很小，如果有区别的地方我会在教程里面详细指出，
如果没有特别备注那么都是一样的。

获取STM32的裸机工程模板
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

STM32的裸机工程模板我们直接使用野火STM32开发板配套的固件库例程即可。
这里我们选取比较简单的例程—“GPIO输出—使用固件库点亮LED”作为裸机工程模板。
该裸机工程模板均可以在对应板子的A盘/程序源码/固件库例程的目录下获取到，下面以野火F103-霸道板子的光盘目录为例，
具体见图 STM32裸机工程模板在光盘资料中的位置_。

.. image:: media/porting_to_stm32/portin002.png
   :align: center
   :name: STM32裸机工程模板在光盘资料中的位置
   :alt: STM32裸机工程模板在光盘资料中的位置


下载μC/OS-III源码
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在移植之前，我们首先要获取到μC/OS-III的官方的源码包，首先，打开Micrium 公司官方网站（http://micrium.com/），
打开网站链接之后，我们单击“Downloads”选项卡进入下载页面，
在“Brouse by MCU Manufacturer”栏目展开“STMicroelectronics”，
单击“Viewall STMicroelectronics”，具体见图 Downloads_ 与图 Viewall_STMicroelectronics_。

.. image:: media/porting_to_stm32/portin003.png
   :align: center
   :name: Downloads
   :alt: Downloads

.. image:: media/porting_to_stm32/portin004.png
   :align: center
   :name: Viewall_STMicroelectronics
   :alt: Viewall STMicroelectronics


μC/OS-III是一个操作系统，其实也可以理解成一个软件库，它可以移植到多种硬件平台，如M3、M4、M7内核的STM32，或者ARM9等等其他芯片。
核心代码肯定是一致的，但是针对不同的处理器肯定要不同的实现部分。
这里选择与我们开发板最为接近的版本（STMicroelectronics STM32F107），
因为我们的野火STM32霸道开发板是M3内核的，而μC/OS-III的这个官方代码就满足我们的需求，目的也在于少花费工夫，要知道，
若要从0开始移植μC/OS-III到目标硬件平台，需要极大的精力和软件水平。

在“ Projects”栏目中选择一个基于 Keil MDK 平台在 cortex-M3 内核 MCU 评估板上测试的 μC/OS-III 源码，单击即可。
我们选择“STMicroelectronics STM32F107”这个项目的代码，单击后进入下载即可，不过μC/OS官网下载这些源码是需要注册账号的，
而我们野火已经将这些源码下载了，在配套源码中，这样子就免去下载这一步了，直接拿来使用即可，
具体见图 STMicroelectronics_STM32F107工程_ 、
图 STMicroelectronics_STM32F107工程下载_ 与图 μCOS-III源码_。

.. image:: media/porting_to_stm32/portin005.png
   :align: center
   :name: STMicroelectronics_STM32F107工程
   :alt: STMicroelectronics_STM32F107工程


.. image:: media/porting_to_stm32/portin006.png
   :align: center
   :name: STMicroelectronics_STM32F107工程下载
   :alt: STMicroelectronics_STM32F107工程下载


.. image:: media/porting_to_stm32/portin007.png
   :align: center
   :name: μCOS-III源码
   :alt: μC/OS-III源码


μC/OS-III源码文件介绍
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们从μC/OS-III源码下面的文件夹夹中看到，里面的文件夹不多，只有4个，分别是EvalBoards、
uC-CPU、uC-LIB、μC/OS-III，下面我们就来介绍一下这几个文件夹的作用。

EvalBoards
^^^^^^^^^^^^^^^^^^^^

EvalBoards文件夹里面包含评估板相关文件，在移植时我们只提取部分文件，具体见图 EvalBoards中提取的代码_ ，
然后我们在我们工程模板中的User文件夹下新建一个APP文件夹，然后将这9个文件复制到APP文件夹下，
具体见图 复制EvalBoards中的文件到APP文件夹下_。

.. image:: media/porting_to_stm32/portin008.png
   :align: center
   :name: EvalBoards中提取的代码
   :alt: EvalBoards中提取的代码


.. image:: media/porting_to_stm32/portin009.png
   :align: center
   :name: 复制EvalBoards中的文件到APP文件夹下
   :alt: 复制EvalBoards中的文件到APP文件夹下


将“EvalBoards\Micrium\uC-Eval-STM32F107\BSP”路径下的bsp.c、bsp.h文件复制到我们工程中的User\BSP文件夹下，
具体见图 提取bsp源码_ 与图 复制到工程中的User_BSP文件夹下_。

.. image:: media/porting_to_stm32/portin010.png
   :align: center
   :name: 提取bsp源码
   :alt: 提取bsp源码



.. image:: media/porting_to_stm32/portin011.png
   :align: center
   :name: 复制到工程中的User_BSP文件夹下
   :alt: 复制到工程中的User\BSP文件夹下


uC-CPU
^^^^^^^^^^^^

这是和CPU紧密相关的文件，里面的一些文件很重要，都是我们需要使用的。在ARM-Cortex-M3文件夹下，存在cpu_c.c一些对不同编译器移植相关的文件，
有GNU、IAR、RealView，里面都有一些很重要的文件，目前我们使用的开发环境是MDK（keil），所以我们选择RealView文件夹下的源码文件作为我们的讲解，
下面具体来介绍一下里面的文件，具体见图 uC-CPU文件夹下的源码文件_。

.. image:: media/porting_to_stm32/portin012.png
   :align: center
   :name: uC-CPU文件夹下的源码文件
   :alt: uC-CPU文件夹下的源码文件


cpu_c.h文件
'''''''''''''''''

包含了一些数据类型的定义，让μC/OSIII与CPU架构和编译器的字宽无关。同时还指定了CPU使用的是大端模式还是小端模式，还包括一些与CPU架构相关的函数的声明。

cpu_c.c文件与cpu_a.asm文件
'''''''''''''''''''''''''''''''''''''''''

这两个文件主要是CPU 底层相关的一些CPU 函数，cpu_c.c 文件中放的是C 函数，包含了一些CPU架构相关的代码，为什么要用C语言实现呢？
μC/OS是为了移植方便而采用C语言编写；而 cpu_a.asm 存放的是汇编代码，有一些代码只能用汇编实现，包含一些用来开关中断，前导零指令等。

cpu_core.c
''''''''''''''''''

cpu_core.c文件包含了适用于所有的CPU架构的C代码，也就是常说的通用代码。是一个很重要的文件。主要包含的函数是CPU 名字的命名，时间戳的计算等等，
跟CPU底层的移植没有太大的关系，主要保留的是 CPU 前导零的 C语言计算函数以及一些其他的函数，因为前导零指令是依靠硬件实现的，这里采用C语言方式实现，
以防止某些CPU不支持前导零指令

cpu_core.h
''''''''''''''''''

主要是对 cpu_core.c 文件里面一些函数的说明，以及一些时间戳相关等待定义。

cpu_def.h
'''''''''''''''''

包含CPU相关的一些宏定义，常量，利用#define进行定义的相关信息。

uC-LIB
^^^^^^^^^^^^

Micrium公司提供的官方库，诸如字符串操作、内存操作等接口，可用可不用。一般能用于代替标准库中的一些函数，使得在嵌入式中应用更加方便安全。

μC/OS-III
^^^^^^^^^^^^^^^^^

这是关键目录，我们下来着重分析的文件位于此目录下。

首先先看看μC/OS-III\Ports\ARM-Cortex-M3\Generic\RealView目录下的文件，
具体见图 μCOS-III_Ports下的文件_。

.. image:: media/porting_to_stm32/portin013.png
   :align: center
   :name: μCOS-III_Ports下的文件
   :alt: μC/OS-III\Ports下的文件


μC/OS是软件，我们的开发板是硬件，软硬件必须有桥梁来连接，这些与处理器架构相关的代码，可以称之为RTOS硬件接口层，
它们位于μC/OS-III\Ports文件夹下，在不同的编译器中选择不同的文件，我们使用了MDK，就使用RealView文件夹下的文件，
这些文件不需要我们去修改，也不需要我们去理会，都是官方给我们写好的，直接使用即可。

os_cpu.h
''''''''''''''''

定义数据类型、处理器相关代码、声明函数原型。

oc_cpu_a.asm
''''''''''''''''''''''''

与处理器相关的汇编代码，主要是与任务切换相关。

os_cpu_c.c
''''''''''''''''''

定义用户钩子函数，提供扩充软件功能的的接口。

打开Source文件，这个是μC/OS的源码文件，我们可以看到这些就是μC/OS核心文件，是非常重要的，
我们在移植的时候必须将这里面的文件添加到我们的工程中去，具体见图 μCOS源码_。

.. image:: media/porting_to_stm32/portin014.png
   :align: center
   :name: μCOS源码
   :alt: μC/OS源码


下面介绍一下每个文件的功能作用，具体见下表

.. image:: media/porting_to_stm32/portin29.png
   :align: center


至此，关于μC/OS-III源码的文件就简单介绍完成，下面我们需要将其复制到我们的工程中，
将uC-CPU、uC-LIB、μC/OS-III这3个文件夹复制到工程中的User文件夹下，然后进行移植，
具体见图 复制源码到工程中_。

.. image:: media/porting_to_stm32/portin015.png
   :align: center
   :name: 复制源码到工程中
   :alt: 复制源码到工程中


移植到STM32工程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在前一章节中，我们看到了μC/OS-III源码中那么多文件，一开始学我们根本看不过来那么多文件，我们需要提取源码中的最简洁的部分代码，
方便同学们学习，更何况我们学习的只是μC/OS-III的实时内核中的知识，因为这才是μC/OS-III的核心，那些demo都是基于此移植而来的，
上一章节我们只是将μC/OS-III的源码放到了本地工程目录下，还没有添加到开发环境里面的组文件夹里面，所以μC/OS-III也就没有移植到我们的工程中去，
现在开始讲μC/OS-III的源码添加到工程中。

在工程中添加文件分组
^^^^^^^^^^^^^^^^^^^^

我们需要先在工程中创建一些分组，方便我们分模块管理μC/OS-III中的文件，就按照μC/OS-III官方的命名方式创建我们的文件分组即可，
具体见图 在工程中添加文件分组_。

.. image:: media/porting_to_stm32/portin016.png
   :align: center
   :name: 在工程中添加文件分组
   :alt: 在工程中添加文件分组

添加文件到对应分组
^^^^^^^^^^^^^^^^^^^^^^

向“APP”分组添加“\User\APP”文件夹下的所有文件，具体见图 APP分组的文件_。

.. image:: media/porting_to_stm32/portin017.png
   :align: center
   :name: APP分组的文件
   :alt: APP分组的文件


向“ BSP”分组添加“ \\User\BSP”文件夹下的所有文件和“ \\User\BSP\led”文件夹下的源文件，具体见图 BSP分组的文件_。

.. image:: media/porting_to_stm32/portin018.png
   :align: center
   :name: BSP分组的文件
   :alt: BSP分组的文件


向“ μC/CPU”分组添加“\User\uC-CPU”文件夹下的所有文件和
“ \\User\uC-CPU\ARM-Cortex-M3\RealView”文件夹下的所有文件，具体见图 μC_CPU分组的文件_。

.. image:: media/porting_to_stm32/portin019.png
   :align: center
   :name: μC_CPU分组的文件
   :alt: μC/CPU分组的文件


向“μC/LIB”分组添加“\User\uC-LIB”文件夹下的所有文件和
“\User\uC-LIB\Ports\ARM-Cortex-M3\RealView”文件夹下的所有文件，具体见图 μC_LIB分组的文件_。

.. image:: media/porting_to_stm32/portin020.png
   :align: center
   :name: μC_LIB分组的文件
   :alt: μC/LIB分组的文件


向“ μC/OS-III Source”分组添加
“ \\User\μC/OS-III\Source”文件夹下的所有文件，具体见图 μCOS-III_Source分组的文件_。

.. image:: media/porting_to_stm32/portin021.png
   :align: center
   :name: μCOS-III_Source分组的文件
   :alt: μC/OS-III Source分组的文件


向“μC/OS-III Port”分组添加
“\User\μC/OS-III\Ports\ARM-Cortex-M3\Generic\RealView”文件夹下的所有文件，
具体见图 μCOS-III_Port分组的文件_。

.. image:: media/porting_to_stm32/portin022.png
   :align: center
   :name: μCOS-III_Port分组的文件
   :alt: μCOS-III_Port分组的文件


至此，我们的源码文件就添加到工程中了，当然此时仅仅是添加而已，并不是移植成功了，如果你编译一下工程就会发现一大堆错误，所以还需努力移植工程才行。

添加头文件路径到工程中
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ΜC/OS-III的源码已经添加到开发环境的组文件夹下面，编译的时候需要为这些源文件指定头文件的路径，不然编译会报错，此时我们先将头文件添加到我们的
工程中，具体见图 添加头文件路径到工程中_。

.. image:: media/porting_to_stm32/portin023.png
   :align: center
   :name: 添加头文件路径到工程中
   :alt: 添加头文件路径到工程中


至此，ΜC/OS的整体工程基本移植完毕，我们需要修改ΜC/OS配置文件，按照我们的需求来进行修改。

具体的工程文件修改
^^^^^^^^^^^^^^^^^^^^

添加完头文件路径后，我们可以编译一下整个工程，但肯定会有错误的， μC/OS-III 的移植尚未完毕，接下来需要对工程文件进行修改。
首先修改工程的启动文件“ startup_stm32f10x_hd.s”。
其中将PendSV_Handler 和 SysTick_Handler 分别改为OS_CPU_PendSVHandler
和OS_CPU_SysTickHandler，共两处，因为μC/OS官方已经给我们处理好对应的中断函数，就无需我们自己处理与系统相关的中断了，
同时我们还需要将stm32f10x_it.c文件中的PendSV_Handler和SysTick_Handler函数注释掉（当然不注释掉也没问题的），
具体见图 修改startup_stm32f10x_hd.s文件第76-77行_ 、图 修改startup_stm32f10x_hd.s文件第193-197行_
与图 注释掉PendSV_Handler和SysTick_Handler函数_。

.. image:: media/porting_to_stm32/portin024.png
   :align: center
   :name: 修改startup_stm32f10x_hd.s文件第76-77行
   :alt: 修改startup_stm32f10x_hd.s文件（第76、77行）


.. image:: media/porting_to_stm32/portin025.png
   :align: center
   :name: 修改startup_stm32f10x_hd.s文件第193-197行
   :alt: 修改startup_stm32f10x_hd.s文件第193-197行


.. image:: media/porting_to_stm32/portin026.png
   :align: center
   :name: 注释掉PendSV_Handler和SysTick_Handler函数
   :alt: 注释掉PendSV_Handler和 SysTick_Handler函数


如果使用的是M4/M7内核带有FPU（浮点运算单元）的处理器，那么还需要修改一下启动文件，如果想要使用FPU，
我们要在启动文件中添加以下代码，处理器必须处于特权模式才能读取和写入CPACR，具体见 代码清单:移植-1_ 与图 启动文件中插入代码_。

注：关于具体的FPU相关说明请参考《ARM-Cortex-M4内核参考手册》第7章相关内容。

.. code-block:: guess
    :caption: 代码清单:移植-1启用FPU（汇编）
    :name: 代码清单:移植-1
    :linenos:

    IF {FPU} != "SoftVFP"
    ; Enable Floating Point Support at reset for FPU
    LDR.W   R0, =0xE000ED88         ; Load address of CPACR register
    LDR     R1, [R0]                ; Read value at CPACR
    ORR     R1,  R1, #(0xF <<20); Set bits 20-23 to enable CP10 and CP11 coprocessors
    ; Write back the modified CPACR value
    STR     R1, [R0]                ; Wait for store to complete
    DSB

    ; Disable automatic FP register content
    ; Disable lazy context switch
    LDR.W   R0, =0xE000EF34         ; Load address to FPCCR register
    LDR     R1, [R0]
    AND     R1,  R1, #(0x3FFFFFFF)  ; Clear the LSPEN and ASPEN bits
    STR     R1, [R0]
    ISB                             ; Reset pipeline now the FPU is enabled
    ENDIF


.. image:: media/porting_to_stm32/portin027.png
   :align: center
   :name: 启动文件中插入代码
   :alt: 启动文件中插入代码

同时将对应芯片头文件中启用FPU的宏定义__FPU_PRESENT配置为1（默认是启用的），
然后在Option->Target->Floating Point Hardware中选择启用浮点运算，具体见图 启用浮点运算_。

.. image:: media/porting_to_stm32/portin028.png
   :align: center
   :name: 启用浮点运算
   :alt: 启用浮点运算


修改源码中的bsp.c与bsp.h文件
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们知道bsp就是板级相关的文件，也就是对应开发板的文件，而μC/OS-III源码的bsp肯定是与我们的板子不一样，
所以就需要进行修改，而且以后我们的板级外设都在bsp.c文件进行初始化，所以按照我们修改好的源码进行修改即可，
具体见 代码清单:移植-2_ 加粗部分。

.. code-block:: c
    :caption: 代码清单:移植-2修改后的bsp.c文件（已删掉注释）
    :emphasize-lines: 22-26
    :name: 代码清单:移植-2
    :linenos:

    #define  BSP_MODULE
    #include <bsp.h>

    CPU_INT32U  BSP_CPU_ClkFreq_MHz;

    #define  DWT_CR      *(CPU_REG32 *)0xE0001000
    #define  DWT_CYCCNT  *(CPU_REG32 *)0xE0001004
    #define  DEM_CR      *(CPU_REG32 *)0xE000EDFC
    #define  DBGMCU_CR   *(CPU_REG32 *)0xE0042004

    #define  DBGMCU_CR_TRACE_IOEN_MASK       0x10
    #define  DBGMCU_CR_TRACE_MODE_ASYNC      0x00
    #define  DBGMCU_CR_TRACE_MODE_SYNC_01    0x40
    #define  DBGMCU_CR_TRACE_MODE_SYNC_02    0x80
    #define  DBGMCU_CR_TRACE_MODE_SYNC_04    0xC0
    #define  DBGMCU_CR_TRACE_MODE_MASK       0xC0

    #define  DEM_CR_TRCENA                   (1 << 24)

    #define  DWT_CR_CYCCNTENA                (1 <<  0)

    void  BSP_Init (void)
    {
        LED_Init ();

    }

    CPU_INT32U  BSP_CPU_ClkFreq (void)
    {
        RCC_ClocksTypeDef  rcc_clocks;


        RCC_GetClocksFreq(&rcc_clocks);

    return ((CPU_INT32U)rcc_clocks.HCLK_Frequency);
    }

    #if ((APP_CFG_PROBE_OS_PLUGIN_EN == DEF_ENABLED) && \
        (OS_PROBE_HOOKS_EN          == 1))
    void  OSProbe_TmrInit (void)
    {
    }
    #endif

    #if ((APP_CFG_PROBE_OS_PLUGIN_EN == DEF_ENABLED) && \
        (OS_PROBE_HOOKS_EN          == 1))
    CPU_INT32U  OSProbe_TmrRd (void)
    {
    return ((CPU_INT32U)DWT_CYCCNT);
    }
    #endif


    #if (CPU_CFG_TS_TMR_EN == DEF_ENABLED)
    void  CPU_TS_TmrInit (void)
    {
        CPU_INT32U  cpu_clk_freq_hz;


        DEM_CR         |= (CPU_INT32U)DEM_CR_TRCENA;
        DWT_CYCCNT      = (CPU_INT32U)0u;
        DWT_CR         |= (CPU_INT32U)DWT_CR_CYCCNTENA;

        cpu_clk_freq_hz = BSP_CPU_ClkFreq();
        CPU_TS_TmrFreqSet(cpu_clk_freq_hz);
    }
    #endif

    #if (CPU_CFG_TS_TMR_EN == DEF_ENABLED)
    CPU_TS_TMR  CPU_TS_TmrRd (void)
    {
    return ((CPU_TS_TMR)DWT_CYCCNT);
    }
    #endif


bsp.h文件中需要添加我们自己的板级驱动头文件，头文件代码具体见 代码清单:移植-3_。

.. code-block:: c
    :caption: 代码清单:移植-3bsp.h文件添加我们自己的板级头文件
    :name: 代码清单:移植-3
    :linenos:

    #include"stm32f10x.h"// Modified by fire

    #include  <app_cfg.h>

    #include"bsp_led.h"// Modified by fire


按需配置最适的工程
~~~~~~~~~~~~~~~~~~~~~~~~~

虽然前面的编译是没有错误的，并且工程模板也是可用的，但是此时还不是我们最适合使用的工程模板，
最适合的工程往往是根据需要进行配置的，而μC/OS提供裁剪的功能，我们可以按需对系统进行裁剪。

os_cfg.h
^^^^^^^^^^^^^^^^

os_cfg.h文件是系统的配置文件，主要是让用户自己配置一些系统默认的功能，用户可以选择某些或者全部的功能，
比如消息队列、信号量、互斥量、事件标志位等，系统默认全部使用的，如果如果用户不需要的话，则可以直接关闭，
在对应的宏定义中设置为0即可，这样子就不会占用系统的SRAM，以节省系统资源，os_cfg.h文件的配置说明具体见 代码清单:移植-4_。

.. code-block:: c
    :caption: 代码清单:移植-4os_cfg.h
    :name: 代码清单:移植-4
    :linenos:

    #ifndef OS_CFG_H
    #define OS_CFG_H


    /* --- 其他配置 --- */
    #define OS_CFG_APP_HOOKS_EN             1u/* 是否使用钩子函数     */
    #define OS_CFG_ARG_CHK_EN               1u/* 是否使用参数检查     */
    #define OS_CFG_CALLED_FROM_ISR_CHK_EN   1u/* 是否使用中断调用检查 */
    #define OS_CFG_DBG_EN                   1u/* 是否使用debug        */
    #define OS_CFG_ISR_POST_DEFERRED_EN     1u/* 是否使用中断延迟post操作*/
    #define OS_CFG_OBJ_TYPE_CHK_EN          1u/* 是否使用对象类型检查   */
    #define OS_CFG_TS_EN                    1u/*是否使用时间戳     */

    #define OS_CFG_PEND_MULTI_EN            1u/*是否使用支持多个任务pend操作*/

    #define OS_CFG_PRIO_MAX                32u/*定义任务的最大优先级 */

    #define OS_CFG_SCHED_LOCK_TIME_MEAS_EN  1u/*是否使用支持测量调度器锁定时间 */
    #define OS_CFG_SCHED_ROUND_ROBIN_EN     1u/* 是否支持循环调度         */
    #define OS_CFG_STK_SIZE_MIN            64u/* 最小的任务栈大小        */


    /* ---------- 事件标志位---------- */
    #define OS_CFG_FLAG_EN                  1u/*是否使用事件标志位    */
    #define OS_CFG_FLAG_DEL_EN           	1u/*是否包含OSFlagDel()的代码 */
    #define OS_CFG_FLAG_MODE_CLR_EN         1u/*是否包含清除事件标志位的代码*/
    #define OS_CFG_FLAG_PEND_ABORT_EN       1u/*是否包含OSFlagPendAbort()的代码*/


    /* --------- 内存管理 --- */
    #define OS_CFG_MEM_EN                   1u/* 是否使用内存管理         */


    /* -------- 互斥量 ----- */
    #define OS_CFG_MUTEX_EN                 1u/*是否使用互斥量 */
    #define OS_CFG_MUTEX_DEL_EN             1u/*是否包含OSMutexDel()的代码*/
    #define OS_CFG_MUTEX_PEND_ABORT_EN      1u/*是否包含OSMutexPendAbort()的代码*/


    /* ------- 消息队列--------------- */
    #define OS_CFG_Q_EN                     1u/* 是否使用消息队列       */
    #define OS_CFG_Q_DEL_EN                 1u/* 是否包含OSQDel()的代码 */
    #define OS_CFG_Q_FLUSH_EN               1u/* 是否包含OSQFlush()的代码 */
    #define OS_CFG_Q_PEND_ABORT_EN          1u/* 是否包含OSQPendAbort()的代码*/


    /* -------------- 信号量 --------- */
    #define OS_CFG_SEM_EN                   1u/*是否使用信号量  */
    #define OS_CFG_SEM_DEL_EN               1u/*是否包含OSSemDel()的代码*/
    #define OS_CFG_SEM_PEND_ABORT_EN        1u/*是否包含OSSemPendAbort()的代码*/
    #define OS_CFG_SEM_SET_EN               1u/*是否包含OSSemSet()的代码  */


    /* ----------- 任务管理 -------------- */
    #define OS_CFG_STAT_TASK_EN             1u/* 是否使用任务统计功能 */
    #define OS_CFG_STAT_TASK_STK_CHK_EN     1u/* 从统计任务中检查任务栈 */

    #define OS_CFG_TASK_CHANGE_PRIO_EN      1u/* 是否包含OSTaskChangePrio()的代码*/
    #define OS_CFG_TASK_DEL_EN              1u/* 是否包含OSTaskDel()的代码*/
    #define OS_CFG_TASK_Q_EN                1u/*是否包含OSTaskQXXXX()的代码*/
    #define OS_CFG_TASK_Q_PEND_ABORT_EN     1u/* 是否包含OSTaskQPendAbort()的代码 */
    #define OS_CFG_TASK_PROFILE_EN          1u/* 是否在OS_TCB中包含变量以进行性能分析 */
    #define OS_CFG_TASK_REG_TBL_SIZE     1u/*任务特定寄存器的数量  */
    #define OS_CFG_TASK_SEM_PEND_ABORT_EN   1u/* 是否包含OSTaskSemPendAbort()的代码 */
    #define OS_CFG_TASK_SUSPEND_EN       1u/*是否包含OSTaskSuspend()和
                            OSTaskResume()的代码*/

    /* ------- 时间管理 ------- */
    #define OS_CFG_TIME_DLY_HMSM_EN      1u/*是否包含OSTimeDlyHMSM()的代码*/
    #define OS_CFG_TIME_DLY_RESUME_EN   1u/*是否包含OSTimeDlyResume()的代码*/


    /* ---------- 定时器管理 ------- */
    #define OS_CFG_TMR_EN                   1u/* 是否使用定时器	*/
    #define OS_CFG_TMR_DEL_EN               1u/* 是否支持OSTmrDel()  */

    #endif


cpu_cfg.h
^^^^^^^^^^^^^^^^^

cpu_cfg.h文件主要是配置CPU相关的一些宏定义，我们可以选择对不同的CPU进行配置，当然，如果我们没有对CPU很熟悉的话，
就直接忽略这个文件即可，在这里我们只需要注意关于时间戳与前导零指令相关的内容，我们使用的CPU是STM32，是32位的CPU，
那么时间戳我们使用32位的变量即可，而且STM32支持前导零指令，可以使用它让系统进行寻找最高优先级的任务更加快捷，
具体见 代码清单:移植-5_。

ΜC/OS支持两种方法选择下一个要执行的任务：一个采用C语言实现前导零指令，这种方法我们通常称为通用方法，
CPU_CFG_LEAD_ZEROS_ASM_PRESENT没有被定义的时候使用才使用通用方法获取下一个即将运行的任务，
通用方法可以用于所有μC/OS支持的硬件平台，因为这种方法是完全用C语言实现，所以效率略低于特殊方法，
但不强制要求限制最大可用优先级数目；另一个是硬件方式查找下一个要运行的任务，
必须定义CPU_CFG_LEAD_ZEROS_ASM_PRESENT这个宏，因为这种方法是必须依赖一个或多个特定架构的汇编指令（一般是类似计算前导零[CLZ]指令，
在M3、M4、M7内核中都有，这个指令是用来计算一个变量从最高位开始的连续零的个数），所以效率略高于通用方法，但受限于硬件平台。

.. code-block:: c
    :caption: 代码清单:移植-5cpu_cfg.h
    :name: 代码清单:移植-5
    :linenos:

    #ifndef  CPU_CFG_MODULE_PRESENT
    #define  CPU_CFG_MODULE_PRESENT

    /*   是否使用CPU名字：DEF_ENABLED或者DEF_DISABLED       */
    #define  CPU_CFG_NAME_EN                        DEF_ENABLED



    /* CPU名字大小（ASCII字符串形式）   */
    #define  CPU_CFG_NAME_SIZE                     16u


    /* CPU时间戳功能配置（只能选择其中一个）  */
    /*  是否使用32位的时间戳变量：DEF_ENABLED或者DEF_DISABLED             */
    #define  CPU_CFG_TS_32_EN                       DEF_ENABLED
    /*  是否使用64位的时间戳变量：DEF_ENABLED或者DEF_DISABLED             */
    #define  CPU_CFG_TS_64_EN                       DEF_DISABLED
    /* *配置CPU时间戳定时器字大小 */
    #define  CPU_CFG_TS_TMR_SIZE                    CPU_WORD_SIZE_32



    /* 是否使用测量CPU禁用中断的时间  */
    #if 0
    #define  CPU_CFG_INT_DIS_MEAS_EN
    #endif
    /* 配置测量的次数*/
    #define  CPU_CFG_INT_DIS_MEAS_OVRHD_NBR                    1u


    /* 是否使用CPU前导零指令（需要硬件支持，在stm32我们可以使用这个指令）  */
    #if 1
    #define  CPU_CFG_LEAD_ZEROS_ASM_PRESENT
    #endif

    #endif


os_cfg_app.h
^^^^^^^^^^^^^^^^^^^^^^^^

os_cfg_app.h是系统应用配置的头文件，简单来说就是系统默认的任务配置，如任务的优先级、栈大小等基本信息，但是有两个任务是必须开启
的，一个就是空闲任务，另一个就是时钟节拍任务，这两个是让系统正常运行的最基本任务，而其他任务我们自己按需配置即可。

.. code-block:: c
    :caption: 代码清单:移植-6os_cfg_app.h
    :name: 代码清单:移植-6
    :linenos:

    #ifndef OS_CFG_APP_H
    #define OS_CFG_APP_H


    /* --------------------- MISCELLANEOUS ------------------ */
    #define  OS_CFG_MSG_POOL_SIZE            100u/* 支持的最大消息数量 */
    #define  OS_CFG_ISR_STK_SIZE             128u/*ISR栈的大小 */
    #define  OS_CFG_TASK_STK_LIMIT_PCT_EMPTY  10u/*检查栈的剩余大小（百分百形式，
                            此处是10%）*/


    /* ---------------------- 空闲任务 --------------------- */
    #define  OS_CFG_IDLE_TASK_STK_SIZE       128u/* 空闲任务栈大小    */


    /* ------------------ 中断处理任务------------------ */
    #define  OS_CFG_INT_Q_SIZE                10u/*中断处理任务队列大小  */
    #define  OS_CFG_INT_Q_TASK_STK_SIZE      128u/* 中断处理任务的栈大小*/


    /* ------------------- 统计任务------------------- */
    #define  OS_CFG_STAT_TASK_PRIO            11u/* 统计任务的优先级  */
    #define  OS_CFG_STAT_TASK_RATE_HZ         10u/* 统计任务的指向频率（10HZ）*/
    #define  OS_CFG_STAT_TASK_STK_SIZE       128u/*统计任务的栈大小*/


    /* ------------------------ 时钟节拍任务 ----------------------- */
    #define  OS_CFG_TICK_RATE_HZ       1000u/*系统的时钟节拍(一般为10 到 1000 Hz) */
    #define  OS_CFG_TICK_TASK_PRIO            1u/*时钟节拍任务的优先级    */
    #define  OS_CFG_TICK_TASK_STK_SIZE       128u/* 时钟节拍任务的栈大小*/
    #define  OS_CFG_TICK_WHEEL_SIZE           17u/* 时钟节拍任务的列表大小 */


    /* ----------------------- 定时器任务 ----------------------- */
    #define  OS_CFG_TMR_TASK_PRIO          11u/*定时器任务的优先级  */
    #define  OS_CFG_TMR_TASK_RATE_HZ        10u/* 定时器频率（10 Hz是典型值） */
    #define  OS_CFG_TMR_TASK_STK_SIZE      128u/* 定时器任务的栈大小    */
    #define  OS_CFG_TMR_WHEEL_SIZE          17u/*定时器任务的列表大小  */

    #endif


此处要注意时钟节拍任务，μC/OS的时钟节拍任务是用于管理时钟节拍的，建议将其优先级设置更高一些，这样子在调度的时候，时钟节拍任务
能抢占其他任务执行，从而能够更新任务，相对于其他操作系统，寻找处于最高优先级的就绪任务都是在中断中，μC/OS将其放于任务中能更好
解决关中断时间过长的问题。

修改app.c
~~~~~~~~~~~~~~~~~~~

我们将原来裸机工程里面app.c的文件内容全部删除，新增如下内容，具体见 代码清单:移植-7_。

.. code-block:: c
    :caption: 代码清单:移植-7app.c文件内容
    :name: 代码清单:移植-7
    :linenos:

    /**
    *********************************************************************
    * @file    app.c
    * @author  fire
    * @version V1.0
    * @date    2018-xx-xx
    * @brief   ΜC/OS-III + STM32 工程模版
    *********************************************************************
    * @attention
    *
    * 实验平台:野火 STM32 开发板
    * 论坛    :http://www.firebbs.cn
    * 淘宝    :https://fire-stm32.taobao.com
    *
    **********************************************************************
    */

    /*
    *************************************************************************
    *                             包含的头文件
    *************************************************************************
    */
    #include <includes.h>



    /*
    *************************************************************************
    *                               变量
    *************************************************************************
    */


    /*
    *************************************************************************
    *                             函数声明
    *************************************************************************
    */



    /*
    *************************************************************************
    *                             main 函数
    *************************************************************************
    */
    /**
    * @brief  主函数
    * @param  无
    * @retval 无
    */
    int main(void)
    {
    /*暂时没有在main里面创建任务应用任务 */
    }


    /********************************END OF FILE****************************/


下载验证
~~~~~~~~~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），
一看，啥现象都没有，一脸懵逼，我说，你急个肾，目前我们还没有在main()函数里面创建任务，系统也没有跑起来，
main()函数中什么都没有，那当然是没有现象。如果要想看现象，得自己在main创建里面应用任务，并且让μC/OS跑起来，
关于如何使用μC/OS创建任务，请看下一章“创建任务”。

