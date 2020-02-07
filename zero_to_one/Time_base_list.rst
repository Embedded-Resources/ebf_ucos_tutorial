.. vim: syntax=rst

实现时基列表
================
从本章开始，我们在OS中加入时基列表，时基列表是跟时间相关的，处于延时的任务和等待事件有超时限制的任务都会从就绪列表中移除，然后插入时基列表。时基列表在OSTimeTick中更新，如果任务的延时时间结束或者超时到期，就会让任务就绪，从时基列表移除，插入就绪列表。到目前为止，我们在OS中只实现了两个列
表，一个是就绪列表，一个是本章将要实现的时基列表，在本章之前，任务要么在就绪列表，要么在时基列表。

.. _实现时基列表-1:

实现时基列表
~~~~~~

定义时基列表变量
^^^^^^^^

时基列表在代码层面上由全局数组OSCfg_TickWheel[]和全局变量OSTickCtr构成，一个空的时基列表示意图见图10‑1，时基列表的代码实现具体见代码清单10‑1。

|Timeba002|

图10‑1空的时基列表

代码清单10‑1时基列表定义

1 /\* 时基列表大小，在os_cfg_app.h 定义 \*/

2 #define OS_CFG_TICK_WHEEL_SIZE 17u

3

4 /\* 在os_cfg_app.c 定义 \*/

5 /\* 时基列表 \*/

6 (1)(2)

7 **OS_TICK_SPOKE** OSCfg_TickWheel[**OS_CFG_TICK_WHEEL_SIZE**];

8 /\* 时基列表大小 \*/

9 OS_OBJ_QTY const OSCfg_TickWheelSize = (OS_OBJ_QTY )OS_CFG_TICK_WHEEL_SIZE;

10

11

12 /\* 在os.h中声明 \*/

13 /\* 时基列表 \*/

14 extern OS_TICK_SPOKE OSCfg_TickWheel[];

15 /\* 时基列表大小 \*/

16 extern OS_OBJ_QTY const OSCfg_TickWheelSize;

17

18

19 /\* Tick 计数器，在os.h中定义 \*/

20 OS_EXT OS_TICK **OSTickCtr**;(3)

代码清单10‑1（1）：OS_TICK_SPOKE为时基列表数组OSCfg_TickWheel[]的数据类型，在os.h文件定义，具体见代码清单10‑2。

代码清单10‑2OS_TICK_SPOKE定义

1 typedefstruct os_tick_spoke OS_TICK_SPOKE;(1)

2

3 struct os_tick_spoke {

4 OS_TCB \*FirstPtr;(2)

5 OS_OBJ_QTY NbrEntries;(3)

6 OS_OBJ_QTY NbrEntriesMax;(4)

7 };

代码清单10‑2（1）：在μC/OS-III中，内核对象的数据类型都会用大写字母重新定义。

代码清单10‑2（2）：时基列表OSCfg_TickWheel[]的每个成员都包含一条单向链表，被插入该条链表的TCB会按照延时时间做升序排列。FirstPtr用于指向这条单向链表的第一个节点。

代码清单10‑2（3）：时基列表OSCfg_TickWheel[]的每个成员都包含一条单向链表，NbrEntries表示该条单向链表当前有多少个节点。

代码清单10‑2（4）：时基列表OSCfg_TickWheel[]的每个成员都包含一条单向链表，NbrEntriesMax记录该条单向链表最多的时候有多少个节点，在增加节点的时候会刷新，在删除节点的时候不刷新。

代码清单10‑1（2）：OS_CFG_TICK_WHEEL_SIZE是一个宏，在os_cfg_app.h中定义，用于控制时基列表的大小。OS_CFG_TICK_WHEEL_SIZE的推荐值为任务数/4，不推荐使用偶数，如果算出来是偶数，则加1变成质数，实际上质数是一个很好的选择。

代码清单10‑1（3）：OSTickCtr为SysTick周期计数器，记录系统启动到现在或者从上一次复位到现在经过了多少个SysTick周期。

修改任务控制块TCB
^^^^^^^^^^

时基列表OSCfg_TickWheel[]的每个成员都包含一条单向链表，被插入该条链表的TCB会按照延时时间做升序排列，为了TCB能按照延时时间从小到大串接在一起，需要在TCB中加入几个成员，具体见代码清单10‑3的加粗部分。

|Timeba003|

图10‑2时基列表中有两个TCB

代码清单10‑3在TCB中加入时基列表相关字段

1 struct os_tcb {

2 CPU_STK \*StkPtr;

3 CPU_STK_SIZE StkSize;

4

5 /\* 任务延时周期个数 \*/

6 OS_TICK TaskDelayTicks;

7

8 /\* 任务优先级 \*/

9 OS_PRIO Prio;

10

11 /\* 就绪列表双向链表的下一个指针 \*/

12 OS_TCB \*NextPtr;

13 /\* 就绪列表双向链表的前一个指针 \*/

14 OS_TCB \*PrevPtr;

15

16 /*时基列表相关字段*/

**17 OS_TCB \*TickNextPtr;**\ (1)

**18 OS_TCB \*TickPrevPtr;**\ (2)

**19 OS_TICK_SPOKE \*TickSpokePtr;**\ (5)

**20**

**21 OS_TICK TickCtrMatch;**\ (4)

**22 OS_TICK TickRemain;**\ (3)

23 };

代码清单10‑3加粗部分的字段可以配合图10‑2一起理解，这样会比较容易。图10‑2是在时基列表OSCfg_TickWheel[]索引11这条链表里面插入了两个TCB，一个需要延时1个时钟周期，另外一个需要延时13个时钟周期。

代码清单10‑3（1）：TickNextPtr用于指向链表中的下一个TCB节点。

代码清单10‑3（2）：TickPrevPtr用于指向链表中的上一个TCB节点。

代码清单10‑3（3）：TickRemain用于设置任务还需要等待多少个时钟周期，每到来一个时钟周期，该值会递减。

代码清单10‑3（4）：TickCtrMatch的值等于时基计数器OSTickCtr的值加上TickRemain的值，当TickCtrMatch的值等于OSTickCtr的值的时候，表示等待到期，TCB会从链表中删除。

代码清单10‑3（5）：每个被插入链表的TCB都包含一个字段TickSpokePtr，用于回指到链表的根部。

实现时基列表相关函数
^^^^^^^^^^

时基列表相关函数在os_tick.c实现，在os.h中声明。如果os_tick.c文件是第一次使用，需要自行在文件夹μC/OS-III\Source中新建并添加到工程的μC/OS-III Source组。

OS_TickListInit()函数
'''''''''''''''''''

OS_TickListInit()函数用于初始化时基列表，即将全局变量OSCfg_TickWheel[]的数据域全部初始化为0，一个初始化为0的的时基列表见图10‑3。

代码清单10‑4OS_TickListInit()函数

1 /\* 初始化时基列表的数据域 \*/

2 void OS_TickListInit (void)

3 {

4 OS_TICK_SPOKE_IX i;

5 OS_TICK_SPOKE \*p_spoke;

6

7 for (i = 0u; i < OSCfg_TickWheelSize; i++) {

8 p_spoke = (OS_TICK_SPOKE \*)&OSCfg_TickWheel[i];

9 p_spoke->FirstPtr = (OS_TCB \*)0;

10 p_spoke->NbrEntries = (OS_OBJ_QTY )0u;

11 p_spoke->NbrEntriesMax = (OS_OBJ_QTY )0u;

12 }

13 }

|Timeba002|

图10‑3时基列表的数据域全部被初始化为0，即为空

OS_TickListInsert()函数
'''''''''''''''''''''

OS_TickListInsert()函数用于往时基列表中插入一个任务TCB，具体实现见代码清单10‑5。代码清单10‑5可配和图10‑4一起阅读，这样理解起来会容易很多。

|Timeba004|

图10‑4时基列表中有三个TCB

代码清单10‑5OS_TickListInsert()函数

1 /\* 将一个任务插入时基列表，根据延时时间的大小升序排列 \*/

2 void OS_TickListInsert (OS_TCB \*p_tcb,OS_TICK time)

3 {

4 OS_TICK_SPOKE_IX spoke;

5 OS_TICK_SPOKE \*p_spoke;

6 OS_TCB \*p_tcb0;

7 OS_TCB \*p_tcb1;

8

9 p_tcb->TickCtrMatch = OSTickCtr + time;(1)

10 p_tcb->TickRemain = time;(2)

11

12 spoke = (OS_TICK_SPOKE_IX)(p_tcb->TickCtrMatch % OSCfg_TickWheelSize);(3)

13 p_spoke = &OSCfg_TickWheel[spoke];(4)

14

15 /\* 插入 OSCfg_TickWheel[spoke] 的第一个节点 \*/

16 if (p_spoke->NbrEntries == (OS_OBJ_QTY)0u) {(5)

17 p_tcb->TickNextPtr = (OS_TCB \*)0;

18 p_tcb->TickPrevPtr = (OS_TCB \*)0;

19 p_spoke->FirstPtr = p_tcb;

20 p_spoke->NbrEntries = (OS_OBJ_QTY)1u;

21 }

22 /\* 如果插入的不是第一个节点，则按照TickRemain大小升序排列 \*/

23 else {(6)

24 /\* 获取第一个节点指针 \*/

25 p_tcb1 = p_spoke->FirstPtr;

26 while (p_tcb1 != (OS_TCB \*)0) {

27 /\* 计算比较节点的剩余时间 \*/

28 p_tcb1->TickRemain = p_tcb1->TickCtrMatch - OSTickCtr;

29

30 /\* 插入比较节点的后面 \*/

31 if (p_tcb->TickRemain > p_tcb1->TickRemain) {

32 if (p_tcb1->TickNextPtr != (OS_TCB \*)0) {

33 /\* 寻找下一个比较节点 \*/

34 p_tcb1 = p_tcb1->TickNextPtr;

35 } else { /\* 在最后一个节点插入 \*/

36 p_tcb->TickNextPtr = (OS_TCB \*)0;

37 p_tcb->TickPrevPtr = p_tcb1;

38 p_tcb1->TickNextPtr = p_tcb;

39 p_tcb1 = (OS_TCB \*)0;(7)

40 }

41 }

42 /\* 插入比较节点的前面 \*/

43 else {

44 /\* 在第一个节点插入 \*/

45 if (p_tcb1->TickPrevPtr == (OS_TCB \*)0) {

46 p_tcb->TickPrevPtr = (OS_TCB \*)0;

47 p_tcb->TickNextPtr = p_tcb1;

48 p_tcb1->TickPrevPtr = p_tcb;

49 p_spoke->FirstPtr = p_tcb;

50 } else {

51 /\* 插入两个节点之间 \*/

52 p_tcb0 = p_tcb1->TickPrevPtr;

53 p_tcb->TickPrevPtr = p_tcb0;

54 p_tcb->TickNextPtr = p_tcb1;

55 p_tcb0->TickNextPtr = p_tcb;

56 p_tcb1->TickPrevPtr = p_tcb;

57 }

58 /\* 跳出while循环 \*/

59 p_tcb1 = (OS_TCB \*)0;(8)

60 }

61 }

62

63 /\* 节点成功插入 \*/

64 p_spoke->NbrEntries++;(9)

65 }

66

67 /\* 刷新NbrEntriesMax的值 \*/

68 if (p_spoke->NbrEntriesMax < p_spoke->NbrEntries) {(10)

69 p_spoke->NbrEntriesMax = p_spoke->NbrEntries;

70 }

71

72 /\* 任务TCB中的TickSpokePtr回指根节点 \*/

73 p_tcb->TickSpokePtr = p_spoke;(11)

74 }

代码清单10‑5（1）：TickCtrMatch的值等于当前时基计数器的值OSTickCtr加上任务要延时的时间time，time由函数形参传进来。OSTickCtr是一个全局变量，记录的是系统自启动以来或者自上次复位以来经过了多少个SysTick周期。OSTickCtr的值每经过一个SysTick
周期其值就加一，当TickCtrMatch的值与其相等时，就表示任务等待时间到期。

代码清单10‑5（2）：将任务需要延时的时间time保存到TCB的TickRemain，它表示任务还需要延时多少个SysTick周期，每到来一个SysTick周期，TickRemain会减一。

代码清单10‑5（3）：由任务的TickCtrMatch 对时基列表的大小OSCfg_TickWheelSize进行求余操作，得出的值spoke作为时基列表OSCfg_TickWheel[]的索引。只要是任务的TickCtrMatch对OSCfg_TickWheelSize求余后得到的值spoke相
等，那么任务的TCB就会被插入OSCfg_TickWheel[spoke]下的单向链表中，节点按照任务的TickCtrMatch值做升序排列。举例：在图10‑4中，时基列表OSCfg_TickWheel[]的大小OSCfg_TickWheelSize等于12，当前时基计数器OSTickCtr的值为1
0，有三个任务分别需要延时TickTemain=1、TickTemain=23和TickTemain=25个时钟周期，三个任务的TickRemain加上OSTickCtr可分别得出它们的TickCtrMatch等于11、23和35，这三个任务的TickCtrMatch对OSCfg_TickWheel
Size求余操作后的值spoke都等于11，所以这三个任务的TCB会被插入OSCfg_TickWheel[11]下的同一条链表，节点顺序根据TickCtrMatch的值做升序排列。

代码清单10‑5（4）：根据刚刚算出的索引值spoke，获取到该索引值下的成员的地址，也叫根指针，因为该索引下对应的成员OSCfg_TickWheel[spoke]会维护一条双向的链表。

代码清单10‑5（5）：将TCB插入链表中分两种情况，第一是当前链表是空的，插入的节点将成为第一个节点，这个处理非常简单；第二是当前链表已经有节点。

代码清单10‑5（6）：当前的链表中已经有节点，插入的时候则根据TickCtrMatch的值做升序排列，插入的时候分三种情况，第一是在最后一个节点之间插入，第二是在第一个节点插入，第三是在两个节点之间插入。

代码清单10‑5（7）（8）：节点成功插入p_tcb1指针，跳出while循环

代码清单10‑5（9）：节点成功插入，记录当前链表节点个数的计数器NbrEntries加一。

代码清单10‑5（10）：刷新NbrEntriesMax的值,NbrEntriesMax用于记录当前链表曾经最多有多少个节点，只有在增加节点的时候才刷新，在删除节点的时候是不刷新的。

代码清单10‑5（11）：任务TCB被成功插入链表，TCB中的TickSpokePtr回指所在链表的根指针。

OS_TickListRemove()函数
'''''''''''''''''''''

OS_TickListRemove()用于从时基列表删除一个指定的TCB节点，具体实现见。代码清单10‑6

代码清单10‑6OS_TickListRemove()函数

1 /\* 从时基列表中移除一个任务 \*/

2 void OS_TickListRemove (OS_TCB \*p_tcb)

3 {

4 OS_TICK_SPOKE \*p_spoke;

5 OS_TCB \*p_tcb1;

6 OS_TCB \*p_tcb2;

7

8 /\* 获取任务TCB所在链表的根指针 \*/

9 p_spoke = p_tcb->TickSpokePtr;(1)

10

11 /\* 确保任务在链表中 \*/

12 if (p_spoke != (OS_TICK_SPOKE \*)0) {

13 /\* 将剩余时间清零 \*/

14 p_tcb->TickRemain = (OS_TICK)0u;

15

16 /\* 要移除的刚好是第一个节点 \*/

17 if (p_spoke->FirstPtr == p_tcb) {(2)

18 /\* 更新第一个节点，原来的第一个节点需要被移除 \*/

19 p_tcb1 = (OS_TCB \*)p_tcb->TickNextPtr;

20 p_spoke->FirstPtr = p_tcb1;

21 if (p_tcb1 != (OS_TCB \*)0) {

22 p_tcb1->TickPrevPtr = (OS_TCB \*)0;

23 }

24 }

25 /\* 要移除的不是第一个节点 \*/(3)

26 else {

27 /\* 保存要移除的节点的前后节点的指针 \*/

28 p_tcb1 = p_tcb->TickPrevPtr;

29 p_tcb2 = p_tcb->TickNextPtr;

30

31 /\* 节点移除，将节点前后的两个节点连接在一起 \*/

32 p_tcb1->TickNextPtr = p_tcb2;

33 if (p_tcb2 != (OS_TCB \*)0) {

34 p_tcb2->TickPrevPtr = p_tcb1;

35 }

36 }

37

38 /\* 复位任务TCB中时基列表相关的字段成员 \*/(4)

39 p_tcb->TickNextPtr = (OS_TCB \*)0;

40 p_tcb->TickPrevPtr = (OS_TCB \*)0;

41 p_tcb->TickSpokePtr = (OS_TICK_SPOKE \*)0;

42 p_tcb->TickCtrMatch = (OS_TICK )0u;

43

44 /\* 节点减1 \*/

45 p_spoke->NbrEntries--;(5)

46 }

47 }

代码清单10‑6（1）：获取任务TCB所在链表的根指针。

代码清单10‑6（2）：要删除的节点是链表的第一个节点，这个操作很好处理，只需更新下第一个节点即可。

代码清单10‑6（3）：要删除的节点不是链表的第一个节点，则先保存要删除的节点的前后节点，然后把这前后两个节点相连即可。

代码清单10‑6（4）：复位任务TCB中时基列表相关的字段成员。

代码清单10‑6（5）：节点删除成功，链表中的节点计数器NbrEntries减一。

OS_TickListUpdate()函数
'''''''''''''''''''''

OS_TickListUpdate()在每个SysTick周期到来时在OSTimeTick()被调用，用于更新时基计数器OSTickCtr，扫描时基列表中的任务延时是否到期，具体实现见代码清单10‑7。

代码清单10‑7OS_TickListUpdate()函数

1 void OS_TickListUpdate (void)

2 {

3 OS_TICK_SPOKE_IX spoke;

4 OS_TICK_SPOKE \*p_spoke;

5 OS_TCB \*p_tcb;

6 OS_TCB \*p_tcb_next;

7 CPU_BOOLEAN done;

8

9 CPU_SR_ALLOC();

10

11 /\* 进入临界段 \*/

12 OS_CRITICAL_ENTER();

13

14 /\* 时基计数器++ \*/

15 OSTickCtr++;(1)

16

17 spoke = (OS_TICK_SPOKE_IX)(OSTickCtr % OSCfg_TickWheelSize);(2)

18 p_spoke = &OSCfg_TickWheel[spoke];

19

20 p_tcb = p_spoke->FirstPtr;

21 done = DEF_FALSE;

22

23 while (done == DEF_FALSE) {

24 if (p_tcb != (OS_TCB \*)0) {(3)

25 p_tcb_next = p_tcb->TickNextPtr;

26

27 p_tcb->TickRemain = p_tcb->TickCtrMatch - OSTickCtr;(4)

28

29 /\* 节点延时时间到 \*/

30 if (OSTickCtr == p_tcb->TickCtrMatch) {(5)

31 /\* 让任务就绪 \*/

32 OS_TaskRdy(p_tcb);

33 } else {(6)

34 /\* 如果第一个节点延时期未满，则退出while循环

35 因为链表是根据升序排列的，第一个节点延时期未满，那后面的肯定未满 \*/

36 done = DEF_TRUE;

37 }

38

39 /\* 如果第一个节点延时期满，则继续遍历链表，看看还有没有延时期满的任务

40 如果有，则让它就绪 \*/

41 p_tcb = p_tcb_next;(7)

42 } else {

43 done = DEF_TRUE;(8)

44 }

45 }

46

47 /\* 退出临界段 \*/

48 OS_CRITICAL_EXIT();

49 }

代码清单10‑7（1）：每到来一个SysTick时钟周期，时基计数器OSTickCtr都要加一操作。

代码清单10‑7（2）：计算要扫描的时基列表的索引，每次只扫描一条链表。时基列表里面有可能有多条链表，为啥只扫描其中一条链表就可以？因为任务在插入时基列表的时候，插入的索引值spoke_insert是通过TickCtrMatch对OSCfg_TickWheelSize求余得出，现在需要扫描的索引值s
poke_update是通过OSTickCtr对OSCfg_TickWheelSize求余得出，TickCtrMatch的值等于OSTickCt加上TickRemain，只有在经过TickRemain个时钟周期后，spoke_update的值才有可能等于spoke_insert。如果算出的spoke
_update小于spoke_insert，且OSCfg_TickWheel[spoke_update]下的链表的任务没有到期，那后面的肯定都没有到期，不用继续扫描。

举例，在图10‑5，时基列表OSCfg_TickWheel[]的大小OSCfg_TickWheelSize等于12，当前时基计数器OSTickCtr的值为7，有三个任务分别需要延时TickTemain=16、TickTemain=28和TickTemain=40个时钟周期，三个任务的TickRema
in加上OSTickCtr可分别得出它们的TickCtrMatch等于23、35和47，这三个任务的TickCtrMatch对OSCfg_TickWheelSize求余操作后的值spoke都等于11，所以这三个任务的TCB会被插入OSCfg_TickWheel[11]下的同一条链表，节点顺序根据Ti
ckCtrMatch的值做升序排列。当下一个SysTick时钟周期到来的时候，会调用OS_TickListUpdate()函数，这时OSTickCtr加一操作后等于8，对OSCfg_TickWheelSize（等于12）求余算得要扫描更新的索引值spoke_update等8，则对OSCfg_Tick
Wheel[8]下面的链表进行扫描，从图10‑5可以得知，8这个索引下没有节点，则直接退出，刚刚插入的三个TCB是在OSCfg_TickWheel[11]下的链表，根本不用扫描，因为时间只是刚刚过了1个时钟周期而已，远远没有达到他们需要的延时时间。

代码清单10‑7（3）：判断链表是否为空，为空则跳转到第（8）步骤。

代码清单10‑7（4）：链表不为空，递减第一个节点的TickRemain。

代码清单10‑7（5）：判断第一个节点的延时时间是否到，如果到期，让任务就绪，即将任务从时基列表删除，插入就绪列表，这两步由函数OS_TaskRdy()来完成，该函数在os_core.c中定义，具体实现见代码清单10‑8。

代码清单10‑8OS_TaskRdy()函数

1 void OS_TaskRdy (OS_TCB \*p_tcb)

2 {

3 /\* 从时基列表删除 \*/

4 OS_TickListRemove(p_tcb);

5

6 /\* 插入就绪列表 \*/

7 OS_RdyListInsert(p_tcb);

8 }

代码清单10‑7（6）：如果第一个节点延时期未满，则退出while循环，因为链表是根据升序排列的，第一个节点延时期未满，那后面的肯定未满。

代码清单10‑7（7）：如果第一个节点延时到期，则继续判断下一个节点延时是否到期。

代码清单10‑7（8）：链表为空，退出扫描，因为其他还没到期。

|Timeba005|

图10‑5时基列表中有三个TCB

修改OSTimeDly()函数
~~~~~~~~~~~~~~~

加入时基列表之后，OSTimeDly()函数需要被修改，具体见代码清单10‑9的加粗部分，被迭代的代码已经用条件编译屏蔽。

代码清单10‑9OSTimeDly()函数

1 void OSTimeDly(OS_TICK dly)

2 {

3 CPU_SR_ALLOC();

4

5 /\* 进入临界区 \*/

6 OS_CRITICAL_ENTER();

7 #if 0

8 /\* 设置延时时间 \*/

9 OSTCBCurPtr->TaskDelayTicks = dly;

10

11 /\* 从就绪列表中移除 \*/

12 //OS_RdyListRemove(OSTCBCurPtr);

13 OS_PrioRemove(OSTCBCurPtr->Prio);

14 #endif

15

**16 /\* 插入时基列表 \*/**

**17 OS_TickListInsert(OSTCBCurPtr, dly);**

**18**

**19 /\* 从就绪列表移除 \*/**

**20 OS_RdyListRemove(OSTCBCurPtr);**

21

22 /\* 退出临界区 \*/

23 OS_CRITICAL_EXIT();

24

25 /\* 任务调度 \*/

26 OSSched();

27 }

修改OSTimeTick()函数
~~~~~~~~~~~~~~~~

加入时基列表之后，OSTimeTick()函数需要被修改，具体见代码清单10‑10的加粗部分，被迭代的代码已经用条件编译屏蔽。

代码清单10‑10OSTimeTick()函数

1 void OSTimeTick (void)

2 {

3 #if 0

4 unsigned int i;

5 CPU_SR_ALLOC();

6

7 /\* 进入临界区 \*/

8 OS_CRITICAL_ENTER();

9

10 for (i=0; i<OS_CFG_PRIO_MAX; i++) {

11 if (OSRdyList[i].HeadPtr->TaskDelayTicks > 0) {

12 OSRdyList[i].HeadPtr->TaskDelayTicks --;

13 if (OSRdyList[i].HeadPtr->TaskDelayTicks == 0) {

14 /\* 为0则表示延时时间到，让任务就绪 \*/

15 //OS_RdyListInsert (OSRdyList[i].HeadPtr);

16 OS_PrioInsert(i);

17 }

18 }

19 }

20

21 /\* 退出临界区 \*/

22 OS_CRITICAL_EXIT();

23

24 #endif

25

**26 /\* 更新时基列表 \*/**

**27 OS_TickListUpdate();**

28

29 /\* 任务调度 \*/

30 OSSched();

31 }

main 函数
~~~~~~~

main()函数同上一章一样。

实验现象
~~~~

实验现象同上一章一样，实验现象虽然一样，但是任务在就是延时状态时，任务的TCB不再继续放在就绪列表，而是放在了时基列表中。

.. |Timeba002| image:: media\Timeba002.png
   :width: 5.76806in
   :height: 1.64931in
.. |Timeba003| image:: media\Timeba003.png
   :width: 4.33333in
   :height: 1.75278in
.. |Timeba002| image:: media\Timeba002.png
   :width: 5.76806in
   :height: 1.64931in
.. |Timeba004| image:: media\Timeba004.png
   :width: 5.76806in
   :height: 2in
.. |Timeba005| image:: media\Timeba005.png
   :width: 5.76806in
   :height: 1.95347in
