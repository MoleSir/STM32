

# STM32 GPIO

## 一、GPIO简介

​	**GPIO**，General-purpose input/output，即通过I/O（输入/输出）端口，是STM32可以控制的引脚。STM32芯片的GPIO引脚与外部设备连接起来，可以实现与外部通讯、控制外部硬件或者采集外部硬件功能。

​	不同的STM32有不同组数的IO口，以STM32F103RC为例，其有四组IO口，分别为GPIOA~GPIOD（如有更多则继续名名下去），每组有16个IO口，通常称为PAx、PBx、PCx，其中x从0到15。



## 二、端口的复用

### 1、什么是端口复用

​	除了作为**默认的IO口**，GPIO还有**复用功能**。

​	芯片中有许多的内置外设，其中有一些也需要与外界交互，比较串口，需要一个发射端口Tx来发送数据到外部，一个Rx端口来接收外部的数据，那么这些内置外设就需要使用 GPIO 来与外界交互，当GPIO用作外设时，就称为**复用**。例如：对STM32中的1号串口，默认其Tx端口复用 PA9，Rx端口复用 PA10，开启这 PA9 与 PA10 的复用功能时，就默认成为了 1 号串口的外设管脚；



### 2、如何开启端口复用



### 3、端口重映射



## 三、GPIO的工作模式

### 1、GPIO内部结构

<img src="./pics/Note5. STM32 GPIO详解.assets/GPIO内部结构.jpg" alt="GPIO内部结构" style="zoom:80%;" />

​	每一个GPIO其内部结构都如上图，通过使用不同的部件来选择GPIO的工作模式，对一些重要元件作简单介绍：

- **保护二极管**：IO引脚上下两边两个二极管用于防止引脚过高过低的电压输入，防止不正常电压引入芯片导致芯片烧毁；
- **上拉、下拉电阻**：控制引脚默认状态电压，开启上拉的时候默认引脚电压为高电平；开启下拉的时候默认引脚电压为低电平；
- **TTL施密特触发器**：IO口信号结果触发器后，只有大于正向阈值的模拟电压才能使触发器输出高电平，小于反向阈值才能使触发器输出低电平；这样就将出入模拟信号转换为高低电平信号；这也是为什么STM32是TTL电平协议的原因；
- **P-MOS管与N-MOS管**：信号由两个管子的G极流入，依据两个管子的工作方式，使得GPIO具有“推挽输出”与“开漏输出”两个模式；P-MOS管在输入高电平时导通，低电平截止，N-MOS管子相反；



### 2、四种输入模式

**（1）GPIO_Mode_IN_FLOATING 浮空输入**

​	               <img src="./pics/Note5. STM32 GPIO详解.assets/浮动输入模式.png" alt="浮动输入模式" style="zoom:80%;" />

​	浮动输入模式下，I/O端口的电平信号直接进入输入寄存器。当输入为高电平时，读入高；输入低电平时读入低。但是当输入端口悬空，读取端口的电平是不确定的（直接用电压表测量是1点多V，是不确定值）以用来作Key识别。

**（2）GPIO_Mode_IPU 上拉输入**

<img src="./pics/Note5. STM32 GPIO详解.assets/上拉输入模式.png" alt="上拉输入模式" style="zoom:80%;" />

​	上拉输入在有确定输入信号时与浮动输入相同，区别在于当输入端口浮空，读入的电压为高电平。值得注意的时STM32内部只能提供“弱上拉”，电流很弱，如果需要大电流需要使用外部上拉。

**（3）GPIO_Mode_IPD 下拉输入**

​					<img src="./pics/Note5. STM32 GPIO详解.assets/下拉输入模式.png" alt="下拉输入模式" style="zoom:80%;" />

​	下拉输入模式在输入确定电压时与浮空输入模式一致，区别在于当输入悬空时，读入的是低电平；

**（4）GPIO_Mode_AIN 模拟输入**

​                   <img src="./pics/Note5. STM32 GPIO详解.assets/模拟输入模式.png" alt="模拟输入模式" style="zoom:80%;" />

​	当GPIO引脚用于ADC采集电压的输入通道时，用作“模拟输入”功能，此时信号不经过施密特触发器，直接进入ADC模块，并且输入数据寄存器为空，CPU不能在输入数据寄存器上读到引脚状态（因为施密特触发器关闭）；

​	当GPIO处于这个模式时，上下拉电阻是不起作用的，这时即使配置了上拉或者下拉模式，也不会影响模拟信号的输入输出；

​	除了ADC与DAC要将IO配置为模拟通道之外，其他的外设功能一律要将端口配置为复用功能；

### 3、四种输出模式

**（1）GPIO_Mode_Out_OD 开漏输出**

<img src="./pics/Note5. STM32 GPIO详解.assets/开漏输出模式.png" alt="开漏输出模式" style="zoom:80%;" />

​	开漏模式下，只有N-MOS管工作，如果我们对栅极写入低电平，则N-MOS导通，输出为低电平；

​	但如果我们对栅极写入高电平，两个管子都截止，输出的高电平不会其作用。此时I/O端口的输出值由端口上下拉的电平决定。而如果没有上拉或下拉，则悬空；

​	此时施密特触发器打开，尽管I/O口用于输出，我们仍可以读取输出电压的值；但是只有我们希望写入低电平时，读取到的才是真正写入的值；如果我们写入的是高，输出值就由外部的上下拉决定，或者悬空；

​	所以想要让开漏模式能输出高低电平，必须外接上拉电源；

**（2）GPIO_Mode_Out_PP 推挽输出**

<img src="./pics/Note5. STM32 GPIO详解.assets/推挽输出模式.png" alt="推挽输出模式" style="zoom:80%;" />

​	推挽模式下，两个管子都工作：

​	当我们输入高电平，P-MOS导通，N-MOS截止，输出高电平；

​	当我们输入低电平，N-MOS导通，P-MOS截止，输出低电平；

​	上拉或下拉的作用是控制没有输出时的I/O电平；

​	此时施密特触发器打开，我们可以读取到输出端口的电压值；并且读取到的一定是我们希望写入的值，因为推挽模式下，无论写入高还是低都可以顺利输出；

**（3）GPIO_Mode_AF_OD 复用开漏输出**

<img src="./pics/Note5. STM32 GPIO详解.assets/复用开漏输出模式.png" alt="复用开漏输出模式" style="zoom:80%;" />

​	此时GPIO复用其他外设，除了输出信号的来源与GPIO_Mode_Out_OD都相同；

**（4）GPIO_Mode_AF_PP 复用推挽输出** 

<img src="./pics/Note5. STM32 GPIO详解.assets/复用推挽输出模式.png" alt="复用推挽输出模式" style="zoom:80%;" />

​	此时GPIO复用其他外设，除了输出信号的来源与GPIO_Mode_AF_OD都相同；



### 4、在STM32中选取IO模式

- 上拉输入、下拉输入可以用来检测外部信号；例如，按键等；
- 模拟输入应用在ADC模拟输入，或者低功耗下省电；
- 开漏输出一般应用在I2C、SMBUS通讯等需要“线与”功能的总线电路中；
- 推挽输出模式一般应用在输出电平为0到3.3V而且需要高速切换开关状态的场合；通常除了必须使用开漏输出的情况，我们都使用推挽输出模式；
- 复用的开漏与推挽输出在片内外设功能时使用；



### 5、F4系列与F1系列的区别

​	本质上二者的区别是F4系列采用了**Cortex-M4内核**， 而F1系列采用了**Cortex-M3内核**

​	**F1系列（M3）IO口的基本结构**

​                	<img src="./pics/Note5. STM32 GPIO详解.assets/F1系列.jpg" alt="F1系列" style="zoom:80%;" />

​	**F4系列（M4）IO口的基本结构**

<img src="./pics/Note5. STM32 GPIO详解.assets/F4系列.jpg" alt="F4系列" style="zoom:80%;" />

​	区别是F4系列将上下拉电阻转移到输入输出驱动的外部，使得输出模式也可以实现上下拉，增加灵活性；

## 四、GPIO的初始化

（以下以F1为例）

### 1、GPIO对象定义

​	当我们需要使用一个或者一些引脚时，我们需要对其进行配置，这需要声明一个描述管脚状态的对象，在STM32标准库中为我们定义了 GPIO_InitTypeDef类型的结构体：

````c
typedef struct
{
    uint64_t GPIO_Pin; 				// 指定要配置的GPIO管脚
    
    GPIOSPeed_TypeDef GPIO_Speed;   // 指定选定管脚的速度
    
    GPIOMode_TypeDef GPIO_Mode;     // 指定所选管脚的操作模式
    
} GPIO_InitTypeDef;
````

​	使用方法就是定义一个对象：

````c
// 定义一个 GPIO_InitTypeDef 类型的对象
GPIO_InitTypeDef GPIO_InitStructure; 
````



### 2、开启GPIO外设时钟

````c
// 开启 GPIOA组的引脚时钟，当然也可以开启 GPIOB、GPIOC的
RCC_AHB2PeriphClockCmd(RCC_AHB2Periph_GPIOA, ENABLE);
````

​	**Q1、那么我们为什么需要设置时钟呢？**

​	任何外设都需要时钟，51单片机、stm32、430等等，因为寄存器是通过D触发器组成的，往触发器写入数据的前提是时钟需要打开。stm32是低功耗的，其默认所有的门时钟disable，用户需要哪些就自己打开，其他没用到的还是关闭节约能源；

​	**Q2、为什么stm32需要多个时钟源**

​	首先stm32本身非常复杂，外设很多，但是不是所有的外设都需要系统时钟那么高的频率。而时钟频率越快，功耗越大，同时抗电磁能力越弱，所以对复杂的系统应该使用多时钟源的方法，各个外设对应开启所需要频率的时钟；

​	**Q3、函数名称前面前面的缩写是什么含义**

​	STM32上的4GB地址空间被分为多块，其中有一块有512MB是用来外设使用的，还有的块分给Flash、SRAM等，具体请看寄存器篇，这里不多赘述。而外设使用的有三条总线：AHB（Advanced High-performance Bus ）、APB1（Advanced Peripheral Bus 1）与 APB2（Advanced Peripheral Bus 2），高速总线AHB来接高速外设，而低速总线APB来接低俗外设。所以这个函数名称的含义就是：某个总线外设的时钟控制函数；

​	注意对于 STM32FC103RC 而言，不同于上面的例子，所有的 GPIO都是在 APB2的总线上的，所以应该使用的是APB2总线的时钟开启函数：

````c
RCC_APB2PeriphClockCmd(RCC_AHB2Periph_GPIOA, ENABLE);
````

​	（相对应的外设功能可以在 stm32f10x_rcc.h中找到）

三条总线时钟的控制函数是不同的，要开启的时钟在哪个总线上就使用哪个函数

````c
void RCC_AHBPeriphClockCmd(uint32_t RCC_AHBPeriph, FunctionalState NewState);
void RCC_APB2PeriphClockCmd(uint32_t RCC_APB2Periph, FunctionalState NewState);
void RCC_APB1PeriphClockCmd(uint32_t RCC_APB1Periph, FunctionalState NewState);
````

​	以下为三条总线上可以打开时钟的位置，写入对应的时钟控制函数的第一个参数；

​	需要开始GPIO的时钟就需要填入关于GPIO的宏；可以注意到，下面只有APB2总线上定义了有关GPIO的宏；这也证明了 GPIO 是在 APB2总线上的；

​	**RCC_AHB**

````c
/** @defgroup AHB_peripheral 
  * @{
  */

#define RCC_AHBPeriph_DMA1               ((uint32_t)0x00000001)
#define RCC_AHBPeriph_DMA2               ((uint32_t)0x00000002)
#define RCC_AHBPeriph_SRAM               ((uint32_t)0x00000004)
#define RCC_AHBPeriph_FLITF              ((uint32_t)0x00000010)
#define RCC_AHBPeriph_CRC                ((uint32_t)0x00000040)

#ifndef STM32F10X_CL
 #define RCC_AHBPeriph_FSMC              ((uint32_t)0x00000100)
 #define RCC_AHBPeriph_SDIO              ((uint32_t)0x00000400)
 #define IS_RCC_AHB_PERIPH(PERIPH) ((((PERIPH) & 0xFFFFFAA8) == 0x00) && ((PERIPH) != 0x00))
#else
 #define RCC_AHBPeriph_OTG_FS            ((uint32_t)0x00001000)
 #define RCC_AHBPeriph_ETH_MAC           ((uint32_t)0x00004000)
 #define RCC_AHBPeriph_ETH_MAC_Tx        ((uint32_t)0x00008000)
 #define RCC_AHBPeriph_ETH_MAC_Rx        ((uint32_t)0x00010000)

 #define IS_RCC_AHB_PERIPH(PERIPH) ((((PERIPH) & 0xFFFE2FA8) == 0x00) && ((PERIPH) != 0x00))
 #define IS_RCC_AHB_PERIPH_RESET(PERIPH) ((((PERIPH) & 0xFFFFAFFF) == 0x00) && ((PERIPH) != 0x00))
#endif /* STM32F10X_CL */
/**
  * @}
  */
````

​	**RCC_APB2**

````c
/** @defgroup APB2_peripheral 
  * @{
  */

#define RCC_APB2Periph_AFIO              ((uint32_t)0x00000001)
#define RCC_APB2Periph_GPIOA             ((uint32_t)0x00000004)
#define RCC_APB2Periph_GPIOB             ((uint32_t)0x00000008)
#define RCC_APB2Periph_GPIOC             ((uint32_t)0x00000010)
#define RCC_APB2Periph_GPIOD             ((uint32_t)0x00000020)
#define RCC_APB2Periph_GPIOE             ((uint32_t)0x00000040)
#define RCC_APB2Periph_GPIOF             ((uint32_t)0x00000080)
#define RCC_APB2Periph_GPIOG             ((uint32_t)0x00000100)
#define RCC_APB2Periph_ADC1              ((uint32_t)0x00000200)
#define RCC_APB2Periph_ADC2              ((uint32_t)0x00000400)
#define RCC_APB2Periph_TIM1              ((uint32_t)0x00000800)
#define RCC_APB2Periph_SPI1              ((uint32_t)0x00001000)
#define RCC_APB2Periph_TIM8              ((uint32_t)0x00002000)
#define RCC_APB2Periph_USART1            ((uint32_t)0x00004000)
#define RCC_APB2Periph_ADC3              ((uint32_t)0x00008000)
#define RCC_APB2Periph_TIM15             ((uint32_t)0x00010000)
#define RCC_APB2Periph_TIM16             ((uint32_t)0x00020000)
#define RCC_APB2Periph_TIM17             ((uint32_t)0x00040000)
#define RCC_APB2Periph_TIM9              ((uint32_t)0x00080000)
#define RCC_APB2Periph_TIM10             ((uint32_t)0x00100000)
#define RCC_APB2Periph_TIM11             ((uint32_t)0x00200000)

#define IS_RCC_APB2_PERIPH(PERIPH) ((((PERIPH) & 0xFFC00002) == 0x00) && ((PERIPH) != 0x00))
/**
  * @}
  */ 
````

​	**RCC_APB1**

````c
/** @defgroup APB1_peripheral 
  * @{
  */

#define RCC_APB1Periph_TIM2              ((uint32_t)0x00000001)
#define RCC_APB1Periph_TIM3              ((uint32_t)0x00000002)
#define RCC_APB1Periph_TIM4              ((uint32_t)0x00000004)
#define RCC_APB1Periph_TIM5              ((uint32_t)0x00000008)
#define RCC_APB1Periph_TIM6              ((uint32_t)0x00000010)
#define RCC_APB1Periph_TIM7              ((uint32_t)0x00000020)
#define RCC_APB1Periph_TIM12             ((uint32_t)0x00000040)
#define RCC_APB1Periph_TIM13             ((uint32_t)0x00000080)
#define RCC_APB1Periph_TIM14             ((uint32_t)0x00000100)
#define RCC_APB1Periph_WWDG              ((uint32_t)0x00000800)
#define RCC_APB1Periph_SPI2              ((uint32_t)0x00004000)
#define RCC_APB1Periph_SPI3              ((uint32_t)0x00008000)
#define RCC_APB1Periph_USART2            ((uint32_t)0x00020000)
#define RCC_APB1Periph_USART3            ((uint32_t)0x00040000)
#define RCC_APB1Periph_UART4             ((uint32_t)0x00080000)
#define RCC_APB1Periph_UART5             ((uint32_t)0x00100000)
#define RCC_APB1Periph_I2C1              ((uint32_t)0x00200000)
#define RCC_APB1Periph_I2C2              ((uint32_t)0x00400000)
#define RCC_APB1Periph_USB               ((uint32_t)0x00800000)
#define RCC_APB1Periph_CAN1              ((uint32_t)0x02000000)
#define RCC_APB1Periph_CAN2              ((uint32_t)0x04000000)
#define RCC_APB1Periph_BKP               ((uint32_t)0x08000000)
#define RCC_APB1Periph_PWR               ((uint32_t)0x10000000)
#define RCC_APB1Periph_DAC               ((uint32_t)0x20000000)
#define RCC_APB1Periph_CEC               ((uint32_t)0x40000000)
 
#define IS_RCC_APB1_PERIPH(PERIPH) ((((PERIPH) & 0x81013600) == 0x00) && ((PERIPH) != 0x00))

/**
  * @}
  */
````



### 3、选择要控制的GPIO引脚

````c
// 选择Pin 9引脚，至于是哪一组，稍后再选择
GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9;
````

​	可以使用的引脚为 0-15一组GPIO口的16个引脚（stm32f10x_gpio.h）；

````c
#define GPIO_Pin_0                 ((uint16_t)0x0001)  /*!< Pin 0 selected */
#define GPIO_Pin_1                 ((uint16_t)0x0002)  /*!< Pin 1 selected */
#define GPIO_Pin_2                 ((uint16_t)0x0004)  /*!< Pin 2 selected */
#define GPIO_Pin_3                 ((uint16_t)0x0008)  /*!< Pin 3 selected */
#define GPIO_Pin_4                 ((uint16_t)0x0010)  /*!< Pin 4 selected */
#define GPIO_Pin_5                 ((uint16_t)0x0020)  /*!< Pin 5 selected */
#define GPIO_Pin_6                 ((uint16_t)0x0040)  /*!< Pin 6 selected */
#define GPIO_Pin_7                 ((uint16_t)0x0080)  /*!< Pin 7 selected */
#define GPIO_Pin_8                 ((uint16_t)0x0100)  /*!< Pin 8 selected */
#define GPIO_Pin_9                 ((uint16_t)0x0200)  /*!< Pin 9 selected */
#define GPIO_Pin_10                ((uint16_t)0x0400)  /*!< Pin 10 selected */
#define GPIO_Pin_11                ((uint16_t)0x0800)  /*!< Pin 11 selected */
#define GPIO_Pin_12                ((uint16_t)0x1000)  /*!< Pin 12 selected */
#define GPIO_Pin_13                ((uint16_t)0x2000)  /*!< Pin 13 selected */
#define GPIO_Pin_14                ((uint16_t)0x4000)  /*!< Pin 14 selected */
#define GPIO_Pin_15                ((uint16_t)0x8000)  /*!< Pin 15 selected */
#define GPIO_Pin_All               ((uint16_t)0xFFFF)  /*!< All pins selected */
````

​	如果需要一次选择多个引脚，将引脚按位 | 即可，因为可以看到每个管脚的定义都是某个二进制位取1；多个|起来后的结果就是相加，比如：

````c
GPIO_InitStruct.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2 | GPIO_Pin_3;
````

​	GPIO_Pin_0到GPIO_Pin_3分别为：
​	0000 0001

​	0000 0010    

​    0000 0100

​	0000 1000

​	按位或后得到 ： 0000 1111



### 4、设置引脚工作模式

````c
// 设置为 推挽输出
GPIO_InitSturcture.GPIO_Mode = GPIO_Mode_Out_PP;
````

在 stm32f10x_gpio.h中可以找到所有的八种模式，每种模式的工作模式详见 [GPIO的工作模式](##  二、GPIO的工作模式)

````c
typedef enum
{ GPIO_Mode_AIN = 0x0,
  GPIO_Mode_IN_FLOATING = 0x04,
  GPIO_Mode_IPD = 0x28,
  GPIO_Mode_IPU = 0x48,
  GPIO_Mode_Out_OD = 0x14,
  GPIO_Mode_Out_PP = 0x10,
  GPIO_Mode_AF_OD = 0x1C,
  GPIO_Mode_AF_PP = 0x18
}GPIOMode_TypeDef;
````



### 5、设置所选引脚的速度

````c
// 配置GPIO口的速度，仅输出有效
GPIO_InitStructure.GPIO_Speed = GPIO_Spedd_50MHz;
````

更多速度选项（stm32f10x_gpio.h）

```c
typedef enum
{ 
  GPIO_Speed_10MHz = 1,
  GPIO_Speed_2MHz, 
  GPIO_Speed_50MHz
}GPIOSpeed_TypeDef;

#define IS_GPIO_SPEED(SPEED) (((SPEED) == GPIO_Speed_10MHz) || ((SPEED) == GPIO_Speed_2MHz) || \
                              ((SPEED) == GPIO_Speed_50MHz))
```



### 6、初始化GPIO

````c
// 初始化 GPIOA 的参数为以上结构体
GPIO_Init(GPIOA, &GPIO_InitStructure);
````

​	在设置好我们需要的引脚号、速度、工作模式后，将这个结构体地址传给官方配置的初始化函数 GPIO_Init函数；这个函数第一个参数是 GPIOx对象，可以是 GPIOA、GPIOB、GPIOC等，传入哪个就代表向初始化的引脚在哪个GPIO组上，第二个参数就是刚刚赋值的GPIO_InitTypeDef对象指针；

​	GPIOA、GPIOB、GPIOC都是官方定义好的 GPIO_TypeDef 对象，GPIO_TypeDef结构体如下：

````c
typedef struct
{
  __IO uint32_t CRL;
  __IO uint32_t CRH;
  __IO uint32_t IDR;
  __IO uint32_t ODR;
  __IO uint32_t BSRR;
  __IO uint32_t BRR;
  __IO uint32_t LCKR;
} GPIO_TypeDef;
````

​	以 GPIOA 为例

​	1、首先，**GPIOA**是将 **GPIOA_BASE** 这个宏强制转换为一个GPIO_TypeDef结构体指针

````
#define GPIOA               ((GPIO_TypeDef *) GPIOA_BASE)
````

​	2、再看 **GPIOA_BASE** 是在 **APB2PERIPH_BASE** 的基础上再偏移 0x0800得到

````
#define GPIOA_BASE            (APB2PERIPH_BASE + 0x0800)
````

​	3、而对 **APB2PERIPH_BASE** 则是在 **PERIPH_BASE** 的基础上偏移 0x10000得到

````c
#define APB2PERIPH_BASE       (PERIPH_BASE + 0x10000)
````

​	4、最后 **PERIPH_BASE** 就是一个 uint32_t 的0x40000000

````c
#define PERIPH_BASE           ((uint32_t)0x40000000)
/*!< Peripheral base address in the alias region */
````

​	综上，我们先从官方定义好的 0x40000000 位置作为 PERIPH 也就是外设区域的基地址，首先在外设区加上0x10000的偏移得到了 APB2PERIPH_BASE 也就是 APB2 总线区域的起始地址，再从APB2中加上 0x0800的偏移最后得到了 GPIOA 这个对象的起始地址，把它转为GPIO_TypeDef结构体指针，我们就可以像使用结构体一样使用 GPIOA，来通过->运算符得到结构体中的寄存器；（具体寄存器的使用见寄存器篇）

````
GPIOA = (GPIO_TypeDef *)((uint32_t)0x40000000 + 0x10000 + 0x0800) 
````

​	其他的GPIOx 形式都是相同的，只是偏移不同，这些都是官方配置好的地址，专门用来保存这个几组GPIO；上述讨论再次验证了 GPIO 是在 APB2这组总线中的；不同的 GPIOx 只是对于 APB2PERIPH_BASE 的偏移不同而已；



### 7、总览

​	以 STM32F103RC 为例：

````c
// 声明一个 GPIO_InitTypeDef 对象
GPIO_InitTypeDef GPIO_InitStructure;

// 打开 GPIOA 的时钟
RCC_APB2PeriphClockCmd(RCC_AHB2Periph_GPIOA, ENABLE);

// 选择管脚
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;

// 选择工作模式
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;

// 选择速度
GPIO_InitStructure.GPIO_Speed = GPIO_Spedd_50MHz;

// 初始化
GPIO_Init(GPIOA, &GPIO_InitStructure);

// 结果将 GPIOA 组上的0号管脚以速度为50MHz设为推挽输出模式
````





## 五、 GPIO的控制函数

### 1、复位IO口

​	**函数原型**

````c
void GPIO_DeInit(GPIO_TypeDef* GPIOx);
````

​	**功能**：将传入的IO口组设置为默认状态；

​	**参数**：

- 参数一：需要设置默认状态的IO口组



### 2、IO初始化

​	**函数原型**

````c
void GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* GPIO_InitStruct);
````

​	**功能**：初始化某个函数组的若干引脚；

​	**参数**：

- 参数一：需要初始化的引脚所在的函数组；
- 参数二：保存待初始化引脚的信息的结构体指针；



### 3、读取引脚电平

​	**函数原型**

````c
uint8_t GPIO_ReadInputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
````

````c
uint8_t GPIO_ReadOutPutDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
````

**功能**：读取指定IO组的指定引脚的电平信号；

​	**参数**：

- 参数一：待读取引脚所在的引脚组；
- 参数二：待读取引脚的引脚号；

​	**返回值**：

​	返回 0 或 1 表示引脚电平的低与高（如果一个 GPIO 组中一个管脚都没有被初始化，直接读取这个 GPIO 组的电平都默认是0，但是只要有一个引脚被初始化，其他没有初始化的引脚读取到的默认是1）；



### 4、读取引脚组电平

​	**函数原型**：

````c
uint16_t GPIO_ReadInputData(GPIO_TypeDef* GPIOx);
````

```c
uint16_t GPIO_ReadOutputData(GPIO_TypeDef* GPIOx);
```

​	**功能**：读取IO组的电平信息；

​	**参数**：

- 参数一：待读取的IO组对象；

​	**返回值**：

​	按照此IO组的引脚号，输出从低到高为一bit代表对应管脚的电平；



### 5、设置引脚输出高电平

​	**函数原型**：

````c
void GPIO_SetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
````

​	**功能**：设定指定 IO 组的指定引脚输出高电平

​	**参数**：

- 参数一：希望修改的引脚所在的 IO 组；
- 参数二：希望修改的引脚号；



### 6、设置引脚输出低电平

​	**函数原型**：

````c
void GPIO_ResetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
````

​	**功能**：设定指定 IO 组的指定引脚输出低电平

​	**参数**：

- 参数一：希望修改的引脚所在的 IO 组；

- 参数二：希望修改的引脚号；

    



### 7、设置引脚的输出电平

​	**函数原型**：

````c
void GPIO_WriteBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, BitAction BitVal);
````

​	**功能**：设定指定 IO 组的指定引脚输出高或低电平；

​	**参数**：

- 参数一：希望修改的引脚所在的 IO 组；
- 参数二：希望修改的引脚号；
- 参数三：指定输出高或低电平；

​	BitAction 是一个简单的枚举：

````c
typedef enum
{ Bit_RESET = 0,
  Bit_SET
}BitAction;
````



### 8、设置一组引脚的输出电平

​	**函数原型**：

````c
void GPIO_Write(GPIO_TypeDef* GPIOx, uint16_t PortVal);
````

​	**功能**：设定指定 IO 组的指定引脚输出高或低电平；

​	**参数**：

- 参数一：希望修改的 IO 组；
- 参数二：希望设置的电平（一个bit一个bit依次付给各个引脚）





## 六、GPIO相关的寄存器

​	GPIO 寄存器用来保存 GPIO 的相关信息，比如速度、工作模式、电平等。

​	官方将每组 GPIO 组的信息用一个 GPIO_TypeDef 结构体来储存，并且已经划分好了一些地址（地址的推导见4.6 初始化 GPIO ）用来保存这些结构体，这也就是在 GPIO初始化时用到的 GPIOA、GPIOB、GPIOC等。所以当我们初始化 GPIO 时，其实就是修改了对应 GPIO 结构体对象中的一些值，这些值我们成为 GPIO 的寄存器；

​	如下代码所示，每个 GPIO_TypeDef 结构体都有 7 个 uint32_t 成员，这些就是寄存器，GPIO 管脚的信息就保存在这里；

````c
typedef struct
{
  __IO uint32_t CRL;
  __IO uint32_t CRH;
  __IO uint32_t IDR;
  __IO uint32_t ODR;
  __IO uint32_t BSRR;
  __IO uint32_t BRR;
  __IO uint32_t LCKR;
} GPIO_TypeDef;
````

​	接下来对这 8 个寄存器中的常用的几个作介绍



### 1、CRL

​	CRL寄存器，即 control register-low，是用来配置此 GPIO 组中低 8 个 GPIO 端口的信息；配置一个引脚需要用 4 个 bits ，那么一个 32-bit 的 CRL 寄存器只能用来配置  8 个引脚；

​	0 - 3 bit 属于 Pin_0 ;  4 - 7 bit 属于 Pin_1 ; 8 - 11 bit 属于 Pin_2 ; 12 - 15 bit 属于 Pin_3 ; 

​	16 - 19 bit 属于 Pin_4 ;  20 - 23 bit 属于 Pin_5 ; 24 - 27 bit 属于 Pin_6 ; 28 - 31 bit 属于 Pin_7 ; 

​	每 4-bit 一组的控制信息又分成两部分，高 2-bit 为配置模式的 MODE ，低 2-bit 为配置工作方式的 CNF 

​	对 MODE 段有：

| 高位 bit | 地位 bit | 配置信息                          |
| -------- | -------- | --------------------------------- |
| 0        | 0        | 设置引脚为输入模式                |
| 0        | 1        | 设置引脚为最大速度10MHz的输出模式 |
| 1        | 0        | 设置引脚为最大速度2MHz的输出模式  |
| 1        | 1        | 设置引脚为最大速度50MHz的输出模式 |

​	对 CNF 段有：

| 高位 bit | 地位 bit | 配置信息（输入模式下） | 配置信息（输出模式下） |
| -------- | -------- | ---------------------- | ---------------------- |
| 0        | 0        | 模拟输入模式           | 通用推挽输出模式       |
| 0        | 1        | 浮空输入模式           | 通用开漏输出模式       |
| 1        | 0        | 上下拉输入模式         | 复用功能推挽输出模式   |
| 1        | 1        | （保留）               | 复用功能开漏输出模式   |



### 2、CRH

​	CRH 寄存器， 即 control register-high，用来配置高八位从 Pin 8 到 Pin 15 的引脚信息，跟 CRL 相似；



### 3、IDR

​	IDR 寄存器，即 input data register，来保存输入模式下的引脚电平信息；因为一组 GPIO 只有 16 个引脚，所以 IDR 的高 16 位是保留位，全0，低 16 位保存对应引脚的电平信息，1 为高电平，0为低电平。



### 4、ODR

​	ODR 寄存器，即 output data register，与 IDR 类型， 高 16 位保留不使用，低 16 位用来配置对应的引脚电平。修改后等待下一个上升或下降沿来到时，会修改对应引脚的值；

​	当对应的引脚配置为输入，那么写入的值就无法写入到引脚上，但是可以来控制输入模式是上拉还是下拉，1表示上拉，0表示下拉；



### 5、BSRR

​	BSRR 寄存器，即 bit set/reset register；用来设置或重载引脚的电平；

​	将 32 bit 分为高 16 位的 Reset 区与低 16 位的 Set 区；

​	对 Reset 区，写入 0 表示无效，写入 1 表示将对应的引脚电平设置为低；

​	对 Set 区，写入 0 表式无效，写入 1 表示将对应的引脚电平设为高；

​	当 Reset 与 Set 区同时起作用，仅 Set 区域起作用；



### 6、BSR

​	BSR 寄存器，即 bit set register; 用来设置引脚电平为高；

​	高 16 位保留不用，低 16 位与 BSRR 的 Set 区功能相同；



