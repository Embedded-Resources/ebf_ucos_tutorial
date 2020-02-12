.. vim: syntax=rst

任务信号量
============

任务信号量的基本概念
~~~~~~~~~~~~~~~~~~~~~~~~~

μC/OS提供任务信号量这个功能，每个任务都有一个32位（用户可以自定义位宽，我们使用32位的CPU，此处就是32位）的信号量值SemCtr，
这个信号量值是在任务控制块中包含的，是任务独有的一个信号量通知值，在大多数情况下，任务信号量可以替代内核对象的二值信号量、计数信号量等。

注：本章主要讲解任务信号量，而非内核对象信号量，如非特别说明，本章中的信号量都指的是内核对象信号量。前面所讲的信号量是单独的内核对象，
是独立于任务存在的；本章要讲述的任务信号量是任务特有的属性，紧紧依赖于一个特定任务。

相对于前面使用μC/OS内核通信的资源，必须创建二进制信号量、计数信号量等情况，使用任务信号量显然更灵活。
因为使用任务信号量比通过内核对象信号量通信方式解除阻塞的任务的速度要快，并且更加节省RAM内存空间，任务信号量的使用无需单独创建信号量。

通过对任务信号量的合理使用，可以在一定场合下替代μC/OS的信号量，用户只需向任务内部的信号量发送一个信号而不用通过外部的信号量进行发送，
这样子处理就会很方便并且更加高效，当然，凡事都有利弊，不然的话μC/OS还要内核的IPC通信机制干嘛，任务信号量虽然处理更快，RAM开销更小，
但也有限制：只能有一个任务接收任务信号量，因为必须指定接收信号量的任务，才能正确发送信号量；而内核对象的信号量则没有这个限制，
用户在释放信号量，可以采用广播的方式，让所有等待信号量的任务都获取到信号量。


在实际任务间的通信中，一个或多个任务发送一个信号量给另一个任务是非常常见的，而一个任务给多个任务发送信号量的情况相对比较少。
这种情况就很适合采用任务信号量进行传递信号，如果任务信号量可以满足设计需求，那么尽量不要使用普通信号量，这样子设计的系统会更加高效。

任务信号量的运作机制与普通信号量一样，没什么差别。

任务信号量的函数接口讲解
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

任务信号量释放函数OSTaskSemPost()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

函数 OSTaskSemPost()用来释放任务信号量，虽然只有拥有任务信号量的任务才可以等待该任务信号量，
但是其他所有的任务或者中断都可以向该任务释放信号量，其源码具体见 代码清单:任务信号量-1_ 。

.. code-block:: c
    :caption: 代码清单:任务信号量-1OSTaskSemPost()
    :name: 代码清单:任务信号量-1
    :linenos:

    OS_SEM_CTR  OSTaskSemPost (OS_TCB  *p_tcb,   	(1)	//目标任务
                            OS_OPT   opt,     	(2)	//选项
                            OS_ERR  *p_err)   	(3)	//返回错误类型
    {
        OS_SEM_CTR  ctr;
        CPU_TS      ts;



    #ifdef OS_SAFETY_CRITICAL//如果启用（默认禁用）了安全检测
    if (p_err == (OS_ERR *)0)           //如果 p_err 为空
        {
            OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数
    return ((OS_SEM_CTR)0);         //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_ARG_CHK_EN > 0u//如果启用（默认启用）了参数检测功能
    switch (opt)                            //根据选项分类处理
        {
    case OS_OPT_POST_NONE:              //如果选项在预期之内
    case OS_OPT_POST_NO_SCHED:
    break;                         //跳出

    default:                            //如果选项超出预期
            *p_err =  OS_ERR_OPT_INVALID;   //错误类型为“选项非法”
    return ((OS_SEM_CTR)0u);       //返回0（有错误），停止执行
        }
    #endif

        ts = OS_TS_GET();                                      //获取时间戳

    #if OS_CFG_ISR_POST_DEFERRED_EN > 0u//如果启用了中断延迟发布
    if (OSIntNestingCtr > (OS_NESTING_CTR)0)  //如果该函数是在中断中被调用
        {
            OS_IntQPost((OS_OBJ_TYPE)OS_OBJ_TYPE_TASK_SIGNAL,
    //将该信号量发布到中断消息队列
                        (void      *)p_tcb,
                        (void      *)0,
                        (OS_MSG_SIZE)0,
                        (OS_FLAGS   )0,
                        (OS_OPT     )0,
                        (CPU_TS     )ts,
                        (OS_ERR    *)p_err);	(4)
    return ((OS_SEM_CTR)0);                           //返回0（尚未发布）
        }
    #endif

        ctr = OS_TaskSemPost(p_tcb,                  //将信号量按照普通方式处理
                            opt,
                            ts,
                            p_err);		(5)

    return (ctr);                                 //返回信号的当前计数值
    }


-   代码清单:任务信号量-1_  **(1)**\ ：目标任务控制块指针，指向要释放任务信号量的任务。

-   代码清单:任务信号量-1_  **(2)**\ ：释放任务信号量的选项。

-   代码清单:任务信号量-1_  **(3)**\ ：用于返回保存错误代码。

-   代码清单:任务信号量-1_  **(4)**\ ：如果启用了中断延迟发布，
    并且该函数在中断中被调用，那就将信号量发布到中断消息队列，由中断消息队列发布任务信号量。

-   代码清单:任务信号量-1_  **(5)**\ ：调用OS_TaskSemPost ()函数将信号量发布到任务中，
    其源码具体见 代码清单:任务信号量-2_

.. code-block:: c
    :caption: 代码清单:任务信号量-2 OS_TaskSemPost()源码
    :name: 代码清单:任务信号量-2
    :linenos:

    OS_SEM_CTR  OS_TaskSemPost (OS_TCB  *p_tcb,   	(1)	//目标任务
                                OS_OPT   opt,     	(2)	//选项
                                CPU_TS   ts,      	(3)	//时间戳
                                OS_ERR  *p_err)   	(4)	//返回错误类型
    {
        OS_SEM_CTR  ctr;
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
    //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
    // SR（临界段关中断只需保存SR），开中断时将该值还原。

        OS_CRITICAL_ENTER();                               //进入临界段
    if (p_tcb == (OS_TCB *)0)             (5)//如果 p_tcb 为空
        {
            p_tcb = OSTCBCurPtr;                   //将任务信号量发给自己（任务）
        }
        p_tcb->TS = ts;                            //记录信号量被发布的时间戳
        *p_err     = OS_ERR_NONE;                           //错误类型为“无错误”
    switch (p_tcb->TaskState)              (6)
    //跟吴目标任务的任务状态分类处理
        {
    case OS_TASK_STATE_RDY:                        //如果目标任务没有等待状态
    case OS_TASK_STATE_DLY:
    case OS_TASK_STATE_SUSPENDED:
    case OS_TASK_STATE_DLY_SUSPENDED:		(7)
    switch (sizeof(OS_SEM_CTR))
            {						//判断是否将导致该信
    case 1u:                                     //号量计数值溢出，如
    if (p_tcb->SemCtr == DEF_INT_08U_MAX_VAL)   //果溢出，则开中断，
                {
                    OS_CRITICAL_EXIT();                     //返回错误类型为“计
                    *p_err = OS_ERR_SEM_OVF;                //数值溢出”，返回0
    return ((OS_SEM_CTR)0);                 //（有错误），不继续
                }                                           //执行。
    break;

    case 2u:
    if (p_tcb->SemCtr == DEF_INT_16U_MAX_VAL)
                {
                    OS_CRITICAL_EXIT();
                    *p_err = OS_ERR_SEM_OVF;
    return ((OS_SEM_CTR)0);
                }
    break;

    case 4u:
    if (p_tcb->SemCtr == DEF_INT_32U_MAX_VAL)
                {
                    OS_CRITICAL_EXIT();
                    *p_err = OS_ERR_SEM_OVF;
    return ((OS_SEM_CTR)0);
                }
    break;

    default:
    break;
            }
            p_tcb->SemCtr++;                (8)//信号量计数值不溢出则加1
            ctr = p_tcb->SemCtr;            (9)//获取信号量的当前计数值
            OS_CRITICAL_EXIT();                           //退出临界段
    break;                                        //跳出

    case OS_TASK_STATE_PEND:                           //如果任务有等待状态
    case OS_TASK_STATE_PEND_TIMEOUT:
    case OS_TASK_STATE_PEND_SUSPENDED:
    case OS_TASK_STATE_PEND_TIMEOUT_SUSPENDED:(10)
    if (p_tcb->PendOn == OS_TASK_PEND_ON_TASK_SEM)   	//如果正等待任务信号量
            {
                OS_Post((OS_PEND_OBJ *)0,                //发布信号量给目标任务
                        (OS_TCB      *)p_tcb,
                        (void        *)0,
                        (OS_MSG_SIZE  )0u,
                        (CPU_TS       )ts);		(11)
                ctr = p_tcb->SemCtr;                   //获取信号量的当前计数值
                OS_CRITICAL_EXIT_NO_SCHED();           //退出临界段（无调度）
    if ((opt & OS_OPT_POST_NO_SCHED) == (OS_OPT)0)   //如果选择了调度任务
                {
                    OSSched();                    (12)//调度任务
                }
            }
    else//如果没等待任务信号量
            {
    switch (sizeof(OS_SEM_CTR))       (13)//判断是否将导致
                {
    case 1u:                                  //该信号量计数值
    if (p_tcb->SemCtr == DEF_INT_08U_MAX_VAL)   //如果溢出，
                    {
                        OS_CRITICAL_EXIT();                  //则开中断，返回
                        *p_err = OS_ERR_SEM_OVF;             //错误类型为“计
    return ((OS_SEM_CTR)0);             //数值溢出”，返
                    }         //回0（有错误），
    break;    //不继续执行。

    case 2u:
    if (p_tcb->SemCtr == DEF_INT_16U_MAX_VAL)
                    {
                        OS_CRITICAL_EXIT();
                        *p_err = OS_ERR_SEM_OVF;
    return ((OS_SEM_CTR)0);
                    }
    break;

    case 4u:
    if (p_tcb->SemCtr == DEF_INT_32U_MAX_VAL)
                    {
                        OS_CRITICAL_EXIT();
                        *p_err = OS_ERR_SEM_OVF;
    return ((OS_SEM_CTR)0);
                    }
    break;

    default:
    break;
                }
                p_tcb->SemCtr++;                       //信号量计数值不溢出则加1
                ctr = p_tcb->SemCtr;                  //获取信号量的当前计数值
                OS_CRITICAL_EXIT();                   //退出临界段
            }
    break;                                    //跳出

    default:                          (14)//如果任务状态超出预期
            OS_CRITICAL_EXIT();                      //退出临界段
            *p_err = OS_ERR_STATE_INVALID;            //错误类型为“状态非法”
            ctr   = (OS_SEM_CTR)0;                 //清零 ctr
    break;                                   //跳出
        }
    return (ctr);  //返回信号量的当前计数值
    }


-   代码清单:任务信号量-2_  **(1)**\ ：目标任务。

-   代码清单:任务信号量-2_  **(2)**\ ：释放任务信号量选项

-   代码清单:任务信号量-2_  **(3)**\ ：时间戳。

-   代码清单:任务信号量-2_  **(4)**\ ：保存返回的错误类型代码。

-   代码清单:任务信号量-2_  **(5)**\ ：如果目标任务为空，则表示将任务信号量释放给自己，那么p_tcb就指向当前任务。

-   代码清单:任务信号量-2_  **(6)**\ ：根据目标任务的任务状态分类处理。

-   代码清单:任务信号量-2_  **(7)**\ ：如果目标任务没有等待状态，判断一下是否即将导致该信号量计数值溢出，
    如果溢出，则开中断，返回错误类型为“计数值溢出”的错误代码，退出不再继续执行。

-   代码清单:任务信号量-2_  **(8)**\ ：如果信号量还没溢出，信号量计数值加1。

-   代码清单:任务信号量-2_  **(9)**\ ：获取信号量的当前计数值，跳出switch语句。

-   代码清单:任务信号量-2_  **(10)**\ ：如果任务有等待状态，并且如果正等待任务信号量。

-   代码清单:任务信号量-2_  **(11)**\ ：调用OS_Post()函数发布信号量给目标任务，该函数在前面章节有讲解。

-   代码清单:任务信号量-2_  **(12)**\ ：如果选择了调度任务，就进行一次任务调度。

-   代码清单:任务信号量-2_  **(13)**\ ：如果不是等待任务信号量，判断一下是否即将导致该信号量计数值溢出，
    如果溢出，则开中断，返回错误类型为“计数值溢出”的错误代码，退出不再继续执行，如果信号量还没溢出，信号量计数值加1。

-   代码清单:任务信号量-2_  **(14)**\ ：如果任务状态超出预期，返回错误类型为“状态非法”的错误代码。

在释放任务信号量的时候，系统首先判断目标任务的状态，
只有处于等待状态并且等待的是任务信号量那就调用OS_Post()函数让等待的任务就绪（如果内核对象信号量的话，还会让任务脱离等待列表），
所以任务信号量的操作是非常高效的；如果没有处于等待状态或者等待的不是任务信号量，那就直接将任务控制块的元素SemCtr 加 1。
最后返回任务信号量计数值。

其实，不管是否启用了中断延迟发布，最终都是调用 OS_TaskSemPost()函数进行释放任务信号量。只是启用了中断延迟发布的释放过程会比较曲折，
中间会有许多插曲，这是中断管理范畴的内容，留到后面再作介绍。在 OS_TaskSemPost()函数中，又会调用OS_Post()函数释放内核对象。
OS_Post()函数是一个底层的释放（发布）函数，它不仅仅用来释放（发布）任务信号量，还可以释放信号量、互斥信号量、消息队列、
事件标志组或任务消息队列。注意：在这里，OS_Post()函数将任务信号量直接释放给目标任务。

释放任务互斥量函数的使用实例具体见 代码清单:任务信号量-3_ 。

.. code-block:: c
    :caption: 代码清单:任务信号量-3OSTaskSemPost()使用实例
    :name: 代码清单:任务信号量-3
    :linenos:

    OSTaskSemPost((OS_TCB  *)&AppTaskPendTCB,          //目标任务
                (OS_OPT   )OS_OPT_POST_NONE,        //没选项要求
                (OS_ERR  *)&err);                  //返回错误类型


获取任务信号量函数OSTaskSemPend()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

与 OSTaskSemPost()任务信号量释放函数相对应，OSTaskSemPend()函数用于获取一个任务信号量，
参数中没有指定某个任务去获取信号量，实际上就是当前运行任务获取它自己拥有的任务信号量，
OSTaskSemPend()源码具体见 代码清单:任务信号量-4_ 。

.. code-block:: c
    :caption: 代码清单:任务信号量-4OSTaskSemPend()源码
    :name: 代码清单:任务信号量-4
    :linenos:

    OS_SEM_CTR  OSTaskSemPend (OS_TICK   timeout, 	(1)	//等待超时时间
                            OS_OPT    opt,      	(2)	//选项
                            CPU_TS   *p_ts,     	(3)	//返回时间戳
                            OS_ERR   *p_err)    	(4)	//返回错误类型
    {
        OS_SEM_CTR    ctr;
        CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和
    //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器
    // SR（临界段关中断只需保存SR），开中断时将该值还原。

    #ifdef OS_SAFETY_CRITICAL//如果启用了安全检测
    if (p_err == (OS_ERR *)0)            //如果错误类型实参为空
        {
            OS_SAFETY_CRITICAL_EXCEPTION();  //执行安全检测异常函数
    return ((OS_SEM_CTR)0);          //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_CALLED_FROM_ISR_CHK_EN > 0u//如果启用了中断中非法调用检测
    if (OSIntNestingCtr > (OS_NESTING_CTR)0)    //如果该函数在中断中被调用
        {
            *p_err = OS_ERR_PEND_ISR;                //返回错误类型为“在中断中等待”
    return ((OS_SEM_CTR)0);                 //返回0（有错误），停止执行
        }
    #endif

    #if OS_CFG_ARG_CHK_EN > 0u//如果启用了参数检测
    switch (opt)                            //根据选项分类处理
        {
    case OS_OPT_PEND_BLOCKING:          //如果选项在预期内
    case OS_OPT_PEND_NON_BLOCKING:
    break;                         //直接跳出

    default:                            //如果选项超出预期
            *p_err = OS_ERR_OPT_INVALID;    //错误类型为“选项非法”
    return ((OS_SEM_CTR)0);        //返回0（有错误），停止执行
        }
    #endif

    if (p_ts != (CPU_TS *)0)        //如果 p_ts 非空
        {
            *p_ts  = (CPU_TS  )0;        //清零（初始化）p_ts
        }

        CPU_CRITICAL_ENTER();                        //关中断
    if (OSTCBCurPtr->SemCtr > (OS_SEM_CTR)0)     //如果任务信号量当前可用
        {
            OSTCBCurPtr->SemCtr--;         (5)//信号量计数器减1
            ctr    = OSTCBCurPtr->SemCtr;    (6)//获取信号量的当前计数值
    if (p_ts != (CPU_TS *)0)                 //如果 p_ts 非空
            {
                *p_ts  = OSTCBCurPtr->TS;       (7)//返回信号量被发布的时间戳
            }
    #if OS_CFG_TASK_PROFILE_EN > 0u	(8)
            OSTCBCurPtr->SemPendTime = OS_TS_GET() - OSTCBCurPtr->TS;     //更新任务等待
    if (OSTCBCurPtr->SemPendTimeMax < OSTCBCurPtr->SemPendTime)   //任务信号量的
            {
                OSTCBCurPtr->SemPendTimeMax = OSTCBCurPtr->SemPendTime;   //最长时间记录。
            }//如果启用任务统计的宏，计算任务信号量从被提交到获取所用时间及最大时间
    #endif
            CPU_CRITICAL_EXIT();                     //开中断
            *p_err = OS_ERR_NONE;                     //错误类型为“无错误”
    return (ctr);                            //返回信号量的当前计数值
        }
    /* 如果任务信号量当前不可用 */			(9)
    if ((opt & OS_OPT_PEND_NON_BLOCKING) != (OS_OPT)0) //如果选择了不阻塞任务
        {
            CPU_CRITICAL_EXIT();                              //开中断
            *p_err = OS_ERR_PEND_WOULD_BLOCK;        //错误类型为“缺乏阻塞”
    return ((OS_SEM_CTR)0);                  //返回0（有错误），停止执行
        }
    else(10)//如果选择了阻塞任务
        {
    if (OSSchedLockNestingCtr > (OS_NESTING_CTR)0)    //如果调度器被锁
            {
                CPU_CRITICAL_EXIT();                          //开中断
                *p_err = OS_ERR_SCHED_LOCKED;//错误类型为“调度器被锁”
    return ((OS_SEM_CTR)0);              //返回0（有错误），停止执行
            }
        }
    /* 如果调度器未被锁 */
        OS_CRITICAL_ENTER_CPU_EXIT();                      //锁调度器，重开中断
        OS_Pend((OS_PEND_DATA *)0,                        //阻塞任务，等待信号量。
                (OS_PEND_OBJ  *)0,                            //不需插入等待列表。
                (OS_STATE      )OS_TASK_PEND_ON_TASK_SEM,
                (OS_TICK       )timeout);		(11)
        OS_CRITICAL_EXIT_NO_SCHED();                          //开调度器（无调度）

        OSSched();                          	(12)//调度任务
    /* 任务获得信号量后得以继续运行 */
        CPU_CRITICAL_ENTER();                  	(13)//关中断
    switch (OSTCBCurPtr->PendStatus)             //根据任务的等待状态分类处理
        {
    case OS_STATUS_PEND_OK:              (14)//如果任务成功获得信号量
    if (p_ts != (CPU_TS *)0)              //返回信号量被发布的时间戳
            {
                *p_ts                    =  OSTCBCurPtr->TS;
    #if OS_CFG_TASK_PROFILE_EN > 0u//更新最长等待时间记录
                OSTCBCurPtr->SemPendTime = OS_TS_GET() - OSTCBCurPtr->TS;
    if (OSTCBCurPtr->SemPendTimeMax < OSTCBCurPtr->SemPendTime)
                {
                    OSTCBCurPtr->SemPendTimeMax = OSTCBCurPtr->SemPendTime;
                }
    #endif
            }
            *p_err = OS_ERR_NONE;                         //错误类型为“无错误”
    break;                                       //跳出

    case OS_STATUS_PEND_ABORT:             (15)//如果等待被中止
    if (p_ts != (CPU_TS *)0)                     //返回被终止时的时间戳
            {
                *p_ts  =  OSTCBCurPtr->TS;
            }
            *p_err = OS_ERR_PEND_ABORT;              //错误类型为“等待被中止”
    break;                                  //跳出

    case OS_STATUS_PEND_TIMEOUT:         (16)//如果等待超时
    if (p_ts != (CPU_TS *)0)                     //返回时间戳为0
            {
                *p_ts  = (CPU_TS  )0;
            }
            *p_err = OS_ERR_TIMEOUT;                      //错误类型为“等待超时”
    break;                                       //跳出

    default:                               (17)//如果等待状态超出预期
            *p_err = OS_ERR_STATUS_INVALID;               //错误类型为“状态非法”
    break;                                       //跳出
        }
        ctr = OSTCBCurPtr->SemCtr;                    //获取信号量的当前计数值
        CPU_CRITICAL_EXIT();                           //开中断
    return (ctr);   (18)//返回信号量的当前计数值
    }


-   代码清单:任务信号量-4_  **(1)**\ ：等待超时时间。

-   代码清单:任务信号量-4_  **(2)**\ ：等待的选项。

-   代码清单:任务信号量-4_  **(3)**\ ：保存返回的时间戳。

-   代码清单:任务信号量-4_  **(4)**\ ：保存返回错误的类型。

-   代码清单:任务信号量-4_  **(5)**\ ：如果任务信号量当前可用，那就信号量计数值SemCtr减一。

-   代码清单:任务信号量-4_  **(6)**\ ：获取信号量的当前计数值保存在ctr变量中，用于返回。

-   代码清单:任务信号量-4_  **(7)**\ ：返回信号量被发布的时间戳。

-   代码清单:任务信号量-4_  **(8)**\ ：如果启用任务统计的宏，计算任务信号量从被释放到获取所用时间及最大时间。

-   代码清单:任务信号量-4_  **(9)**\ ：如果任务信号量当前不可用，并且如果用户选择了不阻塞任务，那么就返回错误类型为“缺乏阻塞”错误代码。

-   代码清单:任务信号量-4_  **(10)**\ ：如果选择了阻塞任务，判断一下调度器是否被锁，如果被锁，则返回错误类型为“调度器被锁”的错误代码。

-   代码清单:任务信号量-4_  **(11)**\ ：如果调度器未被锁，锁调度器，
    重开中断，调用OS_Pend()函数将当前任务进入阻塞状态以等待任务信号量，该函数在前面的章节已经讲解过，
    此处就不再重复赘述。

-   代码清单:任务信号量-4_  **(12)**\ ：进行一次任务调度。

-   代码清单:任务信号量-4_  **(13)**\ ：当程序能执行到这里，
    就说明大体上有两种情况，要么是任务获取到任务信号量了；要么任务还没获取到任务信号量（任务没获取到任务信号量的情况有很多种），
    无论是哪种情况，都先把中断关掉再说，再根据当前运行任务的等待状态分类处理。

-   代码清单:任务信号量-4_  **(14)**\ ：如果任务成功获得任务信号量，返回信号量被发布的时间戳，然后跳出switch语句。

-   代码清单:任务信号量-4_  **(15)**\ ：如果任务在等待中被中止，
    返回被终止时的时间戳，返回错误类型为“等待被中止”的错误代码，跳出switch语句。

-   代码清单:任务信号量-4_  **(16)**\ ：如果任务等待超时，返回错误类型为“等待超时”的错误代码，跳出switch语句。

-   代码清单:任务信号量-4_  **(17)**\ ：如果等待状态超出预期，返回错误类型为“状态非法”的错误代码。

-   代码清单:任务信号量-4_  **(18)**\ ：获取并返回任务信号量的当前计数值。

在调用该函数的时候，系统先判断任务信号量是否可用，即检查任务信号量的计数值是否大于 0，如果大于0，即表示可用，
这个时候获取信号量，即将计数值减 1 后直接返回。如果信号量不可用，且当调度器没有被锁住时，
用户希望在任务信号量不可用的时候进行阻塞任务以等待任务信号量可用，那么系统就会调用OS_Pend()函数将任务脱离就绪列表，
如果用户有指定超时时间，系统还要将该任务插入节拍列表。注意：此处系统并没有将任务插入等待列表。然后切换任务，
处于就绪列表中最高优先级的任务通过任务调度获得 CPU使用权，等到出现任务信号量被释放、任务等待任务信号量被强制停止、
等待超时等情况，任务会从阻塞中恢复，等待任务信号量的任务重新获得 CPU 使用权，返回相关错误代码和任务信号量计数值，
用户可以根据返回的错误知道任务退出等待状态的情况。

获取任务信号量函数的使用实例具体见 代码清单:任务信号量-5_

.. code-block:: c
    :caption: 代码清单:任务信号量-5 OSTaskSemPend()
    :name: 代码清单:任务信号量-5
    :linenos:

    OSTaskSemPend ((OS_TICK   )0,                     //无期限等待
                (OS_OPT    )OS_OPT_PEND_BLOCKING,  //如果信号量不可用就等待
                (CPU_TS   *)&ts,                   //获取信号量被发布的时间戳
                (OS_ERR   *)&err);                 //返回错误类型


任务信号量实验
~~~~~~~~~~~~~~~~~

任务信号量代替二值信号量
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

任务通知代替消息队列是在ΜC/OS中创建了两个任务，其中一个任务是用于接收任务信号量，另一个任务发送任务信号量。
两个任务独立运行，发送任务信号量的任务是通过检测按键的按下情况发送，等待任务在任务信号量中没有可用的信号量之前就一直等待，
获取到信号量以后就继续执行，这样子是为了代替二值信号量，任务同步成功则继续执行，
然后在串口调试助手里将运行信息打印出来，具体见 代码清单:任务信号量-6_ 加粗部分。

.. code-block:: c
    :caption: 代码清单:任务信号量-6任务通知代替二值信号量
    :name: 代码清单:任务信号量-6
    :linenos:

    #include <includes.h>

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


        OSInit(&err);       //初始化μC/OS-III

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
                    (OS_OPT      )(OS_OPT_TASK_STK_CHK |  OS_OPT_TASK_STK_CLR),
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

    //板级初始化
        BSP_Init();

    //初始化 CPU 组件（时间戳、关中断时间测量和主机名）
        CPU_Init();

    //获取 CPU 内核时钟频率（SysTick 工作时钟）
        cpu_clk_freq = BSP_CPU_ClkFreq();
    //根据用户设定的时钟节拍频率计算 SysTick 定时器的计数值
        cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;
    //调用 SysTick 初始化函数，设置定时器计数值和启动定时器
        OS_CPU_SysTickInit(cnts);

        Mem_Init();
    //初始化内存管理组件（堆内存池和内存池表）

    #if OS_CFG_STAT_TASK_EN > 0u
    //如果启用（默认启用）了统计任务
        OSStatTaskCPUUsageInit(&err);
    #endif

        CPU_IntDisMeasMaxCurReset();
    //复位（清零）当前最大关中断时间


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

    uint8_t ucKey1Press = 0;     //记忆按键KEY1状态


        (void)p_arg;


    while (DEF_TRUE)
    //任务体
        {
    if ( Key_Scan ( macKEY1_GPIO_PORT, macKEY1_GPIO_PIN, 1, & ucKey1Press ) )
    //如果KEY1被按下
            {
                printf("发送任务信号量\n");
    /* 发布任务信号量 */
                OSTaskSemPost((OS_TCB  *)&AppTaskPendTCB,
    //目标任务
                            (OS_OPT   )OS_OPT_POST_NONE,
    //没选项要求
                            (OS_ERR  *)&err);
    //返回错误类型


            }

            OSTimeDlyHMSM ( 0, 0, 0, 20, OS_OPT_TIME_DLY, & err );
    //每20ms扫描一次

        }

    }

    static  void  AppTaskPend ( void * p_arg )
    {
        OS_ERR         err;
        CPU_TS         ts;
        CPU_INT32U     cpu_clk_freq;
        CPU_SR_ALLOC();

        (void)p_arg;
        cpu_clk_freq = BSP_CPU_ClkFreq();
    //获取CPU时钟，时间戳是以该时钟计数


    while (DEF_TRUE)                                    //任务体
        {
    /* 阻塞任务，直到KEY1被按下 */
            OSTaskSemPend ((OS_TICK   )0,                     //无期限等待
                            (OS_OPT    )OS_OPT_PEND_BLOCKING,
    //如果信号量不可用就等待
                            (CPU_TS   *)&ts,
    //获取信号量被发布的时间戳
                            (OS_ERR   *)&err);                 //返回错误类型

            ts = OS_TS_GET() - ts;
    //计算信号量从发布到接收的时间差

            macLED1_TOGGLE ();                     //切换LED1的亮灭状态

            OS_CRITICAL_ENTER();
    //进入临界段，避免串口打印被打断

            printf ( "任务信号量从被发送到被接收的时间差是%dus\n\n",
                    ts / ( cpu_clk_freq / 1000000 ) );

            OS_CRITICAL_EXIT();                               //退出临界段

        }

    }


任务信号量代替计数信号量
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

任务通知代替计数信号量是基于计数信号量实验修改而来，模拟停车场工作运行。并且在μC/OS中创建了两个任务：
一个是获取信号量任务，一个是发送信号量任务，两个任务独立运行，获取任务信号量的任务是通过按下KEY1按键获取，
模拟停车场停车操作，其等待时间是0；发送任务信号量的任务则是通过检测KEY2按键按下进行信号量的发送（发送到获取任务），
模拟停车场取车操作，并且在串口调试助手输出相应信息，实验源码具体见 代码清单:任务信号量-7_ 。

.. code-block:: c
    :caption: 代码清单:任务信号量-7任务通知代替计数信号量
    :name: 代码清单:任务信号量-7
    :linenos:

    #include <includes.h>
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
        OSInit(&err);                                  //初始化μC/OS-III

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

        BSP_Init();                                    //板级初始化
        CPU_Init();      //初始化 CPU组件（时间戳、关中断时间测量和主机名）

        cpu_clk_freq = BSP_CPU_ClkFreq();
        //获取 CPU内核时钟频率（SysTick 工作时钟）
        cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;
    //根据用户设定的时钟节拍频率计算 SysTick定时器的计数值
        //调用 SysTick初始化函数，设置定时器计数值和启动定时器
        OS_CPU_SysTickInit(cnts);

        Mem_Init();
    //初始化内存管理组件（堆内存池和内存池表）

    #if OS_CFG_STAT_TASK_EN > 0u
    //如果启用（默认启用）了统计任务
        OSStatTaskCPUUsageInit(&err);
    #endif
    //复位（清零）当前最大关中断时间
        CPU_IntDisMeasMaxCurReset();

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

        OS_SEM_CTR  ctr;

    uint8_t ucKey2Press = 0;     //记忆按键KEY2状态

        CPU_SR_ALLOC();

        (void)p_arg;


    while (DEF_TRUE)
    //任务体
        {
    if ( Key_Scan ( macKEY2_GPIO_PORT, macKEY2_GPIO_PIN, 1, & ucKey2Press ) )
    //如果KEY2被按下
            {

    /* 发布任务信号量 */
                ctr = OSTaskSemPost((OS_TCB  *)&AppTaskPendTCB,
    //目标任务
                                    (OS_OPT   )OS_OPT_POST_NONE,
    //没选项要求
                                    (OS_ERR  *)&err);
    //返回错误类型

                macLED2_TOGGLE();
                OS_CRITICAL_ENTER();
    //进入临界段，避免串口打印被打断

                printf( "KEY2被按下，释放1个停车位，当前车位为 %d 个\n",ctr);


                OS_CRITICAL_EXIT();                               //退出临界段

            }

            OSTimeDlyHMSM ( 0, 0, 0, 20, OS_OPT_TIME_DLY, & err );
    //每20ms扫描一次

        }

    }


    static  void  AppTaskPend ( void * p_arg )
    {
        OS_ERR         err;

        CPU_SR_ALLOC();

        OS_SEM_CTR    ctr;//当前任务信号量计数

    uint8_t ucKey1Press = 0;     //记忆按键KEY1状态

        (void)p_arg;

    while (DEF_TRUE)                                    //任务体
        {

    if ( Key_Scan ( macKEY1_GPIO_PORT, macKEY1_GPIO_PIN, 1, & ucKey1Press ) )
    //如果KEY2被按下
            {
                ctr = OSTaskSemPend ((OS_TICK   )0,               //不等待
                                    (OS_OPT    )OS_OPT_PEND_NON_BLOCKING,
                                    (CPU_TS   *)0,
    //获取信号量被发布的时间戳
            (OS_ERR   *)&err);      //返回错误类型

                macLED1_TOGGLE ();
    //切换LED1的亮灭状态

                OS_CRITICAL_ENTER();
    //进入临界段，避免串口打印被打断

    if (OS_ERR_NONE == err)
                    printf( "KEY1被按下，申请车位成功，当前剩余车位为 %d
    个\n", ctr);
    else
                    printf("申请车位失败，请按KEY2释放车位\n");

                OS_CRITICAL_EXIT();                               //退出临界段
            }

            OSTimeDlyHMSM ( 0, 0, 0, 20, OS_OPT_TIME_DLY, & err );
        }

    }


任务信号量实验现象
~~~~~~~~~~~~~~~~~~~~~~~~


任务信号量代替二值信号量
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

将程序编译好，用USB线连接计算机和开发板的USB接口（对应丝印为USB转串口），
用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），
在计算机上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，它里面输出了信息表明任务正在运行中，
我们按下开发板的按键，串口打印任务运行的信息，表明两个任务同步成功，具体见图 任务信号量代替二值信号量实验现象_ 。

.. image:: media/Task_semaphore/Taskse003.png
   :align: center
   :name: 任务信号量代替二值信号量实验现象
   :alt: 任务信号量代替二值信号量实验现象



任务信号量代替计数信号量
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

将程序编译好，用USB线连接计算机和开发板的USB接口（对应丝印为USB转串口），
用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），
在计算机上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，
按下开发板的KEY1按键获取信号量模拟停车，按下KEY2按键释放信号量模拟取车，因为是使用任务信号量代替信号量，
所以任务通信号量默认为0，表当前车位为0；我们按下KEY1与KEY2试试，在串口调试助手中可以看到运行信息，
具体见图 任务信号量代替计数信号量实验现象_ 。

.. image:: media/Task_semaphore/Taskse003.png
   :align: center
   :name: 任务信号量代替计数信号量实验现象
   :alt: 任务信号量代替计数信号量实验现象

