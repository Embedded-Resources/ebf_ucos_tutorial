.. vim: syntax=rst

事件
========

事件的基本概念
~~~~~~~~~~~~~~~~~~~

事件是一种实现任务间通信的机制，主要用于实现多任务间的同步，但事件通信只能是事件类型的通信，无数据传输。与信号量不同的是，
它可以实现一对多，多对多的同步。即一个任务可以等待多个事件的发生：可以是任意一个事件发生时唤醒任务进行事件处理；
也可以是几个事件都发生后才唤醒任务进行事件处理。同样，也可以是多个任务同步多个事件。

每一个事件组只需要很少的RAM空间来保存事件组的状态。事件组存储在一个OS_FLAGS类型的Flags变量中，该变量在事件结构体中定义。
而变量的宽度由我们自己定义，可以是8位、16位、32位的变量，取决于os_type.h中的OS_FLAGS的位数。在STM32中，我们一般将其定义为32位的变量，
有32个位用来实现事件标志组。每一位代表一个事件，任务通过“逻辑与”或“逻辑或”与一个或多个事件建立关联，形成一个事件组。
事件的“逻辑或”也被称作是独立型同步，指的是任务感兴趣的所有事件任一件发生即可被唤醒；事件“逻辑与”则被称为是关联型同步，
指的是任务感兴趣的若干事件都发生时才被唤醒，并且事件发生的时间可以不同步。

多任务环境下，任务、中断之间往往需要同步操作，一个事件发生会告知等待中的任务，即形成一个任务与任务、
中断与任务间的同步。事件可以提供一对多、多对多的同步操作。一对多同步模型：一个任务等待多个事件的触发，
这种情况是比较常见的；多对多同步模型：多个任务等待多个事件的触发。

任务可以通过设置事件位来实现事件的触发和等待操作。μC/OS的事件仅用于同步，不提供数据传输功能。

μC/OS提供的事件具有如下特点：

    -  事件只与任务相关联，事件相互独立，一个32位（数据宽度由用户定义）的事件集合用于标识该任务发生的事件类型，
       其中每一位表示一种事件类型（0表示该事件类型未发生、1表示该事件类型已经发生），一共32种事件类型。

    -  事件仅用于同步，不提供数据传输功能。

    -  事件无排队性，即多次向任务设置同一事件(如果任务还未来得及读走)，等效于只设置一次。

    -  允许多个任务对同一事件进行读写操作。

    -  支持事件等待超时机制。

    -  支持显式清除事件。

在μC/OS的等待事件中，用户可以选择感兴趣的事件，并且选择等待事件的选项，它有4个属性，分别是逻辑与、逻辑或、等待所有事件清除或者等待任意事件清除。
当任务等待事件同步时，可以通过任务感兴趣的事件位和事件选项来判断当前获取的事件是否满足要求，如果满足则说明任务等待到对应的事件，系统将唤醒等待的任务；
否则，任务会根据用户指定的阻塞超时时间继续等待下去。

事件的应用场景
~~~~~~~~~~~~~~~~~~~

μC/OS的事件用于事件类型的通讯，无数据传输，也就是说，我们可以用事件来做标志位，判断某些事件是否发生了，然后根据结果做处理，
那很多人又会问了，为什么我不直接用变量做标志呢，岂不是更好更有效率？非也非也，若是在裸机编程中，用全局变量是最为有效的方法，
这点我不否认，但是在操作系统中，使用全局变量就要考虑以下问题了：

-  如何对全局变量进行保护呢，如何处理多任务同时对它进行访问？

-  如何让内核对事件进行有效管理呢？使用全局变量的话，就需要在任务中轮询查看事件是否发送，这简直就是在浪费CPU资源啊，
   还有等待超时机制，使用全局变量的话需要用户自己去实现。

所以，在操作系统中，还是使用操作系统给我们提供的通信机制就好了，简单方便还实用。

在某些场合，可能需要多个时间发生了才能进行下一步操作，比如一些危险机器的启动，需要检查各项指标，当指标不达标的时候，
无法启动，但是检查各个指标的时候，不能一下子检测完毕啊，所以，需要事件来做统一的等待，当所有的事件都完成了，
那么机器才允许启动，这只是事件的其中一个应用。

事件可使用于多种场合，它能够在一定程度上替代信号量，用于任务与任务间，中断与任务间的同步。一个任务或中断服务例程发送一个事件给事件对象，
而后等待的任务被唤醒并对相应的事件进行处理。但是它与信号量不同的是，事件的发送操作是不可累计的，而信号量的释放动作是可累计的。事件另外一个特性是，
接收任务可等待多种事件，即多个事件对应一个任务或多个任务。同时按照任务等待的参数，可选择是“逻辑或”触发还是“逻辑与”触发。
这个特性也是信号量等所不具备的，信号量只能识别单一同步动作，而不能同时等待多个事件的同步。

各个事件可分别发送或一起发送给事件对象，而任务可以等待多个事件，任务仅对感兴趣的事件进行关注。当有它们感兴趣的事件发生时并且符合感兴趣的条件，
任务将被唤醒并进行后续的处理动作。

事件运作机制
~~~~~~~~~~~~~~~~~~

等待（接收）事件时，可以根据感兴趣的参事件类型等待事件的单个或者多个事件类型。事件等待成功后，
必须使用OS_OPT_PEND_FLAG_CONSUME选项来清除已接收到的事件类型，否则不会清除已接收到的事件，这样就需要用户显式清除事件位。
用户可以自定义通过传入opt选项来选择读取模式，是等待所有感兴趣的事件还是等待感兴趣的任意一个事件。

设置事件时，对指定事件写入指定的事件类型，设置事件集合的对应事件位为1，可以一次同时写多个事件类型，设置事件成功可能会触发任务调度。

清除事件时，根据入参数事件句柄和待清除的事件类型，对事件对应位进行清零操作。

事件不与任务相关联，事件相互独立，一个32位的变量就是事件的集合，用于标识该任务发生的事件类型，
其中每一位表示一种事件类型（0表示该事件类型未发生、1表示该事件类型已经发生），一共32种事件类型具体见图 事件集合Flags_ 。

.. image:: media/event/event002.png
   :align: center
   :name: 事件集合Flags
   :alt: 事件集合Flags（一个32位的变量）


事件唤醒机制，当任务因为等待某个或者多个事件发生而进入阻塞态，当事件发生的时候会被唤醒，其过程具体见图 事件唤醒任务示意图_ 。

.. image:: media/event/event003.png
   :align: center
   :name: 事件唤醒任务示意图
   :alt: 事件唤醒任务示意图


任务1对事件3或事件5感兴趣（逻辑或），当发生其中的某一个事件都会被唤醒，并且执行相应操作。而任务2对事件3与事件5感兴趣（逻辑与），
当且仅当事件3与事件5都发生的时候，任务2才会被唤醒，如果只有一个其中一个事件发生，那么任务还是会继续等待事件发生。
如果在接收事件函数中设置了清除事件位选项OS_OPT_PEND_FLAG_CONSUME，那么当任务唤醒后将把事件3和事件5的事件标志清零，否则事件标志将依然存在。

事件控制块
~~~~~~~~~~~~~

理论上用户可以创建任意个事件（仅限制于处理器的 RAM大小）。通过设置os_cfg.h中的宏定义 OS_CFG_FLAG_EN为 1即可开启事件功能。
事件是一个内核对象，由数据类型OS_FLAG_GRP定义，该数据类型由 os_flag_grp定义（在os.h文件）。

μC/OS的事件由多个元素组成，在事件被创建时，需要由我们自己定义事件（也可以称之为事件句柄），因为它是用于保存事件的一些信息的，
其数据结构OS_FLAG_GRP除了事件必须的一些基本信息外，还有PendList链表与一个32位的事件组变量Flags等，为的是方便系统来管理事件。
其数据结构具体见 代码清单:事件-1_ ，示意图具体见图 事件控制块数据结构_ 。

.. image:: media/event/event004.png
   :align: center
   :name: 事件控制块数据结构
   :alt: 事件控制块数据结构



.. code-block:: c
    :caption: 代码清单:事件-1事件控制块数据结构
    :name: 代码清单:事件-1
    :linenos:

    struct  os_flag_grp

    {
    /* ------------------ GENERIC  MEMBERS ------------------ */
        OS_OBJ_TYPE          Type;               	(1)

        CPU_CHAR            *NamePtr;            	(2)

        OS_PEND_LIST         PendList;          	(3)

    #if OS_CFG_DBG_EN > 0u
        OS_FLAG_GRP         *DbgPrevPtr;
        OS_FLAG_GRP         *DbgNextPtr;
        CPU_CHAR            *DbgNamePtr;
    #endif
    /* ------------------ SPECIFIC MEMBERS ------------------ */
        OS_FLAGS             Flags;              (4)

        CPU_TS               TS;                 (5)

    };


-   代码清单:事件-1_  **(1)**\ ：事件的类型，用户无需理会，μC/OS用于识别它是一个事件。

-   代码清单:事件-1_  **(2)**\ ：事件的名字，每个内核对象都会被分配一个名，采用字符串形式记录下来。

-   代码清单:事件-1_  **(3)**\ ：因为可以有多个任务同时等待系统中的事件，
    所以事件中包含了一个用于控制挂起任务列表的结构体，用于记录阻塞在此事件上的任务。

-   代码清单:事件-1_  **(4)**\ ：事件中包含了很多标志位，
    Flags这个变量中保存了当前这些标志位的状态。这个变量可以为8位，16位或32位。

-   代码清单:事件-1_  **(5)**\ ：事件中的变量TS用于保存该事件最后一次被释放的时间戳。当事件被释放时，读取时基计数值并存放到该变量中。

注意：用户代码不能直接访问这个结构体，必须通过μC/OS提供的API访问。

事件函数接口
~~~~~~~~~~~~~~~~~~

事件创建函数OSFlagCreate()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

事件创建函数，顾名思义，就是创建一个事件，与其他内核对象一样，都是需要先创建才能使用的资源，μC/OS给我们提供了一个创建事件的函数OSFlagCreate()，
当创建一个事件时，系统会对我们定义的事件控制块进行基本的初始化，所以，在使用创建函数之前，我们需要先定义一个事件控制块（句柄），
事件创建函数的源码具体见 代码清单:事件-2_ 。

.. code-block:: c
    :caption: 代码清单:事件-2OSFlagCreate()源码
    :name: 代码清单:事件-2
    :linenos:

    void  OSFlagCreate (OS_FLAG_GRP  *p_grp,  (1)	//事件指针
                        CPU_CHAR     *p_name, (2)	//命名事件
                        OS_FLAGS      flags,  (3)	//标志初始值
                        OS_ERR       *p_err)  (4)	//返回错误类型
    {
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
    //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
    // SR（临界段关中断只需保存SR），开中断时将该值还原。

    #ifdef OS_SAFETY_CRITICAL(5)//如果启用了安全检测
    if (p_err == (OS_ERR *)0)           //如果错误类型实参为空
        {
            OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数
            return;                         //返回，停止执行
        }
    #endif

    #ifdef OS_SAFETY_CRITICAL_IEC61508(6)//如果启用了安全关键
    if (OSSafetyCriticalStartFlag == DEF_TRUE)   //如果OSSafetyCriticalStart()后创建
        {
            *p_err = OS_ERR_ILLEGAL_CREATE_RUN_TIME;  //错误类型为“非法创建内核对象”
            return;                                  //返回，停止执行
        }
    #endif

    #if OS_CFG_CALLED_FROM_ISR_CHK_EN > 0u(7)//如果启用了中断中非法调用检测
    if (OSIntNestingCtr > (OS_NESTING_CTR)0)   //如果该函数是在中断中被调用
        {
            *p_err = OS_ERR_CREATE_ISR;             //错误类型为“在中断中创建对象”
            return;                                //返回，停止执行
        }
    #endif

    #if OS_CFG_ARG_CHK_EN > 0u(8)//如果启用了参数检测
    if (p_grp == (OS_FLAG_GRP *)0)   //如果 p_grp 为空
        {
            *p_err = OS_ERR_OBJ_PTR_NULL; //错误类型为“创建对象为空”
            return;                      //返回，停止执行
        }
    #endif

        OS_CRITICAL_ENTER();         (9)//进入临界段
        p_grp->Type    = OS_OBJ_TYPE_FLAG; //标记创建对象数据结构为事件
        p_grp->NamePtr = p_name;      (10)//标记事件的名称
        p_grp->Flags   = flags;        (11)//设置标志初始值
        p_grp->TS      = (CPU_TS)0;    (12)//清零事件的时间戳
        OS_PendListInit(&p_grp->PendList);(13)//初始化该事件的等待列表

    #if OS_CFG_DBG_EN > 0u//如果启用了调试代码和变量
        OS_FlagDbgListAdd(p_grp);          //将该事件添加到事件双向调试链表
    #endif
        OSFlagQty++;                 (14)//事件个数加1

        OS_CRITICAL_EXIT_NO_SCHED();       //退出临界段（无调度）
        *p_err = OS_ERR_NONE;         (15)//错误类型为“无错误”
    }


-   代码清单:事件-2_  **(1)**\ ：事件控制块指针，指向我们定义的事件控制块结构体变量，所以在创建之前我们需要先定义一个事件控制块变量。

-   代码清单:事件-2_  **(2)**\ ：事件名称，字符串形式。

-   代码清单:事件-2_  **(3)**\ ：事件标志位的初始值，一般为常为0。

-   代码清单:事件-2_  **(4)**\ ：用于保存返回的错误类型。

-   代码清单:事件-2_  **(5)**\ ：如果启用了安全检测（默认禁用），
    在编译时则会包含安全检测相关的代码，如果错误类型实参为空，系统会执行安全检测异常函数，然后返回，不执行创建互斥量操作。

-   代码清单:事件-2_  **(6)**\ ：如果启用（默认禁用）了安全关键检测，
    在编译时则会包含安全关键检测相关的代码，如果是在调用OSSafetyCriticalStart()后创建该事件，则是非法的，返回错误类型为“非法创建内核对象”错误代码，并且退出，不执行创建事件操作。

-   代码清单:事件-2_  **(7)**\ ：如果启用了中断中非法调用检测（默认启用），
    在编译时则会包含中断非法调用检测相关的代码，如果该函数是在中断中被调用，则是非法的，返回错误类型为“在中断中创建对象”的错误代码，并且退出，不执行创建事件操作。

-   代码清单:事件-2_  **(8)**\ ：如果启用了参数检测（默认启用），
    在编译时则会包含参数检测相关的代码，如果p_grp参数为空，返回错误类型为“创建对象为空”的错误代码，并且退出，不执行创建事件操作。

-   代码清单:事件-2_  **(9)**\ ：进入临界段，标记创建对象数据结构为事件。

-   代码清单:事件-2_  **(10)**\ ：初始化事件的名称。

-   代码清单:事件-2_  **(11)**\ ：设置事件标志的初始值。

-   代码清单:事件-2_  **(12)**\ ：记录时间戳的变量TS初始化为0。

-   代码清单:事件-2_  **(13)**\ ：初始化该事件的等待列表。

-   代码清单:事件-2_  **(14)**\ ：系统事件个数加1。

-   代码清单:事件-2_  **(15)**\ ：退出临界段（无调度），创建事件成功。

如果我们创建一个事件，那么事件创建成功的示意图具体见图 事件创建完成示意图_ 。

.. image:: media/event/event005.png
   :align: center
   :name: 事件创建完成示意图
   :alt: 事件创建完成示意图


事件创建函数的使用实例具体见 代码清单:事件-3_ 。

.. code-block:: c
    :caption: 代码清单:事件-3OSFlagCreate()使用实例
    :name: 代码清单:事件-3
    :linenos:

    OS_FLAG_GRP flag_grp;                   //声明事件

    OS_ERR      err;

    /* 创建事件 flag_grp */
    OSFlagCreate ((OS_FLAG_GRP  *)&flag_grp,        //指向事件的指针
                (CPU_CHAR     *)"FLAG For Test",  //事件的名字
                (OS_FLAGS      )0,                //事件的初始值
                (OS_ERR       *)&err);            //返回错误类型


事件删除函数OSFlagDel()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在很多场合，某些事件只用一次的，就好比在事件应用场景说的危险机器的启动，假如各项指标都达到了，并且机器启动成功了，
那这个事件之后可能就没用了，那就可以进行销毁了。想要删除事件怎么办？μC/OS给我们提供了一个删除事件的函数——OSFlagDel()，
使用它就能将事件进行删除了。当系统不再使用事件对象时，可以通过删除事件对象控制块来进行删除，具体见 代码清单:事件-4_ 。

注意，想要使用删除事件函数则必须将OS_CFG_FLAG_DEL_EN宏定义配置为1，该宏定义在os_cfg.h文件中。

.. code-block:: c
    :caption: 代码清单:事件-4OSFlagDel()源码
    :name: 代码清单:事件-4
    :linenos:

    #if OS_CFG_FLAG_DEL_EN > 0u//如果启用了 OSFlagDel() 函数
    OS_OBJ_QTY  OSFlagDel (OS_FLAG_GRP  *p_grp, (1)	//事件指针
                        OS_OPT        opt,   (2)	//选项
                        OS_ERR       *p_err) (3)	//返回错误类型
    {
        OS_OBJ_QTY        cnt;
        OS_OBJ_QTY        nbr_tasks;
        OS_PEND_DATA     *p_pend_data;
        OS_PEND_LIST     *p_pend_list;
        OS_TCB           *p_tcb;
        CPU_TS            ts;
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
    //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
    // SR（临界段关中断只需保存SR），开中断时将该值还原。

    #ifdef OS_SAFETY_CRITICAL(4)//如果启用（默认禁用）了安全检测
    if (p_err == (OS_ERR *)0)           //如果错误类型实参为空
        {
            OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数
    return ((OS_OBJ_QTY)0);         //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_CALLED_FROM_ISR_CHK_EN > 0u(5)//如果启用了中断中非法调用检测
    if (OSIntNestingCtr > (OS_NESTING_CTR)0)   //如果该函数在中断中被调用
        {
            *p_err = OS_ERR_DEL_ISR;                //错误类型为“在中断中删除对象”
    return ((OS_OBJ_QTY)0);                //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_ARG_CHK_EN > 0u(6)//如果启用了参数检测
    if (p_grp == (OS_FLAG_GRP *)0)        //如果 p_grp 为空
        {
            *p_err  = OS_ERR_OBJ_PTR_NULL;     //错误类型为“对象为空”
    return ((OS_OBJ_QTY)0);           //返回0（有错误），停止执行
        }
    switch (opt)                (7)//根据选项分类处理
        {
    case OS_OPT_DEL_NO_PEND:          //如果选项在预期内
    case OS_OPT_DEL_ALWAYS:
    break;                       //直接跳出

    default:                   (8)//如果选项超出预期
            *p_err = OS_ERR_OPT_INVALID;  //错误类型为“选项非法”
    return ((OS_OBJ_QTY)0);      //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_OBJ_TYPE_CHK_EN > 0u(9)//如果启用了对象类型检测
    if (p_grp->Type != OS_OBJ_TYPE_FLAG)  //如果 p_grp 不是事件类型
        {
            *p_err = OS_ERR_OBJ_TYPE;          //错误类型为“对象类型有误”
    return ((OS_OBJ_QTY)0);           //返回0（有错误），停止执行
        }
    #endif
        OS_CRITICAL_ENTER();                         //进入临界段
        p_pend_list = &p_grp->PendList;       (10)//获取消息队列的等待列表
        cnt         = p_pend_list->NbrEntries;   (11)//获取等待该队列的任务数
        nbr_tasks   = cnt;                          //按照任务数目逐个处理
    switch (opt)                    (12)//根据选项分类处理
        {
    case OS_OPT_DEL_NO_PEND:        (13)//如果只在没任务等待时进行删除
    if (nbr_tasks == (OS_OBJ_QTY)0)     //如果没有任务在等待该事件
            {
    #if OS_CFG_DBG_EN > 0u//如果启用了调试代码和变量
                OS_FlagDbgListRemove(p_grp); (14)//将该事件从事件调试列表移除
    #endif
                OSFlagQty--;           (15)//事件数目减1
                OS_FlagClr(p_grp);      (16)//清除该事件的内容

                OS_CRITICAL_EXIT();             //退出临界段
                *p_err = OS_ERR_NONE;     (17)//错误类型为“无错误”
            }
    else
            {
                OS_CRITICAL_EXIT();             //退出临界段
                *p_err = OS_ERR_TASK_WAITING;  (18)//错误类型为“有任务在等待事件”
            }
    break;                              //跳出

    case OS_OPT_DEL_ALWAYS:           (19)//如果必须删除事件
            ts = OS_TS_GET();             (20)//获取时间戳
    while (cnt > 0u)              (21)//逐个移除该事件等待列表中的任务
            {
                p_pend_data = p_pend_list->HeadPtr;
                p_tcb       = p_pend_data->TCBPtr;
                OS_PendObjDel((OS_PEND_OBJ *)((void *)p_grp),
                            p_tcb,
                            ts);		(22)
                cnt--;
            }
    #if OS_CFG_DBG_EN > 0u//如果启用了调试代码和变量
            OS_FlagDbgListRemove(p_grp);        //将该事件从事件调试列表移除
    #endif
            OSFlagQty--;                   (23)//事件数目减1
            OS_FlagClr(p_grp);             (24)//清除该事件的内容
            OS_CRITICAL_EXIT_NO_SCHED();   		//退出临界段（无调度）
            OSSched();                      (25)//调度任务
            *p_err = OS_ERR_NONE;     (26)//错误类型为“无错误”
    break;                              //跳出

    default:                       (27)//如果选项超出预期
            OS_CRITICAL_EXIT();                 //退出临界段
            *p_err = OS_ERR_OPT_INVALID;         //错误类型为“选项非法”
    break;                              //跳出
        }
    return (nbr_tasks);          (28)//返回删除事件前等待其的任务数
    }
    #endif


-   代码清单:事件-4_  **(1)**\ ：事件控制块指针，指向我们定义的事件控制块结构体变量，
    所以在删除之前我们需要先定义一个事件控制块变量，并且成功创建事件后再进行删除操作。

-   代码清单:事件-4_  **(2)**\ ：事件删除的选项。

-   代码清单:事件-4_  **(3)**\ ：用于保存返回的错误类型。

-   代码清单:事件-4_  **(4)**\ ：如果启用了安全检测（默认），在编译时则会包含安全检测相关的代码，
    如果错误类型实参为空，系统会执行安全检测异常函数，然后返回，不执行删除互斥量操作。

-   代码清单:事件-4_  **(5)**\ ：如果启用了中断中非法调用检测（默认启用），
    在编译时则会包含中断非法调用检测相关的代码，如果该函数是在中断中被调用，则是非法的，返回错误类型为“在中断中删除对象”的错误代码，并且退出，不执行删除事件操作。

-   代码清单:事件-4_  **(6)**\ ：如果启用了参数检测（默认启用），
    在编译时则会包含参数检测相关的代码，如果p_grp参数为空，返回错误类型为“内核对象为空”的错误代码，并且退出，不执行删除事件操作。

-   代码清单:事件-4_  **(7)**\ ：判断opt选项是否合理，该选项有两个，
    OS_OPT_DEL_ALWAYS与OS_OPT_DEL_NO_PEND，在os.h文件中定义。此处是判断一下选项是否在预期之内，如果在则跳出switch语句。

-   代码清单:事件-4_  **(8)**\ ：如果选项超出预期，则返回错误类型为“选项非法”的错误代码，退出，不继续执行。

-   代码清单:事件-4_  **(9)**\ ：如果启用了对象类型检测，在编译时则会包含对象类型检测相关的代码，
    如果 p_grp 不是事件类型，返回错误类型为“内核对象类型错误”的错误代码，并且退出，不执行删除事件操作。

-   代码清单:事件-4_  **(10)**\ ：进入临界段，程序执行到这里，表示可以删除事件了，
    系统首先获取互斥量的等待列表保存到p_pend_list变量中，μC/OS在删事件的时候是通过该变量访问事件等待列表的任务的。

-   代码清单:事件-4_  **(11)**\ ：获取等待该队列的任务数，按照任务个数逐个处理。

-   代码清单:事件-4_  **(12)**\ ：根据选项分类处理。

-   代码清单:事件-4_  **(13)**\ ：如果opt是OS_OPT_DEL_NO_PEND，则表示只在没有任务等待的情况下删除事件，
    如果当前系统中有任务还在等待该事件的某些位，则不能进行删除操作，反之，则可以删除事件。

-   代码清单:事件-4_  **(14)**\ ：如果启用了调试代码和变量，将该事件从事件调试列表移除。

-   代码清单:事件-4_  **(15)**\ ：系统的事件个数减一。

-   代码清单:事件-4_  **(16)**\ ：清除该事件的内容。

-   代码清单:事件-4_  **(17)**\ ：删除成功，返回错误类型为“无错误”的错误代码。

-   代码清单:事件-4_  **(18)**\ ：：如果有任务在等待该事件，则返回错误类型为“有任务在等待该事件”错误代码。

-   代码清单:事件-4_  **(19)**\ ：如果opt是OS_OPT_DEL_ALWAYS，
    则表示无论如何都必须删除事件，那么在删除之前，系统会把所有阻塞在该事件上的任务恢复。

-   代码清单:事件-4_  **(20)**\ ：获取删除时候的时间戳。

-   代码清单:事件-4_  **(21)**\ ：根据前面cnt记录阻塞在该事件上的任务个数，逐个移除该事件等待列表中的任务。

-   代码清单:事件-4_  **(22)**\ ：调用OS_PendObjDel()函数将阻塞在内核对象（如事件）上的任务从阻塞态恢复，
    此时系统在删除内核对象，删除之后，这些等待事件的任务需要被恢复，其源码具体**代码清单:消息队列-1**。

-   代码清单:事件-4_  **(23)**\ ：系统事件数目减1

-   代码清单:事件-4_  **(24)**\ ：清除该事件的内容。

-   代码清单:事件-4_  **(25)**\ ：进行一次任务调度。

-   代码清单:事件-4_  **(26)**\ ：删除事件完成，返回错误类型为“无错误”的错误代码。

-   代码清单:事件-4_  **(27)**\ ：如果选项超出预期则返回错误类型为“任务状态非法”的错误代码。

-   代码清单:事件-4_  **(28)**\ ：返回删除事件前等待其的任务数

事件删除函数OSFlagDel()的使用也是很简单的，只需要传入要删除的事件的句柄与选项还有保存返回的错误类型即可，调用函数时，
系统将删除这个事件。需要注意的是在调用删除事件函数前，系统应存在已创建的事件。如果删除事件时，系统中有任务正在等待该事件，
则不应该进行删除操作，删除事件函数OSFlagDel()的使用实例具体见 代码清单:事件-5_ 。

.. code-block:: c
    :caption: 代码清单:事件-5OSFlagDel()函数使用实例
    :name: 代码清单:事件-5
    :linenos:

    OS_FLAG_GRPflag_grp;;                             //声明事件句柄

    OS_ERR      err;

    /* 删除事件/
    OSFlagDel((OS_FLAG_GRP*)&flag_grp,      //指向事件的指针
    OS_OPT_DEL_NO_PEND,
    (OS_ERR      *)&err);             //返回错误类型


事件设置函数OSFlagPost()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

OSFlagPost()用于设置事件组中指定的位，当位被置位之后，并且满足任务的等待事件，那么等待在事件该标志位上的任务将会被恢复。使用该函数接口时，
通过参数指定的事件标志来设置事件的标志位，然后遍历等待在事件对象上的事件等待列表，判断是否有任务的事件激活要求与当前事件对象标志值匹配，
如果有，则唤醒该任务。简单来说，就是设置我们自己定义的事件标志位为1，并且看看有没有任务在等待这个事件，有的话就唤醒它，
OSFlagPost()函数源码具体见 代码清单:事件-6_ 。

.. code-block:: c
    :caption: 代码清单:事件-6 OSFlagPost()源码
    :name: 代码清单:事件-6
    :linenos:

    OS_FLAGS  OSFlagPost (OS_FLAG_GRP  *p_grp, 		//事件指针
                        OS_FLAGS      flags, 		//选定要操作的标志位
                        OS_OPT        opt,   		//选项
                        OS_ERR       *p_err) 		//返回错误类型
    {
        OS_FLAGS  flags_cur;
        CPU_TS    ts;



    #ifdef OS_SAFETY_CRITICAL//如果启用（默认禁用）了安全检测
    if (p_err == (OS_ERR *)0)           //如果错误类型实参为空
        {
            OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数
    return ((OS_FLAGS)0);           //返回0，停止执行
        }
    #endif

    #if OS_CFG_ARG_CHK_EN > 0u//如果启用（默认启用）了参数检测
    if (p_grp == (OS_FLAG_GRP *)0)       //如果参数 p_grp 为空
        {
            *p_err  = OS_ERR_OBJ_PTR_NULL;    //错误类型为“事件对象为空”
    return ((OS_FLAGS)0);            //返回0，停止执行
        }
    switch (opt)                        //根据选项分类处理
        {
    case OS_OPT_POST_FLAG_SET:       //如果选项在预期之内
    case OS_OPT_POST_FLAG_CLR:
    case OS_OPT_POST_FLAG_SET | OS_OPT_POST_NO_SCHED:
    case OS_OPT_POST_FLAG_CLR | OS_OPT_POST_NO_SCHED:
    break;                      //直接跳出

    default:                         //如果选项超出预期
            *p_err = OS_ERR_OPT_INVALID; //错误类型为“选项非法”
    return ((OS_FLAGS)0);       //返回0，停止执行
        }
    #endif

    #if OS_CFG_OBJ_TYPE_CHK_EN > 0u//如果启用了对象类型检测
    if (p_grp->Type != OS_OBJ_TYPE_FLAG)   //如果 p_grp 不是事件类型
        {
            *p_err = OS_ERR_OBJ_TYPE;           //错误类型“对象类型有误”
    return ((OS_FLAGS)0);              //返回0，停止执行
        }
    #endif

        ts = OS_TS_GET();                             //获取时间戳
    #if OS_CFG_ISR_POST_DEFERRED_EN > 0u(1)//如果启用了中断延迟发布
    if (OSIntNestingCtr > (OS_NESTING_CTR)0)      //如果该函数是在中断中被调用
        {
            OS_IntQPost((OS_OBJ_TYPE)OS_OBJ_TYPE_FLAG,//将该事件发布到中断消息队列
                        (void      *)p_grp,
                        (void      *)0,
                        (OS_MSG_SIZE)0,
                        (OS_FLAGS   )flags,
                        (OS_OPT     )opt,
                        (CPU_TS     )ts,
                        (OS_ERR    *)p_err);
    return ((OS_FLAGS)0);                     //返回0，停止执行
        }
    #endif
    /* 如果没有启用中断延迟发布 */
        flags_cur = OS_FlagPost(p_grp,               //将事件直接发布
                                flags,
                                opt,
                                ts,
                                p_err);	(2)

    return (flags_cur);                         //返回当前标志位的值
    }


注：因为程序大体与之前的程序差不多，此处仅介绍重点。

-   代码清单:事件-6_  **(1)**\ ：如果启用了中断延迟发布并且该函数在中断中被调用，则将该事件发布到中断消息队列。

-   代码清单:事件-6_  **(2)**\ ：如果没有启用中断延迟发布，
    则直接将该事件对应的标志位置位，OS_FlagPost()函数源码具体见 代码清单:事件-7_ 。

.. code-block:: c
    :caption: 代码清单:事件-7OS_FlagPost()源码
    :name: 代码清单:事件-7
    :linenos:

    OS_FLAGS  OS_FlagPost (OS_FLAG_GRP  *p_grp, 	(1)	//事件指针
                        OS_FLAGS      flags, 	(2)	//选定要操作的标志位
                        OS_OPT        opt,   	(3)	//选项
                        CPU_TS        ts,    	(4)	//时间戳
                        OS_ERR       *p_err) 	(5)	//返回错误类型
    {
        OS_FLAGS        flags_cur;
        OS_FLAGS        flags_rdy;
        OS_OPT          mode;
        OS_PEND_DATA   *p_pend_data;
        OS_PEND_DATA   *p_pend_data_next;
        OS_PEND_LIST   *p_pend_list;
        OS_TCB         *p_tcb;
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
        //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
        // SR（临界段关中断只需保存SR），开中断时将该值还原。

        CPU_CRITICAL_ENTER();                                //关中断
        switch (opt)                         (6)//根据选项分类处理
        {
            case OS_OPT_POST_FLAG_SET:          (7)//如果要求将选定位置1
            case OS_OPT_POST_FLAG_SET | OS_OPT_POST_NO_SCHED:
                p_grp->Flags |=  flags;                     //将选定位置1
                break;                                      //跳出

            case OS_OPT_POST_FLAG_CLR:           (8)//如果要求将选定位请0
            case OS_OPT_POST_FLAG_CLR | OS_OPT_POST_NO_SCHED:
                    p_grp->Flags &= ~flags;                     //将选定位请0
                    break;                                      //跳出

            default:                           (9)//如果选项超出预期
                    CPU_CRITICAL_EXIT();                        //开中断
                    *p_err = OS_ERR_OPT_INVALID;                 //错误类型为“选项非法”
            return ((OS_FLAGS)0);                       //返回0，停止执行
        }
        p_grp->TS   = ts;                 (10)//将时间戳存入事件
        p_pend_list = &p_grp->PendList;    (11)//获取事件的等待列表
        if (p_pend_list->NbrEntries == 0u) (12)//如果没有任务在等待事件
        {
            CPU_CRITICAL_EXIT();                             //开中断
            *p_err = OS_ERR_NONE;                        //错误类型为“无错误”
            return (p_grp->Flags);                           //返回事件的标志值
        }
        /* 如果有任务在等待事件 */
        OS_CRITICAL_ENTER_CPU_EXIT();       (13)//进入临界段，重开中断
        p_pend_data = p_pend_list->HeadPtr; (14)//获取等待列表头个等待任务
        p_tcb       = p_pend_data->TCBPtr;
        while (p_tcb != (OS_TCB *)0)        (15)
        //从头至尾遍历等待列表的所有任务
        {
            p_pend_data_next = p_pend_data->NextPtr;
            mode = p_tcb->FlagsOpt & OS_OPT_PEND_FLAG_MASK; //获取任务的标志选项
            switch (mode)                (16)//根据任务的标志选项分类处理
            {
            OS_OPT_PEND_FLAG_SET_ALL:  (17)//如果要求任务等待的标志位都得置1
                flags_rdy = (OS_FLAGS)(p_grp->Flags & p_tcb->FlagsPend);
                if (flags_rdy == p_tcb->FlagsPend) //如果任务等待的标志位都置1了
                {
                    OS_FlagTaskRdy(p_tcb,            //让该任务准备运行
                                flags_rdy,
                                ts);		(18)
                }
                 break;                               //跳出

            case OS_OPT_PEND_FLAG_SET_ANY:     (19)
            //如果要求任务等待的标志位有1位置1即可
                flags_rdy = (OS_FLAGS)(p_grp->Flags & p_tcb->FlagsPend);(20)
                if (flags_rdy != (OS_FLAGS)0)     //如果任务等待的标志位有置1的
                {
                    OS_FlagTaskRdy(p_tcb,            //让该任务准备运行
                                flags_rdy,
                                ts);		(21)
                }
                break;                              //跳出

    #if OS_CFG_FLAG_MODE_CLR_EN > 0u(22)//如果启用了标志位清零触发模式
            case OS_OPT_PEND_FLAG_CLR_ALL: (23)//如果要求任务等待的标志位都得请0
                flags_rdy = (OS_FLAGS)(~p_grp->Flags & p_tcb->FlagsPend);
                if (flags_rdy == p_tcb->FlagsPend)  //如果任务等待的标志位都请0了
                {
                    OS_FlagTaskRdy(p_tcb,           //让该任务准备运行
                                flags_rdy,
                                ts);	(24)
                }
                break;            //跳出

            case OS_OPT_PEND_FLAG_CLR_ANY:     (25)
            //如果要求任务等待的标志位有1位请0即可
                flags_rdy = (OS_FLAGS)(~p_grp->Flags & p_tcb->FlagsPend);
                if (flags_rdy != (OS_FLAGS)0)      //如果任务等待的标志位有请0的
                {
                    OS_FlagTaskRdy(p_tcb,          //让该任务准备运行
                                flags_rdy,
                                ts);	(26)
                }
                break;                            //跳出
    #endif
            default:                      (27)//如果标志选项超出预期
                OS_CRITICAL_EXIT();               //退出临界段
                *p_err = OS_ERR_FLAG_PEND_OPT;     //错误类型为“标志选项非法”
                return ((OS_FLAGS)0);             //返回0，停止运行
            }
            p_pend_data = p_pend_data_next;   (28)//准备处理下一个等待任务
            if (p_pend_data != (OS_PEND_DATA *)0)      //如果该任务存在
            {
                p_tcb = p_pend_data->TCBPtr;   (29)//获取该任务的任务控制块
            }
            else//如果该任务不存在
            {
                p_tcb = (OS_TCB *)0;     (30)//清空 p_tcb，退出 while 循环
            }
        }
        OS_CRITICAL_EXIT_NO_SCHED();                  //退出临界段（无调度）

        if ((opt & OS_OPT_POST_NO_SCHED) == (OS_OPT)0)    //如果 opt没选择“发布时不调度任务”
        {
            OSSched();                 (31)//任务调度
        }

        CPU_CRITICAL_ENTER();        //关中断
        flags_cur = p_grp->Flags;    //获取事件的标志值
        CPU_CRITICAL_EXIT();         //开中断
        *p_err     = OS_ERR_NONE;     //错误类型为“无错误”
        return (flags_cur);       (32)//返回事件的当前标志值

    }


-   代码清单:事件-7_  **(1)**\ ：事件指针。

-   代码清单:事件-7_  **(2)**\ ：选定要操作的标志位。

-   代码清单:事件-7_  **(3)**\ ：设置事件标志位的选项。

-   代码清单:事件-7_  **(4)**\ ：时间戳。

-   代码清单:事件-7_  **(5)**\ ：返回错误类型。

-   代码清单:事件-7_  **(6)**\ ：根据选项分类处理。

-   代码清单:事件-7_  **(7)**\ ：如果要求将选定位置1，则置1即可，然后跳出switch语句。

-   代码清单:事件-7_  **(8)**\ ：如果要求将选定位请0，将选定位清零即可，然后跳出switch语句。

-   代码清单:事件-7_  **(9)**\ ：如果选项超出预期，返回错误类型为“选项非法”的错误代码，退出。

-   代码清单:事件-7_  **(10)**\ ：将时间戳存入事件的TS成员变量中。

-   代码清单:事件-7_  **(11)**\ ：获取事件的等待列表。

-   代码清单:事件-7_  **(12)**\ ：如果当前没有任务在等待事件，置位后直接退出即可，并且返回事件的标志值。

-   代码清单:事件-7_  **(13)**\ ：如果有任务在等待事件，那么进入临界段，重开中断。

-   代码清单:事件-7_  **(14)**\ ：获取等待列表头个等待任务，然后获取到对应的任务控制块，保存在p_tcb变量中。

-   代码清单:事件-7_  **(15)**\ ：当事件等待列表中有任务的时候，就从头至尾遍历等待列表的所有任务。

-   代码清单:事件-7_  **(16)**\ ：获取任务感兴趣的事件标志选项，根据任务的标志选项分类处理。

-   代码清单:事件-7_  **(17)**\ ：如果要求任务等待的标志位都得置1，就获取一下任务已经等待到的事件标志，保存在flags_rdy变量中。

-   代码清单:事件-7_  **(18)**\ ：如果任务等待的标志位都置1了，
    就调用OS_FlagTaskRdy()函数让该任务恢复为就绪态，准备运行，然后跳出switch语句。

-   代码清单:事件-7_  **(19)**\ ：如果要求任务等待的标志位有任意一个位置1即可。

-   代码清单:事件-7_  **(20)**\ ：那么就获取一下任务已经等待到的事件标志，保存在flags_rdy变量中。

-   代码清单:事件-7_  **(21)**\ ：如果任务等待的标志位有置1的，
    也就是满足了任务唤醒的条件，就调用OS_FlagTaskRdy()函数让该任务恢复为就绪态，准备运行，然后跳出switch语句。

-   代码清单:事件-7_  **(22)**\ ：如果启用了标志位清零触发模式，在编译的时候就会包含事件标志位清零触发的代码。

-   代码清单:事件-7_  **(23)**\ ：如果要求任务等待的标志位都得请0，那就看看等待任务对应的标志位是否清零了。

-   代码清单:事件-7_  **(24)**\ ：如果任务等待的标志位都请0了，
    就调用OS_FlagTaskRdy()函数让该任务恢复为就绪态，准备运行，然后跳出switch语句。

-   代码清单:事件-7_  **(25)**\ ：如果要求任务等待的标志位有1位请0即可。

-   代码清单:事件-7_  **(26)**\ ：那么如果任务等待的标志位有请0的，就让任务恢复为就绪态。

-   代码清单:事件-7_  **(27)**\ ：如果标志选项超出预期，返回错误类型为“标志选项非法”的错误代码，并且推出。

-   代码清单:事件-7_  **(28)**\ ：准备处理下一个等待任务。

-   代码清单:事件-7_  **(29)**\ ：如果该任务存在，获取该任务的任务控制块。

-   代码清单:事件-7_  **(30)**\ ：如果该任务不存在，清空 p_tcb，退出 while 循环。

-   代码清单:事件-7_  **(31)**\ ：进行一次任务调度。

-   代码清单:事件-7_  **(32)**\ ：事件标志位设置完成，返回事件的当前标志值。

OSFlagPost()的运用很简单，举个例子，比如我们要记录一个事件的发生，这个事件在事件组的位置是bit0，当它还未发生的时候，那么事件组bit0的值也是0，
当它发生的时候，我们往事件标志组的bit0位中写入这个事件，也就是0x01，那这就表示事件已经发生了，当然，μC/OS也支持事件清零触发。
为了便于理解，一般操作我们都是用宏定义来实现#define EVENT (0x01 << x)，“<< x”表示写入事件集合的bit x ，
在使用该函数之前必须先创建事件，具体见 代码清单:事件-8_ 。

.. code-block:: c
    :caption: 代码清单:事件-8xEventGroupSetBits()函数使用实例
    :name: 代码清单:事件-8
    :linenos:

    #define KEY1_EVENT  (0x01 << 0)//设置事件掩码的位0
    #define KEY2_EVENT  (0x01 << 1)//设置事件掩码的位1

    OS_FLAG_GRP flag_grp;                   //声明事件标志组

    static  void  AppTaskPost ( void * p_arg )
    {
        OS_ERR      err;


        (void)p_arg;


    while (DEF_TRUE) {                            //任务体
    //如果KEY1被按下
    if ( Key_ReadStatus ( macKEY1_GPIO_PORT, macKEY1_GPIO_PIN, 1 ) == 1 )
            {
                macLED1_ON ();                                    //点亮LED1

                OSFlagPost ((OS_FLAG_GRP  *)&flag_grp,
    //将标志组的BIT0置1
                            (OS_FLAGS      )KEY1_EVENT,
                            (OS_OPT        )OS_OPT_POST_FLAG_SET,
                            (OS_ERR       *)&err);

            }
    else//如果KEY1被释放
            {
                macLED1_OFF ();     //熄灭LED1

                OSFlagPost ((OS_FLAG_GRP  *)&flag_grp,
    //将标志组的BIT0清零
                            (OS_FLAGS      )KEY1_EVENT,
                            (OS_OPT        )OS_OPT_POST_FLAG_CLR,
                            (OS_ERR       *)&err);

            }
        //如果KEY2被按下
    if ( Key_ReadStatus ( macKEY2_GPIO_PORT, macKEY2_GPIO_PIN, 1 ) == 1 )
            {
                macLED2_ON ();                              //点亮LED2

                OSFlagPost ((OS_FLAG_GRP  *)&flag_grp,
    //将标志组的BIT1置1
                            (OS_FLAGS      )KEY2_EVENT,
                            (OS_OPT        )OS_OPT_POST_FLAG_SET,
                            (OS_ERR       *)&err);

            }
    else//如果KEY2被释放
            {
            macLED2_OFF ();    //熄灭LED2

                OSFlagPost ((OS_FLAG_GRP  *)&flag_grp,
    //将标志组的BIT1清零
                            (OS_FLAGS      )KEY2_EVENT,
                            (OS_OPT        )OS_OPT_POST_FLAG_CLR,
                            (OS_ERR       *)&err);

            }
        //每20ms扫描一次
            OSTimeDlyHMSM ( 0, 0, 0, 20, OS_OPT_TIME_DLY, & err );

        }

    }


事件等待函数OSFlagPend()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

既然标记了事件的发生，那么我们怎么知道他到底有没有发生，这也是需要一个函数来获取事件是否已经发生，μC/OS提供了一个等待指定事件的函数——OSFlagPend()，
通过这个函数，任务可以知道事件标志组中的哪些位，有什么事件发生了，然后通过“逻辑与”、“逻辑或”等操作对感兴趣的事件进行获取，并且这个函数实现了等待超时机制，
当且仅当任务等待的事件发生时，任务才能获取到事件信息。在这段时间中，如果事件一直没发生，该任务将保持阻塞状态以等待事件发生。
当其他任务或中断服务程序往其等待的事件设置对应的标志位，该任务将自动由阻塞态转为就绪态。当任务等待的时间超过了指定的阻塞时间，即使事件还未发生，
任务也会自动从阻塞态转移为就绪态。这样子很有效的体现了操作系统的实时性，如果事件正确获取（等待到）则返回对应的事件标志位，由用户判断再做处理，
因为在事件超时的时候也可能会返回一个不能确定的事件值，所以最好判断一下任务所等待的事件是否真的发生，
OSFlagPend()函数源码具体见 代码清单:事件-9_ 。

注意：OSFlagPend()函数源码比较长，我们只挑重点进行讲解。

.. code-block:: c
    :caption: 代码清单:事件-9 OSFlagPend()源码
    :name: 代码清单:事件-9
    :linenos:

    OS_FLAGS  OSFlagPend (OS_FLAG_GRP  *p_grp,  (1)	//事件指针
                        OS_FLAGS      flags, (2)	//选定要操作的标志位
                        OS_TICK       timeout,(3)	//等待期限（单位：时钟节拍）
                        OS_OPT        opt,    (4)	//选项
                        CPU_TS       *p_ts,   (5)//返回等到事件标志时的时间戳
                        OS_ERR       *p_err)  (6)	//返回错误类型
    {
        CPU_BOOLEAN   consume;
        OS_FLAGS      flags_rdy;
        OS_OPT        mode;
        OS_PEND_DATA  pend_data;
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
    //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
    // SR（临界段关中断只需保存SR），开中断时将该值还原。

    #ifdef OS_SAFETY_CRITICAL//如果启用（默认禁用）了安全检测
    if (p_err == (OS_ERR *)0)           //如果错误类型实参为空
        {
            OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数
    return ((OS_FLAGS)0);           //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_CALLED_FROM_ISR_CHK_EN > 0u//如果启用了中断中非法调用检测
    if (OSIntNestingCtr > (OS_NESTING_CTR)0)    //如果该函数在中断中被调用
        {
            *p_err = OS_ERR_PEND_ISR;                //错误类型为“在中断中中止等待”
    return ((OS_FLAGS)0);                   //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_ARG_CHK_EN > 0u//如果启用了参数检测
    if (p_grp == (OS_FLAG_GRP *)0)       //如果 p_grp 为空
        {
            *p_err = OS_ERR_OBJ_PTR_NULL;     //错误类型为“对象为空”
    return ((OS_FLAGS)0);            //返回0（有错误），停止执行
        }
    switch (opt)                 (7)//根据选项分类处理
        {
    case OS_OPT_PEND_FLAG_CLR_ALL:   //如果选项在预期内
    case OS_OPT_PEND_FLAG_CLR_ANY:
    case OS_OPT_PEND_FLAG_SET_ALL:
    case OS_OPT_PEND_FLAG_SET_ANY:
    case OS_OPT_PEND_FLAG_CLR_ALL | OS_OPT_PEND_FLAG_CONSUME:
    case OS_OPT_PEND_FLAG_CLR_ANY | OS_OPT_PEND_FLAG_CONSUME:
    case OS_OPT_PEND_FLAG_SET_ALL | OS_OPT_PEND_FLAG_CONSUME:
    case OS_OPT_PEND_FLAG_SET_ANY | OS_OPT_PEND_FLAG_CONSUME:
    case OS_OPT_PEND_FLAG_CLR_ALL | OS_OPT_PEND_NON_BLOCKING:
    case OS_OPT_PEND_FLAG_CLR_ANY | OS_OPT_PEND_NON_BLOCKING:
    case OS_OPT_PEND_FLAG_SET_ALL | OS_OPT_PEND_NON_BLOCKING:
    case OS_OPT_PEND_FLAG_SET_ANY | OS_OPT_PEND_NON_BLOCKING:
    case OS_OPT_PEND_FLAG_CLR_ALL | OS_OPT_PEND_FLAG_CONSUME | OS_OPT_PEND_NON_BLOCKING:
    case OS_OPT_PEND_FLAG_CLR_ANY | OS_OPT_PEND_FLAG_CONSUME | OS_OPT_PEND_NON_BLOCKING:
    case OS_OPT_PEND_FLAG_SET_ALL | OS_OPT_PEND_FLAG_CONSUME | OS_OPT_PEND_NON_BLOCKING:
    ase OS_OPT_PEND_FLAG_SET_ANY | OS_OPT_PEND_FLAG_CONSUME | OS_OPT_PEND_NON_BLOCKING:
    break;                     //直接跳出

    default:                    (8)//如果选项超出预期
            *p_err = OS_ERR_OPT_INVALID;//错误类型为“选项非法”
    return ((OS_OBJ_QTY)0);    //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_OBJ_TYPE_CHK_EN > 0u//如果启用了对象类型检测
    if (p_grp->Type != OS_OBJ_TYPE_FLAG)   //如果 p_grp 不是事件类型
        {
            *p_err = OS_ERR_OBJ_TYPE;           //错误类型为“对象类型有误”
    return ((OS_FLAGS)0);              //返回0（有错误），停止执行
        }
    #endif

    if ((opt & OS_OPT_PEND_FLAG_CONSUME) != (OS_OPT)0)(9)//选择了标志位匹配后自动取反
        {
            consume = DEF_TRUE;
        }
    else(10)//未选择标志位匹配后自动取反
        {
            consume = DEF_FALSE;
        }

    if (p_ts != (CPU_TS *)0)        //如果 p_ts 非空
        {
            *p_ts = (CPU_TS)0;           //初始化（清零）p_ts，待用于返回时间戳
        }

        mode = opt & OS_OPT_PEND_FLAG_MASK; (11)//从选项中提取对标志位的要求
        CPU_CRITICAL_ENTER();                 //关中断
    switch (mode)                (12)//根据事件触发模式分类处理
        {
    case OS_OPT_PEND_FLAG_SET_ALL:   (13)//如果要求所有标志位均要置1
            flags_rdy = (OS_FLAGS)(p_grp->Flags & flags); //提取想要的标志位的值
    if (flags_rdy == flags)        (14)//如果该值与期望值匹配
            {
    if (consume == DEF_TRUE)(15)//如果要求将标志位匹配后取反
                {
                    p_grp->Flags &= ~flags_rdy;           //清零事件的相关标志位
                }
    OSTCBCurPtr->FlagsRdy = flags_rdy; (16)//保存让任务脱离等待的标志值
    if (p_ts != (CPU_TS *)0)            //如果 p_ts 非空
                {
                    *p_ts  = p_grp->TS;          //获取任务等到事件时的时间戳
                }
                CPU_CRITICAL_EXIT();            //开中断
                *p_err = OS_ERR_NONE;            //错误类型为“无错误”
    return (flags_rdy);     (17)//返回让任务脱离等待的标志值
            }
    else(18)
    //如果想要标志位的值与期望值不匹配
            {
    if ((opt & OS_OPT_PEND_NON_BLOCKING) != (OS_OPT)0) //如果选择了不阻塞任务
                {
                    CPU_CRITICAL_EXIT();                  //关中断
                    *p_err = OS_ERR_PEND_WOULD_BLOCK;     //错误类型为“渴求阻塞”
    return ((OS_FLAGS)0);      (19)//返回0（有错误），停止执行
                }
    else(20)//如果选择了阻塞任务
                {
    if (OSSchedLockNestingCtr > (OS_NESTING_CTR)0)   //如果调度器被锁
                    {
                        CPU_CRITICAL_EXIT();              //关中断
                        *p_err = OS_ERR_SCHED_LOCKED;    //错误类型为“调度器被锁”
    return ((OS_FLAGS)0);   (21)//返回0（有错误），停止执行
                    }
                }
    /* 如果调度器未被锁 */
                OS_CRITICAL_ENTER_CPU_EXIT();             //进入临界段，重开中断
                OS_FlagBlock(&pend_data,             //阻塞当前运行任务，等待事件
                            p_grp,
                            flags,
                            opt,
                            timeout);		(22)
                OS_CRITICAL_EXIT_NO_SCHED();              //退出临界段（无调度）
            }
    break;                                        //跳出

    case OS_OPT_PEND_FLAG_SET_ANY:      (23)//如果要求有标志位被置1即可
            flags_rdy = (OS_FLAGS)(p_grp->Flags & flags); //提取想要的标志位的值
    if (flags_rdy != (OS_FLAGS)0)     (24)//如果有位被置1
            {
    if (consume == DEF_TRUE)           //如果要求将标志位匹配后取反
                {
                    p_grp->Flags &= ~flags_rdy;    //清零事件的相关标志位
                }
                OSTCBCurPtr->FlagsRdy = flags_rdy;  //保存让任务脱离等待的标志值
    if (p_ts != (CPU_TS *)0)                  //如果 p_ts 非空
                {
    *p_ts  = p_grp->TS;          //获取任务等到事件时的时间戳
                }
                CPU_CRITICAL_EXIT();                      //开中断
                *p_err = OS_ERR_NONE;                      //错误类型为“无错误”
    return (flags_rdy);      (25)//返回让任务脱离等待的标志值
            }
    else//如果没有位被置1
            {
    if ((opt & OS_OPT_PEND_NON_BLOCKING) != (OS_OPT)0)   //如果没设置阻塞任务
                {
                    CPU_CRITICAL_EXIT();                  //关中断
                    *p_err = OS_ERR_PEND_WOULD_BLOCK;     //错误类型为“渴求阻塞”
    return ((OS_FLAGS)0);     (26)//返回0（有错误），停止执行
                }
    else//如果设置了阻塞任务
                {
    if (OSSchedLockNestingCtr > (OS_NESTING_CTR)0)   //如果调度器被锁
                    {
                        CPU_CRITICAL_EXIT();              //关中断
                        *p_err = OS_ERR_SCHED_LOCKED;  //错误类型为“调度器被锁”
    return ((OS_FLAGS)0);(27)//返回0（有错误），停止执行
                    }
                }
    /* 如果调度器没被锁 */
                OS_CRITICAL_ENTER_CPU_EXIT();      //进入临界段，重开中断
                OS_FlagBlock(&pend_data,         //阻塞当前运行任务，等待事件
                            p_grp,
                            flags,
                            opt,
                            timeout);		(28)
                OS_CRITICAL_EXIT_NO_SCHED();       //退出中断（无调度）
            }
    break;                                        //跳出

    #if OS_CFG_FLAG_MODE_CLR_EN > 0u          (29)
    //如果启用了标志位清零触发模式
    case OS_OPT_PEND_FLAG_CLR_ALL:               //如果要求所有标志位均要清零
            flags_rdy = (OS_FLAGS)(~p_grp->Flags & flags);//提取想要的标志位的值
    if (flags_rdy == flags)            (30)//如果该值与期望值匹配
            {
    if(consume == DEF_TRUE)          //如果要求将标志位匹配后取反
                {
                    p_grp->Flags |= flags_rdy;  (31)//置1事件的相关标志位
                }
                OSTCBCurPtr->FlagsRdy = flags_rdy;  //保存让任务脱离等待的标志值
    if (p_ts != (CPU_TS *)0)                  //如果 p_ts 非空
                {
    *p_ts  = p_grp->TS;           //获取任务等到事件时的时间戳
                }
                CPU_CRITICAL_EXIT();               //开中断
                *p_err = OS_ERR_NONE;              //错误类型为“无错误”
    return (flags_rdy);                //返回0（有错误），停止执行
            }
    else
    //如果想要标志位的值与期望值不匹配
            {
    if ((opt & OS_OPT_PEND_NON_BLOCKING) != (OS_OPT)0) //如果选择了不阻塞任务
                {
                    CPU_CRITICAL_EXIT();                  //关中断
                    *p_err = OS_ERR_PEND_WOULD_BLOCK;    //错误类型为“渴求阻塞”
    return ((OS_FLAGS)0);   (32)//返回0（有错误），停止执行
                }
    else//如果选择了阻塞任务
                {
    if (OSSchedLockNestingCtr > (OS_NESTING_CTR)0)   //如果调度器被锁
                    {
                        CPU_CRITICAL_EXIT();           //关中断
                        *p_err = OS_ERR_SCHED_LOCKED;  //错误类型为“调度器被锁”
    return ((OS_FLAGS)0); (33)//返回0（有错误），停止执行
                    }
                }
    /* 如果调度器未被锁 */
                OS_CRITICAL_ENTER_CPU_EXIT();        //进入临界段，重开中断
                OS_FlagBlock(&pend_data,             //阻塞当前运行任务，等待事件
                            p_grp,
                            flags,
                            opt,
                            timeout);		(34)
                OS_CRITICAL_EXIT_NO_SCHED();        //退出临界段（无调度）
            }
    break;                                 //跳出

    case OS_OPT_PEND_FLAG_CLR_ANY:     (35)//如果要求有标志位被清零即可
            flags_rdy = (OS_FLAGS)(~p_grp->Flags & flags);//提取想要的标志位的值
    if (flags_rdy != (OS_FLAGS)0)                //如果有位被清零
            {
    if (consume == DEF_TRUE)           //如果要求将标志位匹配后取反
                {
                    p_grp->Flags |= flags_rdy; (36)//置1事件的相关标志位
                }
                OSTCBCurPtr->FlagsRdy = flags_rdy;   //保存让任务脱离等待的标志值
    if (p_ts != (CPU_TS *)0)                 //如果 p_ts 非空
                {
                    *p_ts  = p_grp->TS;             //获取任务等到事件时的时间戳
                }
                CPU_CRITICAL_EXIT();                     //开中断
                *p_err = OS_ERR_NONE;                     //错误类型为“无错误”
    return (flags_rdy);        (37)//返回0（有错误），停止执行
            }
    else//如果没有位被清零
            {
    if ((opt & OS_OPT_PEND_NON_BLOCKING) != (OS_OPT)0)   //如果没设置阻塞任务
                {
                    CPU_CRITICAL_EXIT();                 //开中断
                    *p_err = OS_ERR_PEND_WOULD_BLOCK;     //错误类型为“渴求阻塞”
    return ((OS_FLAGS)0);      (38)//返回0（有错误），停止执行
                }
    else//如果设置了阻塞任务
                {
    if (OSSchedLockNestingCtr > (OS_NESTING_CTR)0)   //如果调度器被锁
                    {
                        CPU_CRITICAL_EXIT();             //开中断
                        *p_err = OS_ERR_SCHED_LOCKED;   //错误类型为“调度器被锁”
    return ((OS_FLAGS)0);   (39)//返回0（有错误），停止执行
                    }
                }
    /* 如果调度器没被锁 */
                OS_CRITICAL_ENTER_CPU_EXIT();            //进入临界段，重开中断
                OS_FlagBlock(&pend_data,           //阻塞当前运行任务，等待事件
                            p_grp,
                            flags,
                            opt,
                            timeout);		(40)
                OS_CRITICAL_EXIT_NO_SCHED();             //退出中断（无调度）
            }
    break;                                       //跳出
    #endif

    default:                                 (41)//如果要求超出预期
            CPU_CRITICAL_EXIT();
            *p_err = OS_ERR_OPT_INVALID;                  //错误类型为“选项非法”
    return ((OS_FLAGS)0);                   //返回0（有错误），停止执行
        }

        OSSched();                             (42)//任务调度
    /* 任务等到了事件后得以继续运行 */
        CPU_CRITICAL_ENTER();                                 //关中断
    switch (OSTCBCurPtr->PendStatus)       (43)
    //根据运行任务的等待状态分类处理
        {
    case OS_STATUS_PEND_OK:              (44)//如果等到了事件
    if (p_ts != (CPU_TS *)0)                     //如果 p_ts 非空
            {
                *p_ts  = OSTCBCurPtr->TS;             //返回等到事件时的时间戳
            }
            *p_err = OS_ERR_NONE;                         //错误类型为“无错误”
    break;                                       //跳出

    case OS_STATUS_PEND_ABORT:           (45)//如果等待被中止
    if (p_ts != (CPU_TS *)0)                     //如果 p_ts 非空
            {
                *p_ts  = OSTCBCurPtr->TS;             //返回等待被中止时的时间戳
            }
            CPU_CRITICAL_EXIT();                         //开中断
            *p_err = OS_ERR_PEND_ABORT;                 //错误类型为“等待被中止”
    break;                                       //跳出

    case OS_STATUS_PEND_TIMEOUT:          (46)//如果等待超时
    if (p_ts != (CPU_TS *)0)                     //如果 p_ts 非空
            {
                *p_ts  = (CPU_TS  )0;                     //清零 p_ts
            }
            CPU_CRITICAL_EXIT();                         //开中断
            *p_err = OS_ERR_TIMEOUT;                      //错误类型为“超时”
    break;                                       //跳出

    case OS_STATUS_PEND_DEL:           (47)//如果等待对象被删除
    if (p_ts != (CPU_TS *)0)                     //如果 p_ts 非空
            {
                *p_ts  = OSTCBCurPtr->TS;              //返回对象被删时的时间戳
            }
            CPU_CRITICAL_EXIT();                         //开中断
            *p_err = OS_ERR_OBJ_DEL;                      //错误类型为“对象被删”
    break;                                       //跳出

    default:                             (48)//如果等待状态超出预期
            CPU_CRITICAL_EXIT();                         //开中断
            *p_err = OS_ERR_STATUS_INVALID;               //错误类型为“状态非法”
    break;                                       //跳出
        }
    if (*p_err != OS_ERR_NONE)           (49)//如果有错误存在
        {
    return ((OS_FLAGS)0);               //返回0（有错误），停止执行
        }
    /* 如果没有错误存在 */
        flags_rdy = OSTCBCurPtr->FlagsRdy;    (50)//读取让任务脱离等待的标志值
    if (consume == DEF_TRUE)
    //如果需要取反触发事件的标志位
        {
    switch (mode)                    (51)//根据事件触发模式分类处理
            {
    case OS_OPT_PEND_FLAG_SET_ALL:       //如果是通过置1来标志事件的发生
    case OS_OPT_PEND_FLAG_SET_ANY:
                p_grp->Flags &= ~flags_rdy;  (52)//清零事件里触发事件的标志位
    break;                                   //跳出

    #if OS_CFG_FLAG_MODE_CLR_EN > 0u//如果启用了标志位清零触发模式
    case OS_OPT_PEND_FLAG_CLR_ALL:       //如果是通过清零来标志事件的发生
    case OS_OPT_PEND_FLAG_CLR_ANY:
                p_grp->Flags |=  flags_rdy;  (53)//置1事件里触发事件的标志位
    break;                                   //跳出
    #endif
    default:                                      //如果触发模式超出预期
                CPU_CRITICAL_EXIT();                     //开中断
                *p_err = OS_ERR_OPT_INVALID;              //错误类型为“选项非法”
    return ((OS_FLAGS)0);         (54)//返回0（有错误），停止执行
            }
        }
        CPU_CRITICAL_EXIT();                      //开中断
        *p_err = OS_ERR_NONE;                     //错误类型为“无错误”
    return (flags_rdy);                  (55)//返回让任务脱离等待的标志值
    }


-   代码清单:事件-9_  **(1)**\ ：事件指针。

-   代码清单:事件-9_  **(2)**\ ：选定要等待的标志位。

-   代码清单:事件-9_  **(3)**\ ：等待不到事件时指定阻塞时间（单位：时钟节拍）。

-   代码清单:事件-9_  **(4)**\ ：等待的选项。

-   代码清单:事件-9_  **(5)**\ ：保存返回等到事件标志时的时间戳。

-   代码清单:事件-9_  **(6)**\ ：保存返回错误类型。

-   代码清单:事件-9_  **(7)**\ ：此处是判断一下等待的选项是否在预期内，如果在预期内则继续操作，跳出switch语句。

-   代码清单:事件-9_  **(8)**\ ：如果选项超出预期，返回错误类型为“选项非法”的错误代码，并且退出，不继续执行等待事件操作。

-   代码清单:事件-9_  **(9)**\ ：如果用户选择了标志位匹配后自动取反，变量consume就为DEF_TRUE。

-   代码清单:事件-9_  **(10)**\ ：如果未选择标志位匹配后自动取反，变量consume则为DEF_FALSE。

-   代码清单:事件-9_  **(11)**\ ：从选项中提取对标志位的要求，利用“&”运算操作符获取选项并且保存在mode变量中。

-   代码清单:事件-9_  **(12)**\ ：根据事件触发模式分类处理。

-   代码清单:事件-9_  **(13)**\ ：如果任务要求所有标志位均要置1，那么就提取想要的标志位的值保存在flags_rdy变量中。

-   代码清单:事件-9_  **(14)**\ ：如果该值与任务的期望值匹配。

-   代码清单:事件-9_  **(15)**\ ：如果要求将标志位匹配后取反，就将事件的相关标志位清零。

-   代码清单:事件-9_  **(16)**\ ：保存让任务脱离等待的标志值，此时已经等待到任务要求的事件了，就可以退出了。

-   代码清单:事件-9_  **(17)**\ ：返回错误类型为“无错误”的错误代码与让任务脱离等待的标志值。

-   代码清单:事件-9_  **(18)**\ ：如果想要标志位的值与期望值不匹配。

-   代码清单:事件-9_  **(19)**\ ：并且如果用户选择了不阻塞任务，那么返回错误类型为“渴求阻塞”的错误代码，退出。

-   代码清单:事件-9_  **(20)**\ ：而如果用户选择了阻塞任务。

-   代码清单:事件-9_  **(21)**\ ：那就判断一下调度器是否被锁，如果被锁了，返回错误类型为“调度器被锁”的错误代码，并且退出。

-   代码清单:事件-9_  **(22)**\ ：如果调度器没有被锁，则调用OS_FlagBlock()函数阻塞当前任务，在阻塞中继续等待任务需要的事件。

-   代码清单:事件-9_  **(23)**\ ：如果要求有标志位被置1即可，那就提取想要的标志位的值保存在flags_rdy变量中。

-   代码清单:事件-9_  **(24)**\ ：如果有任何一位被置1，就表示等待到了事件。如果要求将标志位匹配后取反，将事件的相关标志位清零。

-   代码清单:事件-9_  **(25)**\ ：等待成功，就返回让任务脱离等待的标志值。

-   代码清单:事件-9_  **(26)**\ ：如果没有位被置1，并且用户没有设置阻塞时间，那么就返回错误类型为“渴求阻塞”的错误代码，然后退出。

-   代码清单:事件-9_  **(27)**\ ：如果设置了阻塞任务，但是调度器被锁了，返回错误类型为“调度器被锁”的错误代码，并且退出。

-   代码清单:事件-9_  **(28)**\ ：如果调度器没被锁，
    则调用OS_FlagBlock()函数阻塞当前任务，在阻塞中继续等待任务需要的事件。

-   代码清单:事件-9_  **(29)**\ ：如果启用了标志位清零触发模式（宏定义OS_CFG_FLAG_MODE_CLR_EN被配置为1），
    则在编译的时候会包含事件清零触发相关代码。

-   代码清单:事件-9_  **(30)**\ ：如果要求所有标志位均要清零，
    首先提取想要的标志位的值保存在flags_rdy变量中，如果该值与任务的期望值匹配，那么就表示等待的事件。

-   代码清单:事件-9_  **(31)**\ ：如果要求将标志位匹配后取反，
    就置1事件的相关标志位，因为现在是清零触发的，事件标志位取反就是将对应标志位置一。

-   代码清单:事件-9_  **(32)**\ ：如果想要标志位的值与期望值不匹配，
    并且如果用户选择了不阻塞任务，那么返回错误类型为“渴求阻塞”的错误代码，退出。

-   代码清单:事件-9_  **(33)**\ ：如果调度器被锁，返回错误类型为“调度器被锁”的错误代码，并且退出。

-   代码清单:事件-9_  **(34)**\ ：如果调度器没有被锁，则调用OS_FlagBlock()函数阻塞当前任务，在阻塞中继续等待任务需要的事件。

-   代码清单:事件-9_  **(35)**\ ：如果要求有标志位被清零即可，提取想要的标志位的值，如果有位被清零则表示等待到事件。

-   代码清单:事件-9_  **(36)**\ ：如果要求将标志位匹配后取反，将事件的相关标志位置1。

-   代码清单:事件-9_  **(37)**\ ：等待到事件就返回对应的事件标志位。

-   代码清单:事件-9_  **(38)**\ ：如果没有位被清零，并且如果用户没设置阻塞任务，那么就返回错误类型为“渴求阻塞”的错误代码，然后退出。

-   代码清单:事件-9_  **(39)**\ ：如果设置了阻塞任务，但是调度器被锁了，返回错误类型为“调度器被锁”的错误代码，并且退出。

-   代码清单:事件-9_  **(40)**\ ：如果调度器没有被锁，则调用OS_FlagBlock()函数阻塞当前任务，在阻塞中继续等待任务需要的事件。

-   代码清单:事件-9_  **(41)**\ ：如果要求超出预期，返回错误类型为“选项非法”的错误代码，退出。

-   代码清单:事件-9_  **(42)**\ ：执行到这里，说明任务没有等待到事件，并且用户还选择了阻塞任务，那么就进行一次任务调度。

-   代码清单:事件-9_  **(43)**\ ：当程序能执行到这里，就说明大体上有两种情况，
    要么是任务获取到对应的事件了；要么任务还没获取到事件（任务没获取到事件的情况有很多种），无论是哪种情况，都先把中断关掉再说，再根据当前运行任务的等待状态分类处理。

-   代码清单:事件-9_  **(44)**\ ：如果等到了事件，返回等到事件时的时间戳，然后退出。

-   代码清单:事件-9_  **(45)**\ ：如果任务在等待事件中被中止，返回等待被中止时的时间戳，记录错误类型为“等待被中止”的错误代码，然后退出。

-   代码清单:事件-9_  **(46)**\ ：如果等待超时，返回错误类型为“等待超时”的错误代码，退出。

-   代码清单:事件-9_  **(47)**\ ：如果等待对象被删除，返回对象被删时的时间戳，记录错误类型为“对象被删”的错误代码，退出。

-   代码清单:事件-9_  **(48)**\ ：如果等待状态超出预期，记录错误类型为“状态非法”的错误代码，退出。

-   代码清单:事件-9_  **(49)**\ ：如果有错误存在，返回0，表示没有等待到事件。

-   代码清单:事件-9_  **(50)**\ ：如果没有错误存在，如果需要取反触发事件的标志位。

-   代码清单:事件-9_  **(51)**\ ：根据事件触发模式分类处理。

-   代码清单:事件-9_  **(52)**\ ：如果是通过置1来标志事件的发生，将事件里触发事件的标志位清零。

-   代码清单:事件-9_  **(53)**\ ：如果是通过清零来标志事件的发生，那么就将事件里触发事件的标志位置1。

-   代码清单:事件-9_  **(54)**\ ：如果触发模式超出预期，返回错误类型为“选项非法”的错误代码。

-   代码清单:事件-9_  **(55)**\ ：返回让任务脱离等待的标志值。

至此，任务等待事件函数就已经讲解完毕，其实μC/OS这种利用状态机的方法等待事件，根据不一样的情况进行处理，是很好的，省去很多逻辑的代码。

下面简单分析处理过程：当用户调用这个函数接口时，系统首先根据用户指定参数和接收选项来判断它要等待的事件是否发生，
如果已经发生，则根据等待选项来决定是否清除事件的相应标志位，并且返回事件标志位的值，但是这个值可能不是一个稳定的值，
所以在等待到对应事件的时候，我们最好要判断事件是否与任务需要的一致；如果事件没有发生，则把任务添加到事件等待列表中，
将当前任务阻塞，直到事件发生或等待时间超时，事件等待函数OSFlagPend()使用实例具体见 代码清单:事件-10_ 。

.. code-block:: c
    :caption: 代码清单:事件-10OSFlagPend()使用实例
    :name: 代码清单:事件-10
    :linenos:

    #define KEY1_EVENT  (0x01 << 0)//设置事件掩码的位0
    #define KEY2_EVENT  (0x01 << 1)//设置事件掩码的位1

    OS_FLAG_GRP flag_grp;                   //声明事件标志组

    static  void  AppTaskPend ( void * p_arg )
    {
        OS_ERR      err;


        (void)p_arg;

    //任务体
    while (DEF_TRUE)
        {
    //等待标志组的的BIT0和BIT1均被置1
            OSFlagPend ((OS_FLAG_GRP *)&flag_grp,
                        (OS_FLAGS     )( KEY1_EVENT | KEY2_EVENT ),
                        (OS_TICK      )0,
                        (OS_OPT       )OS_OPT_PEND_FLAG_SET_ALL |
                        OS_OPT_PEND_BLOCKING,
                        (CPU_TS      *)0,
                        (OS_ERR      *)&err);

            LED3_ON ();        //点亮LED3

    //等待标志组的的BIT0和BIT1有一个被清零
            OSFlagPend ((OS_FLAG_GRP *)&flag_grp,
                        (OS_FLAGS     )( KEY1_EVENT | KEY2_EVENT ),
                        (OS_TICK      )0,
                        (OS_OPT       )OS_OPT_PEND_FLAG_CLR_ANY |
                        OS_OPT_PEND_BLOCKING,
                        (CPU_TS      *)0,
                        (OS_ERR      *)&err);

            LED3_OFF ();          //熄灭LED3

        }

    }


事件实验
~~~~~~~~~~~~

事件标志组实验是在μC/OS中创建了两个任务，一个是设置事件任务，一个是等待事件任务，两个任务独立运行，
设置事件任务通过检测按键的按下情况设置不同的事件标志位，等待事件任务则获取这两个事件标志位，并且判断两个事件是否都发生，
如果是则输出相应信息，LED进行翻转。等待事件任务一直在等待事件的发生，等待到事件之后清除对应的事件标记位，具体见 代码清单:事件-11_ 。

.. code-block:: c
    :caption: 代码清单:事件-11事件实验
    :name: 代码清单:事件-11
    :linenos:

    #include <includes.h>

    OS_FLAG_GRP flag_grp;                   //声明事件标志组

    #define KEY1_EVENT  (0x01 << 0)//设置事件掩码的位0
    #define KEY2_EVENT  (0x01 << 1)//设置事件掩码的位1

    static  OS_TCB   AppTaskStartTCB;      //任务控制块
    static  OS_TCB   AppTaskPostTCB;
    static  OS_TCB   AppTaskPendTCB;

    static  CPU_STK  AppTaskStartStk[APP_TASK_START_STK_SIZE];       //任务栈
    static  CPU_STK  AppTaskPostStk [ APP_TASK_POST_STK_SIZE ];
    static  CPU_STK  AppTaskPendStk [ APP_TASK_PEND_STK_SIZE ];

    static  void  AppTaskStart  (void *p_arg);               //任务函数声明
    static  void  AppTaskPost   ( void * p_arg );
    static  void  AppTaskPend   ( void * p_arg );

    int  main (void)
    {
        OS_ERR  err;
        OSInit(&err);                       //初始化 μC/OS-III

        /* 创建起始任务 */
        OSTaskCreate((OS_TCB     *)&AppTaskStartTCB,
                    //任务控制块地址
                    (CPU_CHAR   *)"App Task Start",
                    //任务名称
                    (OS_TASK_PTR ) AppTaskStart,
                    //任务函数
                    (void       *) 0,
                    //传递给任务函数（形参p_arg）的实参
                    (OS_PRIO     ) APP_TASK_START_PRIO,
                    //任务的优先级
                    (CPU_STK    *)&AppTaskStartStk[0],
                    //任务栈的基地址
                    (CPU_STK_SIZE) APP_TASK_START_STK_SIZE / 10,
                    //任务栈空间剩下1/10时限制其增长
                    (CPU_STK_SIZE) APP_TASK_START_STK_SIZE,
                    //任务栈空间（单位：sizeof(CPU_STK)）
                    (OS_MSG_QTY  ) 5u,
                    //任务可接收的最大消息数
                    (OS_TICK     ) 0u,
                    //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）
                    (void       *) 0,
                    //任务扩展（0表不扩展）
                    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
                    //任务选项
                    (OS_ERR     *)&err);
                    //返回错误类型

        OSStart(&err);
        //启动多任务管理（交由μC/OS-III控制）
    }

    static  void  AppTaskStart (void *p_arg)
    {
        CPU_INT32U  cpu_clk_freq;
        CPU_INT32U  cnts;
        OS_ERR      err;

        (void)p_arg;

        BSP_Init();
        //板级初始化
        CPU_Init();
        //初始化 CPU组件（时间戳、关中断时间测量和主机名）

        cpu_clk_freq = BSP_CPU_ClkFreq();
        //获取 CPU内核时钟频率（SysTick 工作时钟）
        cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;
        //根据用户设定的时钟节拍频率计算 SysTick定时器的计数值
        OS_CPU_SysTickInit(cnts);
        //调用 SysTick初始化函数，设置定时器计数值和启动定时器

        Mem_Init();
        //初始化内存管理组件（堆内存池和内存池表）

    #if OS_CFG_STAT_TASK_EN > 0u
    //如果启用（默认启用）了统计任务
        OSStatTaskCPUUsageInit(&err);
    #endif


        CPU_IntDisMeasMaxCurReset();
        //复位（清零）当前最大关中断时间


        /* 创建事件标志组 flag_grp */
        OSFlagCreate ((OS_FLAG_GRP  *)&flag_grp,        //指向事件标志组的指针
                    (CPU_CHAR     *)"FLAG For Test",  //事件标志组的名字
                    (OS_FLAGS      )0,                //事件标志组的初始值
                    (OS_ERR       *)&err);            //返回错误类型


        /* 创建 AppTaskPost 任务 */
        OSTaskCreate((OS_TCB     *)&AppTaskPostTCB,
                    //任务控制块地址
                    (CPU_CHAR   *)"App Task Post",
                    //任务名称
                    (OS_TASK_PTR ) AppTaskPost,
                    //任务函数
                    (void       *) 0,
                    //传递给任务函数（形参p_arg）的实参
                    (OS_PRIO     ) APP_TASK_POST_PRIO,
                    //任务的优先级
                    (CPU_STK    *)&AppTaskPostStk[0],
                    //任务栈的基地址
                    (CPU_STK_SIZE) APP_TASK_POST_STK_SIZE / 10,
                    //任务栈空间剩下1/10时限制其增长
                    (CPU_STK_SIZE) APP_TASK_POST_STK_SIZE,
                    //任务栈空间（单位：sizeof(CPU_STK)）
                    (OS_MSG_QTY  ) 5u,
                    //任务可接收的最大消息数
                    (OS_TICK     ) 0u,
                    //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）
                    (void       *) 0,
                    //任务扩展（0表不扩展）
                    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
                    //任务选项
                    (OS_ERR     *)&err);
                    //返回错误类型

        /* 创建 AppTaskPend 任务 */
        OSTaskCreate((OS_TCB     *)&AppTaskPendTCB,
                    //任务控制块地址
                    (CPU_CHAR   *)"App Task Pend",
                    //任务名称
                    (OS_TASK_PTR ) AppTaskPend,
                    //任务函数
                    (void       *) 0,
                    //传递给任务函数（形参p_arg）的实参
                    (OS_PRIO     ) APP_TASK_PEND_PRIO,
                    //任务的优先级
                    (CPU_STK    *)&AppTaskPendStk[0],
                    //任务栈的基地址
                    (CPU_STK_SIZE) APP_TASK_PEND_STK_SIZE / 10,
                    //任务栈空间剩下1/10时限制其增长
                    (CPU_STK_SIZE) APP_TASK_PEND_STK_SIZE,
                    //任务栈空间（单位：sizeof(CPU_STK)）
                    (OS_MSG_QTY  ) 5u,
                    //任务可接收的最大消息数
                    (OS_TICK     ) 0u,
                    //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）
                    (void       *) 0,
                    //任务扩展（0表不扩展）
                    (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
                    //任务选项
                    (OS_ERR     *)&err);
                    //返回错误类型

        OSTaskDel ( & AppTaskStartTCB, & err );
        //删除起始任务本身，该任务不再运行
    }

    static  void  AppTaskPost ( void * p_arg )
    {
        OS_ERR      err;
        (void)p_arg;

        while (DEF_TRUE)                                    //任务体
        {
            if ( Key_ReadStatus ( macKEY1_GPIO_PORT, macKEY1_GPIO_PIN, 1 ) == 1 )
            //如果KEY1被按下
            {
                //点亮LED1
                printf("KEY1被按下\n");
                OSFlagPost ((OS_FLAG_GRP  *)&flag_grp,
                            //将标志组的BIT0置1
                            (OS_FLAGS      )KEY1_EVENT,
                            (OS_OPT        )OS_OPT_POST_FLAG_SET,
                            (OS_ERR       *)&err);

            }

            if ( Key_ReadStatus ( macKEY2_GPIO_PORT, macKEY2_GPIO_PIN, 1 ) == 1 )
            //如果KEY2被按下
            {
                //点亮LED2
                printf("KEY2被按下\n");
                OSFlagPost ((OS_FLAG_GRP  *)&flag_grp,
                            //将标志组的BIT1置1
                            (OS_FLAGS      )KEY2_EVENT,
                            (OS_OPT        )OS_OPT_POST_FLAG_SET,
                            (OS_ERR       *)&err);

            }

            OSTimeDlyHMSM ( 0, 0, 0, 20, OS_OPT_TIME_DLY, & err );

        }

    }



    static  void  AppTaskPend ( void * p_arg )
    {
        OS_ERR      err;
        OS_FLAGS      flags_rdy;

        (void)p_arg;
        while (DEF_TRUE)                                         //任务体
        {
            //等待标志组的的BIT0和BIT1均被置1
            flags_rdy =   OSFlagPend ((OS_FLAG_GRP *)&flag_grp,
                        (OS_FLAGS     )( KEY1_EVENT | KEY2_EVENT ),
                        (OS_TICK      )0,
                        (OS_OPT)OS_OPT_PEND_FLAG_SET_ALL |
                        OS_OPT_PEND_BLOCKING |
                        OS_OPT_PEND_FLAG_CONSUME,
                        (CPU_TS      *)0,
                        (OS_ERR      *)&err);
            if ((flags_rdy & (KEY1_EVENT|KEY2_EVENT)) == (KEY1_EVENT|KEY2_EVENT))
            {
                /* 如果接收完成并且正确 */
                printf ( "KEY1与KEY2都按下\n");
                macLED1_TOGGLE();       //LED1  翻转
            }
        }
    }


事件实验现象
~~~~~~~~~~~~~~~~~~

程序编译好，用USB线连接计算机和开发板的USB接口（对应丝印为USB转串口），
用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），
在计算机上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，按下开发板的KEY1按键发送事件1，
按下KEY2按键发送事件2；我们按下KEY1与KEY2试试，在串口调试助手中可以看到运行结果，并且当事件1与事件2都发生的时候，
开发板的LED会进行翻转，具体见图 事件标志组实验现象_ 。

.. image:: media/event/event006.png
   :align: center
   :name: 事件标志组实验现象
   :alt: 事件标志组实验现象


