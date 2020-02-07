.. vim: syntax=rst

中断管理
===========

异常与中断的基本概念
~~~~~~~~~~

异常是导致处理器脱离正常运行转向执行特殊代码的任何事件，如果不及时进行处理，轻则系统出错，重则会导致系统毁灭性瘫痪。所以正确地处理异常，避免错误的发生是提高软件鲁棒性（稳定性）非常重要的一环，对于实时系统更是如此。

异常是指任何打断处理器正常执行，并且迫使处理器进入一个由有特权的特殊指令执行的事件。异常通常可以分成两类：同步异常和异步异常。由内部事件（像处理器指令运行产生的事件）引起的异常称为同步异常，例如造成被零除的算术运算引发一个异常，又如在某些处理器体系结构中，对于确定的数据尺寸必须从内存的偶数地址进行读
和写操作。从一个奇数内存地址的读或写操作将引起存储器存取一个错误事件并引起一个异常（称为校准异常）。

异步异常主要是指由于外部异常源产生的异常，是一个由外部硬件装置产生的事件引起的异步异常。同步异常不同于异步异常的地方是事件的来源，同步异常事件是由于执行某些指令而从处理器内部产生的，而异步异常事件的来源是外部硬件装置。例如按下设备某个按钮产生的事件。同步异常与异步异常的区别还在于，同步异常触发后，系
统必须立刻进行处理而不能够依然执行原有的程序指令步骤；而异步异常则可以延缓处理甚至是忽略，例如按键中断异常，虽然中断异常触发了，但是系统可以忽略它继续运行（同样也忽略了相应的按键事件）。

中断，中断属于异步异常。所谓中断是指中央处理器CPU正在处理某件事的时候，外部发生了某一事件，请求CPU迅速处理，CPU暂时中断当前的工作，转入处理所发生的事件，处理完后，再回到原来被中断的地方，继续原来的工作，这样的过程称为中断。

中断能打断任务的运行，无论该任务具有什么样的优先级，因此中断一般用于处理比较紧急的事件，而且只做简单处理，例如标记该事件，在使用μC/OS系统时，一般建议使用信号量、消息或事件标志组等标志中断的发生，将这些内核对象发布给处理任务，处理任务再做具体处理。

通过中断机制，在外设不需要CPU介入时，CPU可以执行其他任务，而当外设需要CPU时通过产生中断信号使CPU立即停止当前任务转而来响应中断请求。这样可以使CPU避免把大量时间耗费在等待、查询外设状态的操作上，因此将大大提高系统实时性以及执行效率。

此处读者要知道一点，μC/OS源码中有许多处临界段的地方，临界段虽然保护了关键代码的执行不被打断，但也会影响系统的实时，任何使用了操作系统的中断响应都不会比裸机快。比如，某个时候有一个任务在运行中，并且该任务部分程序将中断屏蔽掉，也就是进入临界段中，这个时候如果有一个紧急的中断事件被触发，这个中断就
会被挂起，不能得到及时响应，必须等到中断开启才可以得到响应，如果屏蔽中断时间超过了紧急中断能够容忍的限度，危害是可想而知的。操作系统的中断在某些时候会产生必要的中断延迟，因此调用中断屏蔽函数进入临界段的时候，也需快进快出。

μC/OS的中断管理支持：

-  开/关中断。

-  恢复中断。

-  中断启用。

-  中断屏蔽。

-  中断嵌套。

-  中断延迟发布。

中断的介绍
^^^^^

与中断相关的硬件可以划分为三类：外设、中断控制器、CPU本身。

外设：当外设需要请求CPU时，产生一个中断信号，该信号连接至中断控制器。

中断控制器：中断控制器是CPU众多外设中的一个，它一方面接收其他外设中断信号的输入，另一方面，它会发出中断信号给CPU。可以通过对中断控制器编程实现对中断源的优先级、触发方式、打开和关闭源等设置操作。在Cortex-M系列控制器中常用的中断控制器是NVIC（内嵌向量中断控制器Nested
Vectored Interrupt Controller）。

CPU：CPU会响应中断源的请求，中断当前正在执行的任务，转而执行中断处理程序。NVIC最多支持240个中断，每个中断最多256个优先级。

和中断相关的名词解释
^^^^^^^^^^

中断号：每个中断请求信号都会有特定的标志，使得计算机能够判断是哪个设备提出的中断请求，这个标志就是中断号。

中断请求：“紧急事件”需向CPU提出申请，要求CPU暂停当前执行的任务，转而处理该“紧急事件”，这一申请过程称为中断请求。

中断优先级：为使系统能够及时响应并处理所有中断，系统根据中断时间的重要性和紧迫程度，将中断源分为若干个级别，称作中断优先级。

中断处理程序：当外设产生中断请求后，CPU暂停当前的任务，转而响应中断申请，即执行中断处理程序。

中断触发：中断源发出并送给CPU控制信号，将中断触发器置“1”，表明该中断源产生了中断，要求CPU去响应该中断，CPU暂停当前任务，执行相应的中断处理程序。

中断触发类型：外部中断申请通过一个物理信号发送到NVIC，可以是电平触发或边沿触发。

中断向量：中断服务程序的入口地址。

中断向量表：存储中断向量的存储区，中断向量与中断号对应，中断向量在中断向量表中按照中断号顺序存储。

临界段：代码的临界段也称为临界区，一旦这部分代码开始执行，则不允许任何中断打断。为确保临界段代码的执行不被中断，在进入临界段之前须关中断，而临界段代码执行完毕后，要立即开中断。

中断的运作机制
~~~~~~~

当中断产生时，处理机将按如下的顺序执行：

1. 保存当前处理机状态信息

2. 载入异常或中断处理函数到PC寄存器

3. 把控制权转交给处理函数并开始执行

4. 当处理函数执行完成时，恢复处理器状态信息

5. 从异常或中断中返回到前一个程序执行点

中断使得CPU可以在事件发生时才给予处理，而不必让CPU连续不断地查询是否有相应的事件发生。通过两条特殊指令：关中断和开中断可以让处理器不响应或响应中断，在关闭中断期间，通常处理器会把新产生的中断挂起，当中断打开时立刻进行响应，所以会有适当的延时响应中断，故用户在进入临界区的时候应快进快出。

中断发生的环境有两种情况：在任务的上下文中，在中断服务函数处理上下文中。

-  任务在工作的时候，如果此时发生了一个中断，无论中断的优先级是多大，都会打断当前任务的执行，从而转到对应的中断服务函数中执行，其过程具体见图26‑1。

图26‑1\ **(1)、(3)**\ ：在任务运行的时候发生了中断，那么中断会打断任务的运行，那么操作系统将先保存当前任务的上下文环境，转而去处理中断服务函数。

图26‑1\ **(2)、(4)**\ ：当且仅当中断服务函数处理完的时候才恢复任务的上下文环境，继续运行任务。

|interr002|

图26‑1中断发生在任务上下文

-  在执行中断服务例程的过程中，如果有更高优先级别的中断源触发中断，由于当前处于中断处理上下文环境中，根据不同的处理器构架可能有不同的处理方式，比如新的中断等待挂起直到当前中断处理离开后再行响应；或新的高优先级中断打断当前中断处理过程，而去直接响应这个更高优先级的新中断源。后面这种情况，称之为中断嵌套
  。在硬实时环境中，前一种情况是不允许发生的，不能使响应中断的时间尽量的短。而在软件处理（软实时环境）上，μC/OS允许中断嵌套，即在一个中断服务例程期间，处理器可以响应另外一个优先级更高的中断，过程如图26‑2所示。

图26‑2\ **(1)**\ ：当中断1的服务函数在处理的时候发生了中断2，由于中断2的优先级比中断1更高，所以发生了中断嵌套，那么操作系统将先保存当前中断服务函数的上下文环境，并且转向处理中断2，当且仅当中断2执行完的时候图26‑2\ **(2)**\ ，才能继续执行中断1。

|interr003|

图26‑2中断嵌套发生

中断延迟的概念
~~~~~~~

即使操作系统的响应很快了，但对于中断的处理仍然存在着中断延迟响应的问题，我们称之为中断延迟(Interrupt Latency) 。

中断延迟是指从硬件中断发生到开始执行中断处理程序第一条指令之间的这段时间。也就是：系统接收到中断信号到操作系统作出响应，并完成换到转入中断服务程序的时间。也可以简单地理解为：（外部）硬件（设备）发生中断，到系统执行中断服务子程序（ISR）的第一条指令的时间。

中断的处理过程是：外界硬件发生了中断后，CPU到中断处理器读取中断向量，并且查找中断向量表，找到对应的中断服务子程序（ISR）的首地址，然后跳转到对应的ISR去做相应处理。这部分时间，我称之为：识别中断时间。

在允许中断嵌套的实时操作系统中，中断也是基于优先级的，允许高优先级中断抢断正在处理的低优先级中断，所以，如果当前正在处理更高优先级的中断，即使此时有低优先级的中断，也系统不会立刻响应，而是等到高优先级的中断处理完之后，才会响应。而即使在不支持中断嵌套，即中断是没有优先级的，中断是不允许被中断的，所以
，如果当前系统正在处理一个中断，而此时另一个中断到来了，系统也是不会立即响应的，而只是等处理完当前的中断之后，才会处理后来的中断。此部分时间，我称其为：等待中断打开时间。

在操作系统中，很多时候我们会主动进入临界段，系统不允许当前状态被中断打断，故而在临界区发生的中断会被挂起，直到退出临界段时候打开中断。此部分时间，我称其为：关闭中断时间。

中断延迟可以定义为，从中断开始的时刻到中断服务例程开始执行的时刻之间的时间段。中断延迟 = 识别中断时间 + [等待中断打开时间] + [关闭中断时间]。

注意：“[ ]”的时间是不一定都存在的，此处为最大可能的中断延迟时间。

此外，中断恢复时间定义为：执行完ISR中最后一句代码后到恢复到任务级代码的这段时间。

任务延迟时间定义为：中断发生到恢复到任务级代码的这段时间。

中断的应用场景
~~~~~~~

中断在嵌入式处理器中应用非常之多，没有中断的系统不是一个好系统，因为有中断，才能启动或者停止某件事情，从而转去做另一间事情。我们可以举一个日常生活中的例子来说明，假如你正在给朋友写信，电话铃响了，这时你放下手中的笔去接电话，通话完毕再继续写信。这个例子就表现了中断及其处理的过程：电话铃声使你暂时中止
当前的工作，而去处理更为急需处理的事情——接电话，当把急需处理的事情处理完毕之后，再回过头来继续原来的事情。在这个例子中，电话铃声就可以称为“中断请求”，而你暂停写信去接电话就叫作“中断响应”，那么接电话的过程就是“中断处理”。由此我们可以看出，在计算机执行程序的过程中，由于出现某个特殊情况(或称为
“特殊事件”)，使得系统暂时中止现行程序，而转去执行处理这一特殊事件的程序，处理完毕之后再回到原来程序的中断点继续向下执行。

为什么说吗没有中断的系统不是好系统呢？我们可以再举一个例子来说明中断的作用。假设有一个朋友来拜访你，但是由于不知何时到达，你只能在门口等待，于是什么事情也干不了；但如果在门口装一个门铃，你就不必在门口等待而可以在家里去做其他的工作，朋友来了按门铃通知你，这时你才中断手中的工作去开门，这就避免了不必要
的等待。CPU也是一样，如果时间都浪费在查询的事情上，那这个CPU啥也干不了，要他何用。在嵌入式系统中合理利用中断，能更好利用CPU的资源。

中断管理讲解
~~~~~~

ARM Cortex-M 系列内核的中断是由硬件管理的，而μC/OS是软件，它并不接管由硬件管理的相关中断（接管简单来说就是，所有的中断都由RTOS的软件管理，硬件来了中断时，由软件决定是否响应，可以挂起中断，延迟响应或者不响应），只支持简单的开关中断等，所以μC/OS中的中断使用其实跟裸机差不多的
，需要我们自己配置中断，并且启用中断，编写中断服务函数，在中断服务函数中使用内核IPC通信机制，一般建议使用信号量、消息或事件标志组等标志事件的发生，将事件发布给处理任务，等退出中断后再由相关处理任务具体处理中断，当然μC/OS为了能让系统更快退出中断，它支持中断延迟发布，将中断级的发布变成任务级（
在后文讲解）。

ARM Cortex-M NVIC支持中断嵌套功能：当一个中断触发并且系统进行响应时，处理器硬件会将当前运行的部分上下文寄存器自动压入中断栈中，这部分的寄存器包括PSR，R0，R1，R2，R3以及R12寄存器。当系统正在服务一个中断时，如果有一个更高优先级的中断触发，那么处理器同样的会打断当前运行的
中断服务例程，然后把老的中断服务例程上下文的PSR，R0，R1，R2，R3和R12寄存器自动保存到中断栈中。这些部分上下文寄存器保存到中断栈的行为完全是硬件行为，这一点是与其他ARM处理器最大的区别（以往都需要依赖于软件保存上下文）。

另外，在ARM Cortex-M系列处理器上，所有中断都采用中断向量表的方式进行处理，即当一个中断触发时，处理器将直接判定是哪个中断源，然后直接跳转到相应的固定位置进行处理。而在ARM7、ARM9中，一般是先跳转进入IRQ入口，然后再由软件进行判断是哪个中断源触发，获得了相对应的中断服务例程入口地址
后，再进行后续的中断处理。ARM7、ARM9的好处在于，所有中断它们都有统一的入口地址，便于OS的统一管理。而ARM Cortex-
M系列处理器则恰恰相反，每个中断服务例程必须排列在一起放在统一的地址上（这个地址必须要设置到NVIC的中断向量偏移寄存器中）。中断向量表一般由一个数组定义（或在起始代码中给出），在STM32上，默认采用起始代码给出：具体见代码清单26‑1。

代码清单26‑1中断向量表（部分）

1 \__Vectors DCD \__initial_sp ; Top of Stack

2 DCD Reset_Handler ; Reset Handler

3 DCD NMI_Handler ; NMI Handler

4 DCD HardFault_Handler ; Hard Fault Handler

5 DCD MemManage_Handler ; MPU Fault Handler

6 DCD BusFault_Handler ; Bus Fault Handler

7 DCD UsageFault_Handler ; Usage Fault Handler

8 DCD 0 ; Reserved

9 DCD 0 ; Reserved

10 DCD 0 ; Reserved

11 DCD 0 ; Reserved

12 DCD SVC_Handler ; SVCall Handler

13 DCD DebugMon_Handler ; Debug Monitor Handler

14 DCD 0 ; Reserved

15 DCD PendSV_Handler ; PendSV Handler

16 DCD SysTick_Handler ; SysTick Handler

17

18 ; External Interrupts

19 DCD WWDG_IRQHandler ; Window Watchdog

20 DCD PVD_IRQHandler ; PVD through EXTI Line detect

21 DCD TAMPER_IRQHandler ; Tamper

22 DCD RTC_IRQHandler ; RTC

23 DCD FLASH_IRQHandler ; Flash

24 DCD RCC_IRQHandler ; RCC

25 DCD EXTI0_IRQHandler ; EXTI Line 0

26 DCD EXTI1_IRQHandler ; EXTI Line 1

27 DCD EXTI2_IRQHandler ; EXTI Line 2

28 DCD EXTI3_IRQHandler ; EXTI Line 3

29 DCD EXTI4_IRQHandler ; EXTI Line 4

30 DCD DMA1_Channel1_IRQHandler ; DMA1 Channel 1

31 DCD DMA1_Channel2_IRQHandler ; DMA1 Channel 2

32 DCD DMA1_Channel3_IRQHandler ; DMA1 Channel 3

33 DCD DMA1_Channel4_IRQHandler ; DMA1 Channel 4

34 DCD DMA1_Channel5_IRQHandler ; DMA1 Channel 5

35 DCD DMA1_Channel6_IRQHandler ; DMA1 Channel 6

36 DCD DMA1_Channel7_IRQHandler ; DMA1 Channel 7

37

37 ………

39

μC/OS在Cortex-M系列处理器上也遵循与裸机中断一致的方法，当用户需要使用自定义的中断服务例程时，只需要定义相同名称的函数覆盖弱化符号即可。所以，μC/OS在Cortex-
M系列处理器的中断控制其实与裸机没什么差别，不过在进入中断与退出中断的时候需要调用一下OSIntEnter()函数与OSIntExit()函数，方便中断嵌套管理。

中断延迟发布
~~~~~~

中断延迟发布的概念
^^^^^^^^^

μC/OS-III有两种方法处理来自于中断的事件，直接发布（或者称为释放）和延迟发布。通过os_cfg.h中的OS_CFG_ISR_POST_DEFERRED_EN来选择，当设置为0时，μC/OS使用直接发布的方法。当设置为1时，使用延迟发布方法，用户可以根据自己设计系统的应用选择其中一种方法即可。

启用中断延时发布，可以将中断级发布转换成任务级发布，而且在进入临界段时也可以使用锁调度器代替关中断，这就大大减小了关中断时间，有利于提高系统的实时性（能实时响应中断而不受中断屏蔽导致响应延迟）。在前面提到的
OSTimeTick()、OSSemPost()、OSQPost()、OSFlagPost()、OSTaskSemPost()、OSTaskQPost()、OSTaskSuspend()和 OSTaskResume()
等这些函数，如果没有使用中断延迟发布，那么调用这些函数意味着进入一段很长的临界段，也就要关中断很长时间。在启用中断延时发布后，如果在中断中调用这些函数，系统就会将这些 post 提交函数必要的信息保存到中断延迟提交的变量中去，为了配合中断延迟，μC/OS还将创建了优先级最高（优先级为
0）的任务——中断发布函数 OS_IntQTask，退出中断后根据之前保存的参数，在任务中再次进行 post 相关操作。这个过程其实就是把中断中的临界段放到任务中来实现，这个时候进入临界段就可以用锁住调度器的方式代替了关中断，因此大大减少了关中断的时间，系统将 post
操作延迟了，中断延迟就是这么来的。

进入临界段的方式可以是关中断或者锁住调度器，系统中有些变量不可能在中断中被访问，所以只要保证其他任务不要使用这些变量即可，这个时候就可以用锁调度启动的方式，用锁住调度代替关中断，大大减少了关中断的时间，也能达到进入临界段的目的。中断延迟就是利用这种思想，让本该在中断中完成的事情切换到任务中完成，而且
进入临界段的方式是锁定调度器，这样子中断就不会被屏蔽，系统能随时响应中断，并且，整个中断延迟发布的过程是不影响post的效果，因为μC/OS已经设定中断发布任务的优先级为最高，在退出中断后会马上进行post操作，这与在中断中直接进行post操作的时间基本一致。

注：操作系统内核相关函数一般为了保证其操作的完整性，一般都会进入或长或短的临界段，所以在中断的要尽量少调用内核函数，部分μC/OS提供的函数是不允许在中断中调用的。

在直接发布方式中，μC/OS访问临界段时是采用关中断方式。然而，在延迟提交方式中，μC/OS访问临界段时是采用锁调度器方式。在延迟提交方式中，访问中断队列时μC/OS仍需要关中断进入临界段，但是这段关中断时间是非常短的且是固定的。

下面来看看中断延迟发布与直接发布的区别，具体见图26‑3图26‑4。

|interr004|

图26‑3中断延迟发布

图26‑3\ **(1)**\ ：进入中断，在中断中需要发布一个内核对象（如消息队列、信号量等），但是使用了中断延迟发布，在中断中值执行OS_IntQPost()函数，在这个函数中，采用关中断方式进入临界段，因此在这个时间段是不能响应中断的。

图26‑3\ **(2)**\ ：已经将内核对象发布到中断消息队列，那么将唤醒OS_IntQTask任务，因为该任务是最高优先级任务，所以能立即被唤醒，然后转到OS_IntQTask任务中发布内核对象，在该任务中，调用OS_IntQRePost()函数进行发布内核对象，进入临界段的方式采用锁调度器方
式，那么在这个阶段，中断是可以被响应的。

图26‑3\ **(3)**\ ：系统正常运行，任务按优先级进行切换。

|interr005|

图26‑4中断直接发布

图26‑4\ **(1)、(2)**\ ：而采用中断直接发布的情况是在中断中直接屏蔽中断以进入临界段，这段时间中，都不会响应中断，直到发布完成，系统任务正常运行才开启中断。

图26‑4\ **(3)**\ ：系统正常运行，任务按照优先级正常切换

从两个图中我们可以看出，很明显，采用中断延迟发布的效果更好，将本该在中断中的处理转变成为在任务中处理，系统关中断的时间大大降低，使得系统能很好地响应外部中断，如果在应用中关中断时间是关键性的，应用中有非常频繁的中断源，且应用不能接受直接发布方式那样较长的关中断时间，推荐使用中断延迟发布方式。

中断队列控制块
^^^^^^^

如果启用中断延迟发布，在中断中调用内核对象发布（释放）函数，系统会将发布的内容存放在中断队列中控制块中，源码具体见代码清单26‑2

代码清单26‑2中断队列信息块

1 #if OS_CFG_ISR_POST_DEFERRED_EN > 0u

2 struct os_int_q

3 {

4 OS_OBJ_TYPE Type; **(1)**

5 OS_INT_Q \*NextPtr; **(2)**

6 void \*ObjPtr; **(3)**

7 void \*MsgPtr; **(4)**

8 OS_MSG_SIZE MsgSize; **(5)**

9 OS_FLAGS Flags; **(6)**

10 OS_OPT Opt; **(7)**

11 CPU_TS TS; **(8)**

12 };

13 #endif

代码清单26‑2\ **(1)**\ ：用于发布的内核对象类型，例如消息队列、信号量、事件等。

代码清单26‑2\ **(2)**\ ：指向下一个中断队列控制块。

代码清单26‑2\ **(3)**\ ：指向内核对象变量指针。

代码清单26‑2\ **(4)**\ ：如果发布的是任务消息或者是内核对象消息，指向发布消息的指针。

代码清单26‑2\ **(5)**\ ：如果发布的是任务消息或者是内核对象消息，记录发布的消息的字节大小。

代码清单26‑2\ **(6)**\ ：如果发布的是事件标志，该成员变量记录要设置事件的标志位。

代码清单26‑2\ **(7)**\ ：记录发布内核对象时的选项。

代码清单26‑2\ **(8)**\ ：记录时间戳。

中断延迟发布任务初始化OS_IntQTaskInit()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在系统初始化的时候，如果我们启用了中断延迟发布，那么系统会根据我们自定义配置中断延迟发布任务的宏定义OS_CFG_INT_Q_SIZE与OS_CFG_INT_Q_TASK_STK_SIZE进行相关初始化，这两个宏定义在os_cfg_app.h文件中，中断延迟发布任务的初始化具体见代码清单26‑3。

代码清单26‑3中断延迟发布任务初始化

1 void OS_IntQTaskInit (OS_ERR \*p_err)

2 {

3 OS_INT_Q \*p_int_q;

4 OS_INT_Q \*p_int_q_next;

5 OS_OBJ_QTY i;

6

7

8

9 #ifdef OS_SAFETY_CRITICAL

10 if (p_err == (OS_ERR \*)0)

11 {

12 OS_SAFETY_CRITICAL_EXCEPTION();

13 return;

14 }

15 #endif

16

17 /\* 清空延迟提交过程中溢出的计数值 \*/

18 OSIntQOvfCtr = (OS_QTY)0u;

19

20 //延迟发布信息队列的基地址必须不为空指针

21 if (OSCfg_IntQBasePtr == (OS_INT_Q \*)0) **(1)**

22 {

23 \*p_err = OS_ERR_INT_Q;

24 return;

25 }

26

27 //延迟发布队列成员必须不小于 2 个

28 if (OSCfg_IntQSize < (OS_OBJ_QTY)2u) **(2)**

29 {

30 \*p_err = OS_ERR_INT_Q_SIZE;

31 return;

32 }

33

34 //初始化延迟发布任务每次运行的最长时间记录变量

35 OSIntQTaskTimeMax = (CPU_TS)0;

36

37 //将定义的数据连接成一个单向链表

38 p_int_q = OSCfg_IntQBasePtr; **(3)**

39 p_int_q_next = p_int_q;

40 p_int_q_next++;

41 for (i = 0u; i < OSCfg_IntQSize; i++)

42 {

43 //每个信息块都进行初始化

44 p_int_q->Type = OS_OBJ_TYPE_NONE;

45 p_int_q->ObjPtr = (void \*)0;

46 p_int_q->MsgPtr = (void \*)0;

47 p_int_q->MsgSize = (OS_MSG_SIZE)0u;

48 p_int_q->Flags = (OS_FLAGS )0u;

49 p_int_q->Opt = (OS_OPT )0u;

50 p_int_q->NextPtr = p_int_q_next;

51 p_int_q++;

52 p_int_q_next++;

53 }

54 //将单向链表的首尾相连组成一个“圈

55 p_int_q--;

56 p_int_q_next = OSCfg_IntQBasePtr;

57 p_int_q->NextPtr = p_int_q_next; **(4)**

58

59 //队列出口和入口都指向第一个

60 OSIntQInPtr = p_int_q_next;

61 OSIntQOutPtr = p_int_q_next; **(5)**

62

63 //清空延迟发布队列中需要进行发布的内核对象个数

64 OSIntQNbrEntries = (OS_OBJ_QTY)0u;

65 //清空延迟发布队列中历史发布的内核对象最大个数

66 OSIntQNbrEntriesMax = (OS_OBJ_QTY)0u;

67

68

69 if (OSCfg_IntQTaskStkBasePtr == (CPU_STK \*)0)

70 {

71 \*p_err = OS_ERR_INT_Q_STK_INVALID;

72 return;

73 }

74

75 if (OSCfg_IntQTaskStkSize < OSCfg_StkSizeMin)

76 {

77 \*p_err = OS_ERR_INT_Q_STK_SIZE_INVALID;

78 return;

79 }

80 //创建延迟发布任务

81 OSTaskCreate((OS_TCB \*)&OSIntQTaskTCB,

82 (CPU_CHAR \*)((void \*)"μC/OS-III ISR Queue Task"),

83 (OS_TASK_PTR )OS_IntQTask,

84 (void \*)0,

85 (OS_PRIO )0u, //优先级最高

86 (CPU_STK \*)OSCfg_IntQTaskStkBasePtr,

87 (CPU_STK_SIZE)OSCfg_IntQTaskStkLimit,

88 (CPU_STK_SIZE)OSCfg_IntQTaskStkSize,

89 (OS_MSG_QTY )0u,

90 (OS_TICK )0u,

91 (void \*)0,

92 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

93 (OS_ERR \*)p_err); **(6)**

94 }

95

96 #endif

代码清单26‑3\ **(1)**\ ：延迟发布信息队列的基地址必须不为空指针，μC/OS在编译的时候就已经静态分配一个存储的空间（大数组），具体见代码清单26‑4。

代码清单26‑4中断延迟发布队列存储空间（位于os_cfg_app.c）

1 #if (OS_CFG_ISR_POST_DEFERRED_EN > 0u)

2 OS_INT_Q OSCfg_IntQ [OS_CFG_INT_Q_SIZE];

3 CPU_STK OSCfg_IntQTaskStk [OS_CFG_INT_Q_TASK_STK_SIZE];

4 #endif

5

6 OS_INT_Q \* const OSCfg_IntQBasePtr = (OS_INT_Q \*)&OSCfg_IntQ[0];

7 OS_OBJ_QTY const OSCfg_IntQSize = (OS_OBJ_QTY )OS_CFG_INT_Q_SIZE;

代码清单26‑3\ **(2)**\ ：延迟发布队列成员（OSCfg_IntQSize = OS_CFG_INT_Q_SIZE）必须不小于2个，该宏在os_cfg_app.h文件中定义。

代码清单26‑3\ **(3)**\ ：将定义的数据连接成一个单向链表，并且初始化每一个信息块的内容。

代码清单26‑3\ **(4)**\ ：将单向链表的首尾相连组成一个“圈”，环形单链表，处理完成示意图具体见图26‑5。

|interr006|

图26‑5中断延迟发布队列初始化完成示意图

代码清单26‑3\ **(5)**\ ：队列出口和入口都指向第一个信息块。

代码清单26‑3\ **(6)**\ ：创建延迟发布任务，任务的优先级是0，是最高优先级任务不允许用户修改。

中断延迟发布过程OS_IntQPost()
^^^^^^^^^^^^^^^^^^^^^

如果启用了中断延迟发布，并且发送消息的函数是在中断中被调用，此时就不该立即发送消息，而是将消息的发送放在指定发布任务中，此时系统就将消息发布到租单消息队列中，等待到中断发布任务唤醒再发送消息，OS_IntQPost()源码具体见代码清单26‑5。

提示：为了阅读方便，将“中断延迟发布队列”简称为“中断队列”。

代码清单26‑5 OS_IntQPost()源码

1 void OS_IntQPost (OS_OBJ_TYPE type, **(1)**//内核对象类型

2 void \*p_obj, **(2)**//被发布的内核对象

3 void \*p_void, **(3)**//消息队列或任务消息

4 OS_MSG_SIZE msg_size, **(4)**//消息的数目

5 OS_FLAGS flags, **(5)**//事件

6 OS_OPT opt, **(6)**//发布内核对象时的选项

7 CPU_TS ts, **(7)**//发布内核对象时的时间戳

8 OS_ERR \*p_err) **(8)**//返回错误类型

9 {

10 CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和定义一个

11 //局部变量，用于保存关中断前的 CPU 状态寄存器 SR（临界段关中断只需保存SR）

12 //，开中断时将该值还原。

13

14 #ifdef OS_SAFETY_CRITICAL\ **(9)**//如果启用（默认禁用）了安全检测

15 if (p_err == (OS_ERR \*)0) { //如果错误类型实参为空

16 OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数

17 return; //返回，不继续执行

18 }

19 #endif

20

21 CPU_CRITICAL_ENTER(); //关中断

22 if (OSIntQNbrEntries < OSCfg_IntQSize) { **(10)**//如果中断队列未占满

23

24 OSIntQNbrEntries++; **(11)**

25 //更新中断队列的最大使用数目的历史记录

26 if (OSIntQNbrEntriesMax < OSIntQNbrEntries) { **(12)**

27 OSIntQNbrEntriesMax = OSIntQNbrEntries;

28 }

29 /\* 将要重新提交的内核对象的信息放入到中断队列入口的信息记录块 \*/**(13)**

30 OSIntQInPtr->Type = type; /*保存要发布的对象类型*/

31 OSIntQInPtr->ObjPtr = p_obj; /*保存指向要发布的对象的指针*/

32 OSIntQInPtr->MsgPtr = p_void;/*将信息保存到消息块的中*/

33 OSIntQInPtr->MsgSize = msg_size; /*保存信息的大小 \*/

34 OSIntQInPtr->Flags = flags; /*如果发布到事件标记组，则保存标志*/

35 OSIntQInPtr->Opt = opt; /*保存选项*/

36 OSIntQInPtr->TS = ts; /*保存时间戳信息*/ **(14)**

37

38 OSIntQInPtr = OSIntQInPtr->NextPtr; **(15)**//指向下一个中断队列入口

39 /\* 让中断队列管理任务 OSIntQTask 就绪 \*/ **(16)**

40 OSRdyList[0].NbrEntries = (OS_OBJ_QTY)1; //更新就绪列表上的优先级0的任务数为1个

41 //就绪列表的头尾指针都指向OSIntQTask 任务

42OSRdyList[0].HeadPtr = &OSIntQTaskTCB;

43 OSRdyList[0].TailPtr = &OSIntQTaskTCB;\ **(17)**

44 OS_PrioInsert(0u); **(18)**//在优先级列表中增加优先级0

45 if (OSPrioCur != 0) { **(19)**//如果当前运行的不是 OSIntQTask 任务

46 OSPrioSaved = OSPrioCur; //保存当前任务的优先级

47 }

48

49 \*p_err = OS_ERR_NONE; **(20)**//返回错误类型为“无错误”

50 } else { //如果中断队列已占满

51 OSIntQOvfCtr++; **(21)**//中断队列溢出数目加1

52 \*p_err = OS_ERR_INT_Q_FULL;//返回错误类型为“中断队列已满”

53 }

54 CPU_CRITICAL_EXIT(); //开中断

55 }

代码清单26‑5\ **(1)**\ ：内核对象类型。

代码清单26‑5\ **(2)**\ ：被发布的内核对象。

代码清单26‑5\ **(3)**\ ：消息队列或任务消息。

代码清单26‑5\ **(4)**\ ：消息的数目、大小。

代码清单26‑5\ **(5)**\ ：事件。

代码清单26‑5\ **(6)**\ ：发布内核对象时的选项。

代码清单26‑5\ **(7)**\ ：发布内核对象时的时间戳。

代码清单26‑5\ **(8)**\ ：返回错误类型。

代码清单26‑5\ **(9)**\ ：如果启用（默认禁用）了安全检测，在编译时则会包含安全检测相关的代码，如果错误类型实参为空，系统会执行安全检测异常函数，然后返回，停止执行。

代码清单26‑5\ **(10)**\ ：如果中断队列未占满，则执行 **(10)~(20)**\ 操作。

代码清单26‑5\ **(11)**\ ：OSIntQNbrEntries用于记录中断队列的入队数量，需要加一表示当前有信息记录块入队。

代码清单26‑5\ **(12)**\ ：更新中断队列的最大使用数目的历史记录。

代码清单26‑5\ **(13)~(14)**\ ：将要重新提交的内核对象的信息放入到中断队列的信息记录块中，记录的信息有发布的对象类型、发布的内核对象、要发布的消息、要发布的消息大小、要发布的事件、选项、时间戳等信息。

代码清单26‑5\ **(15)**\ ：指向下一个中断队列入口。

代码清单26‑5\ **(16)**\ ：让中断队列管理任务 OSIntQTask 就绪，更新就绪列表上的优先级0的任务数为1个。

代码清单26‑5\ **(17)**\ ：就绪列表的头尾指针都指向OSIntQTask 任务。

代码清单26‑5\ **(18)**\ ：调用OS_PrioInsert()函数在优先级列表中增加优先级0。

代码清单26‑5\ **(19)**\ ：如果当前运行的不是 OS_IntQTask 任务，则需要保存当前任务的优先级。

代码清单26‑5\ **(20)**\ ：程序能执行到这里，表示已经正确执行完毕，返回错误类型为“无错误”的错误代码。

代码清单26‑5\ **(21)**\ ：如果中断队列已占满，记录一下中断队列溢出数目，返回错误类型为“中断队列已满”的错误代码。

中断延迟发布任务OS_IntQTask()
^^^^^^^^^^^^^^^^^^^^^

在中断中将消息放入中断队列，那么接下来又怎么样进行发布内核对象呢？原来μC/OS在中断中只是将要提交的内核对象的信息都暂时保存起来，然后就绪优先级最高的中断延迟发布任务，接着继续执行中断，在退出所有中断嵌套后，第一个执行的任务就是延迟发布任务，延迟发布任务源码具体见。

代码清单26‑6延迟发布任务OS_IntQTask()源码

1 void OS_IntQTask (void \*p_arg)

2 {

3 CPU_BOOLEAN done;

4 CPU_TS ts_start;

5 CPU_TS ts_end;

6 CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和

7 //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器

8 // SR（临界段关中断只需保存SR），开中断时将该值还原。

9

10 p_arg = p_arg;

11 while (DEF_ON) //进入死循环

12 {

13 done = DEF_FALSE;

14 while (done == DEF_FALSE)

15 {

16 CPU_CRITICAL_ENTER(); //关中断

17 if (OSIntQNbrEntries == (OS_OBJ_QTY)0u) **(1)**

18 {

19

20 //如果中断队列里的内核对象发布完毕

21 //从就绪列表移除中断队列管理任务OS_IntQTask

22 OSRdyList[0].NbrEntries = (OS_OBJ_QTY)0u;

23 OSRdyList[0].HeadPtr = (OS_TCB \*)0;

24 OSRdyList[0].TailPtr = (OS_TCB \*)0;

25 OS_PrioRemove(0u); **(2)**//从优先级表格移除优先级0

26 CPU_CRITICAL_EXIT(); //开中断

27 OSSched(); **(3)**//任务调度

28 done = DEF_TRUE; //退出循环

29 }

30 else

31 //如果中断队列里还有内核对象

32 {

33 CPU_CRITICAL_EXIT(); //开中断

34 ts_start = OS_TS_GET(); //获取时间戳

35 OS_IntQRePost(); **(4)**//发布中断队列里的内核对象

36 ts_end = OS_TS_GET() - ts_start; //计算该次发布时间

37 if (OSIntQTaskTimeMax < ts_end)

38 //更新中断队列发布内核对象的最大时间的历史记录

39 {

40 OSIntQTaskTimeMax = ts_end;

41 }

42 CPU_CRITICAL_ENTER(); //关中断

43 OSIntQOutPtr = OSIntQOutPtr->NextPtr;\ **(5)**//处理下一个

44 OSIntQNbrEntries--; **(6)**//中断队列的成员数目减1

45 CPU_CRITICAL_EXIT(); //开中断

46 }

47 }

48 }

49 }

代码清单26‑6\ **(1)**\ ：如果中断队列里的内核对象发布完毕（OSIntQNbrEntries变量的值为0），从就绪列表移除中断延迟发布任务OS_IntQTask，这样子的操作相当于挂起OS_IntQTask任务。

代码清单26‑6\ **(2)**\ ：从优先级表格中移除优先级0的任务。

代码清单26‑6\ **(3)**\ ：进行一次任务调度，这就保证了从中断出来后如果需要发布会将相应的内核对象全部进行发布直到全部都发布完成，才会进行一次任务调度，然后让其他的任务占用 CPU。

代码清单26‑6\ **(4)**\ ：如果中断队列里还存在未发布的内核对象，就调用OS_IntQRePost()函数发布中断队列里的内核对象，其实这个函数才是真正的发布操作，该函数源码具体见代码清单26‑7。

代码清单26‑6\ **(5)**\ ：处理下一个要发布的内核对象，直到没有任何要发布的内核对象为止。

代码清单26‑6\ **(6)**\ ：中断队列的成员数目减1。

代码清单26‑7 OS_IntQRePost()源码

1 void OS_IntQRePost (void)

2 {

3 CPU_TS ts;

4 OS_ERR err;

5

6

7 switch (OSIntQOutPtr->Type) **(1)**//根据内核对象类型分类处理

8 {

9 case OS_OBJ_TYPE_FLAG: //如果对象类型是事件标志

10 #if OS_CFG_FLAG_EN > 0u//如果启用了事件标志，则发布事件标志

11 (void)OS_FlagPost((OS_FLAG_GRP \*) OSIntQOutPtr->ObjPtr,

12 (OS_FLAGS ) OSIntQOutPtr->Flags,

13 (OS_OPT ) OSIntQOutPtr->Opt,

14 (CPU_TS ) OSIntQOutPtr->TS,

15 (OS_ERR \*)&err); **(2)**

16 #endif

17 break; //跳出

18

19 case OS_OBJ_TYPE_Q: //如果对象类型是消息队列

20 #if OS_CFG_Q_EN > 0u//如果启用了消息队列，则发布消息队列

21 OS_QPost((OS_Q \*) OSIntQOutPtr->ObjPtr,

22 (void \*) OSIntQOutPtr->MsgPtr,

23 (OS_MSG_SIZE) OSIntQOutPtr->MsgSize,

24 (OS_OPT ) OSIntQOutPtr->Opt,

25 (CPU_TS ) OSIntQOutPtr->TS,

26 (OS_ERR \*)&err); **(3)**

27 #endif

28 break; //跳出

29

30 case OS_OBJ_TYPE_SEM: //如果对象类型是信号量

31 #if OS_CFG_SEM_EN > 0u//如果启用了信号量，则发布信号量

32 (void)OS_SemPost((OS_SEM \*) OSIntQOutPtr->ObjPtr,

33 (OS_OPT ) OSIntQOutPtr->Opt,

34 (CPU_TS ) OSIntQOutPtr->TS,

35 (OS_ERR \*)&err); **(4)**

36 #endif

37 break; //跳出

38

39 case OS_OBJ_TYPE_TASK_MSG: //如果对象类型是任务消息

40 #if OS_CFG_TASK_Q_EN > 0u//如果启用了任务消息，则发布任务消息

41 OS_TaskQPost((OS_TCB \*) OSIntQOutPtr->ObjPtr,

42 (void \*) OSIntQOutPtr->MsgPtr,

43 (OS_MSG_SIZE) OSIntQOutPtr->MsgSize,

44 (OS_OPT ) OSIntQOutPtr->Opt,

45 (CPU_TS ) OSIntQOutPtr->TS,

46 (OS_ERR \*)&err); **(5)**

47 #endif

48 break; //跳出

49

50 case OS_OBJ_TYPE_TASK_RESUME: //如果对象类型是恢复任务

51 #if OS_CFG_TASK_SUSPEND_EN > 0u//如果启用了函数OSTaskResume()，恢复该任务

52 (void)OS_TaskResume((OS_TCB \*) OSIntQOutPtr->ObjPtr,

53 (OS_ERR \*)&err); **(6)**

54 #endif

55 break; //跳出

56

57 case OS_OBJ_TYPE_TASK_SIGNAL://如果对象类型是任务信号量

58 (void)OS_TaskSemPost((OS_TCB \*) OSIntQOutPtr->ObjPtr,//发布任务信号量

59 (OS_OPT ) OSIntQOutPtr->Opt,

60 (CPU_TS ) OSIntQOutPtr->TS,

61 (OS_ERR \*)&err); **(7)**

62 break; //跳出

63

64 case OS_OBJ_TYPE_TASK_SUSPEND://如果对象类型是挂起任务

65 #if OS_CFG_TASK_SUSPEND_EN > 0u//如果启用了函数 OSTaskSuspend()，挂起该任务

66 (void)OS_TaskSuspend((OS_TCB \*) OSIntQOutPtr->ObjPtr,

67 (OS_ERR \*)&err); **(8)**

68 #endif

69 break; //跳出

70

71 case OS_OBJ_TYPE_TICK: **(9)** //如果对象类型是时钟节拍

72 #if OS_CFG_SCHED_ROUND_ROBIN_EN > 0u//如果启用了时间片轮转调度，

73 OS_SchedRoundRobin(&OSRdyList[OSPrioSaved]); //轮转调度进中断前优先级任务

74 #endif

75

76 (void)OS_TaskSemPost((OS_TCB \*)&OSTickTaskTCB,//发送信号量给时钟节拍任务

77 (OS_OPT ) OS_OPT_POST_NONE,

78 (CPU_TS ) OSIntQOutPtr->TS,

79 (OS_ERR \*)&err); **(10)**

80 #if OS_CFG_TMR_EN > 0u

81 //如果启用了软件定时器，发送信号量给定时器任务

82 OSTmrUpdateCtr--;

83 if (OSTmrUpdateCtr == (OS_CTR)0u)

84 {

85 OSTmrUpdateCtr = OSTmrUpdateCnt;

86 ts = OS_TS_GET();

87 (void)OS_TaskSemPost((OS_TCB \*)&OSTmrTaskTCB,

88 (OS_OPT ) OS_OPT_POST_NONE,

89 (CPU_TS ) ts,

90 (OS_ERR \*)&err); **(11)**

91 }

92 #endif

93 break; //跳出

94

95 default: **(12)**//如果内核对象类型超出预期

96 break; //直接跳出

97 }

98 }

代码清单26‑7\ **(1)**\ ：根据内核对象类型分类处理。

代码清单26‑7\ **(2)**\ ：如果对象类型是事件标志，发布事件标志。

代码清单26‑7\ **(3)**\ ：如果对象类型是消息队列，发布消息队列。

代码清单26‑7\ **(4)**\ ：如果对象类型是信号量，发布信号量。

代码清单26‑7\ **(5)**\ ：如果对象类型是任务消息，发布任务消息。

代码清单26‑7\ **(6)**\ ：如果对象类型是恢复任务，恢复该任务。

代码清单26‑7\ **(7)**\ ：如果对象类型是任务信号量，发布任务信号量。

代码清单26‑7\ **(8)**\ ：如果对象类型是挂起任务，挂起该任务。

代码清单26‑7\ **(9)**\ ：如果对象类型是时钟节拍，如果启用了时间片轮转调度，轮转调度进中断前优先级任务。

代码清单26‑7\ **(10)**\ ：发送信号量给时钟节拍任务。

代码清单26‑7\ **(11)**\ ：如果启用了软件定时器，发送信号量给定时器任务。

代码清单26‑7\ **(12)**\ ：如果内核对象类型超出预期，直接跳出。

该函数的整个流程也是非常简单的，首先提取出中断队列中的一个信息块的信息，根据发布的内核对象类型分类处理，在前面我们已经讲解过了全部内核对象发布（释放）的过程，就直接在任务中调用这些发布函数根据对应的内核对象进行发布。值得注意的是时钟节拍类型 OS_OBJ_TYPE_TICK，如果没有启用中断延迟发布
的宏定义，那么所有跟时钟节拍相关的，包括时间片轮转调度，定时器，发送消息给时钟节拍任务等都是在中断中执行，而使用延迟提交就把这些工作都放到延迟发布任务中执行。延迟发布之所以能够减少关中断的时间是因为在这些内核对象发布函数中，进入临界段都是采用锁调度器的方式，如果没有使用延迟发布，提交的整个过程都要关
中断。

至此，中断延迟发布的内容就讲解完毕，无论是否选择中断延迟发布，都不需要我们修改用户代码，这个是μC/OS会根据我们的选择自动处理，无需我们用户理会。

中断管理实验
~~~~~~

中断管理实验是在μC/OS中创建了两个任务分别获取信号量与消息队列，并且定义了两个按键KEY1与KEY2的触发方式为中断触发，其触发的中断服务函数则跟裸机一样，在中断触发的时候通过消息队列将消息传递给任务，任务接收到消息就将信息通过串口调试助手显示出来。而且中断管理实验也实现了一个串口的DMA传输+
空闲中断功能，当串口接收完不定长的数据之后产生一个空闲中断，在中断中将信号量传递给任务，任务在收到信号量的时候将串口的数据读取出来并且在串口调试助手中回显，具体见代码清单26‑8加粗部分。

代码清单26‑8中断管理实验

1 #include <includes.h>

2 #include <string.h>

3

4

5 static OS_TCB AppTaskStartTCB; //任务控制块

6

7 OS_TCB AppTaskUsartTCB;

8 OS_TCB AppTaskKeyTCB;

9

10

11 static CPU_STK AppTaskStartStk[APP_TASK_START_STK_SIZE]; //任务栈

12

13 static CPU_STK AppTaskUsartStk [ APP_TASK_USART_STK_SIZE ];

14 static CPU_STK AppTaskKeyStk [ APP_TASK_KEY_STK_SIZE ];

15

16

17 externchar Usart_Rx_Buf[USART_RBUFF_SIZE];

18

19

20 static void AppTaskStart (void \*p_arg); //任务函数声明

21

22 static void AppTaskUsart ( void \* p_arg );

23 static void AppTaskKey ( void \* p_arg );

24

25

26

27 int main (void)

28 {

29 OS_ERR err;

30

31

32 OSInit(&err); //初始化 μC/OS-III

33

34

35 /\* 创建起始任务 \*/

36 OSTaskCreate((OS_TCB \*)&AppTaskStartTCB,

37 //任务控制块地址

38 (CPU_CHAR \*)"App Task Start",

39 //任务名称

40 (OS_TASK_PTR ) AppTaskStart,

41 //任务函数

42 (void \*) 0,

43 //传递给任务函数（形参p_arg）的实参

44 (OS_PRIO ) APP_TASK_START_PRIO,

45 //任务的优先级

46 (CPU_STK \*)&AppTaskStartStk[0],

47 //任务栈的基地址

48 (CPU_STK_SIZE) APP_TASK_START_STK_SIZE / 10,

49 //任务栈空间剩下1/10时限制其增长

50 (CPU_STK_SIZE) APP_TASK_START_STK_SIZE,

51 //任务栈空间（单位：sizeof(CPU_STK)）

52 (OS_MSG_QTY ) 5u,

53 //任务可接收的最大消息数

54 (OS_TICK ) 0u,

55 //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）

56 (void \*) 0,

57 //任务扩展（0表不扩展）

58 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

59 //任务选项

60 (OS_ERR \*)&err);

61 //返回错误类型

62

63 OSStart(&err);

64 //启动多任务管理（交由μC/OS-III控制）

65

66 }

67

68

69

70 static void AppTaskStart (void \*p_arg)

71 {

72 CPU_INT32U cpu_clk_freq;

73 CPU_INT32U cnts;

74 OS_ERR err;

75

76

77 (void)p_arg;

78 //板级初始化

79 BSP_Init();

80 //初始化 CPU 组件（时间戳、关中断时间测量和主机名）

81 CPU_Init();

82

83 //获取 CPU 内核时钟频率（SysTick 工作时钟）

84 cpu_clk_freq = BSP_CPU_ClkFreq();

85 //根据用户设定的时钟节拍频率计算 SysTick 定时器的计数值

86 cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;

87 //调用 SysTick 初始化函数，设置定时器计数值和启动定时器

88 OS_CPU_SysTickInit(cnts);

89 //初始化内存管理组件（堆内存池和内存池表）

90 Mem_Init();

91

92 //如果启用（默认启用）了统计任务

93 #if OS_CFG_STAT_TASK_EN > 0u

94 OSStatTaskCPUUsageInit(&err);

95 #endif

96

97 //复位（清零）当前最大关中断时间

98 CPU_IntDisMeasMaxCurReset();

99

100

101 /\* 配置时间片轮转调度 \*/

102 OSSchedRoundRobinCfg((CPU_BOOLEAN )DEF_ENABLED,

103 //启用时间片轮转调度

104 (OS_TICK )0,

105 //把 OSCfg_TickRate_Hz/10 设为默认时间片值

106 (OS_ERR \*)&err ); //返回错误类型

107

108

109 /\* 创建 AppTaskUsart 任务 \*/

110 OSTaskCreate((OS_TCB \*)&AppTaskUsartTCB,

111 //任务控制块地址

112 (CPU_CHAR \*)"App Task Usart",

113 //任务名称

114 (OS_TASK_PTR ) AppTaskUsart,

115 //任务函数

116 (void \*) 0,

117 //传递给任务函数（形参p_arg）的实参

118 (OS_PRIO ) APP_TASK_USART_PRIO,

119 //任务的优先级

120 (CPU_STK \*)&AppTaskUsartStk[0],

121 //任务栈的基地址

122 (CPU_STK_SIZE) APP_TASK_USART_STK_SIZE / 10,

123 //任务栈空间剩下1/10时限制其增长

124 (CPU_STK_SIZE) APP_TASK_USART_STK_SIZE,

125 //任务栈空间（单位：sizeof(CPU_STK)）

126 (OS_MSG_QTY ) 50u,

127 //任务可接收的最大消息数

128 (OS_TICK ) 0u,

129 //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）

130 (void \*) 0,

131 //任务扩展（0表不扩展）

132 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

133 //任务选项

134 (OS_ERR \*)&err);

135 //返回错误类型

136

137

138 /\* 创建 AppTaskKey 任务 \*/

139 OSTaskCreate((OS_TCB \*)&AppTaskKeyTCB,

140 //任务控制块地址

141 (CPU_CHAR \*)"App Task Key",

142 //任务名称

143 (OS_TASK_PTR ) AppTaskKey,

144 //任务函数

145 (void \*) 0,

146 //传递给任务函数（形参p_arg）的实参

147 (OS_PRIO ) APP_TASK_KEY_PRIO,

148 //任务的优先级

149 (CPU_STK \*)&AppTaskKeyStk[0],

150 //任务栈的基地址

151 (CPU_STK_SIZE) APP_TASK_KEY_STK_SIZE / 10,

152 //任务栈空间剩下1/10时限制其增长

153 (CPU_STK_SIZE) APP_TASK_KEY_STK_SIZE,

154 //任务栈空间（单位：sizeof(CPU_STK)）

155 (OS_MSG_QTY ) 50u,

156 //任务可接收的最大消息数

157 (OS_TICK ) 0u,

158 //任务的时间片节拍数（0表默认值OSCfg_TickRate_Hz/10）

159 (void \*) 0,

160 //任务扩展（0表不扩展）

161 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

162 //任务选项

163 (OS_ERR \*)&err);

164 //返回错误类型

165

166

167 OSTaskDel ( 0, & err );

168 //删除起始任务本身，该任务不再运行

169

170

171 }

172

173

174

175 static void AppTaskUsart ( void \* p_arg )

176 {

177 OS_ERR err;

178

179 CPU_SR_ALLOC();

180

181

182 (void)p_arg;

183

184

185 while (DEF_TRUE) //任务体

186 {

187

188 OSTaskSemPend ((OS_TICK )0, //无期限等待

189 (OS_OPT )OS_OPT_PEND_BLOCKING,

190 //如果信号量不可用就等待

191 (CPU_TS \*)0,

192 //获取信号量被发布的时间戳

193 (OS_ERR \*)&err); //返回错误类型

194

195 OS_CRITICAL_ENTER();

196 //进入临界段，避免串口打印被打断

197

198 printf("收到数据:%s\n",Usart_Rx_Buf);

199

200 memset(Usart_Rx_Buf,0,USART_RBUFF_SIZE);/\* 清零 \*/

201

202 OS_CRITICAL_EXIT(); //退出临界段

203

204

205

206 }

207

208 }

209

210

211 static void AppTaskKey ( void \* p_arg )

212 {

213 OS_ERR err;

214 CPU_TS_TMR ts_int;

215 CPU_INT32U cpu_clk_freq;

216 CPU_SR_ALLOC();

217

218

219 (void)p_arg;

220

221

222 cpu_clk_freq = BSP_CPU_ClkFreq();

223 //获取CPU时钟，时间戳是以该时钟计数

224

225

226 while (DEF_TRUE) //任务体

227 {

228 /\* 阻塞任务，直到KEY1被按下 \*/

229 OSTaskSemPend ((OS_TICK )0, //无期限等待

230 (OS_OPT )OS_OPT_PEND_BLOCKING,

231 //如果信号量不可用就等待

232 (CPU_TS \*)0,

233 //获取信号量被发布的时间戳

234 (OS_ERR \*)&err); //返回错误类型

235

236 ts_int = CPU_IntDisMeasMaxGet (); //获取最大关中断时间

237

238 OS_CRITICAL_ENTER();

239 //进入临界段，避免串口打印被打断

240

241 printf ( "触发按键中断,最大中断时间是%dus\r\n",

242 ts_int / ( cpu_clk_freq / 1000000 ) );

243

244 OS_CRITICAL_EXIT(); //退出临界段

245

246 }

247

248 }

而中断服务函数则需要我们自己编写，并且中断被触发的时候通过信号量告知任务，具体见代码清单26‑9。

代码清单26‑9中断管理——中断服务函数

1 #include"stm32f10x_it.h"

2 #include <includes.h>

3 #include"bsp_usart1.h"

4 #include"bsp_exti.h"

5

6

7 extern OS_TCB AppTaskUsartTCB;

8 extern OS_TCB AppTaskKeyTCB;

9

10

11 /*\*

12 \* @brief USART 中断服务函数

13 \* @param 无

14 \* @retval 无

15 \*/

16 void macUSART_INT_FUN(void)

17 {

18 OS_ERR err;

19

20 OSIntEnter(); //进入中断

21

22

23 if ( USART_GetITStatus ( macUSARTx, USART_IT_IDLE ) != RESET )

24 {

25

26 DMA_Cmd(USART_RX_DMA_CHANNEL, DISABLE);

27

28 USART_ReceiveData ( macUSARTx ); /\* 清除标志位 \*/

29

30 // 清DMA标志位

31 DMA_ClearFlag( DMA1_FLAG_TC5 );

32 //

33 重新赋值计数值，必须大于等于最大可能接收到的数据帧数目

34 USART_RX_DMA_CHANNEL->CNDTR = USART_RBUFF_SIZE;

35 DMA_Cmd(USART_RX_DMA_CHANNEL, ENABLE);

36

37 //给出信号量，发送接收到新数据标志，供前台程序查询

38

39 /\* 发送任务信号量到任务 AppTaskKey \*/

40 OSTaskSemPost((OS_TCB \*)&AppTaskUsartTCB, //目标任务

41 (OS_OPT )OS_OPT_POST_NONE, //没选项要求

42 (OS_ERR \*)&err); //返回错误类型

43

44 }

45

46 OSIntExit(); //退出中断

47

48 }

49

50

51 /*\*

52 \* @brief EXTI 中断服务函数

53 \* @param 无

54 \* @retval 无

55 \*/

56 void macEXTI_INT_FUNCTION (void)

57 {

58 OS_ERR err;

59

60

61 OSIntEnter(); //进入中断

62

63 if (EXTI_GetITStatus(macEXTI_LINE) != RESET) //确保是否产生了EXTI Line中断

64 {

65 /\* 发送任务信号量到任务 AppTaskKey \*/

66 OSTaskSemPost((OS_TCB \*)&AppTaskKeyTCB, //目标任务

67 (OS_OPT )OS_OPT_POST_NONE, //没选项要求

68 (OS_ERR \*)&err); //返回错误类型

69

70 EXTI_ClearITPendingBit(macEXTI_LINE); //清除中断标志位

71 }

72

73 OSIntExit(); //退出中断

74

75 }

中断管理实验现象
~~~~~~~~

程序编译好，用USB线连接计算机和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），在计算机上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，按下开发板的KEY1按键
触发中断，在串口调试助手中可以看到运行结果，然后通过串口调试助手发送一段不定长信息，触发中断会在中断服务函数发送信号量通知任务，任务接收到信号量的时候将串口信息打印出来，具体见图26‑6。

|interr007|

图26‑6中断管理的实验现象

.. |interr002| image:: media\interr002.png
   :width: 5.40625in
   :height: 2.49583in
.. |interr003| image:: media\interr003.png
   :width: 4.81181in
   :height: 2.94792in
.. |interr004| image:: media\interr004.png
   :width: 5.76806in
   :height: 2.68542in
.. |interr005| image:: media\interr005.png
   :width: 5.76806in
   :height: 2.775in
.. |interr006| image:: media\interr006.png
   :width: 5.76806in
   :height: 2.44236in
.. |interr007| image:: media\interr007.png
   :width: 5.37639in
   :height: 4.31597in
