---
layout: post
os: true
comments: true
title:  "freertos"
excerpt: "..."
tag:
- os
- freertos

---



# reference

FreeRTOS Documentation：https://www.freertos.org/Documentation/RTOS_book.html



# project

- src

  RTOS Source Code Download Instructions：https://www.freertos.org/a00104.html



- files
  |                                           |                                     |
  | ----------------------------------------- | ----------------------------------- |
  | FreeRTOSv10.1.1\FreeRTOS\Source          | os\freertos\Source               |



- modify

  - 补充
    ``` 
    @portmacro.h
    +#include "projdefs.h"	
    +#include "FreeRTOSConfig.h" 
    ```
    
  - 配置接口
      - 系统节拍  xPortSysTickHandler
      - 任务切换函数  xPortPendSVHandler
      - 任务切换函数  vPortSVCHandler

  - 使用了FreeRtos操作系统后，外设中断优先级值不能设置小于configMAX_SYSCALL_INTERRUPT_PRIORITY，否则会导致了宏configASSERT的条件成立，进而卡在`ucCurrentPriority = pcInterruptPriorityRegisters[ ulCurrentInterrupt ];`





    操作系统的最最底层的几个文件也需要用到FreeRTOSConfig.h头文件，而底层文件是用汇编来写的，因此必须在Assembler下添加FreeRTOSConfig.h头文件路

FreeRTOSConfig.h里面几乎都是一些宏定义，关于这些宏定义的具体用法，可以在官网上查阅：<http://www.freertos.org/a00110.html>

- 定义系统底层相关的函数
  SVC中断是操作系统启动时进入的中断
  PendSV中断手动切换任务时进入的中断
  SysTick是作为操作系统的心脏
  由于FreeRTOS对这几个中断的名称做了自己的定义，因此必须要重定义这几个函数才能正常进入中断，但这么做又会跟ST提供的stm32f10x_it.c文件当中定义的中断相冲突，因此必须将stm32f10x_it.c下对应的几个中断服务函数屏蔽掉，否则编译会提示函数重定义

- 修改系统可屏蔽的中断优先级阈值
  FreeRTOS提供的可屏蔽中断优先级阈值是191，对应的十六进制数是0xBF：
  #define configMAX_SYSCALL_INTERRUPT_PRIORITY         191
  由于STM32F103的优先级分组只有4个位，而CM3的优先级是以MSB对齐的，也就是说STM32F103的优先级寄存器只有最高4位有效，低四位是无效的。当操作系统进入临界区时，会把上面的可屏蔽中断优先级阈值写入BASEPRI寄存器以屏蔽部分中断：

  因此当进入临界区时，优先级对应0xB~0xF的中断均被屏蔽，而优先级处于0xB前面的中断不受影响。这个跟CM0有区别，也是最值得注意的地方。

  到底为何要重设可屏蔽的中断优先级阈值，我们重新把思路理一下。根据STM32的中断优先级的设计，只有高4位有效，还有FreeRTOS默认将4个优先级位均划分为抢占优先级。由于FreeRTOS官方提供的中断优先级阈值是191（对应实际的0xB），也就是11~15的优先级均可被操作系统屏蔽。但我们实际使用时设置的中断优先级一般不会使用到11打后的，例如@正点原子的基础例程里面使用最多的1~3，所以我们必须要修改这个值，否则我们要重新修改所有底层驱动的优先级。
  那么怎么修改比较合理？这个就得看实际应用需要了，其实使用宏configMAX_SYSCALL_INTERRUPT_PRIORITY来屏蔽部分中断是比较合理的，相对于CM3，CM0没有中断优先级阈值寄存器，只能简单粗暴的开启全局中断和关闭全局中断。操作系统在执行十分重要的工作时一般不能打断这个工作，尤其是这时在中断里面调用了操作系统的API函数，这都会严重影响系统的稳定性。对configMAX_SYSCALL_INTERRUPT_PRIORITY的理解是，这个数值打后的所有中断均划入操作系统管理，而这个数值打前的中断则归由用户自己管理，但用户必须十分小心地处理这些中断，用户可以使用这些中断来处理一些跟操作系统无关的工作

  由于优先级寄存器是高四位有效，因此上述的屏蔽阈值实际上是0x1，也就是说优先级在1~15之间的中断均可被操作系统屏蔽，而优先级0归由用户自己控制。值得注意的是configMAX_SYSCALL_INTERRUPT_PRIORITY的高四位绝对不能设为0，

  测试：#define configMAX_SYSCALL_INTERRUPT_PRIORITY 0x0F
  系统在进入临界区时BASEPRI寄存器的值一直为0，没有更新

  测试：#define configMAX_SYSCALL_INTERRUPT_PRIORITY 0x1F
  当configMAX_SYSCALL_INTERRUPT_PRIORITY设为0x1F时系统在进入临界区时BASEPRI寄存器的值更新为0x10，中断优先级阈值寄存器起作用了！原因在上面也解释过了，因为STM32F103的优先级寄存器是高四位有效的，对应的BASEPRI也是高四位能够写入而低四位无法写入，而BASEPRI有一个特点——对它写0会取消屏蔽所有中断（相当于禁用了该寄存器），因此BASEPRI的高四位一定不能设为0，否则不会屏蔽任何中断，这点注意下就好了。

  

- 添加参数检测功能

  #define configASSERT( x ) if( ( x ) == 0 ) { taskDISABLE_INTERRUPTS(); for( ;; ); }	

  该参数检测功能是官方提供的一个参考，很多人为了省事不使用参数检测

  





问题：
No space in execution regions with .ANY selector matching hal_cm.o(.data).





问题：

task_algo的心跳包文时间间隔不准确，比预期的间隔长1~2秒

分析：该问题是解决20190506-uplink abort之后产生的，即：

```C
task_lora.c  -  Task_LoRa
case SEND_REQUEST:
{
        DEBUG("SEND_REQUEST");
        LoRa_Port_SendRequest();
+       DelayMs(4000); 
        xTaskNotify(task_led_handle, 0, eSetValueWithOverwrite);
```

心跳包文时间间隔=Task_Algo_Alive_Time：

```C
task_lora.c  -  Task_LoRa
ulNotificationValue = xTaskNotifyWait(0x00,0xffffffff,&event,Task_Algo_Alive_Time);
```

正常来讲，上面的xTaskNotifyWait语句，应该准确地每隔Task_Algo_Alive_Time执行一次，即：
mcu由于xTaskNotifyWait超时而被唤醒
间隔时间：Internal_time，唤醒后进入Task_Algo的非超时执行部分(while(1)中的else部分)，执行心跳报文的发送
运行到 xTaskNotifyWait处，立即开始计时
额外时间：Extral_time，开始计时后再执行一些其他程序，假设这些程序消耗了Extral_time的时间
额外时间过后，mcu进入睡眠，并且设定睡眠时间为Task_Algo_Alive_Time - Extral_time
关键是要保证Internal_time接近于0，准确地每隔Task_Algo_Alive_Time执行一次上面的xTaskNotifyWait语句

现在在debug模式下测得Internal_time有1~2s，显然是alive time inaccurate的原因了
现在的情况：发送心跳包文之后至少延时示4000ms，程序才会再次执行到xTaskNotifyWait处。





//如果使用了freertos，由于systick中断中会执行调度，因此调用vTaskStartScheduler函数之前应该关闭systick中断
//另一方面，mcu的各外设的启动可能会使用到systick中断，所以一般，外设的启动也放到任务中。




# IDE

- Keil

    - source

      os\freertos\Source\ *.c
      os\freertos\Source\portable\MemMang\heap_4.c
      os\freertos\Source\portable\RVDS\ARM_CM4F\port.c
      os\freertos\app\ *.c
      os\freertos\bsp\stm32l433\ *.c

    - include
    
      ..\os\freertos\Source\include
      ..\os\freertos\Source\portable\RVDS\ARM_CM4F
      ..\os\freertos\app
      ..\os\freertos\bsp\stm32l433





# task

##### 任务列表(List)与列表项(Listitem)

- 列表-List_t

  - uxNumberOfItems：记录列表中列表项的数量
  - pxIndex：记录当前列表项索引号，用于遍历列表
  - xListEnd：列表中最后一个列表项，用来表示列表结束，此变量类型为MiniListItem_t，这是一个迷你列表项
  - listSECOND_LIST_INTEGRITY_CHECK_VALUE：检查列表完整性
  - listFIRST_LIST_INTEGRITY_CHECK_VALUE：检查列表完整性

- 列表项-ListItem_t

  - xItemValue ：列表项值
  - pxNext ：指向下一个列表项。
  - pxPrevious ：指向前一个列表项，和pxNext 配合起来实现类似双向链表的功能。
  - pvOwner ：记录此列表项归谁拥有，通常是任务控制块。
  - pvContainer ：用来记录此列表项归哪个列表。注意和pvOwner 的区别，在前面讲解任务控制块TCB_t 的时候说了在TCB_t 中有两个变量xStateListItem 和xEventListItem，这两个变量的类型就是ListItem_t，也就是说这两个成员变量都是列表项。以xStateListItem 为例，当创建一个任务以后xStateListItem 的pvOwner 变量就指向这个任务的任务控制块，表示xSateListItem属于此任务。当任务就绪态以后xStateListItem 的变量pvContainer 就指向就绪列表，表明此列表项在就绪列表中。举个通俗一点的例子：小王在上二年级，他的父亲是老王。如果把小王比作列表项，那么小王的pvOwner 属性值就是老王，小王的pvContainer 属性值就是二年级。列表项结构示意图如图7.1.2.1 所示：

- 迷你列表项-MiniListItem_t，列表项的缩减版(节省内存)

  - xItemValue 

  - pxNext 

  - xPrevious 

    

- 列表初始化
  新创建或者定义的列表需要对其做初始化处理，列表的初始化其实就是初始化列表结构体List_t 中的各个成员变量，列表的初始化通过使函数vListInitialise()来完成，此函数在list.c 中有定义，函数如下：

  - xListEnd 用来表示列表的末尾，而pxIndex 表示列表项的索引号，此时列表只有一个列表项，那就是xListEnd，所以pxIndex 指向xListEnd
  - xListEnd 的列表项值初始化为portMAX_DELAY， portMAX_DELAY 是个宏，在文件portmacro.h 中有定义。根据所使用的MCU 的不同，portMAX_DELAY 值也不相同，可以为0xffff或者0xffffffffUL，本教程中为0xffffffffUL
  - 初始化列表项xListEnd 的pxNext 变量，因为此时列表只有一个列表项xListEnd，因此pxNext 只能指向自身
  - 同(3)一样，初始化xListEnd 的pxPrevious 变量，指向xListEnd 自身
  - 由于此时没有其他的列表项，因此uxNumberOfItems 为0，注意，这里没有算xListEnd
  - 初始化列表项中用于完整性检查字段， 只有宏configUSE_LIST_DATA_INTEGRITY_CHECK_BYTES 为1 的时候才有效。同样的根据所选的MCU 不同其写入的值也不同，可以为0x5a5a 或者0x5a5a5a5aUL。STM32 是32 位系统写入的是0x5a5a5a5aUL，列表初始化完以后如图7.2.1.1 所示：
    注意，图7.2.1.1 为了好分析，将xListEnd 中的各个成员变量都写了出来

- 列表项初始化
  同列表一样，列表项在使用的时候也需要初始化，列表项初始化由函数vListInitialiseItem()来完成，函数如下：
  列表项的初始化很简单，只是将列表项成员变量pvContainer 初始化为NULL，并且给用于完整性检查的变量赋值。有朋友可能会问，列表项的成员变量比列表要多，怎么初始化函数就这么短？其他的成员变量什么时候初始化呢？这是因为列表项要根据实际使用情况来初始化，比如任务创建函数xTaskCreate()就会对任务堆栈中的xStateListItem 和xEventListItem 这两个列表项中的其他成员变量在做初始化，任务创建过程后面会详细讲解

- 列表项插入-vListInsert

  





##### 任务调度

我们会在程序开始先创建若干个任务， 而此时任务调度器还没又开始运行，因此每一次任务创建后都会依据其优先级插入到就绪链表，同时保证全局变量 `pxCurrentTCB` 指向当前创建的所有任务中优先级最高的一个，但是任务还没开始运行。
当初始化完毕后，调用函数 `vTaskStartScheduler`启动任务调度器开始开始调度，此时，`pxCurrentTCB`所指的任务才开始运行。



- vTaskStartScheduler：创建系统所需任务和初始化相关静态变量，然后启动调度器
  - 创建空闲任务：prvIdleTask
  - 创建定时器管理的任务
  - 调用移植层提供的函数 xPortStartScheduler启动调度器 。
- xPortStartScheduler
  - 设置节拍定时器
  - 启动第一个任务
  - 开始系统正常运行调度

与 FreeRTOS 任务优先级相反， Cotex-M3 优先级值越小， 优先级越高。 
Cotex-M3的优先级配置寄存器考虑器件移植而向高位对齐，实际可用的 CPU 会裁掉表达优先级低端的有效位，以减少优先级数。 举例子说， 加入平台支持3bit 表示优先级，则其优先级配置寄存器的高三位可以编程写入，其他位被屏蔽，不管写入何值，重新读回都是0。  另外提供抢占优先级和子优先级分段配置相关，详细阅读 《Cortex-M3权威指南》

在系统调度过程中，主要涉及到的三个异常：

- SVC  系统服务调用  操作系统通常不让用户程序直接访问硬件,而是通过提供一些系统服务函数。 这里主要触发后，在异常服务中启动第一个任务
- PendSV 可悬起系统调用  相比 SVC， PenndSV 异常后可能不会马上响应， 等到其他高优先级中断处理后才响应。 用于上下文切换，同时保证其他中断可以被及时响应处理。
- SysTick 节拍定时器  在没有高优先级任务强制下，同优先级任务按时间片轮流执行，每次SysTick中断，下一个任务将获得一个时间片。

函数中调用了 `prvPortStartFirstTask` 来启动第一个任务， 该函数重新初始化了系统的栈指针，表示 FreeRtos 开始接手平台的控制， 同时通过触发 SVC 系统调用，运行第一个任务。具体实现如下





前面创建任务的文章介绍过， 任务创建后， 对其栈进行了初始化，使其看起来和任务运行过后被系统中断切换了一样。 所以，为了启动第一个任务，触发 SVC 异常后，异常处理函数中直接执行现场恢复， 把 `pxCurrentTCB` "恢复"到运行状态。

（另外，Cotex-M3 具有三级流水线，所以切换任务的时候需要清除预取的指令，避免错误。）

对于 Cotex-M3 ， 其代码实现如下，

```javascript
void vPortSVCHandler( void )
{
    __asm volatile (
    /*取 pxCurrentTCB 的地址*/
    "ldr r3, pxCurrentTCBConst2      \n"
    /*取出 pxCurrentTCB 的值 ： TCB 地址*/
    "ldr r1, [r3]                    \n" 
    /*取出 TCB 第一项 ： 任务的栈顶 */
    "ldr r0, [r1]                   \n"
    /*恢复寄存器数据*/
    "ldmia r0!, {r4-r11}            \n" 
    /*设置线程指针： 任务的栈指针*/
    "msr psp, r0                    \n" 
    /*流水线清洗*/
    "isb                            \n"
    "mov r0, #0                     \n"
    "msr    basepri, r0             \n"
    /*设置返回后进入线程模式*/
    "orr r14, #0xd                  \n"
    "bx r14                         \n"
    "                               \n"
    ".align 4               \n"
    "pxCurrentTCBConst2: .word pxCurrentTCB     \n"
    );
}
```

异常返回后， 系统进入线程模式， 自动从堆栈恢复PC等寄存器，而由于此时栈指针已经更新指向对应准备运行任务的栈，所以，程序会从该任务入口函数开始执行。  到此， 第一个任务启动。

前面提到， 第一个任务启动通过 SVC 异常， 而后续的任务切换， 使用的是 PendSV 异常， 而其对应的服务函数是 `xPortPendSVHandler`。 后续介绍任务切换再分析。



- 调度器(scheduler)

  操作系统的运行由系统节拍时钟驱动，系统延时和阻塞时间都是以系统节拍时钟周期为单位

- configTICK_RATE_HZ

  - 在文件FreeRTOSConfig.h配置系统节拍时钟的中断频率，也间接的改变了系统节拍时钟周期（T=1/f）。
  - 比如设置宏configTICK_RATE_HZ为100，则系统节拍时钟周期为10ms，设置宏configTICK_RATE_HZ为1000，则系统节拍时钟周期为1ms

- xTickCount

  - 变量xTickCount也是在tasks.c文件中定义的静态变量，它在启动调度器时被清零，在每次系统节拍时钟发生中断后加1，用来记录系统节拍时钟中断的次数。内核会将所有阻塞的任务跟这个变量比较，以判断是否超时（超时意味着可以解除阻塞）。
  - 数据类型，跟具体硬件有关，32位架构硬件一般是无符号32位变量、8位或16位架构一般是无符号16位变量。

- uxSchedulerSuspended

  - 调度器挂起的嵌套计数器(嵌套层数)
  - xPortSysTickHandler函数执行调度时，会根据uxSchedulerSuspended的数值判断调度器的状态（正常或挂起），不同的状态对应执行不同的操作。

- vTaskSuspendAll

  - 挂起调度器：uxSchedulerSuspended++

- xTaskResumeAll：

  - 恢复调度器：uxSchedulerSuspended --
  - 执行uxPendedTicks次xTaskIncrementTick()函数，其中uxPendedTicks是本次调用xTaskResumeAll函数之前的调度器挂起期间，系统节拍中断的次数

- 调度器的挂起与恢复必然是配合使用

  ```
  vTaskSuspendAll();
  {
      。。。。。。。
  }
  ( void ) xTaskResumeAll();
  ```

  使用大括号，便于代码的查看。大括号中的程序执行时，正在执行的任务会一直继续执行，内核不再调度，意味着当前任务不会被切换出去，直到该任务调用xTaskResumeAll()函数。

- - 

- 

##### 任务挂起Suspend





- 
- 



##### 任务延时Delay

- 延时列表 xDelayedTaskList1 和 xDelayedTaskList2

  - 延时列表指针：pxDelayedTaskList，指向xDelayedTaskList1

  - 溢出延时列表指针：pxOverflowDelayedTaskList，指向xDelayedTaskList2

  - 延时时间：xTicksToDelay

  - 唤醒时间：xTickCount+ xTicksToDelay，xTickCount是当前的系统节拍累计值，加上延时时间xTicksToDelay，就是下次唤醒任务的时间。xTickCount+xTicksToDelay会被记录到任务TCB中，随着任务一起挂接到延时列表。

  - 溢出处理：判断唤醒时间的数值是否溢出，溢出则将当前任务挂到xDelayedTaskList2，不溢出则挂到xDelayedTaskList1

    任务按照延时时间，顺序的插入到延时列表中。所以当系统节拍中断次数计数器xTickCount溢出时，必须将延时列表指针pxDelayedTaskList和溢出延时列表指针pxOverflowDelayedTaskList交换以便正确处理延时的任务。溢出则执行宏taskSWITCH_DELAYED_LISTS()：

    - pxDelayedTaskList和pxOverflowDelayedTaskList交换
    - 调用prvResetNextTaskUnblockTime函数重新获取下一次解除阻塞的时间，这个时间保存在静态变量xNextTaskUnblockTime中，该变量也是定义在tasks.c中。下面检查延时列表任务是否到期时，会用到这个变量。
    - 检查延时列表，查看延时的任务是否到期。前面我们说过，延时的任务根据延时时间先后，顺序的插入到延时列表中，延时时间短的在前，延时时间长的在后，并且下一个要被唤醒任务的时间数值保存在变量xNextTaskUnblockTime中。所以使用xTickCount与xNextTaskUnblockTime比较就可以知道是否有任务可以被唤醒。
      如果任务被唤醒，则将任务从延时列表中删除，重新加入就绪列表。如果新加入就绪列表的任务优先级大于当前任务优先级，则会触发一次上下文切换。
      FreeRTOS支持多个任务共享同一个优先级，如果设置为抢占式调度（宏configUSE_PREEMPTION设置为1）并且宏configUSE_TIME_SLICING也为1（或未定义），则相同优先级的多个任务间进行任务切换。
      最后还会调用时间片钩子函数vApplicationTickHook()。可以看到时间片钩子函数实在中断服务函数中调用的，所以这个钩子函数必须简洁、不可以调用不带中断保护的API函数。

- prvAddCurrentTaskToDelayedList

  - listSET_LIST_ITEM_VALUE：pxCurrentTCB->xStateListItem.xItemValue=xTickCount+xTicksToWait
  - vListInsert



##### 任务切换

FreeRTOS 支持时间片轮序和优先级抢占。系统调度器通过调度算法确定当前需要获得CPU 使用权的任务并让其处于运行状态。对于嵌入式系统，某些任务需要获得快速的响应，如果使用时间片，该任务可能无法及时被运行，因此抢占调度是必须的，高优先级的任务一旦就绪就能及时运行；而对于同优先级任务，系统根据时间片调度，给予每个任务相同的运行时间片，保证每个任务都能获得CPU 。

- 任务列表

  - xSuspendedTaskList

  - xPendingReadyList

  - pxReadyTasksLists

    任务已经就绪，但是调度被禁止，暂时放到pending列表

  - pxDelayedTaskList

  - pxOverflowDelayedTaskList

  - 

- 任务调度的典型过程

  1. 最高优先级任务 Task 1 运行，直到其被阻塞或者挂起释放CPU
  2. 优先级仅次于Task1的任务Tsak2在就绪链表成为最高优先级任务，它开始运行， 直到... 
     1. 调用接口进入阻塞或者挂起状态
     2. 任务 Task 1 恢复并抢占  CPU 使用权
     3. 同优先级任务TASK 3 就绪，时间片调度
  3. 没有用户任务执行，运行系统空闲任务。



- FreeRTOS 在两种情况下执行任务切换：

  1. 同等级任务时间片用完，提前挂起触发切换  在 SysTick  节拍计数器中断中触发异常
  2. 高优先任务恢复就绪（如信号量，队列等阻塞、挂起状态下退出）时抢占  最终都是通过调用移植层提供的 `portYIELD()` 宏悬起 PendSV  异常

  但是无论何种情况下，都是通过触发系统 PendSV 异常，在该服务程序中完成切换。  

  使用该异常切换上下文的原因是保证切换不会影响到其他中断的及时响应（切换上下文抢占了 ISR 的执行，延时时间不可预知，对于实时系统是无法容忍的），在SysTick 中或其他需要进行任务切换的地方悬起一个 PendSV 异常，系统会直到其他所有 ISR 都完成处理后才执行该异常的服务程序，进行上下文切换。

  系统响应 PendSV 异常，在该中断服务程序中，保存当前任务现场， 选择切换的下一个任务，进行任务切换，退出异常恢复线程模式运行新任务，完成任务切换。

以下是 Cotex-M3 的服务程序，  首先先要明确的是，系统进入异常处理程序的时候，使用的是主堆栈指针 MSP， 而一般情况下运行任务使用的线程模式使用的是进程堆栈指针 PSP。后者使用是系统设置的，前者是硬件强制设置的。  对应这两个指针，系统有两种堆栈，系统内核和异常程序处理使用的是主堆栈，MSP 指向其栈顶。而对应而不同任务，我们在创建时为其分配了空间，作为该任务的堆栈，在该任务运行时，由系统设置进程堆栈 PSP 指向该栈顶。  如下分析该服务函数的执行：

```javascript
void xPortPendSVHandler( void )
{
    /* This is a naked function. */
    __asm volatile
    (
    /*取出当前任务的栈顶指针 也就是 psp -> R0*/
    "   mrs r0, psp                         \n"
    "   isb                                 \n"
    "                                       \n"
    /*取出当前任务控制块指针 -> R2*/
    "   ldr r3, pxCurrentTCBConst           \n"
    "   ldr r2, [r3]                        \n"
    "                                       \n"
    /*R4-R11 这些系统不会自动入栈，需要手动推到当前任务的堆栈*/
    "   stmdb r0!, {r4-r11}                 \n"
    /*最后，保存当前的栈顶指针 
    R0 保存当前任务栈顶地址
    [R2] 是 TCB 首地址，也就是 pxTopOfStack
    下次，任务激活可以重新取出恢复栈顶，并取出其他数据
    */
    "   str r0, [r2]                        \n"
    "                                       \n"
    /*保护现场，调用函数更新下一个准备运行的新任务*/
    "   stmdb sp!, {r3, r14}                \n"
    /*设置优先级 第一个参数，
    即:configMAX_SYSCALL_INTERRUPT_PRIORITY
    进入临界区*/
    "   mov r0, %0                          \n"
    "   msr basepri, r0                     \n"
    "   bl vTaskSwitchContext               \n"
    "   mov r0, #0                          \n"
    "   msr basepri, r0                     \n"
    "   ldmia sp!, {r3, r14}                \n"
    "                                       \n"
    /*函数返回 退出临界区
    pxCurrentTCB 指向新任务
    取出新的 pxCurrentTCB 保存到 R1
    */
    "   ldr r1, [r3]                        \n"
    /*取出新任务的栈顶*/
    "   ldr r0, [r1]                        \n"
    /*恢复手动保存的寄存器*/
    "   ldmia r0!, {r4-r11}                 \n"
    /*设置线程指针 psp 指向新任务栈顶*/
    "   msr psp, r0                         \n"
    "   isb                                 \n"
    /*返回， 硬件执行现场恢复
    开始执行任务
    */
    "   bx r14                              \n"
    "                                       \n"
    "   .align 4                            \n"
    "pxCurrentTCBConst: .word pxCurrentTCB  \n"
    ::"i"(configMAX_SYSCALL_INTERRUPT_PRIORITY)
    );
}
```

在服务程序中，调用了函数 `vTaskSwitchContext` 获取新的运行任务， 该函数会更新当前任务运行时间，检查任务堆栈使用是是否溢出，然后调用宏 `taskSELECT_HIGHEST_PRIORITY_TASK()`设置新的任务。该宏实现分两种情况，普通情况下使用的定义如下

```javascript
UBaseType_t uxTopPriority = uxTopReadyPriority;
while(listLIST_IS_EMPTY(&(pxReadyTasksLists[uxTopPriority])))
{
    --uxTopPriority;
}
    
listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB, 
    &(pxReadyTasksLists[ uxTopPriority]));

uxTopReadyPriority = uxTopPriority;
```

通过 while 查找当前存在就绪任务的最高优先级链表，获取链表项设置任务指针。（通一个链表内多个项目通过指针循环，实现同优先级任务获得相同时间片执行）。

而另外一种方式，需要平台支持，主要差别是查找最高任务优先级，平台支持利用平台特性，效率会更高，但是移植性就不好说了。

发生异常跳转到异常处理服务前，自动执行的现场保护会保留返回模式（线程模式），使用堆栈指针等信息，所以，结束任务切换， 通过执行 `bx r14`返回，系统会自动恢复现场（From stack），开始运行任务。

至此，任务切换完成。



时间片调度和抢占式调度我们一般都是开启了的，这样优先级相同时，使用时间片调度，优先级不同时，使用抢占式调度。

默认在FreeRTOS.h中使能了时间片调度：

```c
#ifndef configUSE_TIME_SLICING
	#define configUSE_TIME_SLICING 1
#endif
```

手动在FreeRTOSConfig..h中使能抢占式调度：

```c
#define configUSE_PREEMPTION			1
```





如果要使能任务的notify机制，需要将configUSE_TASK_NOTIFICATIONS define为1



##### 任务通知 

- notification 

  每一个任务在创建的时候都会有一个32位的notification 值，在创建的时候被初始化为0，RTOS notification 值是一个直接发送给别的任务的可以解除阻塞状态的值（这里需要注意的是接触的这个阻塞是因为等待通知而引起的），当发送到别的任务的时候，会可以配置的更新接收任务的notification值。
  可以配置/更新接收任务的notification值的方式可以是如下的方式：

  直接写一个32位的notification到接收任务中
  让接收任务的notification自加1
  置位notification的某些位
  不改变接收任务的notification

  在用任务通知实现一个轻量级、快速的 二值量/计数信号量 时，需要使用xTaskNotifyGive和ulTaskNotifyTake() 。
  推荐用任务通知来实现二值量/计数信号量，用任务通知更快，用更少的RAM。
  任务通知功能可以被关闭，如果关闭只需要设置configUSE_TASK_NOTIFICATIONS = 0 即可，禁止后每个任务节省8字节的内存。
  有优点，也有缺点：

  只能有一个任务接收notification
  接收任务可以进入阻塞态，但发送任务不能阻塞，即使没有唤醒处于阻塞态的接收任务

- xTaskNotifyWait

  - 参数
    - ulBitsToClearOnEntry：表示任务在获取自身的ulNotifiedValue值时，如果没有其他任务进行notify更新，则首先将ulNotifiedValue的某些位进行清除，相当于进行一个初始化操作。如果传入0xffffffffUL，就将ulNotifiedValue全部清理了，如果传入0，则不改变ulNotifiedValue。
    - ulBitsToClearOnExit：当获取到任务自身的ulNotifiedValue值时，在退出函数前需要对ulNotifiedValue进行的某些位清除操作。
    - pulNotificationValue：用于获取任务自身的ulNotifiedValue。
    - xTicksToWait：阻塞等待的tick数，如果为0，表示非阻塞操作，为portMAX_DELAY表示永久阻塞直到读到notify事件
  - prvAddCurrentTaskToDelayedList

- xTaskNotify

  - xTaskToNotify
  - ulValue：通知值，悬挂的任务接收到该通知后，可以根据通知值，做出不同的响应
  - eAction：如果任务在接收到同时前，程序已经执行了多次xTaskToNotify，那么通知值会根据eAction来设置
  - eSetValueWithoutOverwrite ：如果被通知任务还没取走上一个通知，又接收到了一个通知，则这次通知值未能更新并返回pdFALSE，否则返回pdPASS。

- vTaskNotifyGive

- vTaskNotifyGiveFromISR

  这是vTaskNotifyGive的带中断保护版本，是专门设计用来在某些情况下代替二进制信号量和计数信号量的。

- TaskNotifyFromISR  -  xTaskNotifyAndQueryFromISR （xTaskGenericNotifyFromISR）

  在中断服务程序中被调用来发送通知，与不带中断保护的API函数xTaskGenericNotify()非常相似，只是增加了一些中断保护措施

  

- 





# timers

##### 系统节拍


- configCPU_CLOCK_HZ：CPU主频

- configSYSTICK_CLOCK_HZ：系统时钟（**counts**），Cortex-M3和M4内核的微控制器的实时操作系统一般采用systick做系统时钟，SysTick滴答定时器是一个24bit的递减计数器，有两种时钟源可选择，一个是系统主频，另一个是系统主频的八分频。port.c移植文件默认使用系统主频，**SysTick的计数器以系统主频的速率进行计数**，如果使用该默认配置，请不要定义**configSYSTICK_CLOCK_HZ**

  ```
  #ifndef configSYSTICK_CLOCK_HZ
  	#define configSYSTICK_CLOCK_HZ configCPU_CLOCK_HZ
  	/* Ensure the SysTick is clocked at the same frequency as the core. */
  	#define portNVIC_SYSTICK_CLK_BIT	( 1UL << 2UL )
  #else
  	/* The way the SysTick is clocked is not modified in case it is not the same
  	as the core. */
  	#define portNVIC_SYSTICK_CLK_BIT	( 0 )
  #endif
  ```

- configTICK_RATE_HZ：系统节拍（**ticks**），即SysTick中断的产生频率，也是FreeRTOS的时钟Tick的频率，一般使用1000Hz，即1ms产生一个系统节拍

- ulTimerCountsForOneTick = ( configSYSTICK_CLOCK_HZ / configTICK_RATE_HZ ) ：表示一个系统节拍占据多少个Systick计数器的计数值，

- xMaximumPossibleSuppressedTicks = portMAX_24_BIT_NUMBER / ulTimerCountsForOneTick ：SysTick计数器一次完整计数包含多少个系统节拍

- configOVERRIDE_DEFAULT_TICK_CONFIGURATION

- taskSWITCH_DELAYED_LISTS
  - 交换延时列表指针pxDelayedTaskList和溢出延时列表指针pxOverflowDelayedTaskList
  - prvResetNextTaskUnblockTime：更新下一次解除阻塞的时间



# xTaskNotify

只能发送一个信号量，前一次发送的信号量如果如果在被获取之前，又执行了一次xTaskNotify，那么前一次的无效，以后一次的为准

# xTaskIncrementTick - 任务调度器

移植时，mcu的系统节拍中断服务程序会调用函数xTaskIncrementTick，每次进入Systick中断，就会执行一次xPortSysTickHandler
xPortSysTickHandler函数就是FreeRTOS的调度器。如果该xTaskIncrementTick函数返回值为真（不等于pdFALSE），说明处于就绪态任务的优先级比当前运行的任务优先级高。这会触发一次PendSV中断，进行上下文切换

- xTaskIncrementTick

- 调度器正常（uxSchedulerSuspended = 0 ）

  - xTickCount++

  - xTickCount溢出处理，如32位xTickCount溢出时xTickCount=0xFFFFFFFF+1=0

    溢出则执行宏taskSWITCH_DELAYED_LISTS，交换延时列表指针和溢出延时列表指针。

    不溢出则执行宏mtCOVERAGE_TEST_MARKER，

  - 延时处理，将延时到期的任务从延时任务列表里移除并加入到就绪列表里。
    如果到期的任务优先级>=当前任务则开始一次任务切换
    如果当前任务就绪态里有多个任务，也需要切换任务，优先级相同需要在一个系统时钟周期的时间片里轮流执行每个任务。另外在应用程序里也可以通过设置xYieldPending的值来通知调度器进行任务切换。

- 调度器挂起（uxSchedulerSuspended > 0 ）

  - ++uxPendedTicks，调度器挂起阶段内，uxPendedTicks用来记录挂起期间，系统节拍中断的次数。

- 自动任务切换
  函数的最后几行代码颇让人难以理解，其中局部变量xSwitchRequired是本函数的返回值，在文章开始也说过：“如果该函数返回值为真，说明处于就绪态任务的优先级高于当前运行任务的优先级，则会触发一次PendSV中断，进行上下文切换”，现在如果变量xYieldPending为真，则返回值也会为真，函数结束后会进行上下文切换。这个变量xYieldPending的作用是什么？又是在什么时候被赋值为真呢？还真要从头说起。
  带中断保护的API函数，都会有一个参数pxHigherPriorityTaskWoken。如果API函数导致一个任务解锁，并且解锁的任务优先级高于当前运行的任务，则API函数将*pxHigherPriorityTaskWoken设置成pdTRUE。在中断退出前，老版本的FreeRTOS需要手动触发一次任务切换。比如在《 FreeRTOS系列第15篇---使用任务通知实现命令行解释器》一文中，我们在串口接收中断中调用了带中断保护的API函数vTaskNotifyGiveFromISR()，在函数执行完后，会使用代码portYIELD_FROM_ISR(xHigherPriorityTaskWoken)判断参数xHigherPriorityTaskWoken是否为真，为真则手动强制上下文切换。

     BaseType_txHigherPriorityTaskWoken = pdFALSE;        
     /*收到一帧数据，向命令行解释器任务发送通知*/ 
     vTaskNotifyGiveFromISR(xCmdAnalyzeHandle,&xHigherPriorityTaskWoken); 
          
     /*是否需要强制上下文切换*/ 
     portYIELD_FROM_ISR(xHigherPriorityTaskWoken );  
      
  从FreeRTOSV7.3.0起，pxHigherPriorityTaskWoken成为一个可选参数，并可以设置为NULL。如果将参数xHigherPriorityTaskWoken设置为NULL，并且带中断保护的API函数导致更高优先级任务解锁，任务什么时候、怎么切换呢？
      
  原来从FreeRTOSV7.3.0起，内核增加了一个静态变量xYieldPending，这个变量也是在tasks.c中定义的。如果将变量xYieldPending设置为pdTRUE，则会在下一次系统节拍中断服务函数中，触发一次任务切换，见本小节第一段代码描述。
      
  让我们看一下这个过程是如何实现的。
      
  对于队列以及使用队列机制的信号量、互斥量等，在中断服务程序中调用了这些API函数，将任务从阻塞中解除，则需要调用函数xTaskRemoveFromEventList()将任务的事件列表项从事件列表中移除。在移除事件列表项的过程中，会判断解除的任务优先级是否大于当前任务的优先级，如果解除的任务优先级更高，会将变量xYieldPending设置为pdTRUE。在下一次系统节拍中断服务函数中，触发一次任务切换。代码如下所示：
      
    if(pxUnblockedTCB->uxPriority > pxCurrentTCB->uxPriority)
   {
        /*任务具有更高的优先级，返回pdTRUE。告诉调用这个函数的任务，它需要强制切换上下文。*/
        xReturn= pdTRUE;

  ```
    /*带中断保护的API函数的都会有一个参数参数"xHigherPriorityTaskWoken"，如果用户没有使用这个参数，这里设置任务切换标志。在下个系统中断服务例程中，会检查xYieldPending的值，如果为pdTRUE则会触发一次上下文切换。*/
    xYieldPending= pdTRUE;
  ```

   }        

  对于FreeRTOSV8.2.0新推出的任务通知，也提供了带中断保护版本的API函数。按照逻辑推断，这些API函数的参数xHigherPriorityTaskWoken也可以不使用，变量xYieldPending也应该作用于这些API函数。但事实是，在FreeRTOSV9.0之前的版本，FreeRTOS都没有实现这个功能，如果使用这些API函数解除了一个更高优先级任务，必须手动的进行上下文切换。这可能是一个BUG，因为在FreeRTOS V9.0版本中，已经修复了这个问题，可以使用变量xYieldPending自动切换上下文。这个BUG由QQ昵称为“所长”的网友遇到。
      
  在V9.0以及以上版本中，如果在中断中释放的通知引起更高优先级的任务解锁，API函数会判断参数xHigherPriorityTaskWoken是否有效，有效则将*xHigherPriorityTaskWoken设置为pdTRUE，此时需要手动切换上下文；否则，将变量xYieldPending设置为pdTRUE，在下一次系统节拍中断服务函数中，触发一次任务切换。代码如下所示：



# prvTimerTask - 定时器任务

- 创建：vTaskStartScheduler - xTimerCreateTimerTask

- 触发：

- 执行：prvTimerTask 

  - prvGetNextExpireTime： 取出表头定时器的溢出时间，传递给函数prvProcessTimerOrBlockTask，表头定时器的溢出时间是最小的，它必定是下一个溢出的定时器
  - prvProcessTimerOrBlockTask：
    - xTimeNow = prvSampleTimeNow( &xTimerListsWereSwitched )：先判断系统节拍是否溢出，如果溢出则执行下面的步骤，没有溢出则跳过

      - xLastTime：静态变量，记录上一次调用prvSampleTimeNow函数时的系统节拍值

      - xTimeNow ：本次调用prvSampleTimeNow函数的系统节拍计数

      - xTimeNow  <  xLastTime：系统节拍已发生过溢出，回零重新开始计数

        - prvSwitchTimerLists： 处理当前链表上所有定时器，应为这条链表上的定时器肯定都已经溢出了

          - while( listLIST_IS_EMPTY( pxCurrentTimerList ) == pdFALSE )：使用while循环，在切换链表前，先处理当前链表上的所有执行定时器
            - xNextExpireTime = listGET_ITEM_VALUE_OF_HEAD_ENTRY( pxCurrentTimerList )：表头定时器的溢出时间
            - uxListRemove：从列表中移除该定时器
            - pxTimer->pxCallbackFunction：执行对应的回掉函数
            - 处理自动重载定时器
              - xReloadTime ：该定时器重载后下一次溢出的溢出时间
              - xReloadTime  >   xNextExpireTime：重载后定时器的时间没有溢出
                - 还在当前链表范围内， 继续插回到当前链表，由此保证执行的次数，这是考虑到有些定时器可能因为优先级等原因，虽然早就溢出但是被延迟执行了，但并不会被漏掉。
              - xReloadTime <=  xNextExpireTime：重载后定时器的时间同系统节拍计数器一样溢出了
                - xTimerGenericCommand：将定时器插入到新的链表中， 通过消息发送执行，等到处理消息时，链表已经切换了

          - 切换链表
            - pxTemp = pxCurrentTimerList;
            - pxCurrentTimerList = pxOverflowTimerList;
            - pxOverflowTimerList = pxTemp;

        - 设置溢出标志xTimerListsWereSwitched = pdTRUE

      - xTimeNow >= xLastTime，系统节拍还未溢出

        - 设置溢出标志xTimerListsWereSwitched = pdFALSE

    - 如果xTimeNow > xNextExpireTime ,则表头定时器溢出，执行prvProcessExpiredTimer

      - uxListRemove( &( pxTimer->xTimerListItem ) )：从定时器列表中移除该定时器，更新表头定时器
      - 处理自动重载定时器
        - prvInsertTimerInActiveList - xTimerGenericCommand：如果是自动重载的定时器， 则更新它的下一次溢出时间， 插回链表
      - pxTimer->pxCallbackFunction：执行对应的回调函数

    - 如果表头定时器没有溢出，则

      - vQueueWaitForMessageRestricted：更新表头定时器的溢出时间
      - xTaskResumeAll

- xTimerStart - xTimerGenericCommand

# prvIdleTask - 空闲任务

- 埋葬已删除的任务

  - prvCheckTasksWaitingTermination

- 空闲任务 

  - vApplicationIdleHook

    vApplicationIdleHook() MUST NOT, UNDER ANY CIRCUMSTANCES,CALL A FUNCTION THAT MIGHT BLOCK

    当使能睡眠模式的时候，空闲任务的设置很有必要，在vApplicationIdleHook函数中中可以做睡眠前的检测，确保一切安排妥当后，进入睡眠

- configPRE_SUPPRESS_TICKS_AND_SLEEP_PROCESSING

  - Define the macro to set xExpectedIdleTime to 0if the application does not want portSUPPRESS_TICKS_AND_SLEEP() to be called

- 进入睡眠

  - xExpectedIdleTime：确认睡眠时间是否满足条件，若满足，继续执行，若不满足则退出
  - vTaskSuspendAll：挂起调度器
  - portSUPPRESS_TICKS_AND_SLEEP(vPortSuppressTicksAndSleep)

- 退出睡眠

  - xTaskResumeAll：恢复调度器

- vApplicationIdleHook



- xExpectedIdleTime 睡眠时间
  - xExpectedIdleTime = prvGetExpectedIdleTime() = xNextTaskUnblockTime - xTickCount
  - xTickCount ：当前系统节拍计数值
  - xNextTaskUnblockTime，在下面的函数中取值
    - xTaskCreate：
    - xTaskIncrementTick / prvResetNextTaskUnblockTime ：
      - pxDelayedTaskList.uxNumberOfItems = 0 ：xNextTaskUnblockTime=portMAX_DELAY，
      - pxDelayedTaskList.uxNumberOfItems > 0 ：xNextTaskUnblockTime=pxDelayedTaskList->xListEnd->pxNext->pvOwner->xStateListItem->xItemValue
    - prvAddCurrentTaskToDelayedList：
      - xTickCount+xTicksToWait



### freertos 睡眠

- 睡眠开关

  #define   configUSE_TICKLESS_IDLE    1

  

  #define portSUPPRESS_TICKS_AND_SLEEP( xExpectedIdleTime ) vPortSuppressTicksAndSleep( xExpectedIdleTime )

内部使用函数portSUPPRESS_TICKS_AND_SLEEP，外部定义函数vPortSuppressTicksAndSleep

portSUPPRESS_TICKS_AND_SLEEP  -  portTASK_FUNCTION



##### eTaskConfirmSleepModeStatus



##### 睡眠时间

- prvGetExpectedIdleTime = xNextTaskUnblockTime - xTickCount
- xNextTaskUnblockTime
  - xTaskRemoveFromEventList -> prvResetNextTaskUnblockTime
    - pxDelayedTaskList       Empty： xNextTaskUnblockTime = portMAX_DELAY
    - pxDelayedTaskList Not Empty：xNextTaskUnblockTime = listGET_LIST_ITEM_VALUE( &( ( pxTCB )->xStateListItem ) ) 
  - prvAddCurrentTaskToDelayedList
    - xNextTaskUnblockTime = xTimeToWake **：the task entering the blocked state was at the head of the list of blocked tasks**
  - xTaskCreate
    - xNextTaskUnblockTime = portMAX_DELAY
  - xTaskIncrementTick
    - delayed list       Empty：xNextTaskUnblockTime = portMAX_DELAY 
    - delayed list not Empty：xNextTaskUnblockTime = xItemValue  







**vTaskDelay()** 最常用来产生延时。调用它的任务立即被阻塞，等到经过要求的若干  Timer Tick 过后再恢复到就绪状态。注意，下一次 Tick 的时刻(也就是硬件定时器中断发生时刻)和调用 vTaskDelay()  的时刻有一段不确定的间隔，因此要求精确的延迟时要考虑这个函数是否满足需求。
　　

**vTaskDelayUntil()** 功能类似，只不过它是指定一个绝对时间，而 vTaskDelay() 是从调用时刻计的相对时间。
　　

**xTaskGetTickCount()** 返回调度器已运行的 Tick 数，也就是总执行时间。
　　

**vTaskSuspend()** 和 **vTaskResume()**, 这两个函数可以将某一任务挂起，以及使挂起的任务恢复。
　　

**vTaskSuspendAll()** 和 **vTaskResumeAll()**:  这两个表面上看起来是上两个 API 功能的扩充，实际上原理完全不同。vTaskSuspendAll()  将调度器禁止，当前任务继续执行，中断也是允许的。限制是此时不能调用其它的 API, 直到用 vTaskResumeAll()  恢复调度器之后。和关键区域(critical section)的用法不同，关键区域是硬件上屏蔽中断保证不会发生任务切换，API 还是可以使用的。
　　

**vTaskList()** 接收一个字符缓冲区，生成文本格式的所有任务的列表，包括状态信息。
　　

**小结：**FreeRTOS入门运用不难，只需要把几个文件添加进现有的工程里面，就能改成多任务的。必要的步骤是决定配置选项，编写 FreeRTOSConfig.h 文件，以及决定内存管理方式。




