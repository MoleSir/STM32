# STM32 SysTick

## 1、SysTick简介

### 1.1、何为SysTick

​	Systick是一个位于Cortex-M3内核中的计时器，具有自动重载和溢出中断功能，所有基于Cortex_M3处理器的微控制器都可以由这个定时器获得一定的时间间隔；



### 1.2、SysTick的主要作用

​	在单任务程序中，因为其架构就决定了它执行任务的串行性，这就引出一个问题：当某个任务出现问题时，就会牵连到后续的任务，进而导致整个系统崩溃。

​	要解决这个问题，可以使用实时操作系统（RTOS）。因为RTOS以并行的架构处理任务，单一任务的崩溃并不会牵连到整个系统。这样用户出于可靠性的考虑可能就会基于RTOS来设计自己的应用程序。SYSTICK存在的意义就是提供必要的时钟节拍，为RTOS的任务调度提供一个有节奏的“心跳”。

​	微控制器的定时器资源一般比较丰富，比如STM32存在8个定时器，为啥还要再提供一个SYSTICK？

​	原因就是所有基于ARM Cortex_M3内核的控制器都带有SysTick定时器，这样就方便了程序在不同的器件之间的移植。而使用RTOS的第一项工作往往就是将其移植到开发人员的硬件平台上，由于SYSTICK的存在无疑降低了移植的难度。

​	SysTick定时器除了能服务于操作系统之外，还能用于其它目的：如作为一个闹铃，用于测量时间等。

​	要注意的是，当处理器在调试期间被喊停（halt）时，则SysTick定时器亦将暂停运作。





## 2、SysTick相关寄存器

### 2.1、SysTick_Type结构体

​	在core_cm3.h 可以查看到

````c
typedef struct
{
  __IO uint32_t CTRL;           /*!< Offset: 0x00  SysTick Control and Status Register */
  __IO uint32_t LOAD;           /*!< Offset: 0x04  SysTick Reload Value Register       */
  __IO uint32_t VAL;            /*!< Offset: 0x08  SysTick Current Value Register      */
  __I  uint32_t CALIB;          /*!< Offset: 0x0C  SysTick Calibration Register        */
} SysTick_Type;
````

​	那么根据以往的经验，应该会有一个宏定义来将某个地址解释为 Systick_Type 指针；

````c
#define SCS_BASE            (0xE000E000)        /*!< System Control Space Base Address */
#define SysTick_BASE        (SCS_BASE +  0x0010)              /*!< SysTick Base Address*/
#define SysTick             ((SysTick_Type *)       SysTick_BASE)     
/*!< SysTick configuration struct*/
````

​	啊哈，我们的猜想没有错，并且 SysTick是位于 0xE000 E000 + 0x0010，而 0xE000 0000是属于内核的存储区域，验证了SysTick位于内核；

​	

### 2.2、主要寄存器

​	SysTick 的工作原理很简单，设定一个初值，其开始从初值自减，到0后发出中断，执行服务函数；然后再次回到初值，依次循环；

​	在配置Systick时，主要就是对上述 SysTick_Type 结构体对象的寄存器的配置：

- **CTRL：控制和状态寄存器**：
    - 第16bit (COUNTFLAG)：如果上次读取本寄存器时，SysTick已经到0，则该bit为1；读取该bit后，这bit为0；
    - 第0bit（ENABLE）：SysTick 使能位；
    - 第1bit（TICKINT）：该bit为1表示SysTick减到0时产生SysTick异常，为0表示不发生异常；
    - 第2bit（CLKSOURCE）：该为 bit为1表示使用内核时钟；为0表示使用外部时钟；
- **LOAD：重装载寄存器**：
    - 有效位为 0-23 bit：当SysTick减到0后，重新装载为本数值再次计数；
- **VAL：当前值寄存器**：
    - 有效位为 0-23 bit：表示 SysTick 当前计数到的数字，减到0表示本次计数完毕；
- **CALIB：SysTick校准值寄存器：**
    - 目前无需关心；





## 3、SysTick的时钟源

### 3.1、SysTick时钟源的选择

​	SysTick的时钟源有两种，一个是与 **AHB 高速总线**一致的时钟源，另一种则是将 **AHB 的频率除以 8** 后的频率；

​	我们可以使用在 misc.c 中定义的 SysTick_CLKSourceConfig 函数显式得配置它：

````c
void SysTick_CLKSourceConfig(uint32_t SysTick_CLKSource)
{
  /* Check the parameters */
  assert_param(IS_SYSTICK_CLK_SOURCE(SysTick_CLKSource));
  if (SysTick_CLKSource == SysTick_CLKSource_HCLK)
  {
    SysTick->CTRL |= SysTick_CLKSource_HCLK;
  }
  else
  {
    SysTick->CTRL &= SysTick_CLKSource_HCLK_Div8;
  }
}
````

​	SysTick_CLKSourceConfig 函数只允许我们传入两个参数：

````c
#define SysTick_CLKSource_HCLK ((uint32_t)0x00000004)
#define SysTick_CLKSource_HCLK_Div8 ((uint32_t)0xFFFFFFFB)
````

​	前者表示需要 AHB 时钟，后者则是除 8；

​	配置过程也很简单：

​	先做一个if判断，如果是是 AHB，则将 SysTick->CTRL的第2bit设置为1，表直接使用 AHB 时钟；

​	否则将其设置为0，表示使用外部分频器将 AHB 除以 8 后的时钟源；（具体实现比较简单不再分析）





### 3.2 AHB 高速总线时钟

​	AHB 总线时钟是STM32中最快的，是来自内核的时钟，在启动过程中，内核的时钟就已经被配置，在 system_stm32f10x.c 文件中定义的 SystemInit() 函数中有：

````c
SetSysClock();
````

​	当然 SystemInit函数还有很多其他内容，这里我们只关系其中的 SetSysClokc()函数，继续找:

```c
static void SetSysClock(void)
{
#ifdef SYSCLK_FREQ_HSE
  SetSysClockToHSE();
#elif defined SYSCLK_FREQ_24MHz
  SetSysClockTo24();
#elif defined SYSCLK_FREQ_36MHz
  SetSysClockTo36();
#elif defined SYSCLK_FREQ_48MHz
  SetSysClockTo48();
#elif defined SYSCLK_FREQ_56MHz
  SetSysClockTo56();  
#elif defined SYSCLK_FREQ_72MHz
  SetSysClockTo72();
#endif
 
 /* If none of the define above is enabled, the HSI is used as System clock
    source (default after reset) */ 
}
```

​	这个函数根据不同的宏定义去执行不同的系统时钟配置函数，stm32f10x使用的是 72MHz

​	SetSysClockTo72()函数的内容我们就暂时不再深究，十分复杂，总之就是：

​	在STM32启动时调用 SystemInit ，然后这个函数又调用 SetSysClock；

​	而 SetSysClock 根据芯片不同的宏定义执行到不同的时钟配置函数，在这里，我们调用SetSysClockTo72()将内核时钟设置为 72MHz，即一秒 72M 次的振动；





## 4、SystTck的使用

### 4.1、配置SysTick

​	配置时，需要我们写的只有一个函数：SysTick_Config，以下为 core_cm3.h 中的定义：

````c
static __INLINE uint32_t SysTick_Config(uint32_t ticks)
{ 
  /* Reload value impossible */
  if (ticks > SysTick_LOAD_RELOAD_Msk)  return (1);         
  
  /* set reload register */
  SysTick->LOAD  = (ticks & SysTick_LOAD_RELOAD_Msk) - 1;  
 	
  /* set Priority for Cortex-M0 System Interrupts */
  NVIC_SetPriority (SysTick_IRQn, (1<<__NVIC_PRIO_BITS) - 1);  
    
  /* Load the SysTick Counter Value */
  SysTick->VAL   = 0;      
    
  /* Enable SysTick IRQ and SysTick Timer */
  SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk | 
                   SysTick_CTRL_TICKINT_Msk   | 
                   SysTick_CTRL_ENABLE_Msk;                    
  return (0);                                                  /* Function successful */
}
````

​	简单分析一下代码：

**1、检查参数**

````c
if (ticks > SysTick_LOAD_RELOAD_Msk)  return (1);         
````

​	当传入的 ticks 大于所能设置的最大数量时，函数返回1表示配置失败；

**2、设置重装载寄存器**

```c
SysTick->LOAD  = (ticks & SysTick_LOAD_RELOAD_Msk) - 1;  
```

​	SysTick_LOAD_RELOAD_Msk = 0xFF FFFF，这句话其实就是把 ticks - 1赋值给 LOAD；

​	-1是因为我们0也算一次计数；

**3、设置中断优先级**

```c
NVIC_SetPriority (SysTick_IRQn, (1<<__NVIC_PRIO_BITS) - 1);  
```

​	具体配置了多少，我们就不要关心，这个是定好的；

**4、初始计数寄存器**

```c
SysTick->VAL   = 0;
```

**5、配置控制寄存器**

````c
  SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk | 
                   SysTick_CTRL_TICKINT_Msk   | 
                   SysTick_CTRL_ENABLE_Msk;   
````

​	其中 SysTick_CTRL_CLKSOURCE_Msk = 0x100 ;

​			SysTick_CTRL_TICKINT_Msk = 0x01;

​			SysTick_CTRL_ENABLE_Msk = 0x1;

​	刚好对应[2.1 SysTick主要寄存器](# 2.2、主要寄存器)中的CTRL寄存器的 0，1，2bit的位置，那么这句话就是：开启使能、开始发出中断、使用内核时钟；

**6、配置成功返回0**

````c
return (0);
````

​	可以看到，如果我们不想做太多的修改的话，只需要传入一个合法的参数就可以配置SysTick了；



### 4.2、编写SysTick中断服务函数

​	SysTick 是一次内部中断，可以称它为异常，启动文件中定义好了默认的 SysTick 中断服务函数：

````assembly
DCD     SysTick_Handler            ; SysTick Handler
````

​	与其他中断函数一样，如果我们希望重写，一定要使用相同的函数名；



### 4.3、SysTick中断间隔的计算

​	在4.1中我们介绍了 SysTick_Config 函数，在这里将分析一下其参数；

​	首先看一下参数的范围：

````c
#define SysTick_LOAD_RELOAD_Msk (0xFFFFFFul << SysTick_LOAD_RELOAD_Pos)
````

​	SysTick_LOAD_RELOAD_Pos 是 0，所以参数 ticks 最大为 0xFF FFFF；

​	而这个参数的意义就是重装寄存器的值；

​	而如果我们设置的是 AHB 总线时钟，每 1/72000000 秒，VAL寄存器减1；

​	所以每隔 1/72000000 * ticks 秒，发送一次SysTick 中断，进入一次中断服务函数；

​	那么如果感觉不好算，我们可以直接使用 SystemCoreClock 这个宏定义，它的值就是内核时钟频率，这里就是72000000，那么如果我们想每隔 10ms 发送一次中断，就可以直接填入 SystemCoreClock / 100；

​	因为填 SystemCoreClock 也就是 72000000 是一秒，除100 就是 10 ms；



### 4.4 一个小例子

````c
int softtime=0;
int main(void)
{
  if (SysTick_Config(SystemCoreClock / 100)) //1000 -> 1ms
  { 
    /* Capture error */ 
    while (1);
  }
    // 配置led函数
	led_cfg();
	while(1);
}

// 中断服务函数
void SysTick_Handler(void)
{
	softtime++;
	if(softtime==100){
       	// 每隔 10 * 100 = 1000 ms 执行一次 led_toggle
		led_toggle();
		softtime=0;
	}
}
````

