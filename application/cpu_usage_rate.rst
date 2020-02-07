.. vim: syntax=rst

CPU利用率及栈检测统计
=======================

CPU利用率的基本概念及作用
~~~~~~~~~~~~~~

CPU利用率其实就是系统运行的程序占用的CPU资源，表示机器在某段时间程序运行的情况，如果这段时间中，程序一直在占用CPU的使用权，那么可以认为CPU的利用率是100%。CPU的利用率越高，说明机器在这个时间上运行了很多程序，反之较少。利用率的高低与CPU性能强弱有直接关系，就像一段一模一样的程序，
如果使用运算速度很慢的CPU，它可能要运行1000ms，而使用很运算速度很快的CPU可能只需要10ms，那么在1000ms这段时间中，前者的CPU利用率就是100%，而后者的CPU利用率只有1%，因为1000ms内前者都在使用CPU做运算，而后者只使用10ms的时间做运算，剩下的时间CPU可以做其他
事情。

μC/OS是多任务操作系统，对 CPU 都是分时使用的：比如A任务占用10ms，然后B任务占用30ms，然后空闲60ms，再又是A任务占10ms，B任务占30ms，空闲60ms；如果在一段时间内都是如此，那么这段时间内的利用率为40%，因为整个系统中只有40%的时间是CPU处理数据的时间。

一个系统设计的好坏，可以使用CPU利用率来衡量，一个好的系统必然是能完美响应急需的处理，并且系统的资源不会过于浪费（性价比高）。举个例子，假设一个系统的CPU利用率经常在90%~100%徘徊，那么系统就很少有空闲的时候，这时候突然有一些事情急需CPU的处理，但是此时CPU都很可能被其他任务在占用了，
那么这个紧急事件就有可能无法被相应，即启用被相应，那么占用CPU的任务又处于等待状态，这种系统就是不够完美的，因为资源处理得太过于紧迫；反过来，假如CPU的利用率在1%以下，那么我们就可以认为这种产品的资源过于浪费，搞一个那么好的CPU去干着没啥意义的活（大部分时间处于空闲状态），作为产品的设计，既
不能让资源过于浪费，也不能让资源过于紧迫，这种设计才是完美的，在需要的时候能及时处理完突发事件，而且资源也不会过剩，性价比更高。

μC/OS提供的CPU利用率统计是一个可选功能，只有将OS_CFG_STAT_TASK_EN宏定义启用后用户才能使用CPU利用率统计相关函数，该宏定义位于os_cfg.h文件中。

CPU利用率统计初始化
~~~~~~~~~~~

μC/OS对CPU利用率进行统计是怎么实现的呢？简单来说，CPU利用率统计的原理很简单，我们知道，系统中必须存在空闲任务，当且仅当CPU空闲的时候才会去执行空闲任务，那么我们就可以让CPU在空闲任务中一直做加法运算，假设某段时间T中CPU一直都在空闲任务中做加法运算（变量自加），那么这段时间算出来的
值就是CPU空闲时候的最大值，我们假设为100，那么当系统中有其他任务的时候，CPU就不可能一直处于空闲任务做运算了，那么同样的一段时间T里，空闲任务算出来的值变成了80，那么是不是可以说明空闲任务只占用了系统的80%的资源，剩下的20%被其他任务占用了，这是显而易见的，同样的，利用这个原理，我们就
能知道CPU的利用率大约是多少了（这种计算不会很精确），假设CPU在T时间内空闲任务中运算的最大值为OSStatTaskCtrMax（100），而有其他任务参与时候T时间内空闲任务运算的值为80（OSStatTaskCtr），那么CPU的利用率CPUUsage的公式应该为：CPUUsage（%） =
100*（1- OSStatTaskCtr /
OSStatTaskCtrMax），假设有一次空闲任务运算的值为100（OSStatTaskCtr），说明没有其他任务参与，那么CPU的利用率就是0%，如果OSStatTaskCtr的值为0，那么表示这段时间里CPU都没在空闲任务中运算，那么CPU的利用率自然就是100%。

注意：一般情况下时间T由OS_CFG_STAT_TASK_RATE_HZ宏定义决定，是我们自己在os_cfg_app.h文件中定义的，我们的例程定义为 10，该宏定义决定了统计任务的执行频率，即决定了更新一次 CPU 利用率的时间为 1/
OS_CFG_STAT_TASK_RATE_HZ，单位是秒。此外，统计任务的时钟节拍与软件定时器任务的时钟节拍一样，都是由系统时钟节拍分频得到的，如果统计任务运行的频率设定不是时钟节拍整数倍，那么统计任务实际运行的频率跟设定的就会有误差，这点跟定时器是一样的。

在统计CPU 利用率之前必须先调用OSStatTaskCPUUsageInit()函数进行相关初始化，这个函数的目的就是为了计算只有空闲任务时CPU在某段时间内的运算最大值，也就是OSStatTaskCtrMax，其源码具体见代码清单27‑1。

代码清单27‑1OSStatTaskCPUUsageInit()源码

1 void OSStatTaskCPUUsageInit (OS_ERR \*p_err)

2 {

3 OS_ERR err;

4 OS_TICK dly;

5 CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和

6 //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器

7 // SR（临界段关中断只需保存SR），开中断时将该值还原。

8

9 #ifdef OS_SAFETY_CRITICAL//如果启用了安全检测

10 if (p_err == (OS_ERR \*)0) //如果 p_err 为空

11 {

12 OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数

13 return; //返回，停止执行

14 }

15 #endif

16

17 #if (OS_CFG_TMR_EN > 0u)//如果启用了软件定时器

18 OSTaskSuspend(&OSTmrTaskTCB, &err); **(1)**//挂起软件定时任务

19 if (err != OS_ERR_NONE) //如果挂起失败

20 {

21 \*p_err = err; //返回失败原因

22 return; //返回，停止执行

23 }

24 #endif

25

26 OSTimeDly((OS_TICK )2,

27 //先延时两个节拍，为后面延时同步时钟节拍，增加准确性

28 (OS_OPT )OS_OPT_TIME_DLY,

29 (OS_ERR \*)&err); **(2)**

30 if (err != OS_ERR_NONE) //如果延时失败

31 {

32 \*p_err = err; //返回失败原因

33 return; //返回，停止执行

34 }

35 CPU_CRITICAL_ENTER(); //关中断

36 OSStatTaskCtr = (OS_TICK)0; //清零空闲计数器

37 CPU_CRITICAL_EXIT(); //开中断

38 /\* 根据设置的宏计算统计任务的执行节拍数 \*/

39 dly = (OS_TICK)0; **(3)**

40 if (OSCfg_TickRate_Hz > OSCfg_StatTaskRate_Hz)

41 {

42 dly = (OS_TICK)(OSCfg_TickRate_Hz / OSCfg_StatTaskRate_Hz);

43 }

44 if (dly == (OS_TICK)0)

45 {

46 dly = (OS_TICK)(OSCfg_TickRate_Hz / (OS_RATE_HZ)10);

47 }

48 /\* 延时累加空闲计数器，获取最大空闲计数值 \*/

49 OSTimeDly(dly,

50 OS_OPT_TIME_DLY,

51 &err); **(4)**

52

53 #if (OS_CFG_TMR_EN > 0u)//如果启用了软件定时器

54 OSTaskResume(&OSTmrTaskTCB, &err); **(5)**//恢复软件定时器任务

55 if (err != OS_ERR_NONE) //如果恢复失败

56 {

57 \*p_err = err; //返回错误原因

58 return; //返回，停止执行

59 }

60 #endif

61 /\* 如果上面没产生错误 \*/

62 CPU_CRITICAL_ENTER(); //关中断

63 OSStatTaskTimeMax = (CPU_TS)0; //

64

65 OSStatTaskCtrMax = OSStatTaskCtr; **(6)**//存储最大空闲计数值

66 OSStatTaskRdy = OS_STATE_RDY; **(7)**//准备就绪统计任务

67 CPU_CRITICAL_EXIT(); //开中断

68 \*p_err = OS_ERR_NONE; //错误类型为“无错误”

69 }

代码清单27‑1\ **(1)**\ ：如果启用了软件定时器，那么在系统初始化的时候就会创建软件定时器任务，此处不希望别的任务打扰空闲任务的运算，就暂时将软件定时器任务挂起。

代码清单27‑1\ **(2)**\ ：先延时两个节拍，为后面延时同步时钟节拍，增加准确性，为什么要先延时两个节拍呢？因为是为了匹配后面一个延时的时间起点，当两个时钟节拍到达后，再继续延时dly个时钟节拍，这样子时间就比较精确，程序执行到这里的时候，我们并不知道时间过去了多少，所以此时的延时起点并不
一定与系统的时钟节拍匹配，具体见图27‑1。

代码清单27‑1\ **(3)**\ ：根据设置的宏计算统计任务的执行节拍数，也就是T时间。

代码清单27‑1\ **(4)**\ ：延时dly个时钟节拍（这个时钟节拍的延时会比较准确），将当前任务阻塞，让空闲做累加运算，获取最大空闲运算数值OSStatTaskCtrMax。

代码清单27‑1\ **(5)**\ ：恢复软件定时器任务。

代码清单27‑1\ **(6)**\ ：保存一下空闲任务最大的运算数值OSStatTaskCtrMax

代码清单27‑1\ **(7)**\ ：准备就绪统计任务。

|cpuusa002|

图27‑1延时误差分析

注意，调用OSStatTaskCPUUsageInit()函数进行初始化的时候，一定要在创建用户任务之前，否则当系统有很多任务在调度的时候，空闲任务就没法在某段时间内完成运算并且得到准确的OSStatTaskCtrMax，这样子的CPU利用率计算是不准确的。

注意：统计的过程在后文讲解。

栈溢出检测概念及作用
~~~~~~~~~~

如果处理器有MMU或者MPU，检测栈是否溢出是非常简单的，MMU和MPU是处理器上特殊的硬件设施，可以检测非法访问，如果任务企图访问未被允许的内存空间的话，就会产生警告，但是我们使用的STM32是没有MMU和MPU的，但是可以使用软件模拟栈检测，但是软件的模拟比较难以实现，但是μC/OS为我们提供了
栈使用情况统计的功能，直接使用即可，如果需要使用栈溢出检测的功能，就需要用户自己在App_OS_TaskSwHook()钩子函数中自定义实现（我们不实现该功能），需要使用μC/OS为我们提供的栈检测功能，想要使用该功能就需要在os_cfg_app.h文件中将OS_CFG_STAT_TASK_STK_
CHK_EN宏定义配置为1。

某些处理器中有一些栈溢出检测相关的寄存器，当CPU的栈指针小于（或大于，取决于栈的生长方向）设置于这个寄存器的值时，就会产生一个异常（中断），异常处理程序就需要确保未允许访问空间代码的安全（可能会发送警告给用户，或者其他处理）。任务控制块中的成员变量StkLimitPtr就是为这种目的而设置的，如图
27‑2所示。每个任务的栈必须分配足够大的内存空间供任务使用，在大多数情况下，StkLimitPtr指针的值可以设置接近于栈顶（&TaskStk[0]，假定栈是从高地址往低地址生长的，事实上STM32的栈生长方向就是向下生长，从高地址向低地址生长），StkLimitPtr的值在创建任务的时候由用户指
定。

|cpuusa003|

图27‑2栈溢出检测（硬件）

注意：此处的栈检测是对于带有MPU的处理器。

那么μC/OS中对于没有MPU的处理器是怎么做到栈检测的呢？

当μC/OS从一个任务切换到另一个任务的时候，它会调用一个钩子函数OSTaskSwHook()，它允许用户扩展上下文切换时的功能。所以，如果处理器没有硬件支持溢出检测功能，就可以在该钩子函数中添加代码软件模拟该功能。在切换到任务B前，我们需要检测将要被载入CPU栈指针的值是否超出该任务B的任务控制块
中StkLimitPtr的限制。因为软件不能在溢出时就迅速地做出反应，所以应该设置StkLimitPtr的值尽可能远离栈顶，保证有足够的溢出缓冲，具体见。软件检测不会像硬件检测那样有效，但也可以有效防止栈溢出。

|cpuusa004|

图27‑3栈溢出检测（软件）

栈溢出检测过程
~~~~~~~

在前面的章节中我们已经详细讲解了栈相关的知识，每个任务独立的栈空间对任务来说是至关重要的，栈空间中保存了任务运行过程中需要保存局部变量、寄存器等重要的信息，如果设置的栈太小，任务无法正常运行，可能还会出现各种奇怪的错误，如果发现我们的程序出现奇怪的错误，一定要检查栈空间，包括 MSP
的栈，系统任务的栈，用户任务的栈。

μC/OS是怎么检测任务使用了多少栈的呢？以STM32的栈生长方向为例子（高地址向低地址生长），在任务初始化的时候先将任务所有的栈都置 0，使用后的栈不为
0，在检测的时候只需从栈的低地址开始对为0的栈空间进行计数统计，然后通过计算就可以得出任务的栈使用了多少，这样子用户就可以根据实际情况进行调整任务栈的大小，具体见图27‑4，这些信息同样也会在统计任务每隔 1/OSCfg_StatTaskRate_Hz 秒就进行更新。

|cpuusa005|

图27‑4栈检测示意图

统计任务OS_StatTask()
~~~~~~~~~~~~~~~~~

μC/OS提供了统计任务的函数，该函数为系统内部函数（任务），在启用宏定义OS_CFG_STAT_TASK_EN后，系统会自动创建一个统计任务——OS_StatTask()，它会在任务中计算整个系统的CPU 利用率，各个任务的 CPU 利用率和各个任务的栈使用信息，其源码具体见代码清单27‑2。

代码清单27‑2OS_StatTask()源码

1 void OS_StatTask (void \*p_arg) //统计任务函数

2 {

3 #if OS_CFG_DBG_EN > 0u

4 #if OS_CFG_TASK_PROFILE_EN > 0u

5 OS_CPU_USAGE usage;

6 OS_CYCLES cycles_total;

7 OS_CYCLES cycles_div;

8 OS_CYCLES cycles_mult;

9 OS_CYCLES cycles_max;

10 #endif

11 OS_TCB \*p_tcb;

12 #endif

13 OS_TICK ctr_max;

14 OS_TICK ctr_mult;

15 OS_TICK ctr_div;

16 OS_ERR err;

17 OS_TICK dly;

18 CPU_TS ts_start;

19 CPU_TS ts_end;

20 CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和

21 //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器

22 // SR（临界段关中断只需保存SR），开中断时将该值还原。

23

24 p_arg = p_arg;

25 //没意义，仅为预防编译器警告

26 while (OSStatTaskRdy != DEF_TRUE) //如果统计任务没被允许运行

27 {

28 OSTimeDly(2u \* OSCfg_StatTaskRate_Hz, //一直延时

29 OS_OPT_TIME_DLY,

30 &err);

31 }

32 OSStatReset(&err); **(1)**

33 //如果统计任务已被就绪，复位统计，继续执行

34 /\* 根据设置的宏计算统计任务的执行节拍数 \*/

35 dly = (OS_TICK)0;

36 if (OSCfg_TickRate_Hz > OSCfg_StatTaskRate_Hz)

37 {

38 dly = (OS_TICK)(OSCfg_TickRate_Hz / OSCfg_StatTaskRate_Hz);

39 }

40 if (dly == (OS_TICK)0)

41 {

42 dly = (OS_TICK)(OSCfg_TickRate_Hz / (OS_RATE_HZ)10);

43 } **(2)**

44

45 while (DEF_ON) //进入任务体

46 {

47 ts_start = OS_TS_GET(); //获取时间戳

48 #ifdef CPU_CFG_INT_DIS_MEAS_EN//如果要测量关中断时间

49 OSIntDisTimeMax = CPU_IntDisMeasMaxGet(); //获取最大的关中断时间

50 #endif

51

52 CPU_CRITICAL_ENTER(); //关中断

53 OSStatTaskCtrRun = OSStatTaskCtr; **(3)**//获取上一次空闲任务的计数值

54 OSStatTaskCtr = (OS_TICK)0; //进行下一次空闲任务计数清零

55 CPU_CRITICAL_EXIT(); //开中断

56 /\* 计算CPU利用率 \*/

57 if (OSStatTaskCtrMax > OSStatTaskCtrRun) **(4)**

58 //如果空闲计数值小于最大空闲计数值

59 {

60 if (OSStatTaskCtrMax < 400000u)

61 //这些分类是为了避免计算CPU利用率过程中

62 {

63 ctr_mult = 10000u; //产生溢出，

64 就是避免相乘时超出32位寄存器。

65 ctr_div = 1u;

66 }

67 else if (OSStatTaskCtrMax < 4000000u)

68 {

69 ctr_mult = 1000u;

70 ctr_div = 10u;

71 }

72 else if (OSStatTaskCtrMax < 40000000u)

73 {

74 ctr_mult = 100u;

75 ctr_div = 100u;

76 }

77 else if (OSStatTaskCtrMax < 400000000u)

78 {

79 ctr_mult = 10u;

80 ctr_div = 1000u;

81 }

82 else

83 {

84 ctr_mult = 1u;

85 ctr_div = 10000u;

86 }

87 ctr_max = OSStatTaskCtrMax / ctr_div;

88 OSStatTaskCPUUsage = (OS_CPU_USAGE)((OS_TICK)10000u -

89 ctr_mult \* OSStatTaskCtrRun / ctr_max); **(5)**

90 if (OSStatTaskCPUUsageMax < OSStatTaskCPUUsage)

91 //更新CPU利用率的最大历史记录

92 {

93 OSStatTaskCPUUsageMax = OSStatTaskCPUUsage;

94 }

95 }

96 else\ **(6)**

97 //如果空闲计数值大于或等于最大空闲计数值

98 {

99 OSStatTaskCPUUsage = (OS_CPU_USAGE)10000u; //那么CPU利用率为0

100 }

101

102 OSStatTaskHook(); //用户自定义的钩子函数

103

104 /\* 下面计算各个任务的CPU利用率，原理跟计算整体CPU利用率相似 \*/

105 #if OS_CFG_DBG_EN > 0u//如果启用了调试代码和变量

106 #if OS_CFG_TASK_PROFILE_EN > 0u

107 //如果启用了允许统计任务信息

108 cycles_total = (OS_CYCLES)0;

109

110 CPU_CRITICAL_ENTER(); //关中断

111 p_tcb = OSTaskDbgListPtr;

112 //获取任务双向调试列表的首个任务

113 CPU_CRITICAL_EXIT(); //开中断

114 while (p_tcb != (OS_TCB \*)0) //如果该任务非空

115 {

116 OS_CRITICAL_ENTER(); //进入临界段

117 p_tcb->CyclesTotalPrev = p_tcb->CyclesTotal; **(7)**//保存任务的运行周期

118 p_tcb->CyclesTotal = (OS_CYCLES)0;

119 //复位运行周期，为下次运行做准备

120 OS_CRITICAL_EXIT(); //退出临界段

121

122 cycles_total+=p_tcb->CyclesTotalPrev;\ **(8)**//所有任务运行周期的总和

123

124 CPU_CRITICAL_ENTER(); //关中断

125 p_tcb = p_tcb->DbgNextPtr;

126 //获取列表的下一个任务，进行下一次循环

127 CPU_CRITICAL_EXIT(); //开中断

128 }

129 #endif

130

131 /\* 使用算法计算各个任务的CPU利用率和任务栈用量 \*/

132 #if OS_CFG_TASK_PROFILE_EN > 0u

133 //如果启用了任务的统计功能

134

135 if (cycles_total > (OS_CYCLES)0u) //如果有任务占用过CPU

136 {

137 if (cycles_total < 400000u)

138 //这些分类是为了避免计算CPU利用率过程中

139 {

140 cycles_mult = 10000u; //产生溢出，

141 就是避免相乘时超出32位寄存器。

142 cycles_div = 1u;

143 }

144 else if (cycles_total < 4000000u)

145 {

146 cycles_mult = 1000u;

147 cycles_div = 10u;

148 }

149 else if (cycles_total < 40000000u)

150 {

151 cycles_mult = 100u;

152 cycles_div = 100u;

153 }

154 else if (cycles_total < 400000000u)

155 {

156 cycles_mult = 10u;

157 cycles_div = 1000u;

158 }

159 else

160 {

161 cycles_mult = 1u;

162 cycles_div = 10000u;

163 }

164 cycles_max = cycles_total / cycles_div;

165 }

166 else//如果没有任务占用过CPU

167 {

168 cycles_mult = 0u;

169 cycles_max = 1u;

170 }

171 #endif

172 CPU_CRITICAL_ENTER(); //关中断

173 p_tcb = OSTaskDbgListPtr;

174 //获取任务双向调试列表的首个任务

175 CPU_CRITICAL_EXIT(); //开中断

176 while (p_tcb != (OS_TCB \*)0) //如果该任务非空

177 {

178 #if OS_CFG_TASK_PROFILE_EN > 0u

179 //如果启用了任务控制块的简况变量

180 usage = (OS_CPU_USAGE)(cycles_mult \* //计算任务的CPU利用率

181 p_tcb->CyclesTotalPrev / cycles_max); **(9)**

182 if (usage > 10000u) //任务的CPU利用率为100%

183 {

184 usage = 10000u;

185 }

186 p_tcb->CPUUsage = usage; //保存任务的CPU利用率

187 if (p_tcb->CPUUsageMax < usage)

188 //更新任务的最大CPU利用率的历史记录

189 {

190 p_tcb->CPUUsageMax = usage;

191 }

192 #endif

193 /\* 栈检测 \*/

194 #if OS_CFG_STAT_TASK_STK_CHK_EN > 0u//如果启用了任务栈检测

195 OSTaskStkChk( p_tcb, //计算被激活任务的栈用量

196 &p_tcb->StkFree,

197 &p_tcb->StkUsed,

198 &err); **(10)**

199 #endif

200

201 CPU_CRITICAL_ENTER(); //关中断

202 p_tcb = p_tcb->DbgNextPtr;

203 //获取列表的下一个任务，进行下一次循环

204 CPU_CRITICAL_EXIT(); //开中断

205 }

206 #endif

207

208 if (OSStatResetFlag == DEF_TRUE) //如果需要复位统计

209 {

210 OSStatResetFlag = DEF_FALSE;

211 OSStatReset(&err); //复位统计

212 }

213

214 ts_end = OS_TS_GET() - ts_start; //计算统计任务的执行时间

215 if (OSStatTaskTimeMax < ts_end)

216 //更新统计任务的最大执行时间的历史记录

217 {

218 OSStatTaskTimeMax = ts_end;

219 }

220

221 OSTimeDly(dly,

222 //按照先前计算的执行节拍数延时

223 OS_OPT_TIME_DLY,

224 &err); **(11)**

225 }

226 }

代码清单27‑2\ **(1)**\ ：如果统计任务没被允许运行，就让让它一直延时，直到允许被运行为止，当统计任务准备就绪，就会调用OSStatReset()函数复位。

代码清单27‑2\ **(2)**\ ：根据设置的宏计算统计任务的执行频率，这与我们前面讲解的定时器任务很像。

代码清单27‑2\ **(3)**\ ：进入统计任务主体代码，获取上一次空闲任务的计数值保存在OSStatTaskCtrRun变量中，然后进行下一次空闲任务计数清零。

代码清单27‑2\ **(4)**\ ：计算CPU利用率，如果空闲任务的计数值小于最大空闲的计数值，表示是正常的，然后根据算法得到CPU的利用率，对OSStatTaskCtrMax值的大小进行分类是为了避免计算CPU利用率过程中产生溢出。

代码清单27‑2\ **(5)**\ ：通过算法得到CPU的利用率OSStatTaskCPUUsage。算法很简单，如果不会就代一个数值进去计算一下就能得到。

代码清单27‑2\ **(6)**\ ：如果空闲任务计数值大于或等于最大空闲的计数值，说明CPU利用率为0，CPU一直在空闲任务中计数。

代码清单27‑2\ **(7)**\ ：下面计算各个任务的CPU利用率，原理跟计算整体CPU利用率相似，不过却要启用OS_CFG_DBG_EN与OS_CFG_TASK_PROFILE_EN宏定义，保存任务的运行周期。

代码清单27‑2\ **(8)**\ ：所有被统计的任务运行周期相加得到一个总的运行周期。

代码清单27‑2\ **(9)**\ ：与计算整体CPU利用率一样，计算得到各个任务的CPU利用率。

代码清单27‑2\ **(10)**\ ：如果启用了任务栈检测，调用OSTaskStkChk()函数进行任务的栈检测，在下文讲解该函数。

代码清单27‑2\ **(11)**\ ：按照先前计算的执行节拍数延时，因为统计任务也是按照周期运行的。

栈检测OSTaskStkChk()
~~~~~~~~~~~~~~~~~

μC/OS提供OSTaskStkChk()函数用来进行栈检测，在使用之前必须将宏定义OS_CFG_STAT_TASK_STK_CHK_EN配置为1，对于需要进行任务栈检测的任务，在其被OSTaskCreate()函数创建时，选项参数 opt 还需包含
OS_OPT_TASK_STK_CHK。统计任务会以我们设定的运行频率不断更新栈使用的情况并且保存到任务控制块的StkFree和StkUsed成员变量中，这两个变量分别表示任务栈的剩余空间与已使用空间大小，单位为任务栈大小的单位（在STM32中采用4字节），其源码具体见代码清单27‑3。

代码清单27‑3OSTaskStkChk()源码

1 #if OS_CFG_STAT_TASK_STK_CHK_EN > 0u//如果启用了任务栈检测

2 void OSTaskStkChk (OS_TCB \*p_tcb, **(1)**//目标任务控制块的指针

3 CPU_STK_SIZE \*p_free, **(2)**//返回空闲栈大小

4 CPU_STK_SIZE \*p_used, **(3)**//返回已用栈大小

5 OS_ERR \*p_err) **(4)**//返回错误类型

6 {

7 CPU_STK_SIZE free_stk;

8 CPU_STK \*p_stk;

9 CPU_SR_ALLOC(); //使用到临界段（在关/开中断时）时必须用到该宏，该宏声明和

10 //定义一个局部变量，用于保存关中断前的 CPU 状态寄存器

11 // SR（临界段关中断只需保存SR），开中断时将该值还原。

12

13 #ifdef OS_SAFETY_CRITICAL//如果启用了安全检测

14 if (p_err == (OS_ERR \*)0) //如果 p_err 为空

15 {

16 OS_SAFETY_CRITICAL_EXCEPTION(); //执行安全检测异常函数

17 return; //返回，停止执行

18 }

19 #endif

20

21 #if OS_CFG_CALLED_FROM_ISR_CHK_EN > 0u//如果启用了中断中非法调用检测

22 if (OSIntNestingCtr > (OS_NESTING_CTR)0) //如果该函数是在中断中被调用

23 {

24 \*p_err = OS_ERR_TASK_STK_CHK_ISR; //错误类型为“在中断中检测栈”

25 return; //返回，停止执行

26 }

27 #endif

28

29 #if OS_CFG_ARG_CHK_EN > 0u//如果启用了参数检测

30 if (p_free == (CPU_STK_SIZE*)0) //如果 p_free 为空

31 {

32 \*p_err = OS_ERR_PTR_INVALID; //错误类型为“指针非法”

33 return; //返回，停止执行

34 }

35

36 if (p_used == (CPU_STK_SIZE*)0) //如果 p_used 为空

37 {

38 \*p_err = OS_ERR_PTR_INVALID; //错误类型为“指针非法”

39 return; //返回，停止执行

40 }

41 #endif

42

43 CPU_CRITICAL_ENTER(); //关中断

44 if (p_tcb == (OS_TCB \*)0) **(5)**//如果 p_tcb 为空

45 {

46 p_tcb = OSTCBCurPtr;

47 //目标任务为当前运行任务（自身）

48 }

49

50 if (p_tcb->StkPtr == (CPU_STK*)0) **(6)**//如果目标任务的栈为空

51 {

52 CPU_CRITICAL_EXIT(); //开中断

53 \*p_free = (CPU_STK_SIZE)0; //清零 p_free

54 \*p_used = (CPU_STK_SIZE)0; //清零 p_used

55 \*p_err = OS_ERR_TASK_NOT_EXIST; //错误类型为“任务不存在”

56 return; //返回，停止执行

57 }

58 /\* 如果目标任务的栈非空 \*/

59 if ((p_tcb->Opt & OS_OPT_TASK_STK_CHK) == (OS_OPT)0) **(7)**

60 //如果目标任务没选择检测栈

61 {

62 CPU_CRITICAL_EXIT(); //开中断

63 \*p_free = (CPU_STK_SIZE)0; //清零 p_free

64 \*p_used = (CPU_STK_SIZE)0; //清零 p_used

65 \*p_err = OS_ERR_TASK_OPT;

66 //错误类型为“任务选项有误”

67 return; //返回，停止执行

68 }

69 CPU_CRITICAL_EXIT();

70 //如果任务选择了检测栈，开中断

71 /\* 开始计算目标任务的栈的空闲数目和已用数目 \*/

72 free_stk = 0u; **(8)**//初始化计算栈工作

73 #if CPU_CFG_STK_GROWTH == CPU_STK_GROWTH_HI_TO_LO

74 //如果CPU的栈是从高向低增长

75 p_stk = p_tcb->StkBasePtr; **(9)**

76 //从目标任务栈最低地址开始计算

77 while (*p_stk == (CPU_STK)0) //计算值为0的栈数目

78 {

79 p_stk++;

80 free_stk++; **(10)**

81 }

82 #else

83 //如果CPU的栈是从低向高增长

84 p_stk = p_tcb->StkBasePtr + p_tcb->StkSize - 1u;

85 //从目标任务栈最高地址开始计算

86 while (*p_stk == (CPU_STK)0) //计算值为0的栈数目

87 {

88 free_stk++;

89 p_stk--; **(11)**

90 }

91 #endif

92 \*p_free = free_stk;

93 //返回目标任务栈的空闲数目

94 \*p_used = (p_tcb->StkSize - free_stk); **(12)**

95 //返回目标任务栈的已用数目

96 \*p_err = OS_ERR_NONE; //错误类型为“无错误”

97 }

98 #endif

代码清单27‑3\ **(1)**\ ：目标任务控制块的指针。

代码清单27‑3\ **(2)**\ ：p_free用于保存返回空闲栈大小。

代码清单27‑3\ **(3)**\ ：p_used用于保存返回已用栈大小。

代码清单27‑3\ **(4)**\ ：p_err用于保存返回错误类型。

代码清单27‑3\ **(5)**\ ：如果p_tcb为空，目标任务为当前运行任务（自身）。

代码清单27‑3\ **(6)**\ ：如果目标任务的栈为空，系统将p_free与p_used清零，返回错误类型为“任务不存在”的错误代码。

代码清单27‑3\ **(7)**\ ：如果目标任务的栈非空，但是用户在创建任务的时候没有选择检测栈，那么系统将p_free与p_used清零，返回错误类型为“任务选项有误”的错误代码。

代码清单27‑3\ **(8)**\ ：初始化计算栈工作。

代码清单27‑3\ **(9)**\ ：通过宏定义CPU_CFG_STK_GROWTH选择CPU栈生长的方向，如果CPU的栈是从高向低增长，从目标任务栈最低地址开始计算。

代码清单27‑3\ **(10)**\ ：计算栈空间中内容为0的栈大小，栈空间地址递增。

代码清单27‑3\ **(11)**\ ：如果CPU的栈是从低向高增长，从目标任务栈最高地址开始计算内容为0的栈大小，栈空间地址递减。

代码清单27‑3\ **(12)**\ ：返回目标任务栈的空闲大小与已用大小。

注意：我们自己也可以调用该函数进行统计某个任务的栈空间使用情况。

任务栈大小的确定
~~~~~~~~

任务栈的大小取决于该任务的需求，设定栈大小时，我们就需要考虑：所有可能被栈调用的函数及其函数的嵌套层数，相关局部变量的大小，中断服务程序所需要的空间，另外，栈还需存入CPU寄存器，如果处理器有浮点数单元FPU寄存器的话还需存入FPU寄存器。

嵌入式系统的潜规则，避免写递归函数，这样子可以人为计算出一个任务需要的栈空间大小，逐级嵌套所有可能被调用的函数，计数被调用函数中所有的参数，计算上下文切换时的CPU寄存器空间，计算切换到中断时所需的CPU寄存器空间（假如CPU没有独立的栈用于处理中断），计算处理中断服务函数（ISR）所需的栈空间，将
这些值相加即可得到任务最小的需求空间，但是我们不可能计算出精确的栈空间，我们通常会将这个值再乘以1.5到2.0以确保任务的安全运行。这个计算的值是假定在任务所有的执行路线都是已知的情况下的，但这在真正的应用中并不太可能，比如说，如果调用printf()函数或者其他的函数，这些函数所需要的空间是很难测
得或者说就是不可能知道的，在这种情况下，我们这种人为计算任务栈大小的方法就变得不太可能了，那么我们可以在刚开始创建任务的时候给任务设置一个较大的栈空间，并监测该任务运行时栈空间的实际使用量，运行一段时间后得到任务的最大栈使用情况（或者叫任务栈最坏结果），然后用该值乘1.5到2.0作为栈空间大小就差不
多可以作为任务栈的空间大小，这样子得到的值就会比较精确一点，在调试阶段可以这样子进行测试，发现崩溃就增大任务的栈空间，直到任务能正常稳定运行为止。

CPU利用率及栈检测统计实验
~~~~~~~~~~~~~~

CPU利用率及栈检测统计实验是在μC/OS中创建了四个任务，其中三个任务是普通任务，另一个任务用于获取CPU利用率与任务相关信息并通过串口打印出来。具体见\ **错误！未找到引用源。**\ 。

代码清单27‑4CPU利用率及栈检测统计实验

1 #include <includes.h>

2

3

4 static OS_TCB AppTaskStartTCB;

5

6 static OS_TCB AppTaskLed1TCB;

7 static OS_TCB AppTaskLed2TCB;

8 static OS_TCB AppTaskLed3TCB;

9 static OS_TCB AppTaskStatusTCB;

10

11

12

13 static CPU_STK AppTaskStartStk[APP_TASK_START_STK_SIZE];

14

15 static CPU_STK AppTaskLed1Stk [ APP_TASK_LED1_STK_SIZE ];

16 static CPU_STK AppTaskLed2Stk [ APP_TASK_LED2_STK_SIZE ];

17 static CPU_STK AppTaskLed3Stk [ APP_TASK_LED3_STK_SIZE ];

18 static CPU_STK AppTaskStatusStk [ APP_TASK_STATUS_STK_SIZE ];

19

20 static void AppTaskStart (void \*p_arg);

21

22 static void AppTaskLed1 ( void \* p_arg );

23 static void AppTaskLed2 ( void \* p_arg );

24 static void AppTaskLed3 ( void \* p_arg );

25 static void AppTaskStatus ( void \* p_arg );

26

27

28 int main (void)

29 {

30 OS_ERR err;

31

32

33 OSInit(&err); /\* Init μC/OS-III.
\*/

34

35

36 OSTaskCreate((OS_TCB \*)&AppTaskStartTCB,

37

38 (CPU_CHAR \*)"App Task Start",

39 (OS_TASK_PTR ) AppTaskStart,

40 (void \*) 0,

41 (OS_PRIO ) APP_TASK_START_PRIO,

42 (CPU_STK \*)&AppTaskStartStk[0],

43 (CPU_STK_SIZE) APP_TASK_START_STK_SIZE / 10,

44 (CPU_STK_SIZE) APP_TASK_START_STK_SIZE,

45 (OS_MSG_QTY ) 5u,

46 (OS_TICK ) 0u,

47 (void \*) 0,

48 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

49 (OS_ERR \*)&err);

50

51 OSStart(&err);

52

53

54

55 }

56

57

58 static void AppTaskStart (void \*p_arg)

59 {

60 CPU_INT32U cpu_clk_freq;

61 CPU_INT32U cnts;

62 OS_ERR err;

63

64

65 (void)p_arg;

66

67 BSP_Init();

68

69 CPU_Init();

70

71 cpu_clk_freq = BSP_CPU_ClkFreq();

72

73 cnts = cpu_clk_freq / (CPU_INT32U)OSCfg_TickRate_Hz;

74

75 OS_CPU_SysTickInit(cnts);

76

77

78 Mem_Init();

79

80

81 #if OS_CFG_STAT_TASK_EN > 0u

82

83

84 OSStatTaskCPUUsageInit(&err);

85

86

87 #endif

88

89

90 CPU_IntDisMeasMaxCurReset();

91

92

93

94

95

96 /\* Create the Led1 task \*/

97 OSTaskCreate((OS_TCB \*)&AppTaskLed1TCB,

98 (CPU_CHAR \*)"App Task Led1",

99 (OS_TASK_PTR ) AppTaskLed1,

100 (void \*) 0,

101 (OS_PRIO ) APP_TASK_LED1_PRIO,

102 (CPU_STK \*)&AppTaskLed1Stk[0],

103 (CPU_STK_SIZE) APP_TASK_LED1_STK_SIZE / 10,

104 (CPU_STK_SIZE) APP_TASK_LED1_STK_SIZE,

105 (OS_MSG_QTY ) 5u,

106 (OS_TICK ) 0u,

107 (void \*) 0,

108 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

109 (OS_ERR \*)&err);

110

111 /\* Create the Led2 task \*/

112 OSTaskCreate((OS_TCB \*)&AppTaskLed2TCB,

113 (CPU_CHAR \*)"App Task Led2",

114 (OS_TASK_PTR ) AppTaskLed2,

115 (void \*) 0,

116 (OS_PRIO ) APP_TASK_LED2_PRIO,

117 (CPU_STK \*)&AppTaskLed2Stk[0],

118 (CPU_STK_SIZE) APP_TASK_LED2_STK_SIZE / 10,

119 (CPU_STK_SIZE) APP_TASK_LED2_STK_SIZE,

120 (OS_MSG_QTY ) 5u,

121 (OS_TICK ) 0u,

122 (void \*) 0,

123 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

124 (OS_ERR \*)&err);

125

126 /\* Create the Led3 task \*/

127 OSTaskCreate((OS_TCB \*)&AppTaskLed3TCB,

128 (CPU_CHAR \*)"App Task Led3",

129 (OS_TASK_PTR ) AppTaskLed3,

130 (void \*) 0,

131 (OS_PRIO ) APP_TASK_LED3_PRIO,

132 (CPU_STK \*)&AppTaskLed3Stk[0],

133 (CPU_STK_SIZE) APP_TASK_LED3_STK_SIZE / 10,

134 (CPU_STK_SIZE) APP_TASK_LED3_STK_SIZE,

135 (OS_MSG_QTY ) 5u,

136 (OS_TICK ) 0u,

137 (void \*) 0,

138 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

139 (OS_ERR \*)&err);

140

141 /\* Create the status task \*/

142 OSTaskCreate((OS_TCB \*)&AppTaskStatusTCB,

143 (CPU_CHAR \*)"App Task Status",

144 (OS_TASK_PTR ) AppTaskStatus,

145 (void \*) 0,

146 (OS_PRIO ) APP_TASK_STATUS_PRIO,

147 (CPU_STK \*)&AppTaskStatusStk[0],

148 (CPU_STK_SIZE) APP_TASK_STATUS_STK_SIZE / 10,

149 (CPU_STK_SIZE) APP_TASK_STATUS_STK_SIZE,

150 (OS_MSG_QTY ) 5u,

151 (OS_TICK ) 0u,

152 (void \*) 0,

153 (OS_OPT )(OS_OPT_TASK_STK_CHK \| OS_OPT_TASK_STK_CLR),

154 (OS_ERR \*)&err);

155

156 OSTaskDel ( & AppTaskStartTCB, & err );

157

158

159 }

160

161

162

163 static void AppTaskLed1 ( void \* p_arg )

164 {

165 OS_ERR err;

166 uint32_t i;

167

168 (void)p_arg;

169

170

171 while (DEF_TRUE)

172

173 {

174

175 printf("AppTaskLed1 Running\n");

176

177 for (i=0; i<10000; i++) //模拟任务占用cpu

178 {

179 ;

180 }

181

182 macLED1_TOGGLE ();

183 OSTimeDlyHMSM (0,0,0,500,OS_OPT_TIME_PERIODIC,&err);

184 }

185

186

187 }

188

189

190

191 static void AppTaskLed2 ( void \* p_arg )

192 {

193 OS_ERR err;

194 uint32_t i;

195

196 (void)p_arg;

197

198

199 while (DEF_TRUE)

200

201 {

202 printf("AppTaskLed2 Running\n");

203

204 for (i=0; i<100000; i++) //模拟任务占用cpu

205 {

206 ;

207 }

208 macLED2_TOGGLE ();

209

210 OSTimeDlyHMSM (0,0,0,500,OS_OPT_TIME_PERIODIC,&err);

211 }

212

213

214 }

215

216

217

218 static void AppTaskLed3 ( void \* p_arg )

219 {

220 OS_ERR err;

221

222 uint32_t i;

223 (void)p_arg;

224

225

226 while (DEF_TRUE)

227 {

228

229 macLED3_TOGGLE ();

230

231 for (i=0; i<500000; i++) //模拟任务占用cpu

232 {

233 ;

234 }

235

236 printf("AppTaskLed3 Running\n");

237

238

239 OSTimeDlyHMSM (0,0,0,500,OS_OPT_TIME_PERIODIC,&err);

240

241 }

242

243 }

244

245 static void AppTaskStatus ( void \* p_arg )

246 {

247 OS_ERR err;

248

249 CPU_SR_ALLOC();

250

251 (void)p_arg;

252

253 while (DEF_TRUE)

254 {

255

256 OS_CRITICAL_ENTER();

257 //进入临界段，避免串口打印被打断

258 printf("---------------------------------------------------\n");

259 printf ( "CPU利用率：%d.%d%%\r\n",

260 OSStatTaskCPUUsage / 100, OSStatTaskCPUUsage % 100 );

261 printf ( "CPU最大利用率：%d.%d%%\r\n",

262 OSStatTaskCPUUsageMax / 100, OSStatTaskCPUUsageMax % 100 );

263

264

265 printf ( "LED1任务的CPU利用率：%d.%d%%\r\n",

266 AppTaskLed1TCB.CPUUsageMax / 100, AppTaskLed1TCB.CPUUsageMax % 100 );

267 printf ( "LED1任务的CPU利用率：%d.%d%%\r\n",

268 AppTaskLed2TCB.CPUUsageMax / 100, AppTaskLed2TCB.CPUUsageMax % 100 );

269 printf ( "LED1任务的CPU利用率：%d.%d%%\r\n",

270 AppTaskLed3TCB.CPUUsageMax / 100, AppTaskLed3TCB.CPUUsageMax % 100 );

271 printf ( "统计任务的CPU利用率：%d.%d%%\r\n",

272 AppTaskStatusTCB.CPUUsageMax / 100, AppTaskStatusTCB.CPUUsageMax % 100 ) ;

273

274

275 printf ( "LED1任务的已用和空闲栈大小分别为：%d,%d\r\n",

276 AppTaskLed1TCB.StkUsed, AppTaskLed1TCB.StkFree );

277 printf ( "LED2任务的已用和空闲栈大小分别为：%d,%d\r\n",

278 AppTaskLed2TCB.StkUsed, AppTaskLed2TCB.StkFree );

279 printf ( "LED3任务的已用和空闲栈大小分别为：%d,%d\r\n",

280 AppTaskLed3TCB.StkUsed, AppTaskLed3TCB.StkFree );

281 printf ( "统计任务的已用和空闲栈大小分别为：%d,%d\r\n",

282 AppTaskStatusTCB.StkUsed, AppTaskStatusTCB.StkFree );

283

284 printf("---------------------------------------------------\n");

285 OS_CRITICAL_EXIT(); //退出临界段

286

287 OSTimeDlyHMSM (0,0,0,500,OS_OPT_TIME_PERIODIC,&err);

288

289 }

290 }

CPU利用率及栈检测统计实验现象
~~~~~~~~~~~~~~~~

程序编译好，用USB线连接计算机和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据购买的板子而定，每个型号的板子都配套有对应的程序），在计算机上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，具体见图27‑5。

|cpuusa006|

图27‑5CPU利用率及栈检测统计实验现象

.. |cpuusa002| image:: media\cpuusa002.png
   :width: 5.76806in
   :height: 1.32083in
.. |cpuusa003| image:: media\cpuusa003.png
   :width: 4.15139in
   :height: 3.525in
.. |cpuusa004| image:: media\cpuusa004.png
   :width: 3.98681in
   :height: 3.54167in
.. |cpuusa005| image:: media\cpuusa005.png
   :width: 4.17153in
   :height: 3.81806in
.. |cpuusa006| image:: media\cpuusa006.png
   :width: 5.07014in
   :height: 3.98681in
