.. vim: syntax=rst

就绪列表
=============

在μC/OS-III中，任务被创建后，任务的TCB会被放入就绪列表中，表示任务在就绪，随时可能被运行。就绪列表包含一个表示任务优先级的优先级表，一个存储任务TCB的TCB双向链表。

优先级表
~~~~

优先级表在代码层面上来看，就是一个数组，在文件os_prio.c（os_prio.c第一次使用需要自行在文件夹μC/OS-III\Source中新建并添加到工程的μC/OS-III Source组）的开头定义，具体见代码清单8‑1。

代码清单8‑1优先级表OSPrioTbl[]定义

1 /\* 定义优先级表，在os.h中用extern声明 \*/

2 CPU_DATA OSPrioTbl[OS_PRIO_TBL_SIZE];(1)

代码清单8‑1（1）：正如我们所说，优先级表是一个数组，数组类型为CPU_DATA，在Cortex-M内核芯片的MCU中CPU_DATA为32位整型。数组的大小由宏OS_PRIO_TBL_SIZE控制。OS_PRIO_TBL_SIZE的具体取值与μC/OS-
III支持多少个优先级有关，支持的优先级越多，优先级表也就越大，需要的RAM空间也就越多。理论上μC/OS-III支持无限的优先级，只要RAM控制足够。宏OS_PRIO_TBL_SIZE在os.h文件定义，具体实现见代码清单8‑2。

代码清单8‑2OS_PRIO_TBL_SIZE宏定义

1 (1) (2)

2 #define OS_PRIO_TBL_SIZE((OS_CFG_PRIO_MAX - 1u) / (DEF_INT_CPU_NBR_BITS) + 1u)

代码清单8‑2（1）：OS_CFG_PRIO_MAX表示支持多少个优先级，在os_cfg.h中定义，本书设置为32，即最大支持32个优先级。

代码清单8‑2（2）：DEF_INT_CPU_NBR_BITS定义CPU整型数据有多少位，本书适配的是基于Cortex-M系列的MCU，宏展开为32位。

所以，经过OS_CFG_PRIO_MAX和DEF_INT_CPU_NBR_BITS这两个宏展开运算之后，可得出OS_PRIO_TBL_SIZE的值为1，即优先级表只需要一个成员即可表示32个优先级。如果要支持64个优先级，即需要两个成员，以此类推。如果MCU的类型是16位、8位或者64位，只需要把优
先级表的数据类型CPU_DATA改成相应的位数即可。

那么优先级表又是如何跟任务的优先级联系在一起的？具体的优先级表的示意图见图8‑1。

|readyl002|

图8‑1优先级表

在图8‑1中，优先级表的成员是32位的，每个成员可以表示32个优先级。如果优先级超过32个，那么优先级表的成员就要相应的增加。以本书为例，CPU的类型为32位，支持最大的优先级为32个，优先级表只需要一个成员即可，即只有OSPrioTbl[0]。假如创建一个优先级为Prio的任务，那么就在OSPri
oTbl[0]的位[31-prio]置1即可。如果Prio等于3，那么就将位28置1。OSPrioTbl[0]的位31表示的是优先级最高的任务，以此递减，直到OSPrioTbl[OS_PRIO_TBL_SIZE-1]]的位0，OSPrioTbl[OS_PRIO_TBL_SIZE-1]]的位0表示的是
最低的优先级。

优先级表函数讲解
^^^^^^^^

优先级表相关的函数在os_prio.c文件中实现，在os.h文件中声明，函数汇总具体见表8‑1。

表8‑1优先级表相关函数汇总

================= ======================
函数名称          函数作用
================= ======================
OS_PrioInit       初始化优先级表
OS_PrioInsert     设置优先级表中相应的位
OS_PrioRemove     清除优先级表中相应的位
OS_PrioGetHighest 查找最高的优先级
================= ======================

OS_PrioInit()函数
'''''''''''''''

OS_PrioInit()函数用于初始化优先级表，在OSInit()函数中被调用，具体实现见代码清单8‑3。

代码清单8‑3OS_PrioInit()函数

1 /\* 初始化优先级表 \*/

2 void OS_PrioInit( void )

3 {

4 CPU_DATA i;

5

6 /\* 默认全部初始化为0 \*/

7 for ( i=0u; i<OS_PRIO_TBL_SIZE; i++ ) {

8 OSPrioTbl[i] = (CPU_DATA)0;

9 }

10 }

本书中，优先级表OS_PrioTbl[]只有一个成员，即OS_PRIO_TBL_SIZE等于1经过代码清单8‑3初始化之后，具体示意图见图8‑2。

|readyl003|

图8‑2优先级表初始化后的示意图

OS_PrioInsert()函数
'''''''''''''''''

OS_PrioInsert()函数用于置位优先级表中相应的位，会被OSTaskCreate()函数调用，具体实现见代码清单8‑4。

代码清单8‑4OS_PrioInsert()函数

1 /\* 置位优先级表中相应的位 \*/

2 void OS_PrioInsert (OS_PRIO prio)

3 {

4 CPU_DATA bit;

5 CPU_DATA bit_nbr;

6 OS_PRIO ix;

7

8

9 /\* 求模操作，获取优先级表数组的下标索引 \*/

10 ix = prio / DEF_INT_CPU_NBR_BITS;(1)

11

12 /\* 求余操作，将优先级限制在DEF_INT_CPU_NBR_BITS之内 \*/

13 bit_nbr = (CPU_DATA)prio & (DEF_INT_CPU_NBR_BITS - 1u);(2)

14

15 /\* 获取优先级在优先级表中对应的位的位置 \*/(3)

16 bit = 1u;

17 bit <<= (DEF_INT_CPU_NBR_BITS - 1u) - bit_nbr;

18

19 /\* 将优先级在优先级表中对应的位置1 \*/

20 OSPrioTbl[ix] \|= bit;(4)

21 }

代码清单8‑4（1）：求模操作，获取优先级表数组的下标索引。即定位prio这个优先级对应优先级表数组的哪个成员。假设prio等于3，DEF_INT_CPU_NBR_BITS（用于表示CPU一个整型数有多少位）等于32，那么ix就等于0，即对应OSPrioTBL[0]。

代码清单8‑4（2）：求余操作，将优先级限制在DEF_INT_CPU_NBR_BITS之内，超过DEF_INT_CPU_NBR_BITS的优先级就肯定要增加优先级表的数组成员了。假设prio等于3，DEF_INT_CPU_NBR_BITS（用于表示CPU一个整型数有多少位）等于32，那么bit_nb
r就等于3，但是这个还不是真正需要被置位的位。

代码清单8‑4（3）：获取优先级在优先级表中对应的位的位置。置位优先级对应的位是从高位开始的，不是从低位开始。位31对应的是优先级0，在μC/OS-
III中，优先级数值越小，逻辑优先级就越高。假设prio等于3，DEF_INT_CPU_NBR_BITS（用于表示CPU一个整型数有多少位）等于32，那么bit就等于28。

代码清单8‑4（4）：将优先级在优先级表中对应的位置1。假设prio等于3，DEF_INT_CPU_NBR_BITS（用于表示CPU一个整型数有多少位）等于32，那么置位的就是OSPrioTbl[0]的位28。

在优先级最大是32，DEF_INT_CPU_NBR_BITS等于32的情况下，如果分别创建了优先级3、5、8和11这四个任务，任务创建成功后，优先级表的设置情况是怎么样的？具体见图8‑3。有一点要注意的是，在μC/OS-III中，最高优先级和最低优先级是留给系统任务使用的，用户任务不能使用。

|readyl004|

图8‑3创建优先级3、5、8和11后优先级表的设置情况

OS_PrioRemove()函数
'''''''''''''''''

OS_PrioRemove()函数用于清除优先级表中相应的位，与OS_PrioInsert()函数的作用刚好相反，具体实现见代码清单8‑5，有关代码的讲解参考代码清单8‑4即可，不同的是置位操作改成了清零。

代码清单8‑5OS_PrioRemove()函数

1 /\* 清除优先级表中相应的位 \*/

2 void OS_PrioRemove (OS_PRIO prio)

3 {

4 CPU_DATA bit;

5 CPU_DATA bit_nbr;

6 OS_PRIO ix;

7

8

9 /\* 求模操作，获取优先级表数组的下标索引 \*/

10 ix = prio / DEF_INT_CPU_NBR_BITS;

11

12 /\* 求余操作，将优先级限制在DEF_INT_CPU_NBR_BITS之内 \*/

13 bit_nbr = (CPU_DATA)prio & (DEF_INT_CPU_NBR_BITS - 1u);

14

15 /\* 获取优先级在优先级表中对应的位的位置 \*/

16 bit = 1u;

17 bit <<= (DEF_INT_CPU_NBR_BITS - 1u) - bit_nbr;

18

19 /\* 将优先级在优先级表中对应的位清零 \*/

20 OSPrioTbl[ix] &= ~bit;

21 }

OS_PrioGetHighest()函数
'''''''''''''''''''''

OS_PrioGetHighest()函数用于从优先级表中查找最高的优先级，具体实现见代码清单8‑6。

代码清单8‑6OS_PrioGetHighest()函数

1 /\* 获取最高的优先级 \*/

2 OS_PRIO OS_PrioGetHighest (void)

3 {

4 CPU_DATA \*p_tbl;

5 OS_PRIO prio;

6

7

8 prio = (OS_PRIO)0;

9 /\* 获取优先级表首地址 \*/

10 p_tbl = &OSPrioTbl[0];(1)

11

12 /\* 找到数值不为0的数组成员 \*/(2)

13 while (*p_tbl == (CPU_DATA)0) {

14 prio += DEF_INT_CPU_NBR_BITS;

15 p_tbl++;

16 }

17

18 /\* 找到优先级表中置位的最高的优先级 \*/

19 prio += (OS_PRIO)CPU_CntLeadZeros(*p_tbl);(3)

20 return (prio);

21 }

代码清单8‑6（1）：获取优先级表的首地址，从头开始搜索整个优先级表，直到找到最高的优先级。

代码清单8‑6（2）：找到优先级表中数值不为0的数组成员，只要不为0就表示该成员里面至少有一个位是置位的。我们知道，在图8‑4的优先级表中，优先级按照从左到右，从上到下依次减小，左上角为最高的优先级，右下角为最低的优先级，所以我们只需要找到第一个不是0的优先级表成员即可。

代码清单8‑6（3）：确定好优先级表中第一个不为0的成员后，然后再找出该成员中第一个置1的位（从高位到低位开始找）就算找到最高优先级。在一个变量中，按照从高位到低位的顺序查找第一个置1的位的方法是通过计算前导0函数CPU_CntLeadZeros()来实现的。从高位开始找1叫计算前导0，从低位开始找
1叫计算后导0。如果分别创建了优先级3、5、8和11这四个任务，任务创建成功后，优先级表的设置情况具体见图8‑5。调用CPU_CntLeadZeros()可以计算出OSPrioTbl[0]第一个置1的位前面有3个0，那么这个3就是我们要查找的最高优先级，至于后面还有多少个位置1我们都不用管，只需要找
到第一个1即可。

|readyl005|

图8‑4优先级表

|readyl004|

图8‑5创建优先级3、5、8和11后优先级表的设置情况

CPU_CntLeadZeros()函数可由汇编或者C来实现，如果使用的处理器支持前导零指令CLZ，可由汇编来实现，加快指令运算，如果不支持则由C来实现。在μC/OS-
III中，这两种实现方法均有提供代码，到底使用哪种方法由CPU_CFG_LEAD_ZEROS_ASM_PRESEN这个宏来控制，定义了这个宏则使用汇编来实现，没有定义则使用C来实现。

Cortex-M系列处理器自带CLZ指令，所以CPU_CntLeadZeros()函数默认由汇编编写，具体在cpu_a.asm文件实现，在cpu.h文件声明，具体见代码清单8‑7。

代码清单8‑7CPU_CntLeadZeros()函数实现与声明

1 ;\*

2 ; PUBLIC FUNCTIONS

3 ;\*

4 EXPORT CPU_CntLeadZeros

5 EXPORT CPU_CntTrailZeros

6

7 ;\*

8 ; 计算前导0函数

9 ;

10 ; 描述：

11 ;

12 ; 函数声明： CPU_DATA CPU_CntLeadZeros(CPU_DATA val);

13 ;

14 ;\*

15 CPU_CntLeadZeros

16 CLZ R0, R0 ; Count leading zeros

17 BX LR

18

19

20

21 ;\*

22 ; 计算后导0函数

23 ;

24 ; 描述：

25 ;

26 ; 函数声明： CPU_DATA CPU_CntTrailZeros(CPU_DATA val);

27 ;

28 ;\*

29

30 CPU_CntTrailZeros

31 RBIT R0, R0 ; Reverse bits

32 CLZ R0, R0 ; Count trailing zeros

33 BX LR

1 /\*

2 \\*

3 \* 函数声明

4 \* cpu.h文件

5 \\*

6 \*/

7 #define CPU_CFG_LEAD_ZEROS_ASM_PRESEN

8 CPU_DATA CPU_CntLeadZeros (CPU_DATA val); /\* 在cpu_a.asm定义 \*/

9 CPU_DATA CPU_CntTrailZeros(CPU_DATA val); /\* 在cpu_a.asm定义 \*/

如果处理器不支持前导0指令，CPU_CntLeadZeros()函数就得由C编写，具体在cpu_core.c文件实现，在cpu.h文件声明，具体见代码清单8‑8。

代码清单8‑8由C实现的CPU_CntLeadZeros()函数

1 #ifndef CPU_CFG_LEAD_ZEROS_ASM_PRESENT

2 CPU_DATA CPU_CntLeadZeros (CPU_DATA val)

3 {

4 CPU_DATA nbr_lead_zeros;

5 CPU_INT08U ix;

6

7 /\* 检查高16位 \*/

8 if (val > 0x0000FFFFu) {(1)

9 /\* 检查 bits [31:24] : \*/

10 if (val > 0x00FFFFFFu) {(2)

11

12 /\* 获取bits [31:24]的值，并转换成8位 \*/

13 ix = (CPU_INT08U)(val >> 24u);(3)

14 /\* 查表找到优先级 \*/

15 nbr_lead_zeros=(CPU_DATA)(CPU_CntLeadZerosTbl[ix]+0u);(4)

16

17 }

18 /\* 检查 bits [23:16] : \*/

19 else {

20 /\* 获取bits [23:16]的值，并转换成8位 \*/

21 ix = (CPU_INT08U)(val >> 16u);

22 /\* 查表找到优先级 \*/

23 nbr_lead_zeros = (CPU_DATA )(CPU_CntLeadZerosTbl[ix] + 8u);

24 }

25

26 }

27 /\* 检查低16位 \*/

28 else {

29 /\* 检查 bits [15:08] : \*/

30 if (val > 0x000000FFu) {

31 /\* 获取bits [15:08]的值，并转换成8位 \*/

32 ix = (CPU_INT08U)(val >> 8u);

33 /\* 查表找到优先级 \*/

34 nbr_lead_zeros = (CPU_DATA )(CPU_CntLeadZerosTbl[ix] + 16u);

35

36 }

37 /\* 检查 bits [07:00] : \*/

38 else {

39 /\* 获取bits [15:08]的值，并转换成8位 \*/

40 ix = (CPU_INT08U)(val >> 0u);

41 /\* 查表找到优先级 \*/

42 nbr_lead_zeros = (CPU_DATA )(CPU_CntLeadZerosTbl[ix] + 24u);

43 }

44 }

45

46 /\* 返回优先级 \*/

47 return (nbr_lead_zeros);

48 }

49 #endif

在μC/OS-III中，由C实现的CPU_CntLeadZeros()函数支持8位、16位、32位和64位的变量的前导0计算，但最终的代码实现都是分离成8位来计算。这里我们只讲解32位的，其他几种情况都类似。

代码清单8‑8（1）：分离出高16位，else则为低16位。

代码清单8‑8（2）：分离出高16位的高8位，else则为高16位的低8位。

代码清单8‑8（3）：将高16位的高8位通过移位强制转化为8位的变量，用于后面的查表操作。

代码清单8‑8（4）：将8位的变量ix作为数组CPU_CntLeadZerosTbl[]的索引，返回索引对应的值，那么该值就是8位变量ix对应的前导0，然后再加上（24-右移的位数）就等于优先级。数组CPU_CntLeadZerosTbl[]在cpu_core.c的开头定义，具体见代码清单8‑9。

代码清单8‑9CPU_CntLeadZerosTbl[]定义

1 #ifndef CPU_CFG_LEAD_ZEROS_ASM_PRESENT

2 static const CPU_INT08U CPU_CntLeadZerosTbl[256] = {/\* 索引 \*/

3 8u,7u,6u,6u,5u,5u,5u,5u,4u,4u,4u,4u,4u,4u,4u,4u, /\* 0x00 to 0x0F \*/

4 3u,3u,3u,3u,3u,3u,3u,3u,3u,3u,3u,3u,3u,3u,3u,3u, /\* 0x10 to 0x1F \*/

5 2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u, /\* 0x20 to 0x2F \*/

6 2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u,2u, /\* 0x30 to 0x3F \*/

7 1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u, /\* 0x40 to 0x4F \*/

8 1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u, /\* 0x50 to 0x5F \*/

9 1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u, /\* 0x60 to 0x6F \*/

10 1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u,1u, /\* 0x70 to 0x7F \*/

11 0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u, /\* 0x80 to 0x8F \*/

12 0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u, /\* 0x90 to 0x9F \*/

13 0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u, /\* 0xA0 to 0xAF \*/

14 0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u, /\* 0xB0 to 0xBF \*/

15 0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u, /\* 0xC0 to 0xCF \*/

16 0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u, /\* 0xD0 to 0xDF \*/

17 0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u, /\* 0xE0 to 0xEF \*/

18 0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u,0u /\* 0xF0 to 0xFF \*/

19 };

20 #endif

代码清单8‑8中，对一个32位的变量算前导0个数的时候都是分离成8位的变量来计算，然后将这个8位的变量作为数组CPU_CntLeadZerosTbl[]的索引，索引下对应的值就是这个8位变量的前导0个数。一个8位的变量的取值范围为0~0XFF，这些值作为数组CPU_CntLeadZerosTbl[]
的索引，每一个值的前导0个数都预先算出来作为该数组索引下的值。通过查CPU_CntLeadZerosTbl[]这个表就可以很快的知道一个8位变量的前导0个数，根本不用计算，只是浪费了定义CPU_CntLeadZerosTbl[]这个表的一点点空间而已，在处理器内存很充足的情况下，则优先选择这种空间换
时间的方法。

.. _就绪列表-1:

就绪列表
~~~~

准备好运行的任务的TCB都会被放到就绪列表中，系统可随时调度任务运行。就绪列表在代码的层面上看就是一个OS_RDY_LIST数据类型的数组OSRdyList[]，数组的大小由宏OS_CFG_PRIO_MAX决定，支持多少个优先级，OSRdyList[]就有多少个成员。任务的优先级与OSRdyList
[]的索引一一对应，比如优先级3的任务的TCB会被放到OSRdyList[3]中。OSRdyList[]是一个在os.h文件中定义的全局变量，具体见代码清单8‑10。

代码清单8‑10 OSRdyList[]数组定义

/\* 就绪列表定义 \*/

1 OS_EXT OS_RDY_LIST OSRdyList[OS_CFG_PRIO_MAX];

代码清单8‑10中的数据类型OS_RDY_LIST在os.h中定义，专用于就绪列表，具体实现见代码清单8‑11。

代码清单8‑11OS_RDY_LIST定义

1 typedefstruct os_rdy_list OS_RDY_LIST;(1)

2

3 struct os_rdy_list {

4 OS_TCB \*HeadPtr;(2)

5 OS_TCB \*TailPtr;

6 OS_OBJ_QTY NbrEntries;(3)

7 };

代码清单8‑11（1）：在μC/OS-III中，内核对象的数据类型都会用大写字母重新定义。

代码清单8‑11（2）：OSRdyList[]的成员与任务的优先级一一对应，同一个优先级的多个任务会以双向链表的形式存在OSRdyList[]同一个索引下，那么HeadPtr就用于指向链表的头节点，TailPtr用于指向链表的尾节点，该优先级下的索引成员的地址则称为该优先级下双向链表的根节点，知道根
节点的地址就可以查找到该链表下的每一个节点。

代码清单8‑11（3）：NbrEntries表示OSRdyList[]同一个索引下有多少个任务。

一个空的就绪列表，OSRdyList[]索引下的HeadPtr、TailPtr和NbrEntrie都会被初始化为0，具体见图8‑6。

|readyl006|

图8‑6空的就绪列表

就绪列表相关的所有函数都在os_core.c实现，这些函数都是以“OS_”开头，表示是OS的内部函数，用户不能调用，这些函数的汇总具体见表8‑2。

表8‑2就绪列表相关函数汇总

======================== =============================
函数名称                 函数作用
======================== =============================
OS_RdyListInit           初始化就绪列表为空
OS_RdyListInsert         插入一个TCB到就绪列表
OS_RdyListInsertHead     插入一个TCB到就绪列表的头部
OS_RdyListInsertTail     插入一个TCB到就绪列表的尾部
OS_RdyListMoveHeadToTail 将TCB从就绪列表的头部移到尾部
OS_RdyListRemove         将TCB从就绪列表中移除
======================== =============================

就绪列表函数讲解
^^^^^^^^

在实现就绪列表相关函数之前，我们需要在结构体os_tcb中添加Prio、NextPtr和PrevPtr这三个成员，然后在os.h中定义两个全局变量OSPrioCur和OSPrioHighRdy，具体定义见代码清单8‑12。接下来要实现的就绪列表相关的函数会用到几个变量。

代码清单8‑12就绪列表函数需要用到的变量定义

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

15 };

16

17 /\* 在os.h中定义 \*/

18 OS_EXT OS_PRIO OSPrioCur; /\* 当前优先级 \*/

19 OS_EXT OS_PRIO OSPrioHighRdy; /\* 最高优先级 \*/

OS_RdyListInit()函数
''''''''''''''''''

OS_RdyListInit()用于将就绪列表OSRdyList[]初始化为空，初始化完毕之后具体示意图见图8‑6，具体实现见代码清单8‑13。

代码清单8‑13OS_RdyListInit()函数

1 void OS_RdyListInit(void)

2 {

3 OS_PRIO i;

4 OS_RDY_LIST \*p_rdy_list;

5

6 /\* 循环初始化，所有成员都初始化为0 \*/

7 for ( i=0u; i<OS_CFG_PRIO_MAX; i++ ) {

8 p_rdy_list = &OSRdyList[i];

9 p_rdy_list->NbrEntries = (OS_OBJ_QTY)0;

10 p_rdy_list->HeadPtr = (OS_TCB \*)0;

11 p_rdy_list->TailPtr = (OS_TCB \*)0;

12 }

13 }

OS_RdyListInsertHead()函数
''''''''''''''''''''''''

OS_RdyListInsertHead()用于在链表头部插入一个TCB节点，插入的时候分两种情况，第一种是链表是空链表，第二种是链表中已有节点，具体示意图见图8‑7，具体的代码实现见代码清单8‑14，阅读代码的时候最好配套示意图来理解。

|readyl007|

图8‑7在链表的头部插入一个TCB节点前链表的可能情况

代码清单8‑14OS_RdyListInsertHead()函数

1 void OS_RdyListInsertHead (OS_TCB \*p_tcb)

2 {

3 OS_RDY_LIST \*p_rdy_list;

4 OS_TCB \*p_tcb2;

5

6

7

8 /\* 获取链表根部 \*/

9 p_rdy_list = &OSRdyList[p_tcb->Prio];

10

11 /\* CASE 0: 链表是空链表 \*/

12 if (p_rdy_list->NbrEntries == (OS_OBJ_QTY)0) {

13 p_rdy_list->NbrEntries = (OS_OBJ_QTY)1;

14 p_tcb->NextPtr = (OS_TCB \*)0;

15 p_tcb->PrevPtr = (OS_TCB \*)0;

16 p_rdy_list->HeadPtr = p_tcb;

17 p_rdy_list->TailPtr = p_tcb;

18 }

19 /\* CASE 1: 链表已有节点 \*/

20 else {

21 p_rdy_list->NbrEntries++;

22 p_tcb->NextPtr = p_rdy_list->HeadPtr;

23 p_tcb->PrevPtr = (OS_TCB \*)0;

24 p_tcb2 = p_rdy_list->HeadPtr;

25 p_tcb2->PrevPtr = p_tcb;

26 p_rdy_list->HeadPtr = p_tcb;

27 }

28 }

OS_RdyListInsertTail()函数
''''''''''''''''''''''''

OS_RdyListInsertTail()用于在链表尾部插入一个TCB节点，插入的时候分两种情况，第一种是链表是空链表，第二种是链表中已有节点，具体示意图见图8‑8，具体的代码实现见，阅读代码的时候最好配套示意图来理解。

|readyl008|

图8‑8在链表的尾部插入一个TCB节点前链表的可能情况

代码清单8‑15OS_RdyListInsertTail()函数

1 void OS_RdyListInsertTail (OS_TCB \*p_tcb)

2 {

3 OS_RDY_LIST \*p_rdy_list;

4 OS_TCB \*p_tcb2;

5

6

7 /\* 获取链表根部 \*/

8 p_rdy_list = &OSRdyList[p_tcb->Prio];

9

10 /\* CASE 0: 链表是空链表 \*/

11 if (p_rdy_list->NbrEntries == (OS_OBJ_QTY)0) {

12 p_rdy_list->NbrEntries = (OS_OBJ_QTY)1;

13 p_tcb->NextPtr = (OS_TCB \*)0;

14 p_tcb->PrevPtr = (OS_TCB \*)0;

15 p_rdy_list->HeadPtr = p_tcb;

16 p_rdy_list->TailPtr = p_tcb;

17 }

18 /\* CASE 1: 链表已有节点 \*/

19 else {

20 p_rdy_list->NbrEntries++;

21 p_tcb->NextPtr = (OS_TCB \*)0;

22 p_tcb2 = p_rdy_list->TailPtr;

23 p_tcb->PrevPtr = p_tcb2;

24 p_tcb2->NextPtr = p_tcb;

25 p_rdy_list->TailPtr = p_tcb;

26 }

27 }

OS_RdyListInsert()函数
''''''''''''''''''''

OS_RdyListInsert()用于将任务的TCB插入就绪列表，插入的时候分成两步，第一步是根据优先级将优先级表中的相应位置位，这个调用OS_PrioInsert()函数来实现，第二步是根据优先级将任务的TCB放到OSRdyList[优先级]中，如果优先级等于当前的优先级则插入链表的尾部，否则插
入链表的头部，具体实现见代码清单8‑16。

代码清单8‑16OS_RdyListInsert()函数

1 /\* 在就绪链表中插入一个TCB \*/

2 void OS_RdyListInsert (OS_TCB \*p_tcb)

3 {

4 /\* 将优先级插入优先级表 \*/

5 OS_PrioInsert(p_tcb->Prio);

6

7 if (p_tcb->Prio == OSPrioCur) {

8 /\* 如果是当前优先级则插入链表尾部 \*/

9 OS_RdyListInsertTail(p_tcb);

10 } else {

11 /\* 否则插入链表头部 \*/

12 OS_RdyListInsertHead(p_tcb);

13 }

14 }

OS_RdyListMoveHeadToTail()函数
''''''''''''''''''''''''''''

OS_RdyListMoveHeadToTail()函数用于将节点从链表头部移动到尾部，移动的时候分四种情况，第一种是链表为空，无事可做；第二种是链表只有一个节点，也是无事可做；第三种是链表只有两个节点；第四种是链表有两个以上节点，具体示意图见图8‑9，具体代码实现见代码清单8‑17，阅读代码的时候
最好配套示意图来理解。

|readyl009|

图8‑9将节点从链表头部移动到尾部前链表的可能情况

代码清单8‑17OS_RdyListMoveHeadToTail()函数

1 void OS_RdyListMoveHeadToTail (OS_RDY_LIST \*p_rdy_list)

2 {

3 OS_TCB \*p_tcb1;

4 OS_TCB \*p_tcb2;

5 OS_TCB \*p_tcb3;

6

7

8

9 switch (p_rdy_list->NbrEntries) {

10 case 0:

11 case 1:

12 break;

13

14 case 2:

15 p_tcb1 = p_rdy_list->HeadPtr;

16 p_tcb2 = p_rdy_list->TailPtr;

17 p_tcb1->PrevPtr = p_tcb2;

18 p_tcb1->NextPtr = (OS_TCB \*)0;

19 p_tcb2->PrevPtr = (OS_TCB \*)0;

20 p_tcb2->NextPtr = p_tcb1;

21 p_rdy_list->HeadPtr = p_tcb2;

22 p_rdy_list->TailPtr = p_tcb1;

23 break;

24

25 default:

26 p_tcb1 = p_rdy_list->HeadPtr;

27 p_tcb2 = p_rdy_list->TailPtr;

28 p_tcb3 = p_tcb1->NextPtr;

29 p_tcb3->PrevPtr = (OS_TCB \*)0;

30 p_tcb1->NextPtr = (OS_TCB \*)0;

31 p_tcb1->PrevPtr = p_tcb2;

32 p_tcb2->NextPtr = p_tcb1;

33 p_rdy_list->HeadPtr = p_tcb3;

34 p_rdy_list->TailPtr = p_tcb1;

35 break;

36 }

37 }

OS_RdyListRemove()函数
''''''''''''''''''''

OS_RdyListRemove()函数用于从链表中移除一个节点，移除的时候分为三种情况，第一种是链表为空，无事可做；第二种是链表只有一个节点；第三种是链表有两个以上节点，具体示意图见图8‑10，具体代码实现见，阅读代码的时候最好配套示意图来理解。

|readyl010|

图8‑10从链表中移除一个节点前链表的可能情况

代码清单8‑18OS_RdyListRemove()函数

1 void OS_RdyListRemove (OS_TCB \*p_tcb)

2 {

3 OS_RDY_LIST \*p_rdy_list;

4 OS_TCB \*p_tcb1;

5 OS_TCB \*p_tcb2;

6

7

8

9 p_rdy_list = &OSRdyList[p_tcb->Prio];

10

11 /\* 保存要删除的TCB节点的前一个和后一个节点 \*/

12 p_tcb1 = p_tcb->PrevPtr;

13 p_tcb2 = p_tcb->NextPtr;

14

15 /\* 要移除的TCB节点是链表中的第一个节点 \*/

16 if (p_tcb1 == (OS_TCB \*)0) {

17 /\* 且该链表中只有一个节点 \*/

18 if (p_tcb2 == (OS_TCB \*)0) {

19 /\* 根节点全部初始化为0 \*/

20 p_rdy_list->NbrEntries = (OS_OBJ_QTY)0;

21 p_rdy_list->HeadPtr = (OS_TCB \*)0;

22 p_rdy_list->TailPtr = (OS_TCB \*)0;

23

24 /\* 清除在优先级表中相应的位 \*/

25 OS_PrioRemove(p_tcb->Prio);

26 }

27 /\* 该链表中不止一个节点 \*/

28 else {

29 /\* 节点减1 \*/

30 p_rdy_list->NbrEntries--;

31 p_tcb2->PrevPtr = (OS_TCB \*)0;

32 p_rdy_list->HeadPtr = p_tcb2;

33 }

34 }

35 /\* 要移除的TCB节点不是链表中的第一个节点 \*/

36 else {

37 p_rdy_list->NbrEntries--;

38 p_tcb1->NextPtr = p_tcb2;

39

40 /\* 如果要删除的节点的下一个节点是0，即要删除的节点是最后一个节点 \*/

41 if (p_tcb2 == (OS_TCB \*)0) {

42 p_rdy_list->TailPtr = p_tcb1;

43 } else {

44 p_tcb2->PrevPtr = p_tcb1;

45 }

46 }

47

48 /\* 复位从就绪列表中删除的TCB的PrevPtr和NextPtr这两个指针 \*/

49 p_tcb->PrevPtr = (OS_TCB \*)0;

50 p_tcb->NextPtr = (OS_TCB \*)0;

51 }

main()函数
~~~~~~~~

本章main()函数没有添加新的测试代码，只需理解章节内容即可。

实验现象
~~~~

本章没有实验，只需理解章节内容即可。

.. |readyl002| image:: media\readyl002.png
   :width: 5.76806in
   :height: 1.35903in
.. |readyl003| image:: media\readyl003.png
   :width: 5.76806in
   :height: 0.77778in
.. |readyl004| image:: media\readyl004.png
   :width: 5.76806in
   :height: 0.77778in
.. |readyl005| image:: media\readyl005.png
   :width: 5.76806in
   :height: 1.35903in
.. |readyl004| image:: media\readyl004.png
   :width: 5.76806in
   :height: 0.77778in
.. |readyl006| image:: media\readyl006.png
   :width: 3.75694in
   :height: 2.08611in
.. |readyl007| image:: media\readyl007.png
   :width: 4.16806in
   :height: 3.09514in
.. |readyl008| image:: media\readyl008.png
   :width: 4.36319in
   :height: 3.23403in
.. |readyl009| image:: media\readyl009.png
   :width: 4.43472in
   :height: 2.92222in
.. |readyl010| image:: media\readyl010.png
   :width: 4.71389in
   :height: 2.73472in
