.. vim: syntax=rst

临界段
========

临界段简介
~~~~~~~~~~~~~

临界段代码，也称作临界域，是一段不可分割的代码。μC/OS中包含了很多临界段代码。如果临界段可能被中断，
那么就需要关中断以保护临界段。如果临界段可能被任务级代码打断，那么需要锁调度器保护临界段。

临界段用一句话概括就是一段在执行的时候不能被中断的代码段。在μC/OS里面，这个临界段最常出现的就是对全局变量的操作，
全局变量就好像是一个枪把子，谁都可以对他开枪，但是我开枪的时候，你就不能开枪，否则就不知道是谁命中了靶子。
可能有人会说我可以在子弹上面做个标记，我说你能不能不要瞎扯淡。

那么什么情况下临界段会被打断？一个是系统调度，还有一个就是外部中断。在μC/OS的系统调度，最终也是产生PendSV中断，
在PendSV Handler里面实现任务的切换，所以还是可以归结为中断。既然这样，μC/OS对临界段的保护最终还是回到对中断的开和关的控制。

μC/OS中定义了一个进入临界段的宏和两个出临界段的宏，用户可以通过这些宏定义进入临界段和退出临界段。

-  OS_CRITICAL_ENTER()

-  OS_CRITICAL_EXIT()

-  OS_CRITICAL_EXIT_NO_SCHED()

此外还有一个开中断但是锁定调度器的宏定义OS_CRITICAL_ENTER_CPU_EXIT()。

Cortex-M内核快速关中断指令
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了快速地开关中断， Cortex-M内核专门设置了一条 CPS 指令，有 4 种用法，具体见 代码清单:临界段-1_。

.. code-block::
    :caption: 代码清单:临界段-1 CPS 指令用法
    :name: 代码清单:临界段-1
    :linenos:

    CPSID I ;PRIMASK=1     ;关中断
    CPSIE I ;PRIMASK=0     ;开中断
    CPSID F ;FAULTMASK=1   ;关异常
    CPSIE F ;FAULTMASK=0   ;开异常


代码清单:临界段-1_ 中PRIMASK和FAULTMAST是Cortex-M内核里面三个中断屏蔽寄存器中的两个，还有一个是BASEPRI，
有关这三个寄存器的详细用法见表 Cortex-M内核中断屏蔽寄存器组描述_。

表7‑1Cortex-M内核中断屏蔽寄存器组描述

.. list-table::
   :widths: 50 50
   :name: Cortex-M内核中断屏蔽寄存器组描述
   :header-rows: 0


   * - 名字
     - 功能描述

   * - PRIMASK
     - 这是个只有单一比特的寄存器。在它被置1 后，就关掉所有可屏蔽的异常，只剩下NMI 和硬FAULT可以响应。它的默认值是0，表示没有关中断。

   * - FAULTMASK
     - 这是个只有1 个位的寄存器。当它置1 时，只有NMI 才能响应，所有其他的异常，甚至是硬FAULT，也通通闭嘴。它的默认值也是0，表示没有关异常。

   * - BASEPRI
     - 这个寄存器最多有9 位（由表达优先级的位数决定）。它定义了被屏蔽优先级的阈值。当它被设成某个值后，所有优先级号大于等于此值的中断都被关（优先级号越大，优先级越低）。但若被设成0，则不关闭任何中断，0 也是默认值。                                         |


但是，在μC/OS中，对中断的开和关是通过操作PRIMASK寄存器来实现的，使用CPSID I指令就能立即关闭中断。很是方便。

关中断
~~~~~~~

μC/OS中关中断的函数在cpu_a.asm中定义，无论上层的宏定义是怎么实现的，底层操作关中断的函数还是CPU_SR_Save()，
具体实现见 代码清单:临界段-2_。

.. code-block::
    :caption: 代码清单:临界段-2关中断
    :name: 代码清单:临界段-2
    :linenos:

    CPU_SR_Save
            MRSR0, PRIMASK       	(1)
            CPSID   I			(2)
            BX      LR			(3)


-   代码清单:临界段-2_  **(1)**\：通过MRS指令将特殊寄存器PRIMASK寄存器的值存储到通用寄存器r0。当在C中调用汇编的子程序返回时，
    会将r0作为函数的返回值。所以在C中调用CPU_SR_Save()的时候，需要事先声明一个变量用来存储CPU_SR_Save()的返回值，即r0寄存器的值，
    也就是PRIMASK的值。

-   代码清单:临界段-2_  **(2)**\ ：关闭中断，即使用CPS指令将PRIMASK寄存器的值置1。在这里，我敢肯定，一定会有人有这样一个疑
    问：关中断，不就是直接使用 CPSID I 指令就行了嘛，为什么还要第一步，即在执行CPSIDI指令前，要先把PRIMASK的值保存起来？这个
    疑问接下来在“临界段代码的应用”这个小结揭晓。

-   代码清单:临界段-2_  **(3)**\ ：子程序返回。

开中断
~~~~~~~

开中断要与关中断配合使用，μC/OS中开中断的函数在cpu_a.asm中定义，无论上层的宏定义是怎么实现的，
底层操作关中断的函数还是CPU_SR_Restore()，具体实现见 代码清单:临界段-3_。

.. code-block::
    :caption: 代码清单:临界段-3开中断
    :name: 代码清单:临界段-3
    :linenos:

    CPU_SR_Restore
            MSR     PRIMASK, R0			(1)
            BX      LR				(2)


-   代码清单:临界段-2_  **(1)**\ ：通过MSR指令将通用寄存器r0的值存储到特殊寄存器PRIMASK。当在C中调用汇编的子程序返回时，
    会将第一个形参传入到通用寄存器r0。所以在C中调用CPU_SR_Restore()的时候，需要传入一个形参，
    该形参是进入临界段之前保存的PRIMASK的值。这个时候又有人会问，开中断，不就是使用CPSIE I指令就行了嘛，
    为啥跟我等凡人想的不一样？其中奥妙将在接下来“临界段代码的应用”这个小结揭晓。

-   代码清单:临界段-2_  **(2)**\ ：子程序返回。

临界段代码的应用
~~~~~~~~~~~~~~~~~~~~~~~~

在进入临界段之前，我们会先把中断关闭，退出临界段时再把中断打开。而且Cortex-M内核设置了快速关中断的CPS指令，
那么按照我们的第一思维，开关中断的函数的实现和临界段代码的保护应该是像 代码清单:临界段-4_ 那样的。

.. code-block::
    :caption: 代码清单:临界段-4开关中断的函数的实现和临界段代码的保护
    :name: 代码清单:临界段-4
    :linenos:

    ;//开关中断函数的实现
    ;/*
    ; * void CPU_SR_Save();
    ; */
    CPU_SR_Save
            CPSID   I				(1)
            BX      LR

    ;/*
    ; * void CPU_SR_Restore(void);
    ; */
    CPU_SR_Restore
            CPSIE   I 				(2)
            BX      LR

    PRIMASK = 0;             /* PRIMASK初始值为0,表示没有关中断 */(3)

    /* 临界段代码保护 */
    {
        /* 临界段开始 */
        CPU_SR_Save();     /* 关中断,PRIMASK = 1 */(4)
        {
            /* 执行临界段代码，不可中断 */(5)
        }
        /* 临界段结束 */
        CPU_SR_Restore();      /* 开中断,PRIMASK = 0 */(6)
    }


-   代码清单:临界段-4_  **(1)**\ ：关中断直接使用了CPSID I，没有跟代码清单:临界段-2一样事先将PRIMASK的值保存在r0中。

-   代码清单:临界段-4_  **(2)**\ ：开中断直接使用了CPSIE I，而不是像代码清单:临界段-3那样从传进来的形参来恢复PRIMASK的值。

-   代码清单:临界段-4_  **(4)**\ ：假设PRIMASK初始值为0，表示没有关中断。

-   代码清单:临界段-4_  **(4)**\ ：临界段开始，调用关中断函数CPU_SR_Save()，此时PRIMASK的值等于1，确实中断已经关闭。

-   代码清单:临界段-4_  **(5)**\ ：执行临界段代码，不可中断。

-   代码清单:临界段-4_  **(5)**\ ：临界段结束，
    调用开中断函数CPU_SR_Restore()，此时PRIMASK的值等于0，确实中断已经开启。

乍一看， 代码清单:临界段-4_ 的这种实现开关中断的方法确实有效，没有什么错误，但是我们忽略了一种情况，
就是当临界段是出现嵌套的时候，这种开关中断的方法就不行了，具体怎么不行具体见 代码清单:临界段-5_。

.. code-block::
    :caption: 代码清单:临界段-5开关中断的函数的实现和嵌套临界段代码的保护（有错误，只为讲解）
    :name: 代码清单:临界段-5
    :linenos:

    ;//开关中断函数的实现
    ;/*
    ; * void CPU_SR_Save();
    ; */
    CPU_SR_Save
            CPSID   I
            BX      LR

    ;/*
    ; * void CPU_SR_Restore(void);
    ; */
    CPU_SR_Restore
            CPSIE   I
            BX      LR

    PRIMASK = 0;                 /* PRIMASK初始值为0,表示没有关中断 */

    /* 临界段代码 */
    {
        /* 临界段1开始 */
        CPU_SR_Save();           /* 关中断,PRIMASK = 1 */
        {
            /* 临界段2 */
            CPU_SR_Save();       /* 关中断,PRIMASK = 1 */
            {

            }
            CPU_SR_Restore();        /* 开中断,PRIMASK = 0 */(注意)
        }
        /* 临界段1结束 */
        CPU_SR_Restore();            /* 开中断,PRIMASK = 0 */
    }


-   代码清单:临界段-5_  **(注意)**\ ：当临界段出现嵌套的时候，这里以一重嵌套为例。
    临界段1开始和结束的时候PRIMASK分别等于1和0，表示关闭中断和开启中断，这是没有问题的。临界段2开始的时候，
    PRIMASK等于1，表示关闭中断，这是没有问题的，问题出现在临界段2结束的时候，PRIMASK的值等于0，如果单纯对于临界段2来说，
    这也是没有问题的，因为临界段2已经结束，可是临界段2是嵌套在临界段1中，虽然临界段2已经结束，但是临界段1还没有结束，
    中断是不能开启的，如果此时有外部中断来临，那么临界段1就会被中断，违背了我们的初衷，那应该怎么办？
    正确的做法具体见 代码清单:临界段-6_。

.. code-block::
    :caption: 代码清单:临界段-6开关中断的函数的实现和嵌套临界段代码的保护（正确）
    :name: 代码清单:临界段-6
    :linenos:

    ;//开关中断函数的实现
    ;/*
    ; * void CPU_SR_Save();
    ; */
    CPU_SR_Save
            MRS     R0, PRIMASK
            CPSID   I
            BX      LR

    ;/*
    ; * void CPU_SR_Restore(void);
    ; */
    CPU_SR_Restore
            MSR     PRIMASK, R0
            BX      LR

    PRIMASK = 0;        /* PRIMASK初始值为0,表示没有关中断 */		(1)

    CPU_SR  cpu_sr1 = (CPU_SR)0
    CPU_SR  cpu_sr2 = (CPU_SR)0				(2)

    /* 临界段代码 */
    {
        /* 临界段1开始 */
        cpu_sr1 = CPU_SR_Save();    /* 关中断,cpu_sr1=0,PRIMASK=1 */(3)
        {
            /* 临界段2 */
            cpu_sr2 = CPU_SR_Save();/*关中断,cpu_sr2=1,PRIMASK=1 */(4)
            {

            }
            CPU_SR_Restore(cpu_sr2); /*开中断,cpu_sr2=1,PRIMASK=1 */(5)
        }
        /* 临界段1结束 */
        CPU_SR_Restore(cpu_sr1);    /* 开中断,cpu_sr1=0,PRIMASK=0 */(6)
    }


-   代码清单:临界段-6_  **(1)**\ ：假设PRIMASK初始值为0,表示没有关中断。

-   代码清单:临界段-6_  **(2)**\ ：定义两个变量，留着后面用。

-   代码清单:临界段-6_  **(3)**\ ：临界段1开始，调用关中断函数CPU_SR_Save()，
    CPU_SR_Save()函数先将PRIMASK的值存储在通用寄存器r0，
    一开始我们假设PRIMASK的值等于0，所以此时r0的值即为0。然后执行汇编指令 CPSIDI关闭中断，即设置PRIMASK等于1，
    在返回的时候r0当做函数的返回值存储在cpu_sr1，所以cpu_sr1等于r0等于0。

-   代码清单:临界段-6_  **(4)**\ ：临界段2开始，调用关中断函数CPU_SR_Save()，
    CPU_SR_Save()函数先将PRIMASK的值存储在通用寄存器r0，
    临界段1开始的时候我们关闭了中断，即设置PRIMASK等于1，
    所以此时r0的值等于1。然后执行汇编指令 CPSIDI关闭中断，即设置PRIMASK等于1，
    在返回的时候r0当做函数的返回值存储在cpu_sr2，所以cpu_sr2等于r0等于1。

-   代码清单:临界段-6_  **(5)**\ ：临界段2结束，调用开中断函数CPU_SR_Restore(cpu_sr2)，
    cpu_sr2作为函数的形参传入到通用寄存器r0，
    然后执行汇编指令 MSR r0, PRIMASK 恢复PRIMASK的值。此时PRIAMSK = r0 = cpu_sr2 =1。
    关键点来了，为什么临界段2结束了，
    PRIMASK还是等于1，按道理应该是等于0。因为此时临界段2是嵌套在临界段1中的，还是没有完全离开临界段的范畴，所以不能把中断打开，
    如果临界段是没有嵌套的，使用当前的开关中断的方法的话，那么PRIMASK确实是等于1，具体举例见 代码清单:临界段-7_。

.. code-block::
    :caption: 代码清单:临界段-7开关中断的函数的实现和一重临界段代码的保护（正确）
    :name: 代码清单:临界段-7
    :linenos:

    ;//开关中断函数的实现
    ;/*
    ; * void CPU_SR_Save();
    ; */
    CPU_SR_Save
            MRS     R0, PRIMASK
            CPSID   I
            BX      LR

    ;/*
    ; * void CPU_SR_Restore(void);
    ; */
    CPU_SR_Restore
            MSR     PRIMASK, R0
            BX      LR

    PRIMASK = 0;                   /* PRIMASK初始值为0,表示没有关中断 */

    CPU_SR  cpu_sr1 = (CPU_SR)0

    /* 临界段代码 */
    {
        /* 临界段开始 */
        cpu_sr1 = CPU_SR_Save();/* 关中断,cpu_sr1=0,PRIMASK=1 */
        {

        }
        /* 临界段结束 */
        CPU_SR_Restore(cpu_sr1);    /* 开中断,cpu_sr1=0,PRIMASK=0 */(注意点)
    }


-   代码清单:临界段-6_  **(6)**\ ：临界段1结束，PRIMASK等于0，开启中断，与进入临界段1遥相呼应。

测量关中断时间
~~~~~~~~~~~~~~~~~~~

μC/OS提供了测量关中断时间的功能，通过设置cpu_cfg.h中的宏定义CPU_CFG_INT_DIS_MEAS_EN为1就表示启用该功能。

系统会在每次关中断前开始测量，开中断后结束测量，测量功能保存了 2个方面的测量值，总的关中断时间与最近一次关中断的时间。
因此，用户可以根据得到的关中断时间对其加以优化。时间戳的速率决定于CPU的速率。例如，如果CPU速率为72MHz，
时间戳的速率就为72MHz，那么时间戳的分辨率为1/72M微秒，大约为13.8纳秒（ns）。显然，
系统测出的关中断时间还包括了测量时消耗的额外时间，那么测量得到的时间减掉测量时所耗时间就是实际上的关中断时间。
关中断时间跟处理器的指令、速度、内存访问速度有很大的关系。

测量关中断时间初始化
^^^^^^^^^^^^^^^^^^^^

关中断之前要用函数 CPU_IntDisMeasInit()函数进行初始化，
可以直接调用函数 CPU_Init()函数进行初始化，具体见 代码清单:临界段-8_。

.. code-block::
    :caption: 代码清单:临界段-8CPU_IntDisMeasInit()源码
    :name: 代码清单:临界段-8
    :linenos:

    #ifdef  CPU_CFG_INT_DIS_MEAS_EN
    static  void  CPU_IntDisMeasInit (void)
    {
        CPU_TS_TMR  time_meas_tot_cnts;
        CPU_INT16U  i;
        CPU_SR_ALLOC();

        CPU_IntDisMeasCtr         = 0u;
        CPU_IntDisNestCtr         = 0u;
        CPU_IntDisMeasStart_cnts  = 0u;
        CPU_IntDisMeasStop_cnts   = 0u;
        CPU_IntDisMeasMaxCur_cnts = 0u;
        CPU_IntDisMeasMax_cnts    = 0u;
        CPU_IntDisMeasOvrhd_cnts  = 0u;

        time_meas_tot_cnts = 0u;
        CPU_INT_DIS();                        /* 关中断 */
        for (i = 0u; i < CPU_CFG_INT_DIS_MEAS_OVRHD_NBR; i++)
        {
            CPU_IntDisMeasMaxCur_cnts = 0u;
            CPU_IntDisMeasStart();        /* 执行多个连续的开始/停止时间测量  */
            CPU_IntDisMeasStop();
            time_meas_tot_cnts += CPU_IntDisMeasMaxCur_cnts; /* 计算总的时间 */
        }

        CPU_IntDisMeasOvrhd_cnts  = (time_meas_tot_cnts + (CPU_CFG_INT_DIS_MEAS_OVRHD_NBR / 2u))/CPU_CFG_INT_DIS_MEAS_OVRHD_NBR;
        /*得到平均值，就是每一次测量额外消耗的时间  */
        CPU_IntDisMeasMaxCur_cnts =  0u;
        CPU_IntDisMeasMax_cnts    =  0u;
        CPU_INT_EN();
    }
    #endif


因为关中断测量本身也会耗费一定的时间，这些时间实际是加入到我们测量到的最大关中断时间里面，如果能够计算出这段时间，
后面计算的时候将其减去可以得到更加准确的结果。这段代码的核心思想很简单，就是重复多次开始测量与停止测量，
然后多次之后，取得平均值，那么这个值就可以看作一次开始测量与停止测量的时间，保存在CPU_IntDisMeasOvrhd_cnts变量中。

测量最大关中断时间
^^^^^^^^^^^^^^^^^^^^

如果用户启用了CPU_CFG_INT_DIS_MEAS_EN这个宏定义，那么系统在关中断的时候会调用了开始测量关中断最大时间的函数
CPU_IntDisMeasStart()，开中断的时候调用停止测量关中断最大时间的函数CPU_IntDisMeasStop()。从代码中我们能看到，
只要在关中断且嵌套层数 OSSchedLockNestingCtr为0的时候保存下时间戳，如果嵌套层数不为0，肯定不是刚刚进入中断，
退出中断且嵌套层数为 0 的时候，这个时候才算是真正的退出中断，把测得的时间戳减去一次测量额外消耗的时间，
便得到这次关中断的时间，再将这个时间跟历史保存下的最大的关中断的时间对比，刷新最大的关中断时间，
源码具体见 代码清单:临界段-9_。

.. code-block:: c
    :caption: 代码清单:临界段-9开始/停止测量关中断时间
    :name: 代码清单:临界段-9
    :linenos:

    /* 开始测量关中断时间  */
    #ifdef  CPU_CFG_INT_DIS_MEAS_EN
    void  CPU_IntDisMeasStart (void)
    {
        CPU_IntDisMeasCtr++;
        if (CPU_IntDisNestCtr == 0u)                   /* 嵌套层数为0   */
        {
            CPU_IntDisMeasStart_cnts = CPU_TS_TmrRd();  /* 保存时间戳  */
        }
        CPU_IntDisNestCtr++;
    }
    #endif

    /* 停止测量关中断时间  */
    #ifdef  CPU_CFG_INT_DIS_MEAS_EN
    void  CPU_IntDisMeasStop (void)
    {
        CPU_TS_TMR  time_ints_disd_cnts;
        CPU_IntDisNestCtr--;
        if (CPU_IntDisNestCtr == 0u)                /* 嵌套层数为0*/
        {
            CPU_IntDisMeasStop_cnts = CPU_TS_TmrRd();    /* 保存时间戳  */

            time_ints_disd_cnts = CPU_IntDisMeasStop_cnts -
            CPU_IntDisMeasStart_cnts;/* 得到关中断时间  */
            /* 更新最大关中断时间  */
            if (CPU_IntDisMeasMaxCur_cnts < time_ints_disd_cnts)
            {
                CPU_IntDisMeasMaxCur_cnts = time_ints_disd_cnts;
            }
            if (CPU_IntDisMeasMax_cnts    < time_ints_disd_cnts)
            {
                CPU_IntDisMeasMax_cnts    = time_ints_disd_cnts;
            }
        }
    }
    #endif


获取最大关中断时间
^^^^^^^^^^^^^^^^^^^^^

现在得到了关中断时间，那么μC/OS也提供了三个与获取关中断时间有关的函数，分别是：

-  CPU_IntDisMeasMaxCurReset()

-  CPU_IntDisMeasMaxCurGet()

-  CPU_IntDisMeasMaxGet()

如果想直接获取整个程序运行过程中最大的关中断时间的话，直接调用函数 CPU_IntDisMeasMaxGet()获取即可。

如果想要测量某段程序执行的最大关中断时间，那么在这段程序的前面调用CPU_IntDisMeasMaxCurReset()函数将
CPU_IntDisMeasMaxCur_cnts 变量清 0，在这段程序结束的时候调用函数CPU_IntDisMeasMaxCurGet()即可。

这些函数的源码很简单，具体见 代码清单:临界段-10_。

.. code-block:: c
    :caption: 代码清单:临界段-10获取最大关中断时间相关源码
    :name: 代码清单:临界段-10
    :linenos:

    #ifdef  CPU_CFG_INT_DIS_MEAS_EN//如果启用了关中断时间测量
    CPU_TS_TMR  CPU_IntDisMeasMaxCurGet (void) //获取测量的程序段的最大关中断时间
    {
        CPU_TS_TMR  time_tot_cnts;
        CPU_TS_TMR  time_max_cnts;
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
        //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
        // SR（临界段关中断只需保存SR），开中断时将该值还原。
        CPU_INT_DIS();                                       //关中断
        time_tot_cnts = CPU_IntDisMeasMaxCur_cnts;
        //获取未处理的程序段最大关中断时间
        CPU_INT_EN();                                        //开中断
        time_max_cnts = CPU_IntDisMeasMaxCalc(time_tot_cnts);
        //获取减去测量时间后的最大关中断时间

        return (time_max_cnts);                    //返回程序段的最大关中断时间
    }
    #endif

    #ifdef  CPU_CFG_INT_DIS_MEAS_EN//如果启用了关中断时间测量
    CPU_TS_TMR  CPU_IntDisMeasMaxGet (void)
    //获取整个程序目前最大的关中断时间
    {
        CPU_TS_TMR  time_tot_cnts;
        CPU_TS_TMR  time_max_cnts;
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
        //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
        // SR（临界段关中断只需保存SR），开中断时将该值还原。
        CPU_INT_DIS();                                        //关中断
        time_tot_cnts = CPU_IntDisMeasMax_cnts;
        //获取尚未处理的最大关中断时间
        CPU_INT_EN();                                         //开中断
        time_max_cnts = CPU_IntDisMeasMaxCalc(time_tot_cnts);
        //获取减去测量时间后的最大关中断时间

        return (time_max_cnts);                      //返回目前最大关中断时间
    }
    #endif

    #ifdef  CPU_CFG_INT_DIS_MEAS_EN//如果启用了关中断时间测量
    CPU_TS_TMR  CPU_IntDisMeasMaxCurReset (void)
    //初始化（复位）测量程序段的最大关中断时间
    {
        CPU_TS_TMR  time_max_cnts;
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
        //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
        // SR（临界段关中断只需保存SR），开中断时将该值还原。
        time_max_cnts=CPU_IntDisMeasMaxCurGet();//获取复位前的程序段最大关中断时间
        CPU_INT_DIS();                             //关中断
        CPU_IntDisMeasMaxCur_cnts = 0u;            //清零程序段的最大关中断时间
        CPU_INT_EN();                              //开中断

        return (time_max_cnts);                //返回复位前的程序段最大关中断时间
    }
    #endif


main()函数
~~~~~~~~~~~~~~~

本章main()函数没有添加新的测试代码，只需理解章节内容即可。

实验现象
~~~~~~~~~~~~

本章没有实验，只需理解章节内容即可。
