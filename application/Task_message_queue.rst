.. vim: syntax=rst

任务消息队列
===============

任务消息队列的基本概念
~~~~~~~~~~~

任务消息队列跟任务信号量一样，均隶属于某一个特定任务，不需单独创建，任务在则任务消息队列在，只有该任务才可以获取（接收）这个任务消息队列的消息，其他任务只能给这个任务消息队列发送消息，却不能获取。任务消息队列与前面讲解的（普通）消息队列极其相似，只是任务消息队列已隶属于一个特定任务，所以它不具有等待
列表，在操作的过程中省去了等待任务插入和移除列表的动作，所以工作原理相对更简单一点，效率也比较高一些。

注意：本书所提的“消息队列”，若无特别说明，均指前面的（普通）消息队列（属于内核对象），而非任务消息队列。

通过对任务消息队列的合理使用，可以在一定场合下替代μC/OS的消息队列，用户只需向任务内部的消息队列发送一个消息而不用通过外部的消息队列进行发送，这样子处理就会很方便并且更加高效，当然，凡事都有利弊，任务消息队列虽然处理更快，RAM开销更小，但也有限制：只能指定消息发送的对象，有且只有一个任务接收消
息；而内核对象的消息队列则没有这个限制，用户在发送消息的时候，可以采用广播消息的方式，让所有等待该消息的任务都获取到消息。

在实际任务间的通信中，一个或多个任务发送一个消息给另一个任务是非常常见的，而一个任务给多个任务发送消息的情况相对比较少，前者就很适合采用任务消息队列进行传递消息，如果任务消息队列可以满足设计需求，那么尽量不要使用普通消息队列，这样子设计的系统会更加高效。

（内核对象）消息队列是用结构体OS_Q来管理的，包含了管理消息的元素 MsgQ 和管理等待列表的元素 PendList
等。而任务消息队列的结构体成员变量就少了PendList，因为等待任务消息队列只有拥有任务消息队列本身的任务才可以进行获取，故任务消息队列不需要等待列表的相关数据结构，具体见代码清单24‑1。

注意：想要使用任务消息队列，就必须将OS_CFG_TASK_Q_EN宏定义配置为1，该宏定义位于os_cfg.h文件中。

代码清单24‑1任务消息队列数据结构

1 struct os_msg_q

2 {

3 OS_MSG \*InPtr; **(1)**

4 OS_MSG \*OutPtr; **(2)**

5 OS_MSG_QTY NbrEntriesSize; **(3)**

6 OS_MSG_QTY NbrEntries; **(4)**

7 OS_MSG_QTY NbrEntriesMax; **(5)**

8 };

代码清单24‑1\ **(1)、(2)**\ ：任务消息队列中进出消息指针。

代码清单24‑1\ **(3)**\ ：任务消息队列中最大可用的消息个数，在创建任务的时候由用户指定这个值的大小。

代码清单24‑1\ **(4)**\ ：记录任务消息队列中当前的消息个数，每当发送一个消息到任务消息队列的时候，若任务没有在等待该消息，那么新发送的消息被插入任务消息队列后此值加1，NbrEntries 的大小不能超过NbrEntriesSize。

代码清单24‑1\ **(5)**\ ：记录任务消息队列最多的时候拥有的消息个数。

任务消息队列的运作机制与普通消息队列一样，没什么差别。

任务消息队列的函数接口讲解
~~~~~~~~~~~~~

任务消息队列发送函数OSTaskQPost()
^^^^^^^^^^^^^^^^^^^^^^^

函数 OSTaskQPost()用来发送任务消息队列，参数中有指向消息要发送给的任务控制块的指针，任何任务都可以发送消息给拥有任务消息队列的任务（任务在被创建的时候，要设置参数 q_size 大于 0），其源码具体见代码清单24‑2。

代码清单24‑2 OSTaskQPost()源码

1 #if OS_CFG_TASK_Q_EN > 0u //如果启用了任务消息队列

2 void OSTaskQPost (OS_TCB \*p_tcb, **(1)** //目标任务

3 void \*p_void, **(2)** //消息内容地址

4 OS_MSG_SIZE msg_size, **(3)** //消息长度

5 OS_OPT opt, **(4)** //选项

6 OS_ERR \*p_err) **(5)** //返回错误类型

7 {

8 CPU_TS ts;

9

10

11

12 #ifdef OS_SAFETY_CRITICAL//如果启用（默认禁用）了安全检测

13 if (p_err == (OS_ERR \*)0) //如果错误类型实参为空

14 {

15 OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数

16 return; //返回，停止执行

17 }

18 #endif

19

20 #if OS_CFG_ARG_CHK_EN > 0u//如果启用了参数检测

21 switch (opt) //根据选项分类处理

22 {

23 case OS_OPT_POST_FIFO: //如果选项在预期内

24 case OS_OPT_POST_LIFO:

25 case OS_OPT_POST_FIFO \| OS_OPT_POST_NO_SCHED:

26 case OS_OPT_POST_LIFO \| OS_OPT_POST_NO_SCHED:

27 break; //直接跳出

28

29 default: //如果选项超出预期

30 \*p_err = OS_ERR_OPT_INVALID; //错误类型为“选项非法”

31 return; //返回，停止执行

32 }

33 #endif

34

35 ts = OS_TS_GET(); //获取时间戳

36

37 #if OS_CFG_ISR_POST_DEFERRED_EN > 0u//如果启用了中断延迟发布

38 if (OSIntNestingCtr > (OS_NESTING_CTR)0) //如果该函数在中断中被调用

39 {

40 OS_IntQPost((OS_OBJ_TYPE)OS_OBJ_TYPE_TASK_MSG, //将消息先发布到中断消息队列

41 (void \*)p_tcb,

42 (void \*)p_void,

43 (OS_MSG_SIZE)msg_size,

44 (OS_FLAGS )0,

45 (OS_OPT )opt,

46 (CPU_TS )ts,

47 (OS_ERR \*)p_err); **(6)**

48 return; //返回

49 }

50 #endif

51

52 OS_TaskQPost(p_tcb, //将消息直接发布

53 p_void,

54 msg_size,

55 opt,

56 ts,

57 p_err); **(7)**

58 }

59 #endif

代码清单24‑2\ **(1)**\ ：目标任务。

代码清单24‑2\ **(2)**\ ：任务消息内容指针。

代码清单24‑2\ **(3)**\ ：任务消息的大小。

代码清单24‑2\ **(4)**\ ：发送的选项。

代码清单24‑2\ **(5)**\ ：用于保存返回的错误类型。

代码清单24‑2\ **(6)**\ ：如果启用了中断延迟发布，并且如果该函数在中断中被调用，就先将消息先发布到中断消息队列。

代码清单24‑2\ **(7)**\ ：调用OS_TaskQPost()函数将消息直接发送，其源码具体见代码清单24‑3。

代码清单24‑3 OS_TaskQPost()源码

1 #if OS_CFG_TASK_Q_EN > 0u//如果启用了任务消息队列

2 void OS_TaskQPost (OS_TCB \*p_tcb, //目标任务

3 void \*p_void, //消息内容地址

4 OS_MSG_SIZE msg_size, //消息长度

5 OS_OPT opt, //选项

6 CPU_TS ts, //时间戳

7 OS_ERR \*p_err) //返回错误类型

8 {

9 CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和

10 //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器

11 // SR（临界段关中断只需保存SR），开中断时将该值还原。

12

13 OS_CRITICAL_ENTER(); //进入临界段

14 if (p_tcb == (OS_TCB \*)0) **(1)**//如果 p_tcb 为空

15 {

16 p_tcb = OSTCBCurPtr; //目标任务为自身

17 }

18 \*p_err = OS_ERR_NONE; //错误类型为“无错误”

19 switch (p_tcb->TaskState) **(2)**//根据任务状态分类处理

20 {

21 case OS_TASK_STATE_RDY: //如果目标任务没等待状态

22 case OS_TASK_STATE_DLY:

23 case OS_TASK_STATE_SUSPENDED:

24 case OS_TASK_STATE_DLY_SUSPENDED:

25 OS_MsgQPut(&p_tcb->MsgQ, //把消息放入任务消息队列

26 p_void,

27 msg_size,

28 opt,

29 ts,

30 p_err); **(3)**

31 OS_CRITICAL_EXIT(); //退出临界段

32 break; //跳出

33

34 case OS_TASK_STATE_PEND: //如果目标任务有等待状态

35 case OS_TASK_STATE_PEND_TIMEOUT:

36 case OS_TASK_STATE_PEND_SUSPENDED:

37 case OS_TASK_STATE_PEND_TIMEOUT_SUSPENDED:

38 if (p_tcb->PendOn == OS_TASK_PEND_ON_TASK_Q) //如果等的是任务消息队列

39 {

40 OS_Post((OS_PEND_OBJ \*)0, //把消息发布给目标任务

41 p_tcb,

42 p_void,

43 msg_size,

44 ts); **(4)**

45 OS_CRITICAL_EXIT_NO_SCHED(); //退出临界段（无调度）

46 if ((opt & OS_OPT_POST_NO_SCHED) == (OS_OPT)0u) //如果要调度任务

47 {

48 OSSched(); //调度任务

49 }

50 }

51 else\ **(5)**//如果没在等待任务消息队列

52 {

53 OS_MsgQPut(&p_tcb->MsgQ, //把消息放入任务消息队列

54 p_void,

55 msg_size,

56 opt,

57 ts,

58 p_err);

59 OS_CRITICAL_EXIT(); //退出临界段

60 }

61 break; //跳出

62

63 default: **(6)**//如果状态超出预期

64 OS_CRITICAL_EXIT(); //退出临界段

65 \*p_err = OS_ERR_STATE_INVALID; //错误类型为“状态非法”

66 break; //跳出

67 }

68 }

69 #endif

代码清单24‑3\ **(1)**\ ：如果目标任务为空，则表示将任务消息释放给自己，那么p_tcb就指向当前任务。

代码清单24‑3\ **(2)**\ ：根据任务状态分类处理。

代码清单24‑3\ **(3)**\ ：如果目标任务没等待状态，就调用OS_MsgQPut()函数将消息放入队列中，执行完毕就退出，OS_MsgQPut()源码具体见代码清单18‑13。

代码清单24‑3\ **(4)**\ ：如果目标任务有等待状态，那就看看是不是在等待任务消息队列，如果是的话，调用OS_Post()函数把任务消息发送给目标任务，其源码具体见代码清单18‑14。

代码清单24‑3\ **(5)**\ ：如果任务并不是在等待任务消息队列，那么调用OS_MsgQPut()函数将消息放入任务消息队列中即可。

代码清单24‑3\ **(6)**\ ：如果状态超出预期，返回错误类型为“状态非法”的错误代码。

任务消息队列的发送过程是跟消息队列发送过程差不多，先检查目标任务的状态，如果该任务刚刚好在等待任务消息队列的消息，那么直接让任务脱离等待状态即可。如果任务没有在等待任务消息队列的消息，那么就将消息插入要发送消息的任务消息队列。

任务消息队列发送函数的使用实例具体见代码清单24‑4。

代码清单24‑4 OSTaskQPost()使用实例

1 OS_ERR err;

2

3 /\* 发布消息到任务 AppTaskPend \*/

4 OSTaskQPost ((OS_TCB \*)&AppTaskPendTCB, //目标任务的控制块

5 (void \*)"YeHuo μC/OS-III", //消息内容

6 (OS_MSG_SIZE )sizeof ( "YeHuo μC/OS-III" ), //消息长度

7 (OS_OPT )OS_OPT_POST_FIFO,

8 //发布到任务消息队列的入口端

9 (OS_ERR \*)&err); //返回错误类型

任务消息队列获取函数OSTaskQPend()
^^^^^^^^^^^^^^^^^^^^^^^

与OSTaskQPost()任务消息队列发送函数相对应，OSTaskQPend()函数用于获取一个任务消息队列，函数的参数中没有指定哪个任务获取任务消息，实际上就是当前执行的任务，当任务调用了这个函数就表明这个任务需要获取任务消息，OSTaskQPend()源码具体见代码清单24‑5。

代码清单24‑5OSTaskQPend()源码

1 #if OS_CFG_TASK_Q_EN > 0u//如果启用了任务消息队列

2 void \*OSTaskQPend (OS_TICK timeout, **(1)**//等待期限（单位：时钟节拍）

3 OS_OPT opt, **(2)** //选项

4 OS_MSG_SIZE \*p_msg_size, **(3)** //返回消息长度

5 CPU_TS \*p_ts, **(4)** //返回时间戳

6 OS_ERR \*p_err) **(5)** //返回错误类型

7 {

8 OS_MSG_Q \*p_msg_q;

9 void \*p_void;

10 CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和

11 //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器

12 // SR（临界段关中断只需保存SR），开中断时将该值还原。

13

14 #ifdef OS_SAFETY_CRITICAL//如果启用（默认禁用）了安全检测

15 if (p_err == (OS_ERR \*)0) //如果错误类型实参为空

16 {

17 OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数

18 return ((void \*)0); //返回0（有错误），停止执行

19 }

20 #endif

21

22 #if OS_CFG_CALLED_FROM_ISR_CHK_EN > 0u//如果启用了中断中非法调用检测

23 if (OSIntNestingCtr > (OS_NESTING_CTR)0) //如果该函数在中断中被调用

24 {

25 \*p_err = OS_ERR_PEND_ISR; //错误类型为“在中断中中止等待”

26 return ((void \*)0); //返回0（有错误），停止执行

27 }

28 #endif

29

30 #if OS_CFG_ARG_CHK_EN > 0u//如果启用了参数检测

31 if (p_msg_size == (OS_MSG_SIZE \*)0) //如果 p_msg_size 为空

32 {

33 \*p_err = OS_ERR_PTR_INVALID; //错误类型为“指针不可用”

34 return ((void \*)0); //返回0（有错误），停止执行

35 }

36 switch (opt) //根据选项分类处理

37 {

38 case OS_OPT_PEND_BLOCKING: //如果选项在预期内

39 case OS_OPT_PEND_NON_BLOCKING:

40 break; //直接跳出

41

42 default: //如果选项超出预期

43 \*p_err = OS_ERR_OPT_INVALID; //错误类型为“选项非法”

44 return ((void \*)0); //返回0（有错误），停止执行

45 }

46 #endif

47

48 if (p_ts != (CPU_TS \*)0) //如果 p_ts 非空

49 {

50 \*p_ts = (CPU_TS )0; //初始化（清零）p_ts，待用于返回时间戳

51 }

52

53 CPU_CRITICAL_ENTER(); //关中断

54 p_msg_q = &OSTCBCurPtr->MsgQ; **(6)**//获取当前任务的消息队列

55 p_void = OS_MsgQGet(p_msg_q, //从队列里获取一个消息

56 p_msg_size,

57 p_ts,

58 p_err); **(7)**

59 if (*p_err == OS_ERR_NONE) //如果获取消息成功

60 {

61 #if OS_CFG_TASK_PROFILE_EN > 0u

62

63 if (p_ts != (CPU_TS \*)0)

64 {

65 OSTCBCurPtr->MsgQPendTime = OS_TS_GET() - \*p_ts;

66 if (OSTCBCurPtr->MsgQPendTimeMax < OSTCBCurPtr->MsgQPendTime)

67 {

68 OSTCBCurPtr->MsgQPendTimeMax = OSTCBCurPtr->MsgQPendTime;

69 }

70 }

71 #endif

72 CPU_CRITICAL_EXIT(); //开中断

73 return (p_void); //返回消息内容

74 }

75 /\* 如果获取消息不成功（队列里没有消息） \*/ **(8)**

76 if ((opt & OS_OPT_PEND_NON_BLOCKING) != (OS_OPT)0) //如果选择了不阻塞任务

77 {

78 \*p_err = OS_ERR_PEND_WOULD_BLOCK; //错误类型为“缺乏阻塞”

79 CPU_CRITICAL_EXIT(); //开中断

80 return ((void \*)0); //返回0（有错误），停止执行

81 }

82 else\ **(9)**//如果选择了阻塞任务

83 {

84 if (OSSchedLockNestingCtr > (OS_NESTING_CTR)0) //如果调度器被锁

85 {

86 CPU_CRITICAL_EXIT(); //开中断

87 \*p_err = OS_ERR_SCHED_LOCKED; //错误类型为“调度器被锁”

88 return ((void \*)0); //返回0（有错误），停止执行

89 }

90 }

91 /\* 如果调度器未被锁 \*/

92 OS_CRITICAL_ENTER_CPU_EXIT(); **(10)**//锁调度器，重开中断

93 OS_Pend((OS_PEND_DATA \*)0, **(11)**//阻塞当前任务，等待消息

94 (OS_PEND_OBJ \*)0,

95 (OS_STATE )OS_TASK_PEND_ON_TASK_Q,

96 (OS_TICK )timeout);

97 OS_CRITICAL_EXIT_NO_SCHED(); //解锁调度器（无调度）

98

99 OSSched(); **(12)**//调度任务

100 /\* 当前任务（获得消息队列的消息）得以继续运行 \*/

101 CPU_CRITICAL_ENTER(); **(13)**//关中断

102 switch (OSTCBCurPtr->PendStatus) //根据任务的等待状态分类处理

103 {

104 case OS_STATUS_PEND_OK: **(14)**//如果任务已成功获得消息

105 p_void = OSTCBCurPtr->MsgPtr; //提取消息内容地址

106 \*p_msg_size = OSTCBCurPtr->MsgSize; //提取消息长度

107 if (p_ts != (CPU_TS \*)0) //如果 p_ts 非空

108 {

109 \*p_ts = OSTCBCurPtr->TS; //获取任务等到消息时的时间戳

110 #if OS_CFG_TASK_PROFILE_EN > 0u

111

112 OSTCBCurPtr->MsgQPendTime = OS_TS_GET() - OSTCBCurPtr->TS;

113 if (OSTCBCurPtr->MsgQPendTimeMax < OSTCBCurPtr->MsgQPendTime)

114 {

115 OSTCBCurPtr->MsgQPendTimeMax = OSTCBCurPtr->MsgQPendTime;

116 }

117 #endif

118 }

119 \*p_err = OS_ERR_NONE; //错误类型为“无错误”

120 break; //跳出

121

122 case OS_STATUS_PEND_ABORT: **(15)**//如果等待被中止

123 p_void = (void \*)0; //返回消息内容为空

124 \*p_msg_size = (OS_MSG_SIZE)0; //返回消息大小为0

125 if (p_ts != (CPU_TS \*)0) //如果 p_ts 非空

126 {

127 \*p_ts = (CPU_TS )0; //清零 p_ts

128 }

129 \*p_err = OS_ERR_PEND_ABORT; //错误类型为“等待被中止”

130 break; //跳出

131

132 case OS_STATUS_PEND_TIMEOUT: **(16)**//如果等待超时，

133 default: //或者任务状态超出预期。

134 p_void = (void \*)0; //返回消息内容为空

135 \*p_msg_size = (OS_MSG_SIZE)0; //返回消息大小为0

136 if (p_ts != (CPU_TS \*)0) //如果 p_ts 非空

137 {

138 \*p_ts = OSTCBCurPtr->TS;

139 }

140 \*p_err = OS_ERR_TIMEOUT; //错误类为“等待超时”

141 break; //跳出

142 }

143 CPU_CRITICAL_EXIT(); //开中断

144 return (p_void); **(17)**//返回消息内容地址

145 }

146 #endif

代码清单24‑5\ **(1)**\ ：指定超时时间（单位：时钟节拍）。

代码清单24‑5\ **(2)**\ ：获取任务消息队列的选项。

代码清单24‑5\ **(3)**\ ：返回消息大小。

代码清单24‑5\ **(4)**\ ：返回时间戳。

代码清单24‑5\ **(5)**\ ：返回错误类型。

代码清单24‑5\ **(6)**\ ：获取当前任务的消息队列保存在p_msg_q变量中。

代码清单24‑5\ **(7)**\ ：调用OS_MsgQGet()函数从消息队列获取一个消息，其源码具体见代码清单18‑17，如果获取消息成功，则返回指向消息的指针。

代码清单24‑5\ **(8)**\ ：如果获取消息不成功（任务消息队列里没有消息），并且如果用户选择了不阻塞任务，那么返回错误类型为“缺乏阻塞”的错误代码，然后退出。

代码清单24‑5\ **(9)**\ ：如果选择了阻塞任务，先判断一下调度器是否被锁，如果被锁了也就不能继续执行。

代码清单24‑5\ **(10)**\ ：如果调度器未被锁，系统会锁调度器，重开中断。

代码清单24‑5\ **(11)**\ ：调用OS_Pend()函数将当前任务脱离就绪列表，并根据用户指定的阻塞时间插入节拍列表，但是不会插入队列等待列表，然后打开调度器，但不进行调度，OS_Pend()源码具体见代码清单18‑18。

代码清单24‑5\ **(12)**\ ：进行一次任务调度。

代码清单24‑5\ **(13)**\ ：程序能执行到这里，就说明大体上有两种情况，要么是任务获取到消息了；任务还没获取到消息（任务没获取到消息的情况有很多种），无论是哪种情况，都先把中断关掉再说，然后根据当前运行任务的等待状态分类处理。

代码清单24‑5\ **(14)**\ ：如果任务状态是OS_STATUS_PEND_OK，则表示任务获取到消息了，那么就从任务控制块中提取消息，这是因为在发送消息给任务的时候，会将消息放入任务控制块的MsgPtr成员变量中，然后继续提取消息大小，如果p_ts非空，记录获取任务等到消息时的时间戳，返
回错误类型为“无错误”的错误代码，跳出switch语句。

代码清单24‑5\ **(15)**\ ：如果任务在等待（阻塞）重被中止，则返回消息内容为空，返回消息大小为0，返回错误类型为“等待被中止”的错误代码，跳出switch语句。

代码清单24‑5\ **(16)**\ ：如果任务等待（阻塞）超时，说明等待的时间过去了，任务也没获取到消息，则返回消息内容为空，返回消息大小为0，返回错误类型为“等待超时”的错误代码，跳出switch语句。

代码清单24‑5\ **(17)**\ ：打开中断，返回消息内容。

任务消息队列实验
~~~~~~~~

任务通知代替消息队列是在ΜC/OS中创建了两个任务，其中一个任务是用于接收任务消息，另一个任务发送任务消息。两个任务独立运行，发送消息任务每秒发送一次任务消息，接收任务在就一直在等待消息，一旦获取到消息通知就把消息打印在串口调试助手里，具体见代码清单24‑6。

代码清单24‑6任务通知代替消息队列

1 #include <includes.h>

2

3 static OS_TCB AppTaskStartTCB; //任务控制块

4

5 static OS_TCB AppTaskPostTCB;

6 static OS_TCB AppTaskPendTCB;

7

8

9 static CPU_STK AppTaskStartStk[APP_TASK_START_STK_SIZE]; //任务栈

10

11 static CPU_STK AppTaskPostStk [ APP_TASK_POST_STK_SIZE ];

12 static CPU_STK AppTaskPendStk [ APP_TASK_PEND_STK_SIZE ];

13

14

15 static void AppTaskStart (void \*p_arg); //任务函数声明

16

17 static void AppTaskPost ( void \* p_arg );

18 static void AppTaskPend ( void \* p_arg );

19

20

21

22

23 int main (void)

24 {

25 OS_ERR err;

26

27

28 OSInit(&err); //初始化 μC/OS

29

30

31 /\* 创建起始任务 \*/

32 OSTaskCreate((OS_TCB \*)&AppTaskStartTCB,

33 //任务控制块地址

34 (CPU_CHAR \*)"App Task Start", //任务名称

35 (OS_TASK_PTR ) AppTaskStart, //任务函数

36 (void \*) 0,

37 //传递给任务函数（形参p_arg）的实参

38 (OS_PRIO ) APP_TASK_START_PRIO, //任务的优先级

39 (CPU_STK \*)&AppTaskStartStk[0],

40 //任务栈的基地址

41 (CPU_STK_SIZE) APP_TASK_START_STK_SIZE / 10,

42 //任务栈空间剩下1/10时限制其增长

43 (CPU_STK_SIZE) APP_TASK_START_STK_SIZE,

44 //任务栈空间（单位：sizeof(CPU_STK)）

45 (OS_MSG_QTY ) 5u,

46 //任务可接收的最大消息数

47 (OS_TICK ) 0u,

48 //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）

49 (void \*) 0,

50 //任务扩展（0表不扩展）

51 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),//任务选项

52 (OS_ERR \*)&err); //返回错误类型

53

54 OSStart(&err);

55 //启动多任务管理（交由μC/OS-III控制）

56

57 }

58

59

60 static void AppTaskStart (void \*p_arg)

61 {

62 CPU_INT32U cpu_clk_freq;

63 CPU_INT32U cnts;

64 OS_ERR err;

65

66

67 (void)p_arg;

68

69 BSP_Init(); //板级初始化

70 CPU_Init(); //初始化 CPU组件（时间戳、关中断时间测量和主机名）

71

72

73 cpu_clk_freq = BSP_CPU_ClkFreq();

74 //获取 CPU内核时钟频率（SysTick 工作时钟）

75 cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;

76 //根据用户设定的时钟节拍频率计算 SysTick 定时器的计数值

77 OS_CPU_SysTickInit(cnts); //调用 SysTick

78 初始化函数，设置定时器计数值和启动定时器

79

80 Mem_Init();

81 //初始化内存管理组件（堆内存池和内存池表）

82

83 #if OS_CFG_STAT_TASK_EN > 0u

84 //如果启用（默认启用）了统计任务

85 OSStatTaskCPUUsageInit(&err);

86

87

88 #endif

89

90

91 CPU_IntDisMeasMaxCurReset();

92 //复位（清零）当前最大关中断时间

93

94

95 /\* 创建 AppTaskPost 任务 \*/

96 OSTaskCreate((OS_TCB \*)&AppTaskPostTCB,

97 //任务控制块地址

98 (CPU_CHAR \*)"App Task Post", //任务名称

99 (OS_TASK_PTR ) AppTaskPost, //任务函数

100 (void \*) 0,

101 //传递给任务函数（形参p_arg）的实参

102 (OS_PRIO ) APP_TASK_POST_PRIO, //任务的优先级

103 (CPU_STK \*)&AppTaskPostStk[0],

104 //任务栈的基地址

105 (CPU_STK_SIZE) APP_TASK_POST_STK_SIZE / 10,

106 //任务栈空间剩下1/10时限制其增长

107 (CPU_STK_SIZE) APP_TASK_POST_STK_SIZE,

108 //任务栈空间（单位：sizeof(CPU_STK)）

109 (OS_MSG_QTY ) 5u,

110 //任务可接收的最大消息数

111 (OS_TICK ) 0u,

112 //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）

113 (void \*) 0,

114 //任务扩展（0表不扩展）

115 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

116 (OS_ERR \*)&err); //返回错误类型

117

118 /\* 创建 AppTaskPend 任务 \*/

119 OSTaskCreate((OS_TCB \*)&AppTaskPendTCB,

120 //任务控制块地址

121 (CPU_CHAR \*)"App Task Pend", //任务名称

122 (OS_TASK_PTR ) AppTaskPend, //任务函数

123 (void \*) 0,

124 //传递给任务函数（形参p_arg）的实参

125 (OS_PRIO ) APP_TASK_PEND_PRIO, //任务的优先级

126 (CPU_STK \*)&AppTaskPendStk[0],

127 //任务栈的基地址

128 (CPU_STK_SIZE) APP_TASK_PEND_STK_SIZE / 10,

129 //任务栈空间剩下1/10时限制其增长

130 (CPU_STK_SIZE) APP_TASK_PEND_STK_SIZE,

131 //任务栈空间（单位：sizeof(CPU_STK)）

132 (OS_MSG_QTY ) 50u,

133 //任务可接收的最大消息数

134 (OS_TICK ) 0u,

135 //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）

136 (void \*) 0,

137 //任务扩展（0表不扩展）

138 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR), //任务选项

139 (OS_ERR \*)&err); //返回错误类型

140

141 OSTaskDel ( & AppTaskStartTCB, & err );

142 //删除起始任务本身，该任务不再运行

143

144

145 }

146

147

148

149 static void AppTaskPost ( void \* p_arg )

150 {

151 OS_ERR err;

152

153

154 (void)p_arg;

155

156

157 while (DEF_TRUE) //任务体

158 {

159 /\* 发送消息到任务 AppTaskPend \*/

160 OSTaskQPost ((OS_TCB \*)&AppTaskPendTCB, //目标任务的控制块

161 (void \*)"Fire μC/OS-III", //消息内容

162 (OS_MSG_SIZE )sizeof( "Fire μC/OS-III" ), //消息长度

163 (OS_OPT )OS_OPT_POST_FIFO,

164 //发送到任务消息队列的入口端

165 (OS_ERR \*)&err); //返回错误类型

166

167 OSTimeDlyHMSM ( 0, 0, 1, 0, OS_OPT_TIME_DLY, & err );

168

169 }

170

171 }

172

173

174

175 static void AppTaskPend ( void \* p_arg )

176 {

177 OS_ERR err;

178 OS_MSG_SIZE msg_size;

179 CPU_TS ts;

180 CPU_INT32U cpu_clk_freq;

181 CPU_SR_ALLOC();

182

183 char \* pMsg;

184

185

186 (void)p_arg;

187

188

189 cpu_clk_freq = BSP_CPU_ClkFreq();

190 //获取CPU时钟，时间戳是以该时钟计数

191

192

193 while (DEF_TRUE) //任务体

194 {

195 /\* 阻塞任务，等待任务消息 \*/

196 pMsg = OSTaskQPend ((OS_TICK )0, //无期限等待

197 (OS_OPT )OS_OPT_PEND_BLOCKING, //没有消息就阻塞任务

198 (OS_MSG_SIZE \*)&msg_size, //返回消息长度

199 (CPU_TS \*)&ts,

200 //返回消息被发送的时间戳

201 (OS_ERR \*)&err); //返回错误类型

202

203 ts = OS_TS_GET() - ts;

204 //计算消息从发送到被接收的时间差

205

206 macLED1_TOGGLE (); //切换LED1的亮灭状态

207

208 OS_CRITICAL_ENTER();

209 //进入临界段，避免串口打印被打断

210

211 printf ( "\r\n接收到的消息的内容为：%s，长度是：%d字节。",

212 pMsg, msg_size );

213

214 printf ( "\r\n任务消息从被发送到被接收的时间差是%dus\r\n",

215 ts / ( cpu_clk_freq / 1000000 ) );

216

217 OS_CRITICAL_EXIT(); //退出临界段

218

219 }

220

221 }

任务消息队列实验现象
~~~~~~~~~~

将程序编译好，用USB线连接计算机和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），在计算机上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的运行打印信息，具体见图24‑1。

|Taskme002|

图24‑1任务通知代替消息队列实验现象

.. |Taskme002| image:: media\Taskme002.png
   :width: 5.10486in
   :height: 4.15556in
