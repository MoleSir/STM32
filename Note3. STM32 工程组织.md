# STM32工程组织

## 一、建目录拷代码

- 创建一个工程文件夹，在这个文件夹下新建四个文件夹：**st_lib、cmsis、user、driver**；

- 将标准库下：\Libraries\STM32F10x_StdPeriph_Driver\ 里的 **inc** 与 **src** 文件夹拷贝到自己工程文件夹下的 stb_lib 目录；
- 将标准库下：\Libraries\CMSIS\CM3\CoreSupport\ 里面的 **core_cm3.c** 与 **core_cm3.h** 拷贝到自己工程文件夹下的 cmsis 目录；
- 将标准库下：\Libraries\CMSIS\CM3\DeviceSupport\ST\STM32F10x\startup\arm 里的 **startup_stm32f10x_md.s** 文件拷贝到自己工程文件夹下的 cmsis 目录；
- 在标准库下官方提高的ex中选择一个将里面的内容拷贝到自己工程文件夹下的 user 目录，这里使用\Project\STM32F10x_StdPeriph_Examples\GPIO\IOToggle；
- 在标准库下：\Libraries\CMSIS\CM3\DeviceSupport\ST\STM32F10x 里的三个文件：stm3210x.h、system_stm32f10x.h、system_stm3210fx.c 拷贝到自己工程文件夹下的 user 目录，如果有文件重复，点击替换；



## 二、建工程、添群组

- 打开 Keil MDK 软件 $\to$ 点击 Project 下的 New  $\mu$Vision Project $\to$ 弹出新建工程文件的对话框；
- 选择自己新建的工程目录下的 user 文件夹，在这里新建一个 .uvproj 的工程启动文件，名字可自己取；
- 新建后会弹出新的对话框，选择对应的芯片：STMircroelectronics $\to$ STM32F Series $\to$ STM32F 103 $\to$ STM32F103RC 后点击ok；
- 新建完毕后开始添加群组。在 Keil MDK 软件中 Project 栏中右击 Target1 $\to$ Manage Project items $\to$ 弹出对话框；
- 删除默认的群组 source group ，在 Group 中添加 st_lib、cmsis、startup、user 四个群组；
- 把自己工程文件中 st_lib/src、cmsis、user中的 .c 文件加入刚刚创建的群组，注意文件名对应群组名；
- 特别得，对 startup 群组，需要加入工程文件夹下 cmsis 目录下的 startup_stm32f10x_hd.s 文件；



## 三、启动文件的选择问题

​	在上述步骤中，我们选择了 startup_stm3210fx_hd.s 作为启动文件，是因为我们使用了 stm32f103RC 这个型号的芯片；

​	对芯片名称：STM32F103XY，X表示芯片拥有的引脚数量：C = 48PIN，R = 64PIN，V = 100PIN，Z= 144PIN；而 Y 表示 FLASH 容量的大小：4 = 16KB，6 = 32KB，8 = 64KB，B = 128KB，C = 256KB，D = 384KB，E = 512KB。那么 STM32F103RC 就表示有 64PIN、256KB 的芯片；

​	对于不同的芯片要使用不同的启动文件，因为启动文件需要划分内存；通常有：

|        启动文件        |                         区别                         |
| :--------------------: | :--------------------------------------------------: |
| startup_stm32f10x_ld.s |  ld：low-density  小容量，  FLASH 在 16 - 32K 之间   |
| startup_stm32f10x_md.s | md：medium-density 中容量，  FLASH 在 64 - 128K 之间 |
| startup_stm32f10x_hd.s |  hd：high-density 大容量，  FLASH 在256 - 512K 之间  |
| startup_stm32f10x_xl.s |      xl：超大容量，  FLASH 在512K - 1024K 之间       |

​	所以 STM32F103RC 使用大容量的启动文件 startup_stm32f10x_hd.s；



## 四、慎配置、看编译

- 打开刚刚建立好的工程文件；
- 点击魔术棒图标（Option for Target）弹出对话框
- 点击 Device，选择芯片：这里选 STM32F103RC ；
- 点击 Target，修改 Xtal(MHz) 为 8；
- 点击 C/C++，添加宏定义： STM3210F_HD,USE_STDPERIPH_DRIVER；
- 添加头文件路径（include path）：主要需要包含：
    - 自己工程文件夹下的 std_lib\inc；
    - 自己工程文件夹下的 user；
    - 自己工程文件夹下的 cmsis；
- 去掉 main.c 中 多余的部分；