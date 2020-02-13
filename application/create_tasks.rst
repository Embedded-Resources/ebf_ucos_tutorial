.. vim: syntax=rst

创建任务
=============

在上一章，我们已经基于野火STM32开发板创建好了μC/OS-III的工程模板，这章开始我们将真正进入如何使用μC/OS的征程，
先从最简单的创建任务开始，点亮一个LED，以慰藉下尔等初学者弱小的心灵。

硬件初始化
~~~~~~~~~~~~~

本章创建的任务需要用到开发板上的LED，所以先要将LED相关的函数初始化好，为了方便以后统一管理板级外设的初始化，
我们在bsp.c文件中创建一个BSP_Init()函数，专门用于存放板级外设初始化函数，具体见 代码清单:创建任务-1_ 的加粗部分。

.. code-block:: c
    :caption: 代码清单:创建任务-1 BSP_Init()中添加硬件初始化函数
    :emphasize-lines: 7-21
    :name: 代码清单:创建任务-1
    :linenos:

    /***********************************************************************
    * @ 函数名： BSP_Init
    * @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面
    * @ 参数：
    * @ 返回值：无
    *********************************************************************/
    static void BSP_Init(void)
    {
    /*
        * STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15
        * 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，
        * 都统一用这个优先级分组，千万不要再分组，切忌。
        */
        NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

        /* LED 初始化 */
        LED_GPIO_Config();

        /* 串口初始化	*/
        USART_Config();
    }


执行到BSP_Init()函数的时候，操作系统完全都还没有涉及，即BSP_Init()函数所做的工作跟我们以前编写的裸机工程里面的硬件初始化工作是一模一样的。
运行完BSP_Init ()函数，接下来才慢慢启动操作系统，最后运行创建好的任务。有时候任务创建好，整个系统跑起来了，
可想要的实验现象就是出不来，比如LED不会亮，串口没有输出，LCD没有显示等等。如果是初学者，这个时候就会心急如焚，
四处求救，那怎么办？这个时候如何排除是硬件的问题还是系统的问题，这里面有个小小的技巧，即在硬件初始化好之后，
顺便测试下硬件，测试方法跟裸机编程一样，具体实现见 代码清单:创建任务-2_ 的加粗部分。

.. code-block:: c
    :caption: 代码清单:创建任务-2BSP_Init()中添加硬件测试函数
    :name: 代码清单:创建任务-2
    :linenos:

    /* 开发板硬件bsp头文件 */
    #include"bsp_led.h"
    #include"bsp_usart.h"

    /***********************************************************************
    * @ 函数名： BSP_Init
    * @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面
    * @ 参数：
    * @ 返回值：无
    *********************************************************************/
    static void BSP_Init(void)
    {
    /*
        * STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15
        * 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，
        * 都统一用这个优先级分组，千万不要再分组，切忌。
        */
        NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

    /* LED 初始化 */
        LED_GPIO_Config();			(1)

    /* 测试硬件是否正常工作 */		(2)
        LED1_ON;

    /* 其他硬件初始化和测试 */

    /* 让程序停在这里，不再继续往下执行 */
    while (1);			(3)

    /* 串口初始化	*/
        USART_Config();

    }


-   代码清单:创建任务-2_  **(1)**\ ：初始化硬件后，顺便测试硬件，看下硬件是否正常工作。

-   代码清单:创建任务-2_  **(2)**\ ：可以继续添加其他的硬件初始化和测试。
    硬件确认没有问题之后，硬件测试代码可删可不删，因为BSP_Init()函数只执行一遍。

-   代码清单:创建任务-2_  **(3)**\ ：方便测试硬件好坏，让程序停在这里，不再继续往下执行，当测试完毕后，这个while(1);必须删除。

注意：以上仅仅是测试代码，以实际工程代码为准。

创建单任务
~~~~~~~~~~~~~

这里，我们创建一个单任务，任务使用的栈和任务控制块都使用静态内存，即预先定义好的全局变量，这些预先定义好的全局变量都存在内部的SRAM中。

定义任务栈
^^^^^^^^^^^^^

目前我们只创建了一个任务，当任务进入延时的时候，因为没有另外就绪的用户任务，那么系统就会进入空闲任务，空闲任务是μC/OS系统自己创建并且启动的一个任务，
优先级最低。当整个系统都没有就绪任务的时候，系统必须保证有一个任务在运行，空闲任务就是为这个设计的。当用户任务延时到期，又会从空闲任务切换回用户任务。

在μC/OS系统中，每一个任务都是独立的，他们的运行环境都单独的保存在他们的栈空间当中。那么在定义好任务函数之后，我们还要为任务定义一个栈，
目前我们使用的是静态内存，所以任务栈是一个独立的全局变量，具体见 代码清单:创建任务-3_。任务的栈占用的是MCU内部的RAM，当任务越多的时候，
需要使用的栈空间就越大，即需要使用的RAM空间就越多。一个MCU能够支持多少任务，就得看你的RAM空间有多少。

.. code-block:: c
    :caption: 代码清单:创建任务-3定义任务栈
    :name: 代码清单:创建任务-3
    :linenos:

    #define  APP_TASK_START_STK_SIZE                    128

    static  CPU_STK  AppTaskStartStk[APP_TASK_START_STK_SIZE];


定义任务控制块
^^^^^^^^^^^^^^^^^^^

定义好任务函数和任务栈之后，我们还需要为任务定义一个任务控制块，通常我们称这个任务控制块为任务的身份证。在C代码上，任务控制块就是一个结构体，
里面有非常多的成员，这些成员共同描述了任务的全部信息，具体见 代码清单:创建任务-4_。

.. code-block:: c
    :caption: 代码清单:创建任务-4定义任务控制块
    :name: 代码清单:创建任务-4
    :linenos:

    static OS_TCB AppTaskStartTCB;


定义任务主体函数
^^^^^^^^^^^^^^^^^^^

任务实际上就是一个无限循环且不带返回值的C函数。目前，我们创建一个这样的任务，
让开发板上面的LED灯以500ms的频率闪烁，具体实现见 代码清单:创建任务-5_。

.. code-block:: c
    :caption: 代码清单:创建任务-5定义任务函数（此处为伪代码，以工程代码为准）
    :name: 代码清单:创建任务-5
    :linenos:

    static voidLED_Task (void* parameter)
    {
    while (1)					(1)
        {
            LED1_ON;
    OSTimeDly (500,OS_OPT_TIME_DLY,&err);/* 延时500个tick */(2)

            LED1_OFF;
    OSTimeDly (500,OS_OPT_TIME_DLY,&err);/* 延时500个tick */

        }
    }


-   代码清单:创建任务-5_  **(1)**\ ：任务必须是一个死循环，否则任务将通过LR返回，
    如果LR指向了非法的内存就会产生HardFault_Handler，
    而μC/OS指向一个任务退出函数OS_TaskReturn()，它如果支持任务删除的话，则进行任务删除操作，否则就进入死循环中，
    这样子的任务是不安全的，所以避免这种情况，任务一般都是死循环并且无返回值的，只执行一次的任务在执行完毕要记得及时删除。

-   代码清单:创建任务-5_  **(2)**\ ：任务里面的延时函数必须使用μC/OS里面提供的阻塞延时函数，并不能使用我们裸机编程中的那种延时。
    这两种的延时的区别是μC/OS里面的延时是阻塞延时，即调用OSTimeDly()函数的时候，当前任务会被挂起，调度器会切换到其他就绪的任务，
    从而实现多任务。如果还是使用裸机编程中的那种延时，那么整个任务就成为了一个死循环，如果恰好该任务的优先级是最高的，
    那么系统永远都是在这个任务中运行，比它优先级更低的任务无法运行，根本无法实现多任务，因此任务中必须有能阻塞任务的函数，
    才能切换到其他任务中。


创建任务
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一个任务的三要素是任务主体函数，任务栈，任务控制块，那么怎么样把这三个要素联合在一起？μC/OS里面有一个叫任务创建函数OSTaskCreate()，
它就是干这个活的。它将任务主体函数，任务栈和任务控制块这三者联系在一起，让任务在创建之后可以随时被系统启动与调度，
具体见 代码清单:创建任务-6_。

.. code-block::
    :caption: 代码清单:创建任务-6创建任务
    :name: 代码清单:创建任务-6
    :linenos:

    OSTaskCreate((OS_TCB     *)&AppTaskStartTCB, 			(1)
                (CPU_CHAR   *)"App Task Start",			(2)
                (OS_TASK_PTR ) AppTaskStart,			(3)
                (void       *) 0,					(4)
                (OS_PRIO     ) APP_TASK_START_PRIO,		(5)
                (CPU_STK    *)&AppTaskStartStk[0],			(6)
                (CPU_STK_SIZE) APP_TASK_START_STK_SIZE / 10,	(7)
                (CPU_STK_SIZE) APP_TASK_START_STK_SIZE,		(8)
                (OS_MSG_QTY  ) 5u,				(9)
                (OS_TICK     ) 0u,				(10)
                (void       *) 0,					(11)
    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR), (12)
                (OS_ERR     *)&err);					(13)


-   代码清单:创建任务-6_  **(1)**\ ：任务控制块，由用户自己定义。

-   代码清单:创建任务-6_  **(2)**\ ：任务名称，字符串形式，这里任务名称最好要与任务函数入口名字一致，方便进行调试。

-   代码清单:创建任务-6_  **(3)**\ ：任务入口函数，即任务函数的名称，需要我们自己定义并且实现。

-   代码清单:创建任务-6_  **(4)**\ ：任务入口函数形参，不用的时候配置为0或者NULL即可，p_arg是指向可选数据区域的指针，
    用于将参数传递给任务，因为任务一旦执行，那必须是在一个死循环中，所以传参只在首次执行时有效。

-   代码清单:创建任务-6_  **(5)**\ ：任务的优先级，由用户自己定义。

-   代码清单:创建任务-6_  **(6)**\ ：指向栈基址的指针（即栈的起始地址）。

-   代码清单:创建任务-6_  **(7)**\ ：设置栈深度的限制位置。这个值表示任务的栈满溢之前剩余的栈容量。
    例如，指定stk_size值的10％表示将达到栈限制，当栈达到90％满就表示任务的栈已满。

-   代码清单:创建任务-6_  **(8)**\ ：任务栈大小，单位由用户决定，如果CPU_STK 被设置为CPU_INT08U，
    则单位为字节，而如果CPU_STK 被设置为CPU_INT16U，则单位为半字，同理，如果CPU_STK被设置为CPU_INT32U，
    单位为字。在32位的处理器下（STM32），一个字等于4个字节，那么任务大小就为APP_TASK_START_STK_SIZE \* 4字节。

-   代码清单:创建任务-6_  **(9)**\ ：设置可以发送到任务的最大消息数，按需设置即可。

-   代码清单:创建任务-6_  **(10)**\ ：在任务之间循环时的时间片的时间量（以滴答为单位）。指定0则使用默认值。

-   代码清单:创建任务-6_  **(11)**\ ：是指向用户提供的内存位置的指针，用作TCB扩展。例如，
    该用户存储器可以保存浮点寄存器的内容在上下文切换期间，每个任务执行的时间，次数、任务已经切换等。

-   代码清单:创建任务-6_  **(12)**\ ：用户可选的任务特定选项，具体见 代码清单:创建任务-7_。

.. code-block:: c
    :caption: 代码清单:创建任务-7任务特定选项
    :name: 代码清单:创建任务-7
    :linenos:

    #define  OS_OPT_TASK_NONE         (OS_OPT)(0x0000u)	(1)
    #define  OS_OPT_TASK_STK_CHK      (OS_OPT)(0x0001u)	(2)
    #define  OS_OPT_TASK_STK_CLR      (OS_OPT)(0x0002u)	(3)
    #define  OS_OPT_TASK_SAVE_FP      (OS_OPT)(0x0004u)	(4)
    #define  OS_OPT_TASK_NO_TLS       (OS_OPT)(0x0008u)	(5)


-   代码清单:创建任务-7_  **(1)**\ ：未选择任何选项。

-   代码清单:创建任务-7_  **(2)**\ ：启用任务的栈检查。

-   代码清单:创建任务-7_  **(3)**\ ：任务创建时清除栈。

-   代码清单:创建任务-7_  **(4)**\ ：保存任何浮点寄存器的内容，这需要CPU硬件的支持，
    CPU需要有浮点运算硬件与专门保存浮点类型数据的寄存器。

-   代码清单:创建任务-7_  **(5)**\ ：指定任务不需要TLS支持。

-   代码清单:创建任务-6_  **(13)**\ ：用于保存返回的错误代码。

启动任务
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当任务创建好后，是处于任务就绪，在就绪态的任务可以参与操作系统的调度。任务调度器只启动一次，
之后就不会再次执行了，μC/OS中启动任务调度器的函数是OSStart()，并且启动任务调度器的时候就不会返回，
从此任务都由μC/OS管理，此时才是真正进入实时操作系统中的第一步，具体见 代码清单:创建任务-8_。

.. code-block:: c
    :caption: 代码清单:创建任务-8启动任务
    :name: 代码清单:创建任务-8
    :linenos:

    /* 启动任务，开启调度 */
    OSStart(&err);


app.c全貌
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

现在我们把任务主体，任务栈，任务控制块这三部分代码统一放到app.c中，我们在app.c文件中创建一个AppTaskStart任务，
这个任务是仅是用于测试用户任务，以后为了方便管理，我们的所有的任务创建都统一放在这个任务中，
在这个任务中创建成功的任务就可以直接参与任务调度了，具体内容见 代码清单:创建任务-9_。

.. code-block:: c
    :caption: 代码清单:创建任务-9app.c全貌
    :name: 代码清单:创建任务-9
    :linenos:

    #include <includes.h>


    /*
    *******************************************************
    *                           LOCAL DEFINES
    ************************************************************************
    */

    /*
    *************************************************************************
    *                             TCB
    ************************************************************************
    */

    static  OS_TCB   AppTaskStartTCB;


    /*
    ************************************************************
    *                              STACKS
    *************************************************************************
    */

    static  CPU_STK  AppTaskStartStk[APP_TASK_START_STK_SIZE];


    /*
    **************************************************************
    *                            FUNCTION PROTOTYPES
    ***********************************************************************
    */

    static  void  AppTaskStart  (void *p_arg);


    /*
    *************************************************************************
    *                                                main()
    *
    * Description : This is the standard entry point for C code.
    *              It is assumed that your code will callmain() once
    *		you have performed all necessary initialization.
    * Arguments   : none
    *
    * Returns     : none
    ************************************************************************
    */

    int  main (void)
    {
        OS_ERR  err;


        OSInit(&err);                     /* Init μC/OS-III.
            */

        OSTaskCreate((OS_TCB     *)&AppTaskStartTCB, /* Create the start task
                    */
                    (CPU_CHAR   *)"App Task Start",
                    (OS_TASK_PTR ) AppTaskStart,
                    (void       *) 0,
                    (OS_PRIO     ) APP_TASK_START_PRIO,
                    (CPU_STK    *)&AppTaskStartStk[0],
                    (CPU_STK_SIZE) APP_TASK_START_STK_SIZE / 10,
                    (CPU_STK_SIZE) APP_TASK_START_STK_SIZE,
                    (OS_MSG_QTY  ) 5u,
                    (OS_TICK     ) 0u,
                    (void       *) 0,
    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
                    (OS_ERR     *)&err);

    OSStart(&err);/* Start multitasking(i.e.give control to μC/OS-III).*/



    }


    /*
    *********************************************************************
    *                                          STARTUP TASK
    * Description : This is an example of a startup task.  As mentioned in
    *		 the book's text, you MUSTinitialize the ticker only once
    *              multitasking has started.
    *
    * Arguments   : p_arg   is the argument passed to 'AppTaskStart()' by
    *		'OSTaskCreate()'.
    * Returns     : none
    * Notes: 1) The first line of code is used to prevent a compiler warning
    * 		because 'p_arg' is not
    *used.  The compiler should not generate any code for this statement.
    *****************************************************************
    */

    static  void  AppTaskStart (void *p_arg)
    {
        CPU_INT32U  cpu_clk_freq;
        CPU_INT32U  cnts;
        OS_ERR      err;


        (void)p_arg;

        BSP_Init();  /* Initialize BSP functions
                */
        CPU_Init();
        /*Determine SysTick reference freq*/
    cpu_clk_freq = BSP_CPU_ClkFreq();
    /* Determine nbr SysTick increments */
        cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;

        OS_CPU_SysTickInit(cnts); /* Init μC/OS periodic time src (SysTick).
        */

        Mem_Init();   /* Initialize Memory Management Module
        */

    #if OS_CFG_STAT_TASK_EN > 0u
        OSStatTaskCPUUsageInit(&err); /* Compute CPU capacity with no task
            running  */
    #endif

        CPU_IntDisMeasMaxCurReset();


    while (DEF_TRUE) {  	/* Task body, always written as an
                infinite loop.*/
            macLED1_TOGGLE ();
            OSTimeDly ( 5000, OS_OPT_TIME_DLY, & err );
        }


下载验证
~~~~~~~~~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），
可以看到板子上面的LED灯已经在闪烁，说明我们创建的单任务已经跑起来了。

创建多任务
~~~~~~~~~~~~~

创建多任务只需要按照创建单任务的套路依葫芦画瓢即可，接下来我们创建四个任务，分别是起始任务、 LED1 任务、 LED2 任务和 LED3任务。
任务1让一个LED灯闪烁，任务2让另外一个LED闪烁，两个LED闪烁的频率不一样，三个任务的优先级不一样。主函数运行时创建起始任务，
起始任务运行时进行创建三个LED 灯的任务和删除自身，之后就运行三个 LED 灯的任务。三个 LED 灯的任务优先级不一样，
LED1任务为 LED1 每隔 1秒切换一次亮灭状态， LED2 任务为 LED2 每隔 5 秒切换一次亮灭状态，
LED3 任务为 LED3 每隔 10 秒切换一次亮灭状态，首先在“ app_cfg.h”里，增加定义三个 LED 灯任务的优先级和栈空间大小，
然后修改app.c的源码，具体见 代码清单:创建任务-10_ 加粗部分。

.. code-block:: c
    :caption: 代码清单:创建任务-10app.c全貌
    :emphasize-lines: 27-31,39-43,65-78,129-173,181-193,201-213,221-233
    :name: 代码清单:创建任务-10
    :linenos:

    #include <includes.h>


    /*
    ****************************************************************
    *                       LOCAL DEFINES
    *****************************************************************
    */

    /*
    **************************************************************
    *                               TCB
    **************************************************************
    */

    static  OS_TCB   AppTaskStartTCB;

    static  OS_TCB   AppTaskLed1TCB;
    static  OS_TCB   AppTaskLed2TCB;
    static  OS_TCB   AppTaskLed3TCB;


    /*
    *************************************************************************
    *                      	 STACKS
    *************************************************************************
    */

    static  CPU_STK  AppTaskStartStk[APP_TASK_START_STK_SIZE];

    static  CPU_STK  AppTaskLed1Stk [ APP_TASK_LED1_STK_SIZE ];
    static  CPU_STK  AppTaskLed2Stk [ APP_TASK_LED2_STK_SIZE ];
    static  CPU_STK  AppTaskLed3Stk [ APP_TASK_LED3_STK_SIZE ];


    /*
    ************************************************************************
    *                          FUNCTION PROTOTYPES
    *************************************************************************
    */

    static  void  AppTaskStart  (void *p_arg);

    static  void  AppTaskLed1  ( void * p_arg );
    static  void  AppTaskLed2  ( void * p_arg );
    static  void  AppTaskLed3  ( void * p_arg );


    /*
    ************************************************************************
    *                                                main()
    *
    * Description : This is the standard entry point for C code.  It is
    *               assumed that your code will call main() once you have
    * 		performed all necessary initialization.
    * Arguments   : none
    *
    * Returns     : none
    *************************************************************************
    */

    int  main (void)
    {
    OS_ERR  err;


    OSInit(&err);             /* Init μC/OS-III.
            */

    OSTaskCreate((OS_TCB     *)&AppTaskStartTCB,	/*Create the
                    start task  */
                    (CPU_CHAR   *)"App Task Start",
                    (OS_TASK_PTR ) AppTaskStart,
                    (void       *) 0,
                    (OS_PRIO     ) APP_TASK_START_PRIO,
                    (CPU_STK    *)&AppTaskStartStk[0],
                    (CPU_STK_SIZE) APP_TASK_START_STK_SIZE / 10,
                    (CPU_STK_SIZE) APP_TASK_START_STK_SIZE,
                    (OS_MSG_QTY  ) 5u,
                    (OS_TICK     ) 0u,
                    (void       *) 0,
                    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
                    (OS_ERR     *)&err);

    OSStart(&err);                /* Start multitasking (i.e. give
                    control to μC/OS-III). */


    }


    /*
    ***********************************************************************
    *                                          STARTUP TASK
    *
    * Description : This is an example of a startup task.  As mentioned in
    *               the book's text, you MUST initialize the ticker only once
    * 		multitasking has started.
    * Arguments   : p_arg   is the argument passed to 'AppTaskStart()' by
    *		'OSTaskCreate()'.
    * Returns     : none
    *
    * Notes       : 1) The first line of code is used to prevent a compiler
    * 		warning because 'p_arg' is not used.  The compiler should
    *		not generate any code for this statement.
    *********************************************************************
    */
    static  void  AppTaskStart (void *p_arg)
    {
        CPU_INT32U  cpu_clk_freq;
        CPU_INT32U  cnts;
        OS_ERR      err;


        (void)p_arg;

        BSP_Init();                  /* Initialize BSP functions
            */
        CPU_Init();

        cpu_clk_freq = BSP_CPU_ClkFreq();     /* Determine SysTick reference
                freq. */
        cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz; /* Determine
                    nbrSysTick increme nts */
        OS_CPU_SysTickInit(cnts); /*Init μC/OS periodic time src(SysTick).*/


        Mem_Init();     /* Initialize Memory Management Module
                */

    #if OS_CFG_STAT_TASK_EN > 0u
        OSStatTaskCPUUsageInit(&err);  /* Compute CPU capacity with no task
        running */
    #endif

        CPU_IntDisMeasMaxCurReset();


        OSTaskCreate((OS_TCB     *)&AppTaskLed1TCB,/*Create the Led1 task */
                    (CPU_CHAR   *)"App Task Led1",
                    (OS_TASK_PTR ) AppTaskLed1,
                    (void       *) 0,
                    (OS_PRIO     ) APP_TASK_LED1_PRIO,
                    (CPU_STK    *)&AppTaskLed1Stk[0],
                    (CPU_STK_SIZE) APP_TASK_LED1_STK_SIZE / 10,
                    (CPU_STK_SIZE) APP_TASK_LED1_STK_SIZE,
                    (OS_MSG_QTY  ) 5u,
                    (OS_TICK     ) 0u,
                    (void       *) 0,
    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
                    (OS_ERR     *)&err);

        OSTaskCreate((OS_TCB     *)&AppTaskLed2TCB, /*Create the Led2 task*/
                    (CPU_CHAR   *)"App Task Led2",
                    (OS_TASK_PTR ) AppTaskLed2,
                    (void       *) 0,
                    (OS_PRIO     ) APP_TASK_LED2_PRIO,
                    (CPU_STK    *)&AppTaskLed2Stk[0],
                    (CPU_STK_SIZE) APP_TASK_LED2_STK_SIZE / 10,
                    (CPU_STK_SIZE) APP_TASK_LED2_STK_SIZE,
                    (OS_MSG_QTY  ) 5u,
                    (OS_TICK     ) 0u,
                    (void       *) 0,
    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
                    (OS_ERR     *)&err);

        OSTaskCreate((OS_TCB     *)&AppTaskLed3TCB, /*Create the Led3 task*/
                    (CPU_CHAR   *)"App Task Led3",
                    (OS_TASK_PTR ) AppTaskLed3,
                    (void       *) 0,
                    (OS_PRIO     ) APP_TASK_LED3_PRIO,
                    (CPU_STK    *)&AppTaskLed3Stk[0],
                    (CPU_STK_SIZE) APP_TASK_LED3_STK_SIZE / 10,
                    (CPU_STK_SIZE) APP_TASK_LED3_STK_SIZE,
                    (OS_MSG_QTY  ) 5u,
                    (OS_TICK     ) 0u,
                    (void       *) 0,
    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
                    (OS_ERR     *)&err);


        OSTaskDel ( & AppTaskStartTCB, & err );


    }


    /*
    ********************************************************************
    *                         LED1 TASK
    ********************************************************************
    */

    static  void  AppTaskLed1 ( void * p_arg )
    {
        OS_ERR      err;


        (void)p_arg;


    while (DEF_TRUE) {    /* Task body, always written as an infinite
            loop.*/
            macLED1_TOGGLE ();
            OSTimeDly ( 1000, OS_OPT_TIME_DLY, & err );
        }


    }


    /*
    ***********************************************************************
    *                              LED2 TASK
    ************************************************************************
    */

    static  void  AppTaskLed2 ( void * p_arg )
    {
        OS_ERR      err;


        (void)p_arg;


    while (DEF_TRUE) {          /* Task body, always written as an
                infinite loop.  */
            macLED2_TOGGLE ();
            OSTimeDly ( 5000, OS_OPT_TIME_DLY, & err );
        }


    }


    /*
    **********************************************************************
    *                            LED3 TASK
    **********************************************************************
    */

    static  void  AppTaskLed3 ( void * p_arg )
    {
        OS_ERR      err;


        (void)p_arg;


    while (DEF_TRUE) {       /* Task body, always written as an infinite
            loop.     */
            macLED3_TOGGLE ();
            OSTimeDly ( 10000, OS_OPT_TIME_DLY, & err );
        }


    }



下载验证
~~~~~~~~~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），
可以看到板子上面的三个LED灯以不同的频率在闪烁，说明我们创建的多任务已经跑起来了。
