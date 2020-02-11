.. vim: syntax=rst

任务的删除
============

本章开始，我们让OS的任务支持删除操作，一个任务被删除后就进入休眠态，要想继续运行必须创新创建。

实现任务删除
~~~~~~~~~~~~~~~~~~

编写任务删除函数
^^^^^^^^^^^^^^^^^^^^^^^^

OSTaskDel()函数
'''''''''''''''''''''''''

任务删除函数OSTaskDel()用于删除一个指定的任务，也可以删除自身，在os_task.c中定义，具体实现见 代码清单:任务的删除_。

.. code-block:: c
    :caption: 代码清单:任务的删除OSTaskDel()函数
    :name: 代码清单:任务的删除
    :linenos:

    #if OS_CFG_TASK_DEL_EN > 0u(1)
    void  OSTaskDel (OS_TCB  *p_tcb,
                    OS_ERR  *p_err)
    {
        CPU_SR_ALLOC();

    /* 不允许删除空闲任务 */(2)
    if (p_tcb == &OSIdleTaskTCB) {
            *p_err = OS_ERR_TASK_DEL_IDLE;
    return;
        }

    /* 删除自己 */
    if (p_tcb == (OS_TCB *)0) {(3)
            CPU_CRITICAL_ENTER();
            p_tcb  = OSTCBCurPtr;
            CPU_CRITICAL_EXIT();
        }

        OS_CRITICAL_ENTER();

    /* 根据任务的状态来决定删除的动作 */
    switch (p_tcb->TaskState) {
    case OS_TASK_STATE_RDY:(4)
            OS_RdyListRemove(p_tcb);
    break;

    case OS_TASK_STATE_SUSPENDED:(5)
    break;

    /* 任务只是在延时，并没有在任何等待列表*/
    case OS_TASK_STATE_DLY:(6)
    case OS_TASK_STATE_DLY_SUSPENDED:
            OS_TickListRemove(p_tcb);
    break;

    case OS_TASK_STATE_PEND:(7)
    case OS_TASK_STATE_PEND_SUSPENDED:
    case OS_TASK_STATE_PEND_TIMEOUT:
    case OS_TASK_STATE_PEND_TIMEOUT_SUSPENDED:
            OS_TickListRemove(p_tcb);

    #if 0/* 目前我们还没有实现等待列表，暂时先把这部分代码注释 */
    /* 看看在等待什么 */
    switch (p_tcb->PendOn) {
    case OS_TASK_PEND_ON_NOTHING:
    /* 任务信号量和队列没有等待队列，直接退出 */
    case OS_TASK_PEND_ON_TASK_Q:
    case OS_TASK_PEND_ON_TASK_SEM:
    break;

    /* 从等待列表移除 */
    case OS_TASK_PEND_ON_FLAG:
    case OS_TASK_PEND_ON_MULTI:
    case OS_TASK_PEND_ON_MUTEX:
    case OS_TASK_PEND_ON_Q:
    case OS_TASK_PEND_ON_SEM:
                OS_PendListRemove(p_tcb);
    break;

    default:
    break;
            }
    break;
    #endif
    default:
            OS_CRITICAL_EXIT();
            *p_err = OS_ERR_STATE_INVALID;
    return;
        }

    /* 初始化TCB为默认值 */
        OS_TaskInitTCB(p_tcb);(8)
    /* 修改任务的状态为删除态，即处于休眠 */
        p_tcb->TaskState = (OS_STATE)OS_TASK_STATE_DEL;(9)

        OS_CRITICAL_EXIT_NO_SCHED();
    /* 任务切换，寻找最高优先级的任务 */
        OSSched();(10)

        *p_err = OS_ERR_NONE;
    }
    #endif/* OS_CFG_TASK_DEL_EN > 0u */


-   代码清单:任务的删除_ （1）：任务删除是一个可选功能，由OS_CFG_TASK_DEL_EN控制，该宏在os_cfg.h中定义。

-   代码清单:任务的删除_ （2）：空闲任务不能被删除。系统必须至少有一个任务在运行，当没有其他用户任务运行的时候，系统就会运行空闲任务。

-   代码清单:任务的删除_ （3）：删除自己。

-   代码清单:任务的删除_ （4）：任务只在就绪态，则从就绪列表移除。

-   代码清单:任务的删除_ （5）：任务只是被挂起，则退出返回，不用做什么。

-   代码清单:任务的删除_ （6）：任务在延时或者是延时加挂起，则从时基列表移除。

-   代码清单:任务的删除_ （7）：任务在多种状态，但只要有一种是等待状态，就需要从等待列表移除。如果任务等待是任务自身的信号量和消息，
    则直接退出返回，因为任务信号量和消息是没有等待列表的。等待列表我们暂时还没实现，所以暂时将等待部分相关的代码用条件编译屏蔽掉。

-   代码清单:任务的删除_ （8）：初始化TCB为默认值。

-   代码清单:任务的删除_ （9）：修改任务的状态为删除态，即处于休眠。

-   代码清单:任务的删除_ （10）：任务调度，寻找优先级最高的任务来运行。

main()函数
~~~~~~~~~~~~~~~~~~~~~~~~

本章main()函数没有添加新的测试代码，只需理解章节内容即可。

实验现象
~~~~~~~~~~~~

本章没有实验，只需理解章节内容即可。
