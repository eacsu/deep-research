好的，我将为你编写一份通俗易懂的 FreeRTOS 教程文档，面向熟悉裸机开发的 STM32 初学者。内容将基于 STM32F4 系列 MCU，使用 STM32CubeMX 配置，Keil 作为开发工具，最终实现一个最小可运行的闭环控制系统，使用 PID 算法控制电机运行。

我会涵盖：

* 使用 CubeMX 创建 FreeRTOS 工程的步骤
* 如何在 FreeRTOS 中设计任务结构
* 如何编写一个基本的 PID 控制器任务
* 如何使用队列或信号量传递电机反馈数据
* 如何组织主控逻辑和控制闭环

完成后我会告诉你查看文档草稿。


# 在 STM32F4 上使用 FreeRTOS 实现最小闭环 PID 控制系统教程

本教程面向熟悉裸机开发但初次接触 RTOS 的 STM32 初学者，讲解如何使用 STM32F4 系列 MCU、STM32CubeMX 和 Keil MDK 构建一个**最小可迭代的闭环控制系统**。我们将通过 FreeRTOS 实现对电机的 PID 闭环控制，并帮助初学者理解如何从**裸机思维**转向**RTOS 架构**思维。

**教学目标：** 通过一个简单的直流电机速度（或位置）控制例子，学习 FreeRTOS 工程创建、任务划分、任务间通信以及 PID 控制算法在 RTOS 下的实现方法。文中配有详细步骤、截图和示例代码，方便读者跟随实践。

## 简介：从裸机到 RTOS 的思维转变

在裸机开发中，我们通常在 `while(1)` 主循环中以**顺序执行**方式处理所有功能，并使用中断处理实时事件。随着系统复杂度增加，这种\*\*“超级循环 + 中断”架构\*\*会变得难以维护：各功能相互影响、调试困难，而且手动确保实时性很繁琐。

RTOS（Real-Time Operating System）提供了\*\*任务（Task/Thread）\*\*机制，将不同功能分解到独立并发执行的任务中，由内核统一调度管理。每个任务相当于一个小的无限循环，RTOS 内核负责根据任务优先级和状态切换运行它们，从而更好地利用 CPU 资源。这意味着：

* **并行处理：** 原本在裸机单循环按顺序执行的代码，现在可以分散到多个任务“同时”运行。比如一个任务专门采集传感器数据，另一个任务专门进行电机控制计算，互不干扰。
* **实时调度：** 高优先级任务可以及时打断低优先级任务执行（抢占式调度），保证关键控制环节的实时性；而低优先级任务在空闲时段运行，不会阻塞关键流程。
* **同步通信：** RTOS 提供**信号量、消息队列**等机制安全地在线程间同步数据，避免裸机开发中全局变量共享带来的不确定性和风险。

**思维转变要点：** 裸机开发者需要习惯让出CPU控制权——在裸机中可能习惯写死循环不停查询或等待事件，而在 RTOS 中，每个任务应在适当时刻**挂起自己**（例如调用延时或等待信号/队列) 来让出 CPU，让其它任务运行。这种\*\*“主动让步”**理念对充分发挥多任务并发非常重要。当进入 RTOS 架构，我们更多考虑**任务划分与通信\*\*，而不是在一个大循环里插入各种功能和延时了。

接下来，我们以构建一个电机闭环控制系统为例，详细演示从 CubeMX 工程配置到 FreeRTOS 多任务实现的全过程。

## 1. 使用 STM32CubeMX 创建 FreeRTOS 工程

我们首先使用 STM32CubeMX 图形化配置工具创建启用了 FreeRTOS 的 STM32F4 工程，并进行基础外设和 RTOS 设置。以下步骤以 **STM32F407VG** 芯片为例（读者可根据自己硬件选择相近的 STM32F4 芯片或开发板）。

### 1.1 新建 CubeMX 工程并选择 MCU/开发板

打开 STM32CubeMX，点击“New Project”新建工程。在器件选择界面中，选择目标 MCU（例如 STM32F407VG）或直接选择所用的开发板型号。配置系统时钟为合适频率（如使用 HSE 外部晶振并启用 PLL 让 SYSCLK 跑到 168MHz），确保 MCU 工作在**全速运行**模式，以获得精确的控制周期。

> **提示：** 在 CubeMX 中可以使用**时钟树配置**窗口直观设置和查看主频。对于电机控制，推荐开启硬件 FPU（如F4的单精度 FPU）以提升浮点计算效率。CubeMX 自动根据芯片使能 FPU，无需特别手动配置。

### 1.2 配置基础外设 (GPIO、UART、Timer 等)

根据闭环控制需要，配置以下外设（可根据具体硬件调整）：

* **调试接口：** 启用 **SWD** 调试接口，以便后续下载和调试程序。在 *Pinout* 界面将 Debug 设置为 Serial Wire。
* **通信接口：** 可启用一个 USART/UART（如 USART1）用于调试打印或与上位机通信，波特率默认即可。CubeMX 上将对应 TX 引脚设置为复用 UART功能，并在 *Configuration* 中启用异步模式。
* **电机 PWM 输出：** 选定一个定时器的通道用于产生 PWM 信号控制电机驱动。如 TIM3 通道 1（默认映射在 PA6 引脚）连接到电机驱动的 PWM 输入。将 TIM3 的 Channel1 模式设为 PWM Generation CH1，并在参数中配置预分频和计数值以得到合适的PWM频率（例如 20kHz 左右，或根据电机/驱动需求）。**注意**记下 PWM 定时器及通道，将在代码中通过 HAL 库启动 PWM 输出。
* **传感器反馈输入：** 如果电机带有编码器，可使用定时器的编码器模式读取速度/位置信号。例如使用 TIM4 配置为 Encoder Mode，将 A/B 信号引脚映射到对应的 TIM4\_CH1/CH2（如 PB6/PB7）。或者如果使用霍尔传感器/光耦，考虑将信号接 ExtI 中断或 Timer 输入捕获。为了简化，本例假定采用**定时器编码器接口**读取速度。配置 TIM4 的 Encoder Mode 为 mode2 (X4 encoding)，并启用其计数中断或准备在定时器重装时读取计数值。
* **ADC 输入（可选）：** 若做位置控制且通过电位器提供目标设定或反馈位置，可配置一个 ADC 通道采集模拟量。但本示例主要聚焦速度闭环，不详细展开 ADC。读者可视需要启用。

完成引脚及外设设置后，**确认时钟**：确保相应外设时钟（如 TIM3、TIM4、USART1 等）已在 RCC 设置中打开。

### 1.3 启用 FreeRTOS 中间件

在 **Middleware** 菜单下找到 **FreeRTOS**，点击启用该模块。在版本选择下拉框中，选择 **CMSIS-RTOS v2** 接口（CubeMX 目前提供 CMSIS-RTOS V1 和 V2，两者 API 稍有区别，V2 更新）。同时勾选 **USE\_NEWLIB\_REENTRANT** 选项（启用 Newlib 的线程安全）。如图所示：


*图1：在 STM32CubeMX 中启用 FreeRTOS 中间件，并选择 CMSIS-RTOS V2 接口（同时启用 USE\_NEWLIB\_REENTRANT）。启用该选项确保多任务下使用 `printf` 等 C 库函数是线程安全的。*

CubeMX 启用 FreeRTOS 后，会自动加入必要的 RTOS 内核源码，并基于 CMSIS-OS 接口生成初始化代码。接下来我们将对 FreeRTOS 进行具体配置。

### 1.4 FreeRTOS 配置与资源分配

启用 FreeRTOS 后，CubeMX 主窗口下方会出现 FreeRTOS 的配置分类项。主要分为【Tasks and Queues】、【Timers and Semaphores】、【Mutexes】、【Events】、【Heap Usage】、【Config Parameters】等。各部分功能如下：

* **Tasks and Queues：** 配置**任务**（线程）及**消息队列**
* **Timers and Semaphores：** 配置**软件定时器**和**信号量**（含二值信号量和计数信号量）
* **Mutexes：** 配置**互斥量**（用于资源互斥访问）
* **Events：** 配置**事件标志组**
* **FreeRTOS Heap Usage：** 查看当前各任务和系统占用的堆大小
* **Config Parameters：** 核心**系统参数**配置（调度方式、内存、钩子函数、中断优先级等）
* **Include Parameters：** 功能裁剪配置（决定编译哪些RTOS功能，精简代码）
* **Advanced Settings：** 代码预配置选项及用户自定义常量等

对于本教程，我们无需修改大部分默认参数，但有几点关键配置值得关注：

* **调度器类型：** 在 Config Parameters 中，`USE_PREEMPTION` 默认启用，表示使用**抢占式调度**。这意味着高优先级任务可中断低优先级任务，保障实时性。一般保持抢占式即可。
* **时基选择：** FreeRTOS 默认使用 SysTick 定时器作为其心跳时钟（时基源）。CubeMX 会自动将 SysTick 中断设置为最低优先级，并处理与 HAL 库时基的兼容。无需手动更改，但**注意**：一旦启用 FreeRTOS，**不要再使用 `HAL_Delay` 实现延时**，而应使用 RTOS 提供的延时函数（后续介绍），否则可能造成冲突。
* **内存分配和堆大小：** 默认为 heap4 算法，堆大小 (`TOTAL_HEAP_SIZE`) 默认值通常较小（例如 0x2000 = 8192 字节），需确保足够容纳所有任务、队列等。对于本例的简单系统，默认 heap 大小通常足够。如遇内存不足，可在 Config Parameters 中增大 **Heap Size**。

一般初学者可以暂时保留其他参数默认，CubeMX 默认配置已适合大多数情况。下面我们重点在 CubeMX 中**创建任务和队列**，以便生成相应代码框架。

### 1.5 添加任务（Threads）配置

进入 **Tasks and Queues** 配置页面。可以看到 CubeMX 已经默认添加了一个名为 `DefaultTask` 的任务（通常函数名为 `StartDefaultTask`），优先级为普通 (osPriorityNormal)，栈大小默认 128 字（即512字节，因为对32位MCU 1个字=4字节）。

我们可以在此页面**新增自定义任务**：点击 “Add” 按钮来添加一个新任务。假设我们创建两个任务：

* **MotorControlTask：** 用于执行 PID 运算并更新电机 PWM 输出。
* **SensorTask：** 用于读取传感器（编码器）得到当前速度，并通过队列传递给控制任务。
  *(如果需要，我们还可以添加比如 CommTask 来通过串口输出调试信息或接收命令，但本例中可选，用于演示多个任务并发。)*

添加第一个任务：点击 Add 后，填写任务参数：


*图2：通过 CubeMX 图形界面添加 FreeRTOS 任务。这里添加了名为 “MotorTask” 的任务，优先级设置为 Normal（普通），栈大小 256 字（即1KB），入口函数名为 `MotorTask`（CubeMX 将自动生成此函数框架）。任务默认使用动态内存分配（Allocation = Dynamic）。*

如上图所示，设置 **Task Name**（任务名称）为 “MotorTask” 作为标识，**Priority** 选择 `osPriorityAboveNormal`（比默认任务稍高，让控制任务更及时），**Stack Size** 设为 256 words（即 1024 字节栈，用于存放任务运行时局部变量、函数调用栈等），**Entry Function** 填写 `MotorTask`（这是任务函数名，CubeMX 会生成一个空的 `MotorTask(void *argument)` 函数）。Code Generation 选择默认即可（产生一个有默认实现的函数体，我们稍后填入功能）。点击 OK 完成添加。

用同样方法添加第二个任务 “SensorTask”，优先级可设为 `osPriorityNormal`（稍低于电机控制任务，让传感器采集在后台运行），栈大小根据需要（128\~256 words均可，传感器任务逻辑简单就不需要太大）。Entry Function 名称填 `SensorTask`。

如果计划添加通信任务也类似进行：如 “CommTask” 优先级更低 (osPriorityBelowNormal)，用于UART打印状态等。

任务配置完成后，可在 Tasks 列表看到新增的任务条目。**确认**默认的 `StartDefaultTask` 可以保留用于空闲演示，或取消勾选使其不生成。本例保留默认任务用于观察系统运行（例如让它闪LED或打印心跳）。

### 1.6 创建消息队列配置

电机控制是一个经典的**生产者-消费者**模型：传感器任务定期产生最新的速度数据，控制任务消费该数据计算控制输出。因此我们需要一个**消息队列**在两任务间传递数据。

在 **Tasks and Queues** 页面下方，找到 **Message Queues** 区域。点击 “Add” 新增一个队列。设定参数：


*图3：通过 CubeMX 添加一个消息队列。这里我们新增队列 `SpeedQueue`，长度（Queue Size）为 1，元素大小（Item Size）为 2 字节(uint16\_t)。队列采用动态分配（Dynamic）方式。*

如上图，将 **Queue Name** 命名为 “SpeedQueue”，**Queue Size** 设为 1（队列长度为1表示只保存最新的一条数据，有新的数据会覆盖或等待），**Item Size** 默认是 16 位 (uint16\_t)。我们假定速度数据用 `uint16_t` 表示（足够存储转速或位置计数）。设置完点击 OK。这样CubeMX会生成一个队列句柄 `SpeedQueueHandle` 以及初始化代码。

> **说明：** 队列的长度和元素大小可以根据需求调整。如果要缓冲多笔数据可加大长度；元素类型如果要传复杂结构，可以在 CubeMX 的队列属性里改为指针大小（如 32 位）并传递指向结构的指针。但对于初学者，传递简单的数值最直观。

### 1.7 生成代码并打开工程

完成上述配置后，点击工具栏的 “Generate Code” 按钮。在弹出的对话框中，选择 **MDK-ARM** (即 Keil uVision) 作为目标工具链，并指定工程名称和保存路径。确认后 CubeMX 将生成 Keil MDK 项目文件和代码。

生成完毕后，CubeMX 会提示打开项目，在 Keil uVision 中用相应版本（如 MDK 5）的编译器打开 `.uvprojx` 工程文件。**确保**已安装对应的 STM32F4xx Device Pack 和 Keil Middleware packages（CubeMX 生成的 FreeRTOS 代码需要 Keil 支持 CMSIS-OS v2 接口）。

在项目资源管理器中，可以看到生成的文件结构，其中包括:

* `Core/Src/main.c` – 主程序入口
* `Core/Src/freertos.c` – FreeRTOS 初始化和任务创建代码
* `Core/Src/stm32f4xx_hal_init.c` 等 – 硬件初始化代码 (CubeMX自动生成)
* `Middlewares/Third_Party/FreeRTOS` – FreeRTOS 内核源码

CubeMX 已经帮我们在 `freertos.c` 的 `MX_FREERTOS_Init()` 函数中创建了配置的任务和队列。例如，可打开 `freertos.c` 查看：CubeMX 会用我们设定的参数调用 `osThreadNew(MotorTask, NULL, &MotorTask_attributes);` 等创建任务，并调用 `osMessageQueueNew` 创建消息队列等。同时，在 `MotorTask` 和 `SensorTask` 函数的框架中，CubeMX已填充了无限循环结构和 `osDelay` 占位延时，提示用户在其中实现具体功能。

## 2. 在 Keil MDK 中编译和运行工程

打开生成的 Keil 工程后，我们需要进行编译、下载和调试。以下是一些注意事项：

* **选择合适的编译选项：** 默认CubeMX生成的工程配置通常为 `Debug` 优化级别-O0。这利于调试但生成代码较大。一般先保持Debug模式编译，待功能OK后再切换Optimize空间或速度。**注意：** FreeRTOS对优化不敏感，但过度优化可能使调试单步有偏差。
* **Heap 和 Stack 设置：** 在 Keil 的 Target Options中，确保 Heap 和 Stack 大小合适。FreeRTOS 动态内存使用heap4算法从其内部定义的heap数组分配，不直接用Keil设置的heap，但`main`启动时的C库heap/stack应足够大以防printf等。默认配置一般够用。
* **编译工程：** 点击 Build 按钮，首次编译可能会稍久，因为要编译FreeRTOS源码。应无错误。如果出现 `error: limit 32K exceeded` 则可能是代码超出了评估版Keil的32KB限制——可以尝试优化代码或使用MDK完整版。一般启用FreeRTOS后代码量仍很小（几十KB），通常在32K以内除非加了复杂功能。
* **下载程序：** 将 STM32 开发板通过 ST-LINK 与PC连接。确保 Target Options 中调试器选为 ST-Link/V2，Flash 编程算法正确。点击 Download 将程序烧录进MCU。FreeRTOS 初始化后会立即启动任务并运行闭环控制逻辑。

> **提示：** 初次调试可先简单验证 RTOS任务是否运行正常。例如在 `DefaultTask` 的循环中调用 `HAL_GPIO_TogglePin` 切换板上LED，加入 `osDelay` 制造1Hz闪烁，通过观察LED确认RTOS调度正常。

* **调试 FreeRTOS 任务：** Keil MDK 支持 FreeRTOS 内核感知调试。如果安装了“Keil RTX5 Thread Viewer”插件，可以在调试模式下查看线程列表、各线程状态和栈使用情况。这样有助于验证任务优先级和运行状态。如果没有插件，也可通过在任务中 `printf` 打印简单消息来调试（需确保 USE\_NEWLIB\_REENTRANT 已开启，否则多任务下 `printf` 需要慎用).

当工程成功运行时，系统将创建我们定义的任务并开始调度。接下来，我们将填充任务函数代码，实现闭环控制逻辑。在此之前，先简要介绍 FreeRTOS 的基础知识，以便更好理解代码含义。

## 3. FreeRTOS 基础知识简介

在编写多任务代码前，让我们回顾一些 FreeRTOS 的核心概念和 API，使初学者对 RTOS 有基本了解：

* **任务（Task/Thread）：** RTOS 中的任务类似于操作系统线程，是调度的基本单元。每个任务都有自己的独立**栈**空间和运行上下文。创建任务时需提供任务函数、栈大小、优先级等。任务函数一般是一个无限循环，在里面执行特定功能并通过延时或等待事件来让出CPU。
* **调度器（Scheduler）：** FreeRTOS 内核包含一个任务调度器，它按照设定的调度算法管理任务运行。默认采用**抢占式调度**（可选时间片轮转）。调度器通过定期时钟中断（SysTick）或任务主动让出，实现任务切换。高优先级任务准备就绪时可抢占低优先级任务。调度器在调用 `osKernelStart()` 后开始运行，此调用不会返回——意味着 `osKernelStart` 后面的代码永远不应被执行（除非RTOS停止，这是错误情况）。
* **任务优先级：** 每个任务有一个优先级（数值越高优先级越高）。高优先级任务总是会抢占低优先级任务执行（前提是高优先级任务处于Ready就绪态）。合理设置优先级确保关键任务及时运行。需要注意避免**优先级反转**或不必要的高优先级长时间占用CPU导致低优先级任务饥饿。
* **任务状态：** 任务可能处于运行 (Running)、就绪 (Ready)、阻塞 (Blocked) 或挂起 (Suspended) 等状态。调用阻塞式API（如等待队列/信号或调用 `osDelay`）会使任务进入阻塞态，直到超时或事件发生，这期间调度器会切换运行其他任务。
* **延时函数：** FreeRTOS 提供了非阻塞延时，如 `osDelay(ms)` 或 `vTaskDelay(ticks)`，用于让任务睡眠指定时间片。在延时期间任务处于阻塞状态，让出CPU。与裸机 `HAL_Delay` 不同，RTOS延时不会忙等待而是把CPU让给其他任务，非常高效。
* **任务栈和内存：** 每个任务的栈大小是在创建时指定的（words数）。要确保足够大以容纳任务内的局部变量和调用深度。如果栈不足会导致溢出错误，可通过设置 `configCHECK_FOR_STACK_OVERFLOW` 来调试。FreeRTOS 会从其内置堆（heap）分配任务栈和控制块（TCB）。内置堆大小由 Config 参数里的 `TOTAL_HEAP_SIZE` 定义。若系统创建多个任务/队列，需要相应增大该值防止内存不足。
* **线程安全：** 多任务访问共享资源时需要同步机制。FreeRTOS 提供**信号量/互斥量**来协调任务对资源的使用，以避免竞态条件（稍后会介绍队列和信号量）。直接使用全局变量在多任务间传递数据是**不安全**的，“看似正常运行，但可能由于寄存器或内存访问冲突在不经意的时候引发崩溃”。因此应使用 RTOS 提供的同步机制进行通信。
* **中断和 RTOS：** FreeRTOS 可以与硬件中断配合使用。需要注意中断服务程序中只能调用一部分 FreeRTOS 提供的函数（主要是 “FromISR” 结尾的API）且中断优先级高于 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 的中断**不能**使用RTOS服务。通常我们将与RTOS交互的中断优先级设为较低（数值高）以防与内核冲突。

以上知识对理解 FreeRTOS 行为很重要。对于本例，我们会主要用到**任务创建**、**延时**、**消息队列**和**信号量**几个API。

## 4. 创建多个任务实践

在CubeMX生成代码的基础上，我们现在开始编写实际的任务函数代码，实现电机控制闭环逻辑。首先明确我们系统中的任务及功能划分：

* **SensorTask** (传感器任务): 定期读取传感器获取电机当前速度（或位置信息）。这里假定使用编码器读取速度，每隔一定周期读取一次增量计数。读取后通过**消息队列**将最新速度发送给控制任务。它相当于“数据采集”角色。
* **MotorTask** (电机控制任务): 等待来自传感器任务的速度数据，通过与目标值比较计算 PID 输出，更新PWM占空比以驱动电机调整速度。它相当于“控制计算+执行”角色。
* **CommTask** (通信任务，可选): 低优先级任务，负责通过串口打印关键参数（如当前速度、PWM输出等）或者接收用户指令（如修改目标速度）。本例主要演示前两者，通信任务可选实现用于调试。

这样划分的好处是各任务单一职责、逻辑清晰，使用 RTOS 的队列/信号实现同步，而不像裸机那样所有逻辑串在一起或通过复杂状态机维护。这也是**RTOS架构思维**的体现：把问题分解成并行的、协作的多个任务，而非一个大循环反复查询。

### 4.1 填写任务代码框架

打开 `Core/Src/freertos.c` 文件，找到 CubeMX 为我们生成的 `SensorTask(void *argument)` 和 `MotorTask(void *argument)` 函数。我们将编辑它们：

* 在 SensorTask 中，实现定时采样并发送队列；
* 在 MotorTask 中，实现从队列接收数据、PID计算和输出。

编写代码前，先在文件顶部 `#include` 部分确保包含了所需的头文件，比如 `<stdio.h>`（如果用到printf）和 HAL驱动的头文件（CubeMX已包含大部分）。另外，在文件开头还可以定义或声明一些用于控制的全局变量，例如 PID参数、期望目标值、以及从CubeMX生成的队列句柄（如 `extern osMessageQueueId_t SpeedQueueHandle;`，CubeMX可能已在 freertos.c 顶部定义了队列句柄）。

以下是我们要编写的关键代码段：

```c
/* 用户代码区域 0 */
// 假设CubeMX在此已定义了消息队列句柄
// extern osMessageQueueId_t SpeedQueueHandle;

// 设置 PID 参数
float Kp = 0.5f, Ki = 0.1f, Kd = 0.0f;    // 简单起见，先取D=0仅用PI控制
uint16_t targetSpeed = 1000;             // 目标速度 (例如编码器计数值，1000为示例)

// 全局变量用于存储PWM占空比（0~100或定时器计数范围）
volatile uint16_t motorPWM = 0;
```

上述变量中，`targetSpeed` 可以理解为希望电机达到的速度（由用户设定，假设以编码器计数每采样周期为单位），初值示例为1000。实际应用中这个目标值可根据需要改变（比如通过串口命令修改）。

接下来是 **SensorTask** 实现：该任务应每隔固定周期读取一次当前速度。本例假设我们使用 TIM Encoder 模式获取编码器脉冲，在每个周期读取并清零计数器得到速度脉冲数。

伪代码如下：

```c
void SensorTask(void *argument) {
    // 启动编码器计数定时器 (如果未在MX_Init中启动)
    HAL_TIM_Encoder_Start(&htim4, TIM_CHANNEL_ALL);

    uint16_t speed;
    for(;;) {
        // 读取编码器当前计数并清零计数器
        speed = __HAL_TIM_GET_COUNTER(&htim4);
        __HAL_TIM_SET_COUNTER(&htim4, 0);
        // 将读取的speed值发送到队列，等待空间（此处队列长1，可以用覆盖或等待）
        osMessageQueuePut(SpeedQueueHandle, &speed, 0, 0);
        // 周期为10ms (100Hz 控制频率)
        osDelay(10);
    }
}
```

上述代码中，每循环一次读取TIM4计数器的值作为`speed`，然后立即清零计数器，为下一周期累积做准备。这样如果10ms内编码器产生了N个脉冲，则 speed=N（假设编码器每圈脉冲数已考虑进目标和PID计算中）。然后通过 `osMessageQueuePut` 将 speed 放入队列。队列长度为1，若上一次数据尚未被取走，`osMessageQueuePut` 默认会覆盖最旧的数据或者等待（可以设置超时，这里传0表示不等待直接返回）。我们并不担心数据丢弃，因为速度最新值覆盖旧值即可。

最后 `osDelay(10)` 实现10ms循环周期（即100Hz采样）。**注意：** 这里的 `osDelay(10)` 是按照RTOS节拍算10个tick，默认FreeRTOS节拍频率1kHz，因此10tick≈10ms。

再看 **MotorTask** 实现：该任务需要持续等待有新的速度数据到来，然后执行PID计算调整PWM：

```c
void MotorTask(void *argument) {
    // 启动电机PWM输出定时器通道 (假设TIM3 CH1)
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);

    uint16_t currSpeed;
    int32_t error, prevError = 0;
    int32_t integral = 0;
    uint16_t pwmOut;
    for(;;) {
        // 从队列等待接收速度数据（无限等待直到收到）
        if(osMessageQueueGet(SpeedQueueHandle, &currSpeed, NULL, osWaitForever) == osOK) {
            // 计算误差 (目标-当前)
            error = targetSpeed - currSpeed;
            // 累积误差用于积分项（简单积分）
            integral += error;
            // 限制积分累积防止溢出或过饱和
            if(integral > 10000) integral = 10000;
            if(integral < -10000) integral = -10000;
            // 计算微分项
            int32_t derivative = error - prevError;
            prevError = error;
            // PID 控制输出计算 (这里假设循环周期固定10ms，未额外乘除dt，常数可吸收于参数)
            float output = Kp*error + Ki*integral + Kd*derivative;
            // 输出限幅 [0, 100] 或计数值上下限
            if(output < 0) output = 0;
            if(output > 1000) output = 1000;
            pwmOut = (uint16_t)output;
            motorPWM = pwmOut;  // 更新全局用于监控（可选）
            // 设置PWM占空比（将PWM输出占空比调整为 pwmOut对应的值）
            __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, pwmOut);
        }
        // 控制任务可不主动延时，因为osMessageQueueGet阻塞控制了节奏
        // 若无数据，它一直等待；一旦收到数据，立刻计算输出然后循环等待下一次
    }
}
```

在上述 `MotorTask` 代码中：

* 使用 `osMessageQueueGet` 从 `SpeedQueue` 获取数据。我们设置 `osWaitForever` 表示如果没有新数据就一直阻塞等待，这样实现与 SensorTask 的**同步**：只有当SensorTask放入新速度后，这里才会解除阻塞进行计算。这确保控制环路和传感器采样**周期一致**（都为10ms）而不会跑飞。
* PID 算法部分：计算当前误差 `error`，积分累加 `integral`，并简单防止积分累积过大（避免积分饱和的问题）。微分项 `derivative` 用当前误差减去上一次误差 `prevError`（\*\*注意：\*\*这样实际上未除以周期Δt，相当于把 Δt=常量 融合进 Kd，需要调参时注意单位）。然后用比例、积分、微分系数计算输出 `output = Kp*error + Ki*integral + Kd*derivative`。这里Kp, Ki, Kd需根据系统特性调试合适值。本例设置较小Kp避免过冲，Ki积累消除稳态差，Kd暂不使用。
* 将计算得到的 `output` 限制在合理范围，例如 0到1000。**输出单位**：这里假设TIM3 PWM的计数器周期对应1000为100%占空比（具体取决于TIM3初始化arr值），或者您可以按百分比转换。总之确保 pwmOut 在 PWM允许范围内。
* 使用 `__HAL_TIM_SET_COMPARE` 将 TIM3\_CH1 通道的占空比更新为 pwmOut值，从而改变电机驱动电压占空比，影响电机速度。这实现了将 PID 控制器输出转换为 PWM 信号去调节电机转速。
* 更新 `prevError`，然后循环等待下一次数据。由于队列同步，MotorTask实际上每收到一笔数据就立刻处理一次，与SensorTask周期相同，不需要显式延时。

这样，闭环控制任务就搭建完成。SensorTask定期采样->发送队列，MotorTask等待->计算->输出，实现闭环。

### 4.2 任务间通信与同步

在上面的实现中，我们使用**消息队列**完成了 SensorTask 和 MotorTask 之间的数据通信：一个放入，一个取出。这种方式**线程安全**且简洁。在 RTOS 架构中，与裸机不断检查变量不同，任务可以**阻塞等待**队列消息，大大提高效率。正如我们代码所示，当 MotorTask 调用 `osMessageQueueGet` 并队列为空时，任务自动进入阻塞态，无需消耗CPU轮询，直到SensorTask发送消息将其唤醒。这也是 RTOS 的**事件驱动**思想体现。

**为什么不用全局变量直接传递 speed?** 正如前面提到，全局变量在多任务访问下可能出现不可预知的问题，比如任务读取的瞬间另一个任务修改了它，或者编译优化导致的可见性问题。使用**消息队列**可以确保完整的数据拷贝和同步。此外，队列还能**自动缓冲**多笔数据，具有先进先出顺序。本例中队列长度为1相当于每周期最新值覆盖旧值。如果控制任务暂时跟不上采样（例如执行时间过长），队列也会保存最近的一个值防止完全丢失采样信息。

除了消息队列，FreeRTOS 还提供**信号量**用于任务间简单同步。当只需通知事件、不需要传递数据时，可考虑使用**二值信号量**（binary semaphore）。例如，我们也可以这样设计：SensorTask每次采集完数据后给一个二值信号量 “DataReady” post，MotorTask等待这个信号量 `osSemaphoreAcquire`，拿到后再读取共享的速度变量和计算。这相当于通知+共享内存的方式，也能实现同步。但是队列更直接地把数据和通知合二为一处理了。因此对于有数据传递的情况，**消息队列是更明智的选择**。

> **小知识：** FreeRTOS的**二值信号量**本质上也是用队列机制实现的（长度为1，item为空的特殊队列)，只是只用于表示资源可用性。**互斥量**则是一种特殊的二值信号量，用于保护临界资源，同一时间只能一个任务持有，典型用于串口输出等互斥访问。本例中如果我们的 MotorTask 和 CommTask 都需要调用 `printf` 输出串口，则需要用一个互斥量锁定串口打印，防止两任务同时输出造成乱码。CubeMX 也支持直接添加信号量、互斥量配置，这里不展开。

**小结：** 通过消息队列，我们实现了传感器数据在实时性和完整性都得到保证的传输，同时利用 RTOS 阻塞机制让控制任务准确地按周期运行。这种任务划分和同步方式是 RTOS 编程的精髓之一，也充分体现了与裸机方式的不同。

## 5. 实现一个简单 PID 控制器

现在，我们详细看看 PID 控制器本身的实现。在闭环系统中，PID（比例-积分-微分）控制器根据**当前误差**以及误差的历史和变化趋势，计算出控制输出，从而驱动被控对象达到目标。PID 控制因其算法简单、可调参数直观而被广泛应用于工业控制。

**PID基本概念：**
设定值为 \$R\$，测量反馈值为 \$Y\$，误差 \$e(t)=R-Y\$。则 PID 控制输出 \$U(t)\$ 定义为：

$U(t) = K_p \cdot e(t) + K_i \cdot \int_{0}^{t} e(\tau)d\tau + K_d \cdot \frac{de(t)}{dt}$

其中：

* \$K\_p\$ 为**比例系数**（P），反映对当前误差的放大作用。误差越大，输出校正越大。P控制可以提高响应速度，缩小稳态误差，但单独使用会存在剩余误差且系数过大时引起振荡。
* \$K\_i\$ 为**积分系数**（I），对过去累积误差进行作用。积分项能逐渐消除稳态误差（让系统最终无偏差），但过大会造成超调和振荡，需要权衡。
* \$K\_d\$ 为**微分系数**（D），对误差的变化趋势进行预测。微分项相当于阻尼，抑制误差变化过快导致的超调。在误差快速减小（向目标逼近）时，D项产生负输出抵消部分控制，减小冲击；在误差迅速增大时，D项产生正输出，提供额外纠正。因此D可以提高系统稳定性，减少振荡。

在数字实现中，我们将积分和微分通过**离散采样**近似计算。例如每隔 Δt 采样一次，则积分可以用误差累加和表示，微分用相邻两次误差差分表示。常见的实现方式有：

* **位置式 PID：** 直接根据上述公式计算 \$U(k)\$，需要保存累计误差和前一误差等。
* **增量式 PID：** 计算本次输出相对于上次输出的增量 \$\Delta U = U(k)-U(k-1)\$，其公式等价于位置式差分。在实现上，增量式PID每次根据当前误差、上次误差和上上次误差计算控制增量。增量式的好处是对控制量进行累加，更适合控制有约束的执行器（如只输出增量命令）。

本例代码中采用的是**位置式PID**的简单实现，每次用最新误差更新积分和微分。也可以理解为一种增量实现：因为我们在计算时将 Δt 视为常数1，故参数需要根据实际采样周期调整。

**PID参数整定：** 初学者可以采用**试凑法**：先将 Ki, Kd 设为0，只调 Kp，看系统能否基本跟随，有振荡则减小Kp；再调 Ki 消除静差，Ki从小到大增加，观察稳定误差减小但振荡不要明显加剧；最后酌情加入小量 Kd 缓和振荡。因为本例电机系统具有惯性，通常需要一定的积分作用消除稳态误差，同时D项在简单速度控制中可不使用或者很小。

**代码实现注意：**

* 积分项可能导致**积分饱和**（误差长期为正则积分累积极大，释放后造成很大超调）。所以代码里对 `integral` 做了限幅处理。
* 微分项对测量噪声很敏感，编码器的量化误差会放大高频抖动。因此 D 项常在速度控制中设得较小或者使用滤波后的误差。
* 控制输出需要限定范围，如 PWM 输出的占空比0~~100%对应计数器0~~ARR的值。我们在代码中简单假设了0\~1000的范围，根据实际PWM定时器配置修改即可。

通过 PID 控制算法，我们的 MotorTask 能根据传感器提供的反馈不断调整电机PWM输出，使电机转速逐渐逼近目标值并稳定下来。这就完成了闭环控制的核心。闭环控制系统通常包括**采样-控制-执行**三个环节不断循环——SensorTask完成采样反馈，MotorTask完成控制计算和执行输出，二者协同构成闭环。

## 6. 控制回路的组织方式

一个最小闭环系统在 RTOS 下的组织方式，可以总结如下：


*图4：闭环控制系统示意图（逻辑上）。PID 控制器的输出经PWM驱动电路作用于电机，被控电机的速度通过传感器反馈，形成闭环。在RTOS任务划分中，一个任务周期性获取反馈值，另一个任务计算控制输出，二者通过队列/信号进行同步通信。*

1. **数据采集环节（SensorTask）**：周期性（每Δt）获取被控对象的当前状态。如本例中读取编码器计数得到电机速度。采样间隔Δt应根据系统动态特性选取，如电机机械时间常数、响应速度等。采样过慢则控制滞后，过快则受限于噪声和计算资源。通过 RTOS 的延时或定时器，我们能精确地以固定频率执行采样任务。
2. **控制计算环节（MotorTask）**：获取最新的反馈值后，与目标值比较，执行 PID 算法算出控制量。这里利用RTOS通信确保每次计算使用新的数据。一些情况下，为了精确定时，也可以将控制计算放在**定时中断**或**RTOS定时器**中执行（例如上面参考资料中用 FreeRTOS软件定时器每10ms回调进行控制）。本例直接用任务阻塞同步简化设计。
3. **执行输出环节（MotorTask）**：将控制器输出转换为对执行机构的操作。如将 PID 输出通过PWM更新电机驱动电压占空比。在实际硬件中，这一步通常涉及对寄存器/外设的访问，因此需要考虑互斥（但本例中只有一个任务操作PWM，没有冲突）。RTOS 下也能通过**通知**或**事件**触发专门的执行任务，不过在简单系统中计算和执行常在一个任务里完成。
4. **循环往复**：上述过程不断循环，使得每个控制周期都校正一次输出，系统逐步逼近目标。这就是闭环控制（反馈控制）的持续工作流程。

在 RTOS 中，实现闭环控制还有一个好处：可以利用多任务并行处理**辅助功能**而不影响主控制环路。例如我们添加了一个 CommTask 在后台运行，它每隔100ms读取一些全局变量（如当前速度currSpeed、输出占空比motorPWM等）通过串口打印，或监听串口修改 targetSpeed。因为CommTask优先级低于控制任务，控制环的实时性不会受打印影响——这在裸机里如果直接在主循环打印调试信息，可能会扰乱控制周期。而 RTOS 可以保证高优先级任务稳定按周期执行，低优先级任务仅在空闲时运行，使系统更**可预测**和**可维护**。

综上，RTOS 架构下闭环控制系统的组织强调**解耦**与**同步**：采集、控制、通信各司其职，通过消息/信号连接。在时间上利用 RTOS tick 精确定时或阻塞等待，实现周期一致和相互配合。这种模型对于初学者理解实时系统设计很有帮助——它清晰地展示了如何把原来裸机一个大循环拆成多个并行模块且运行更加高效可靠。

## 7. 完整代码示例和讲解

最后，我们给出简化的**完整代码片段**，涵盖主要任务的实现，以及系统初始化的关键部分。请注意以下代码基于 CubeMX 生成的框架进行填写，为突出重点，省略了一些初始化样板代码（如SystemClock\_Config等），关注 FreeRTOS 和控制算法相关部分。

```c
/* ============ 头文件和全局变量 ============ */
#include "cmsis_os.h"
#include "tim.h"    // 定时器硬件驱动
#include "usart.h"  // 串口硬件驱动（如果需要printf调试）
#include <stdio.h>

osThreadId_t sensorTaskHandle;
osThreadId_t motorTaskHandle;
osMessageQueueId_t SpeedQueueHandle;

// PID参数及控制目标
float Kp=0.5f, Ki=0.1f, Kd=0.0f;
uint16_t targetSpeed = 1000;
volatile uint16_t motorPWM = 0;

/* ============ FreeRTOS 对象初始化 ============ */
void MX_FREERTOS_Init(void) {
  // 创建消息队列 (长度1, 元素2字节)
  SpeedQueueHandle = osMessageQueueNew(1, sizeof(uint16_t), NULL);
  
  // 创建任务 - SensorTask
  const osThreadAttr_t sensorTask_attributes = {
    .name = "SensorTask",
    .priority = (osPriority_t) osPriorityNormal,
    .stack_size = 128 * 4
  };
  sensorTaskHandle = osThreadNew(SensorTask, NULL, &sensorTask_attributes);
  
  // 创建任务 - MotorTask
  const osThreadAttr_t motorTask_attributes = {
    .name = "MotorTask",
    .priority = (osPriority_t) osPriorityAboveNormal,
    .stack_size = 256 * 4
  };
  motorTaskHandle = osThreadNew(MotorTask, NULL, &motorTask_attributes);
  
  // （如果有通信任务，也可在此创建，略）
}

/* ============ 任务函数实现 ============ */
void SensorTask(void *argument) {
  // 启动编码器接口计数（假设htim4配置为Encoder模式）
  HAL_TIM_Encoder_Start(&htim4, TIM_CHANNEL_ALL);
  uint16_t speed;
  for(;;) {
    // 读取编码器增量
    speed = __HAL_TIM_GET_COUNTER(&htim4);
    __HAL_TIM_SET_COUNTER(&htim4, 0);
    // 发送速度值到队列（覆盖旧值）
    osMessageQueuePut(SpeedQueueHandle, &speed, 0, 0);
    // 10ms 周期
    osDelay(10);
  }
}

void MotorTask(void *argument) {
  // 启动PWM输出（假定htim3 CH1已配置PWM模式）
  HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
  uint16_t currSpeed;
  int32_t error, prevError = 0;
  int32_t integral = 0;
  for(;;) {
    // 等待从队列收到速度数据
    if(osMessageQueueGet(SpeedQueueHandle, &currSpeed, NULL, osWaitForever) == osOK) {
      // PID计算
      error = targetSpeed - currSpeed;
      integral += error;
      if(integral > 10000) integral = 10000;
      if(integral < -10000) integral = -10000;
      int32_t derivative = error - prevError;
      prevError = error;
      float output = Kp*error + Ki*integral + Kd*derivative;
      // 限制输出范围[0,999]对应PWM占空比0%-100%
      if(output < 0) output = 0;
      if(output > 999) output = 999;
      motorPWM = (uint16_t)output;
      // 更新PWM占空比
      __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, motorPWM);
      // （可选）通过UART打印调试：printf("Speed:%d, PWM:%d\n", currSpeed, motorPWM);
    }
    // 循环继续等待下一数据
  }
}
```

上述代码展示了核心逻辑。在 `MX_FREERTOS_Init` 中我们直接使用代码方式创建了任务和队列（CubeMX 通常已在 freertos.c 生成等效代码，亦可直接使用）。SensorTask 和 MotorTask 则按照前文设计实现。代码运行后，理论上电机将被驱动尝试达到设定的 targetSpeed（编码器计数值 1000）。在调试环境下，可以通过**实时观察** `currSpeed` 和 `motorPWM` 的变化（例如设置断点或通过串口打印）来验证 PID 控制效果。如果参数合理，应该会看到 currSpeed 从0逐渐上升逼近1000的过程，motorPWM 会相应调整大小来维持速度。

**注意：** 实际效果取决于电机特性和参数调校，如负载、摩擦等都会影响，需要调整 Kp、Ki、Kd 以获得满意的响应。这部分工作可离线分析或在线实验调整。

## 8. 总结

通过本教程，我们完成了一个基于 STM32F4 + FreeRTOS 的最小闭环控制系统，从CubeMX工程配置到代码实现逐步讲解。关键要点回顾：

* **CubeMX 配置 FreeRTOS：** 使用图形工具快捷生成 RTOS 初始化代码和任务框架，大幅降低了移植和配置难度。初学者应掌握在 CubeMX 中启用RTOS、添加任务/队列并生成Keil工程的流程。
* **Keil 工程编译运行：** 熟悉 MDK 工具链设置，注意开启 USE\_NEWLIB\_REENTRANT 以安全使用 `printf` 等C库。通过仿真器/调试器可以观察 RTOS 任务运行状态，帮助理解调度机制。
* **FreeRTOS 基础概念：** 理解任务的概念及其类似线程的本质、调度器的作用和优先级的影响、RTOS延时与裸机延时的区别。这些概念有助于正确编写多任务代码而不陷入传统裸机思维误区。
* **任务划分与通信：** 学会将复杂问题拆解为并行任务：本例中分成传感器采集和控制计算两个主要任务，各自独立死循环运行，通过消息队列同步数据。使用消息队列避免了直接共享全局变量的风险，确保数据交换可靠实时。同时这也是从裸机转向RTOS的关键一步——充分利用任务并发来简化设计，而非一锅烩。
* **PID 控制实现：** 了解闭环控制系统组成和 PID 算法原理，能够将数学公式转换为离散的代码实现。尤其是要在**固定周期**调用PID算法，才能保证控制器稳定发挥作用。通过调试不同的参数体会P、I、D各部分对系统的影响，理解比例让系统响应快速但有余差，积分消除余差但过多会引起振荡，微分抑制振荡但对噪声敏感。
* **RTOS 架构优势：** 体会RTOS带来的结构化好处：我们可以轻松加入更多任务（如通信、故障监测等）而不担心扰乱控制节奏，因为 FreeRTOS 调度会按优先级保障实时任务运行。同时低优先级任务利用空闲时间完成附加功能，提高系统资源利用率。相对于裸机繁琐的定时器中断+状态机方案，RTOS 使程序逻辑更直观易维护。

希望通过本教程，读者对 STM32 上使用 FreeRTOS 实现电机闭环控制有了全面认识，并掌握从裸机思路过渡到 RTOS 思路的方法。在实际应用中，可以在此基础上扩展更复杂的功能，比如速度-位置双环控制、多个传感器融合、任务间更丰富的通信同步等。祝愿大家在 STM32 + FreeRTOS 开发之旅中不断收获新的技能和经验！

**参考资料：**

* ST 官方文档: *UM1722 《在 STM32Cube 上使用 RTOS 开发应用》*（包含 FreeRTOS CMSIS 接口使用介绍）
* CSDN 博客: 《在STM32Cube中使用FreeRTOS：入门体验》
* 博客园文章: 《CubeMX使用FreeRTOS编程指南》
* CSDN 博客: 《Freertos-小车开发笔记3 -- PID 控制编码电机》 (软件定时器+PID 实战)
* CSDN 博客: 《PID控制电机》 (PID 原理离散化讲解)
