# STM32启动文件

​	对于常用的桌面操作系统而言，如最常见的Windows系统，我们并不关心系统的初始化，我们只要点击开机按钮，由微软编写的优质代码就可以直接运行在我们的电脑上，为我们提供很多系统调用等功能；

​	然而对于嵌入式设备而言，在设备上电后，所有的一切都需要由开发者来设置，这里处理器是没有堆栈，没有中断，更没有外围设备，这些工作是需要软件来指定的，而且不同的CPU类型、不同大小的内存和不同种类的外设，其初始化工作都是不同的。

​	所以我们需要了解在STM32进入main之前都做了什么，为什么我们的程序会被加载，启动文件里都有些什么？



## 1、启动文件分析

### 1.1 概述

​	每一个STM32工程都必须包含一个启动文件 startup_stm32f10x_xx.s（根据所用的芯片不同使用不同的启动文件），一个汇编程序；

​	当STM32上电后，内核会首先执行这个文件的内容，完成一些基本的工作；



### 1.2 启动文件的重要工作

​	每一个启动文件都必须完成：

- 设置堆栈指针MSP = _initial_sp；
- 设置PC指针 = Reset_Handler；
- 配置系统时钟；
- 配置外部 SRAM 用于程序变量等数据的存储（可选）；
- 调用C库中的__main函数，最终调用main；



### 1.3 ARM 常用汇编

​	启动文件是使用汇编语言编写的，这里列出几个接下来会使用的ARM汇编指令：

|   指令名称    |                             作用                             |
| :-----------: | :----------------------------------------------------------: |
|      EQU      |                给数字常量取名，类似于 #define                |
|     AREA      |                  汇编一个新的代码段或数据段                  |
|     SPACE     |                         分配内存空间                         |
|   PRESERVES   |                 当前文件堆栈需按照8字节对齐                  |
|    EXPORT     |         声明一个变量具有全局属性，可以被外部文件引用         |
|      DCD      |      以字节为单位分配内存，要求4字节对齐，并且要初始化       |
|     PROC      |                  定义子程序，与ENDP配合使用                  |
|     WEAK      | 若定义，当外部文件定义了同名符号就使用外部，若没有则使用此文件内部的 |
|    IMPORT     |              声明符号来自外部文件，类似 extern               |
|       B       |                        跳转到某个符号                        |
|     ALIGN     |         对指令或数据段对齐，一般需要再跟上一个立即数         |
|      END      |                        表示文件的终止                        |
| IF ELSE ENDIF |                        类似于 if else                        |



### 1.4 启动文件代码分析

#### 1.4.1 开辟栈空间

```assembly
Stack_Size      EQU     0x00000400 
                AREA    STACK, NOINIT, READWRITE, ALIGN=3 
Stack_Mem       SPACE   Stack_Size
__initial_sp
```

1、首先，定义了 Stack_Size 这个符号为 0x00000400，表示栈的大小：

2、接着，使用 AREA 开辟一段空间名为 STACK，后面跟着的表示属性：NOINIT（不初始化）、READWRITE（可	读可写）、ALIGN（要求2^3 = 8字节对齐）；

3、最后，使用 SPACE 分配了一段 Stack_Size 也就是 0x00000400 字节的空间；

​	并且这个空间的最高位命名为 __initial_sp；



#### 1.4.2 开辟堆空间

```assembly
Heap_Size       EQU     0x00000200
                AREA    HEAP, NOINIT, READWRITE, ALIGN=3
__heap_base
Heap_Mem        SPACE   Heap_Size
__heap_limit
```

1、首先，定义了 Heap_Size 这个符号为 0x00000200，表示堆的大小：

2、接着，使用 AREA 开辟一段空间名为 HEAP，后面跟着的表示属性：NOINIT（不初始化）、READWRITE（可	读可写）、ALIGN（要求2^3 = 8字节对齐）；

3、最后，使用 SPACE 分配了一段 Heap_Size 也就是 0x00000200 字节的空间；

​	并且这个空间的最低为命名为 \__heap_base, 最高位命名为 __heap_limit；



#### 1.4.3 定义向量表符号

````assembly
				PRESERVE8
                THUMB

; Vector Table Mapped to Address 0 at Reset
                AREA    RESET, DATA, READONLY
                EXPORT  __Vectors
                EXPORT  __Vectors_End
                EXPORT  __Vectors_Size
````

​	1、PRESERVE8 指定当前文件保存堆栈8字节对齐；

​	2、THUMB 表示之后的指令是 THUMB 指令集，CM4采用的是 THUMB - 2指令集；

​	3、接下来使用 AREA 开辟了一个 RESET 的空间，并且附上 DATA（数据段）、READONLY（只读）；

​	4、之后三句表明，____Vectors,  ____Vectors_End、__Vectors_Size 这个三个符号可以被外部使用；

​		这三个符号都是关于中断向量表的；



#### 1.4.4 建立中断向量表

```assembly
__Vectors       DCD     __initial_sp               ; Top of Stack
                DCD     Reset_Handler              ; Reset Handler
                DCD     NMI_Handler                ; NMI Handler
                DCD     HardFault_Handler          ; Hard Fault Handler
                DCD     MemManage_Handler          ; MPU Fault Handler
                DCD     BusFault_Handler           ; Bus Fault Handler
                DCD     UsageFault_Handler         ; Usage Fault Handler
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     SVC_Handler                ; SVCall Handler
                DCD     DebugMon_Handler           ; Debug Monitor Handler
                DCD     0                          ; Reserved
                DCD     PendSV_Handler             ; PendSV Handler
                DCD     SysTick_Handler            ; SysTick Handler

                ; External Interrupts
                DCD     WWDG_IRQHandler            ; Window Watchdog
                DCD     PVD_IRQHandler             ; PVD through EXTI Line detect
                DCD     TAMPER_IRQHandler          ; Tamper
                DCD     RTC_IRQHandler             ; RTC
                DCD     FLASH_IRQHandler           ; Flash
                DCD     RCC_IRQHandler             ; RCC
                DCD     EXTI0_IRQHandler           ; EXTI Line 0
                
                ;........
                
                DCD     EXTI15_10_IRQHandler       ; EXTI Line 15..10
                DCD     RTCAlarm_IRQHandler        ; RTC Alarm through EXTI Line
                DCD     USBWakeUp_IRQHandler       ; USB Wakeup from suspend
__Vectors_End

__Vectors_Size  EQU  __Vectors_End - __Vectors
```

​	1、中间还省略了很多，主要就是在创建中断向量表；

​	2、在\__Vectors 与 __Vectors之间分配许多的符号，这些符号每个占4字节，都是中断处理函数的函数名称，这	里先建立起一张由函数地址组成的中断向量表；

​	3、定义了 \__Vectors_Szie 是由 \__Vectors_End - __Vectors 得到，既表尾减去表头得到表的大小；

​	特别需要注意的是，中断向量表的第一各表项是放置栈顶 `__initial_sp` ，而第二行放置了 `Reset_Handler`	函数地址；



#### 1.4.5 编写Reset_Hanlder

````assembly
                AREA    |.text|, CODE, READONLY

; Reset handler
Reset_Handler    PROC
                 EXPORT  Reset_Handler             [WEAK]
     IMPORT  __main
     IMPORT  SystemInit
                 LDR     R0, =SystemInit
                 BLX     R0
                 LDR     R0, =__main
                 BX      R0
                 ENDP
````

​	1、首先开辟了一个 .text 段附加 CODE（代码段）、READONLY（只读）属性；

​	2、齐次，声明一个符号 Reset_Handler 具有 EXPORT（可被外部文件引用属性），并且是[WEAK]，既如果外部文件也定义了 Reset_Hanlder 这个符号，优先使用外部符号；

​	3、接着声明 __main 与 SystemInit 这两个符号是来自外部的；

​	4、之后利用 PROC 与 ENDP 为 Reset_Hanlder 定义函数体，函数体执行：

​		4.1、`LDR R0, =SystemInit`--> 将 SystemInit 符号的地址传入 R0 寄存器；

​		4.2、`BLX R0`--> PC跳转到 R0 寄存器上的值 --> 执行 SystemInit；

​		4.3、`LDR R0, =__main`--> 将 __main 符号的地址传入 R0 寄存器；

​		4.4、`BLX R0`--> PC跳转到 R0 寄存器上的值 --> 执行 __main；

​	这里出现的两个函数，SystemInit 定义在 system_stm32f10x.c 里，其主要负责初始化系统时钟与重定位中断向量表之后会再细说；

​	而__main主要是负责初始化堆栈，完成库函数加载，最后跳转到main()函数，之后会细说；



#### 1.4.6 编写中断处理函数

```assembly
NMI_Handler     PROC
                EXPORT  NMI_Handler                [WEAK]
                B       .
                ENDP
HardFault_Handler\
                PROC
                EXPORT  HardFault_Handler          [WEAK]
                B       .
                ENDP
MemManage_Handler\
                PROC
                EXPORT  MemManage_Handler          [WEAK]
                B       .
                ENDP
BusFault_Handler\
                PROC
                EXPORT  BusFault_Handler           [WEAK]
                B       .
                ENDP
                ;................
DMA1_Channel7_IRQHandler
ADC1_2_IRQHandler
USB_HP_CAN1_TX_IRQHandler
USB_LP_CAN1_RX0_IRQHandler
CAN1_RX1_IRQHandler
CAN1_SCE_IRQHandler
EXTI9_5_IRQHandler
TIM1_BRK_IRQHandler
TIM1_UP_IRQHandler
TIM1_TRG_COM_IRQHandler
TIM1_CC_IRQHandler
TIM2_IRQHandler
TIM3_IRQHandler
TIM4_IRQHandler
I2C1_EV_IRQHandler
I2C1_ER_IRQHandler
I2C2_EV_IRQHandler
I2C2_ER_IRQHandler
SPI1_IRQHandler
SPI2_IRQHandler
USART1_IRQHandler
USART2_IRQHandler
USART3_IRQHandler
EXTI15_10_IRQHandler
RTCAlarm_IRQHandler
USBWakeUp_IRQHandler

                B       .

                ENDP

                ALIGN
```

​	1、还记得1.4.3中创建了一个表吗，当时给每个表项一个中断处理函数名，而现在就是编写这些函数；

​	2、对每个函数都是一样的，先规定这些函数是 EXPORT（可以被外部引用），[WEAK]（若外部有定义优先使用	外部的符号）；

​	3、把这个函数都写为`B  .`表示无限循环；

​	所以，如果我们不自己写中断处理函数，或者函数名称写错了，那么自动调用到这些无限循环的函数中去；



#### 1.4.7 堆栈初始化

````assembly
                 IF      :DEF:__MICROLIB           
                
                 EXPORT  __initial_sp
                 EXPORT  __heap_base
                 EXPORT  __heap_limit
                
                 ELSE
                
                 IMPORT  __use_two_region_memory
                 EXPORT  __user_initial_stackheap
                 
__user_initial_stackheap

                 LDR     R0, =  Heap_Mem
                 LDR     R1, =(Stack_Mem + Stack_Size)
                 LDR     R2, = (Heap_Mem +  Heap_Size)
                 LDR     R3, = Stack_Mem
                 BX      LR

                 ALIGN

                 ENDIF

                 END
````

​	1、判断是否定义了 __MICROLIB 这个宏

​		1.1 如果有将 `__initial_sp, __heap_base, __heap_limit`三个符号设置为可被外部使用；这是因为如果		我们定义了__MICROLIB ，说明我们拥有微软为我们写好的初始化堆栈函数，我们就只要把之前定义好的一些		堆栈信息给这个函数调用即可；具体如何初始化我们不需要关心；

​		1.2 如果没有，那么就设定  `__use_two_region_memory`这个符号是来自外部，并且定义了一个可以被外部		使用的符号 `__user_initial_stackheap`；这是因为如果我们没有微软写好的初始化堆栈程序，那么就必须		自己写一个   `__use_two_region_memory`；

​	2、最后定义了 `__user_initial_stackheap` 的函数体，就是将一些堆栈信道放在寄存器上，给我们在外部定		义的堆栈初始化函数 `__use_two_region_memory`使用；



#### 1.4.8 总结

​	当内核执行完启动文件，内核已经在SRAM里分配好了栈区与堆区，完成了一张中断向量表，特别要注意的是中断向量表的第一个表项为栈顶指针，第二个为 Reset_Handler 地址，编写了一段中断复位程序到FLASH中；



## 2、启动过程分析

### 2.1 上电取址

​	在STM32上电后，Cortex-M3 内核会去内存绝对地址中的0x00处取出其数值，赋值给栈指针寄存器MSP，再去0x04处取出其数据，赋值给程序计数器PC；

​	其实这里的取到的两个值就是中断向量表的前两个表项，但是我们知道中断向量表是放置在FLASH --> 0x0800 0000，那为什么去0x00取地可以获取到这个位置呢，是因为STM32外部做了一些重定位，使得内核去0x00取值时刚好也能取到 0x0800 0000 上；

​	这样内核就获得了1、帧顶地址，2、复位程序 Reset_Handler 地址；



### 2.2 执行复位程序

​	接下来，内核执行 PC 处的程序——Reset_Handler，即执行 SystemInit 与 __main；

​	最后进入 main 中完成我们编写的 C 程序；

​	

​	

