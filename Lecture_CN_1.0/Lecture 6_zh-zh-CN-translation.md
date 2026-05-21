## 学习目标

*   探索实时操作系统（RTOS）
*   探索 ESP32-S3 上的内存结构、使用与操作

## 实时操作系统（RTOS）

在构建简单的嵌入式项目时，我们通常将代码编写在一个单一无限循环中，这常被称为"超级循环"或"裸机"编程。在这种方法中，微控制器逐条执行指令。如果某个特定函数需要很长时间才能完成，或者我们使用了阻塞函数，整个处理器就会停止，在该函数完成之前无法运行其他任何代码。
随着嵌入式系统通过整合 Wi-Fi、传感器读取、显示更新和用户输入而变得日益复杂，超级循环方法开始显得效率低下。系统会变得响应迟钝且难以管理。
为克服这一局限，我们采用实时操作系统（RTOS）。RTOS 充当微控制器的高效管理者。我们不再使用单一的大型循环，而是将程序分解为多个称为任务的独立小程序。RTOS 根据时序和优先级，在处理器上快速切换这些任务的执行。这种快速切换营造出同时执行的假象，使系统具备高响应性和确定性。

### FreeRTOS 与 ESP32-S3

ESP-IDF 框架采用 FreeRTOS 作为其核心操作系统。FreeRTOS 是嵌入式系统的行业标准。然而，传统的 FreeRTOS 部署通常为单核架构。由于 ESP32-S3 采用双核处理器，乐鑫科技对 FreeRTOS 进行了大幅修改以支持 SMP（对称多处理）。

这意味着 ESP-IDF 版本的 FreeRTOS 不仅允许我们将程序拆分为多个任务，还能明确指定任务运行在核心 0（通常用于系统任务和通信任务，如 Wi-Fi/蓝牙）或核心 1（保留给应用程序代码）上。

### 调度器行为

FreeRTOS 的核心是调度器，这个组件负责决定在任意时刻哪个任务能访问 CPU。调度器通过一种称为滴答中断的周期性信号来运作。微控制器内部的硬件定时器以固定频率生成此中断，在 ESP-IDF 系统中通常为每 1 毫秒一次（1 kHz）。每次发生此中断时，FreeRTOS 会短暂暂停正常执行，并重新评估系统中所有任务的状态。

这种周期性重新评估赋予了 FreeRTOS“实时”特性。操作系统不会让单个程序流程无限运行，而是持续检查是否有其他任务应当接管处理器控制权。任务可处于*运行* 、 *就绪* 、 *阻塞*或*挂起*等不同状态。调度器的职责就是检查这些状态，并选择优先级最高且处于就绪状态的任务来执行。

FreeRTOS 采用抢占式调度模型，这意味着任务优先级直接影响 CPU 访问权限。当高优先级任务进入就绪状态时，若当前正在运行低优先级任务，调度器会立即中断低优先级任务，将执行权切换至高优先级任务。这一过程被称为抢占。

调度器还管理多个任务共享相同优先级的情况。在这种情况下，FreeRTOS 通常采用一种称为**时间片轮转**的技术。调度器不会让某个任务独占 CPU，而是基于系统滴答将处理器时间划分为小的时间片。一个任务可以运行一个滴答周期，之后另一个相同优先级的任务获得执行机会。只要这两个任务都处于就绪状态，这种交替模式就会持续进行。

一个重要细节是上下文切换发生得非常迅速。FreeRTOS 在恢复下一个任务的上下文之前，会保存当前任务的执行上下文，包括 CPU 寄存器和栈信息。由于这个过程非常轻量，操作系统能够在任务之间快速切换，即使在单核处理器上，多任务处理看起来也像是同时进行的。

### 任务

任务是 FreeRTOS 最基本的构建模块。我们可以将任务视为一个独立的程序，它拥有自己的无限循环、自己的优先级以及自己分配的内存（称为栈）。

FreeRTOS 中的任务运行于不同状态：

*   **运行中：** 任务当前正在 CPU 核心上执行。
*   **就绪：** 任务已准备好运行，但因更高优先级的任务正在占用 CPU 而处于等待状态。
*   **阻塞：** 任务正在等待某个事件发生（如定时器到期，或等待传感器数据）。处于阻塞状态时，任务消耗**零** CPU 时间。
*   **已挂起：** 任务被程序员明确暂停，在显式恢复前不会再次运行。

#### 任务优先级

每个任务都会被分配一个从 0 到 `configMAX_PRIORITIES - 1` 的优先级。在 ESP-IDF 中，最大值通常为 24。数字越大表示优先级越高。FreeRTOS 调度器严格遵循优先级规则：当高优先级任务准备就绪时，会立即暂停低优先级任务的执行。

#### 在 ESP-IDF 中编写任务

让我们构建一个简单的项目来演示任务。我们希望以完全不同的频率闪烁两个 LED。我们可以使用带延迟的超循环来实现，这种方法确实可行，但会产生阻塞代码。如果频率不相关，我们可能会遇到问题。使用 FreeRTOS，我们可以为每个 LED 分配各自独立的任务。

首先，我们定义任务函数。FreeRTOS 中的每个任务必须返回 `void`，接受一个 `void *` 参数，并包含一个无限循环。关键在于，我们必须使用 `vTaskDelay()` 而不是标准延迟。`vTaskDelay()` 会将任务置于**阻塞**状态，从而释放 CPU 核心来运行其他任务。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"

#define LED_1 4
#define LED_2 5

// 任务 1：每 500 毫秒闪烁 LED 1
void blink_task_1(void *pvParameters) {
gpio_set_direction(LED_1, GPIO_MODE_OUTPUT);

while (1) {
gpio_set_level(LED_1, 1);
vTaskDelay(pdMS_TO_TICKS(500)); // 阻塞 500 毫秒
gpio_set_level(LED_1, 0);
vTaskDelay(pdMS_TO_TICKS(500));
    }
}

// 任务 2：每 1000 毫秒闪烁 LED 2
void blink_task_2(void *pvParameters) {
gpio_set_direction(LED_2, GPIO_MODE_OUTPUT);

while (1) {
gpio_set_level(LED_2, 1);
vTaskDelay(pdMS_TO_TICKS(1000)); // 阻塞 1000 毫秒
gpio_set_level(LED_2, 0);
vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

现在，在 `app_main()` 函数内部，我们使用 `xTaskCreatePinnedToCore` 创建这些任务。该函数需要：

1.  任务函数名称。
2.  用于调试的人类可读字符串名称。
3.  栈大小（以字节为单位），决定了任务所需的内存空间。
4.  传递给任务的参数，此处传入 `NULL`。
5.  优先级级别；我们对两个任务都使用 `1`。
6.  任务句柄；我们传递 `NULL`，因为后续不需要引用它。
7.  核心 ID 为 `0` 或 `1`，或者使用 `tskNO_AFFINITY` 让 RTOS 自行选择。

```c
void app_main(void) {
// 在核心1上创建任务1
xTaskCreatePinnedToCore(blink_task_1, "任务 1", 2048, NULL, 1, NULL, 1);

// 在核心1上创建任务2
xTaskCreatePinnedToCore(blink_task_2, "任务 2", 2048, NULL, 1, NULL, 1);
}
```

#### 任务管理：挂起、恢复与删除

我们不仅能创建和运行任务，还能控制其状态与生命周期。要实现这一点，需要在创建任务时捕获任务句柄。

*   **`vTaskSuspend(TaskHandle_t xTaskToSuspend)`**：暂停某个任务。该任务将进入挂起状态，并忽略所有 RTOS 事件，直至被恢复。
*   **`vTaskResume(TaskHandle_t xTaskToResume)`**：将任务从挂起状态恢复至就绪状态。
*   **`vTaskDelete(TaskHandle_t xTaskToDelete)`**：彻底销毁一个任务，并释放其分配的栈内存。（若要“重启”任务，需先删除它，再调用 `xTaskCreate` 重新创建）。

```c
TaskHandle_t myTaskHandle = NULL;

void app_main(void) {
// 1. 创建任务并保存其句柄
xTaskCreatePinnedToCore(blink_task_1, "Task 1", 2048, NULL, 1, &myTaskHandle, 1);

// 2. 挂起任务（停止它）
vTaskSuspend(myTaskHandle);
// 3. 恢复任务
vTaskResume(myTaskHandle);
// 4. 删除任务（传入 NULL 将删除调用该函数的任务）
vTaskDelete(myTaskHandle);
}
```

### 任务间通信：队列

当多个任务同时运行时，它们不可避免地需要共享数据。例如，"传感器任务"读取温度数据，而"显示任务"将其打印到 OLED 屏幕上。

我们可以使用全局变量来传递这些数据。但在 RTOS 中，全局变量是危险的。如果传感器任务正在更新全局变量的过程中，RTOS 突然切换到显示任务，显示任务可能会读取到损坏或不完整的数据。

为了解决这个问题，FreeRTOS 提供了队列（Queues）。队列是任务之间一种安全的先进先出（FIFO）管道。

*   发送任务将数据推入队列的尾部。
*   接收任务从队列的头部取出数据。
*   如果队列为空，接收任务会自动进入阻塞状态，直到有数据到达，从而不浪费任何 CPU 时间。

#### 编程队列

要使用队列，首先需要创建一个类型为 `QueueHandle_t` 的全局变量。之后，我们可以通过三个主要函数来操作队列：

*   `xQueueCreate()` 用于创建一个队列对象。它接受两个参数：队列长度（即队列能容纳的最大元素数量）和队列中每个元素的大小。
*   `xQueueSend()` 用于向队列发送新数据。它接受三个参数：全局队列变量、要添加的数据，以及超时值——该参数指定当队列已满时函数应等待的时间。
*   `xQueueReceive()` 用于从队列中读取和获取数据。与上一个函数类似，它接受三个参数：队列句柄、指向接收数据存储位置的指针，以及指定函数等待数据时长的超时值。

让我们使用队列和任务重新构建第三讲中创建的自动照明系统。首先，搭建电路。

![](./attachments/ldr_circuit.png)

现在我们可以创建程序了。首先导入所需的库并创建全局队列变量。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_adc/adc_oneshot.h"
#include "driver/gpio.h"

QueueHandle_t data_queue;
```

之后，我们创建两个代表任务的函数：

*   **生产者任务**从 LDR 传感器读取数据并将其发送到队列中。
*   **消费者任务**从队列接收数据，并根据接收到的值控制 LED。

```C

void producer_task(void *pvParameters) {


adc_oneshot_unit_handle_t adc1_handle;

adc_oneshot_unit_init_cfg_t init_config1 = { .unit_id = ADC_UNIT_1 };

adc_oneshot_new_unit(&init_config1, &adc1_handle);

adc_oneshot_chan_cfg_t 配置 = {

.bitwidth = ADC_BITWIDTH_DEFAULT, .atten = ADC_ATTEN_DB_12

    };

adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_4, &配置);


int ldrValue = 0;

while (1) {

adc_oneshot_read(adc1_handle, ADC_CHANNEL_4, &ldrValue);

xQueueSend(data_queue, &ldrValue , pdMS_TO_TICKS(100));

vTaskDelay(pdMS_TO_TICKS(500)); 

    }

}


void consumer_task(void *pvParameters) {

int received_data = 0;

while (1) {

// 永远等待（portMAX_DELAY）队列中有数据到达

xQueueReceive(data_queue, &received_data, portMAX_DELAY);

如果 (接收到的数据 > 1600) {

gpio_set_level(7, 1); 

} else {

gpio_set_level(7, 0);

        }


    }

}
```

最后，在 `app_main()` 函数内部，我们将 GPIO 7 初始化为输出引脚，创建队列，然后启动这两个任务。

```c
void app_main(void) {
gpio_reset_pin(7);
gpio_set_direction(7, GPIO_MODE_OUTPUT);

data_queue = xQueueCreate(5, sizeof(int));

如果 (data_queue != NULL) {
xTaskCreatePinnedToCore(producer_task, "生产者", 2048, NULL, 1, NULL, 1);
xTaskCreatePinnedToCore(consumer_task, "消费者", 2048, NULL, 1, NULL, 1);
    }
}
```

### 资源管理：互斥锁

队列非常适合移动数据，但如果两个任务需要访问完全相同的物理硬件资源呢？
想象一下，两个任务试图在同一毫秒内同时使用 I²C 总线或 UART 终端。发送到终端的数据会变得混乱，将两个任务的输出混合在一起。这被称为竞态条件。

为防止这种情况，FreeRTOS 提供了互斥锁（Mutual Exclusion）。互斥锁就像锁住房间的实体钥匙。

1.  任务在使用共享资源前，必须先获取互斥锁。
2.  若其他任务尝试使用该资源，会发现互斥锁已不存在，并立即进入阻塞状态，在门外等待。
3.  当第一个任务完成后，它会“归还”互斥锁。
4.  等待中的任务随后获取互斥锁，锁上门，然后继续执行。

#### 编程实现互斥锁

要使用互斥锁，我们首先需要创建一个类型为 `SemaphoreHandle_t` 的全局变量。之后，我们可以通过两个主要函数来操作互斥锁：

*   `xSemaphoreCreateMutex()` 创建一个互斥锁对象，并返回其句柄。
*   `xSemaphoreTake()` 尝试在访问共享资源前锁定互斥锁。它接受两个参数：互斥锁句柄和一个超时值，该值指定当互斥锁已被锁定时任务应等待的时间。
*   `xSemaphoreGive()` 在共享资源不再被使用后释放互斥锁，允许其他任务访问该资源。

让我们创建一个示例，其中两个任务尝试同时向终端写入消息。由于终端是共享资源，我们将使用互斥锁确保一次只有一个任务能够写入。

我们首先导入所需的库并创建全局互斥锁变量。

```c
#include "freertos/FreeRTOS.h"  
#include "freertos/task.h"  
#include "freertos/semphr.h"  
#include <stdio.h>

SemaphoreHandle_t terminal_mutex;
```

之后，我们创建两个任务：

*   第一个任务向终端打印一条长消息。
*   第二个任务则打印另一条不同的消息。
*   两个任务在使用 `printf()` 之前都必须锁定互斥锁。

```C
void print_task_1(void *pvParameters) {  

while (1) {    
xSemaphoreTake(terminal_mutex, portMAX_DELAY);
printf("任务 1 正在写一条长消息...\n");  
vTaskDelay(pdMS_TO_TICKS(100));  
printf("任务 1 写完了。\n");  
xSemaphoreGive(terminal_mutex);
vTaskDelay(pdMS_TO_TICKS(500));  
	}  
}  

void print_task_2(void *pvParameters) {

while (1) {  
xSemaphoreTake(terminal_mutex, portMAX_DELAY);  
printf("任务 2 正在写入一些不同的内容...\n");  
vTaskDelay(pdMS_TO_TICKS(100));
printf("任务 2 完成写入。\n");  
xSemaphoreGive(terminal_mutex);  
vTaskDelay(pdMS_TO_TICKS(300));
	}  
}
```

最后，在 `app_main()` 函数内部，我们创建互斥锁，然后启动两个任务。

```c
void app_main(void) {    // 创建互斥锁    
terminal_mutex = xSemaphoreCreateMutex();
如果（terminal_mutex != NULL）{
xTaskCreatePinnedToCore(print_task_1, "Printer1", 2048, NULL, 1, NULL, 1);
xTaskCreatePinnedToCore(print_task_2, "Printer2", 2048, NULL, 1, NULL, 1);
	}
}
```

没有互斥锁时，两个任务可能同时尝试使用 `printf()`，导致终端输出混乱或混杂。通过使用互斥锁保护终端，每次只能有一个任务访问它。在处理变量和数据时，这种方法会变得更加强大。

#### 优先级反转

使用互斥锁时，RTOS 常见的一个问题是**优先级反转** 。当三个不同优先级的任务相互交互时，就会发生这种情况：

*   一个**低优先级**任务获取了一个互斥锁。
*   随后，一个**高优先级**任务也需要同一个互斥锁，因此它在等待该锁时被阻塞。
*   与此同时，一个**中优先级**任务进入就绪状态，准备运行。

由于中优先级任务的优先级高于低优先级任务，它会抢占低优先级任务并占用 CPU。结果导致高优先级任务仍被阻塞，等待互斥锁，而低优先级任务却无法获得足够的运行时间来释放该锁。

换句话说，高优先级任务会被中等优先级任务间接延迟，尽管中等优先级任务根本没有使用该互斥锁。

这类错误极难调试，并可能在实时系统中引发严重的时序问题。

为解决此问题，FreeRTOS 互斥锁实现了优先级继承机制。当高优先级任务因等待低优先级任务持有的互斥锁而阻塞时，RTOS 会临时将低优先级任务的优先级提升至高优先级任务的级别。这使得低优先级任务能够立即运行，完成其临界区，释放互斥锁，然后恢复至原始优先级。

### 信号与同步：二值信号量

互斥锁设计用于资源保护（锁门），而二值信号量主要用于任务间或中断与任务之间的信号传递与同步。与互斥锁不同，二值信号量不涉及资源所有权，而是用于通知某个事件已发生。它像一个简单的标志，状态可以是“可用”（1）或“不可用”（0）。系统的一部分“给出”信号量以发送信号，另一部分则“获取”信号量以等待该信号。

二值信号量最常用于处理中断服务程序（ISR）。在嵌入式系统中，我们希望 ISR 尽可能快速执行。当按钮被按下或数据包到达时，不应在中断内部进行繁重处理，因为这会阻塞整个处理器。

相反，我们采用一种称为延迟中断处理的技术：

1.  硬件触发中断。
2.  ISR 仅执行最基础的操作，并"给予"一个二进制信号量。
3.  正在"获取"该信号量的任务会被唤醒，并执行繁重的工作。

#### 二进制信号量的编程

要使用二进制信号量，我们需要用到以下函数：

*   **`xSemaphoreCreateBinary()`**：此函数用于初始化信号量对象，但理解其初始状态至关重要。与可能初始化为“可用”状态的简单标志不同，新创建的二进制信号量初始状态为空（不可用）。这意味着首次执行 `Take` 操作时，任务将进入阻塞状态，直到系统其他部分显式发出信号。
    
*   **`xSemaphoreGiveFromISR()`**：此函数用于在中断服务程序（ISR）中发送信号。其作用仅是通知系统某个事件已发生，而不执行任何繁重处理。
    
*   **`xSemaphoreTake()`**：由任务调用以等待信号。调用此函数的任务将进入阻塞状态，直到信号量被释放。一旦 ISR（或其他任务）发出信号，等待的任务会立即解除阻塞并恢复执行以处理该事件。
    

让我们通过创建一个按下 BOOT 按钮时切换 LED 的简单项目，来了解这些函数的工作原理。

首先包含所需的库并定义常量。然后声明一个变量 `xGuiSemaphore`，它将作为二进制信号量用于同步按钮中断与任务。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "driver/gpio.h"

#define BUTTON_GPIO 0
#define LED_GPIO    7

SemaphoreHandle_t xGuiSemaphore;
```

之后，我们定义了中断服务例程（ISR），该例程会在按钮中断发生时自动执行。该函数通过 `IRAM_ATTR` 放置在 IRAM 中，以确保在中断期间能够快速可靠地执行。

在 ISR 内部，我们声明了一个名为 `xHigherPriorityTaskWoken` 的变量，并将其初始化为 `pdFALSE`。FreeRTOS 使用该变量来指示释放信号量是否导致更高优先级的任务脱离阻塞状态并进入就绪状态。

函数 `xSemaphoreGiveFromISR()` 以 ISR 安全的方式释放信号量。如果有更高优先级的任务正在等待该信号量，该函数会将 `xHigherPriorityTaskWoken` 更新为 `pdTRUE`。

最后，`portYIELD_FROM_ISR()` 检查该标志，并在退出中断前请求立即进行上下文切换。这使得被唤醒的更高优先级任务能够立即运行，而无需等待下一个调度器节拍。

```c
static void IRAM_ATTR gpio_isr_handler(void* arg) {
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
xSemaphoreGiveFromISR(xGuiSemaphore, &xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

接下来，我们创建任务。该任务负责处理按钮按下事件后应执行的实际操作。它首先将 LED 的 GPIO 配置为输出模式，然后进入一个无限循环，等待来自信号量的信号。当中断服务程序（ISR）释放信号量时，该任务被唤醒，并切换 LED 的状态。

```c
void led_toggle_task(void *pvParameters) {
gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
int state = 0;

while (1) {
if (xSemaphoreTake(xGuiSemaphore, portMAX_DELAY) == pdPASS) {
state = !state;
gpio_set_level(LED_GPIO, state);
        }
    }
}
```

最后，在 `app_main()` 函数中，我们首先使用 `xSemaphoreCreateBinary()` 创建一个二值信号量。该信号量将用于中断服务程序（ISR）与任务之间的同步。创建后，我们通过检查返回的句柄是否为 `NULL` 来验证信号量是否成功分配。若为 `NULL`，则表示系统未能分配所需资源。

接下来，我们使用 `gpio_config_t` 结构体配置 GPIO。该结构体定义了所选引脚的所有设置：

*   `intr_type = GPIO_INTR_NEGEDGE` 将中断配置为下降沿触发（即信号从高电平变为低电平时），这通常用于检测按键按下——当引脚被上拉至高电平，按下时接地。
*   `mode = GPIO_MODE_INPUT` 将引脚设置为输入模式，使其能够读取外部信号，例如按键状态。
*   `pin_bit_mask = (1ULL << BUTTON_GPIO)` 通过设置 64 位掩码中的对应位，来选择正在配置的 GPIO 引脚。
*   `pull_up_en = 1` 启用内部上拉电阻，确保按钮未按下时引脚保持稳定的高电平状态。

填充配置结构体后，我们使用 `gpio_config(&io_conf)` 应用该配置，此操作会将设置写入硬件寄存器并激活 GPIO 配置。

接着，通过调用 `gpio_install_isr_service(0)` 初始化 GPIO 中断系统。该函数会建立中断服务框架，以便后续注册 GPIO 中断处理程序。参数 `0` 表示使用默认配置标志。

最后，我们使用 `gpio_isr_handler_add(BUTTON_GPIO, gpio_isr_handler, NULL)` 将中断处理程序附加到选定的 GPIO 引脚上。这会将指定的 GPIO 引脚与 ISR 函数 `gpio_isr_handler` 关联起来，当配置的中断事件发生时，该函数将自动执行。最后一个参数（`NULL`）是传递给 ISR 的用户自定义参数，在此例中未使用。

```c
void app_main(void) {

xGuiSemaphore = xSemaphoreCreateBinary();

if (xGuiSemaphore != NULL) {

gpio_config_t io_conf = {
.intr_type = GPIO_INTR_NEGEDGE, // 按下时触发（高电平到低电平）
.mode = GPIO_MODE_INPUT,
.pin_bit_mask = (1ULL << BUTTON_GPIO),
.pull_up_en = 1,
        };
gpio_config(&io_conf);


gpio_install_isr_service(0);
gpio_isr_handler_add(BUTTON_GPIO, gpio_isr_handler, NULL);
xTaskCreatePinnedToCore(led_toggle_task, "LED_Task", 2048, NULL, 10, NULL, 1);
    }
}
```

## 内存

在使用 ESP32-S3 时，理解内存的运作方式是基础。内存是芯片运行时程序代码、变量和数据驻留的地方。为了直观理解，我们可以将内存划分为两个主要工作区：闪存（Flash memory）作为永久文件柜，而 RAM（随机存取存储器）则充当快速临时工作台。

ESP32-S3 通过灵活的内存映射和两条不同的路径来管理这些工作区：一条专用于执行代码的指令总线，以及一条用于逐字节读写信息的数据总线。

### 闪存

闪存是我们的长期存储器。它安全地保存着我们的程序代码、已保存的设置和永久数据，这样即使电路板断电，所有内容也能保持安全。由于内部 RAM 有限，我们严重依赖闪存来容纳应用程序的主体部分。

*   **IROM（从闪存执行的代码）：** 这是我们大部分可执行二进制代码所在的位置。由于无法将所有内容放入内部 RAM，ESP32-S3 使用一种特殊的缓存系统，将这些代码直接映射到指令空间。这使我们能够直接从闪存执行应用程序，速度几乎与在内部存储器中执行一样快。
*   **DROM（存储在闪存中的数据）：** 这是我们的只读数据部分。我们使用 DROM 来存储程序需要读取但永远不会更改或执行的常量变量和固定数据。

### RAM

RAM 速度极快，主要用于实时计算和临时变量。但标准 RAM 在断电后会丢失所有数据。ESP32-S3 根据我们的需求，将 RAM 工作台划分为多个专用区域：

*   **DRAM（数据随机存取存储器）：** 这是非恒定静态数据、零初始化数据以及运行时堆的主要工作空间。DRAM 严格连接至数据总线，因此无法在此执行代码。若需某些数据在软件重启（热启动）后仍保留，可在 DRAM 中设置"无初始化"区域，防止其在启动时被擦除。此外，当硬件外设需要通过直接内存访问（DMA）快速收发数据时，必须将内存缓冲区妥善放置在 DRAM 中，确保其对齐和格式正确。
*   **IRAM（指令内存）：** 与 DRAM 不同，IRAM 是可执行的。我们将这一极其宝贵的空间保留给最关键的任务，例如中断处理程序或对时序敏感的代码。通过将这些操作放置在 IRAM 中，我们避免了从 Flash 读取指令时的微小延迟。然而，这需要权衡取舍：任何用于 IRAM 的空间都会减少可用于静态数据和堆的 DRAM 容量。
*   **RTC 内存（实时时钟）：** 这是一种特殊的低功耗 RAM 子集，即使我们将芯片的其他部分置于深度睡眠状态，它也能保持活动状态。
    *   **RTC 快速内存：** 我们使用这一区域来存放芯片从深度睡眠唤醒时必须立即执行的代码和数据。如果不需要将其用于唤醒程序，我们可以将剩余空间添加到通用数据堆中，不过其运行速度略低于标准 DRAM。
    *   **RTC 慢速内存：** 我们利用这一更深层的存储区域来存放全局变量和静态变量，这些变量必须在整个深度睡眠周期中保持其值，或者用于存放主处理器休眠期间需要由超低功耗（ULP）协处理器访问的数据。

### 使用 RAM

了解了 ESP32-S3 中内存的总体结构后，我们来探讨 RAM 的工作原理以及如何操作它。ESP32-S3 将 RAM 分为两个主要区域：堆和栈。

#### 栈

在之前的课程中，我们一直在不知不觉地使用栈。每当我们在函数内部创建变量时，它通常会被存储在栈中，栈用于局部和自动内存分配。每当调用一个函数时，微控制器会自动为该函数的局部变量、函数参数和返回地址预留一部分栈内存。一旦函数执行完毕，这部分内存就会被自动释放。

栈采用**后进先出（LIFO）** 机制运作，意味着最新添加的数据会最先被移除。由于其结构简单，栈内存的运行速度极快且效率极高。不过，与堆相比，栈的容量是有限的。声明过大的局部变量或进行深度嵌套的函数调用，可能会耗尽可用的栈空间，从而导致栈溢出。在 ESP32-S3 上，栈溢出通常会导致系统恐慌、崩溃或自动重启。

#### 使用栈

让我们创建一个简单的示例来理解栈的工作原理。在这个示例中，一个函数调用另一个函数，使我们能够观察函数执行期间栈是如何被管理的。

```c
#include <stdio.h>

void secondFunction() {  
int b = 20;  
printf("b = %d\n a = %d\n", b,a); // 我们可以访问 a
}  

void firstFunction() {  
int a = 10;  
printf("a = %d\n", a); 
//b = 5 + a  we cant access be
secondFunction();  
}  

void app_main(void) {  
firstFunction();
}
```

当 `app_main()` 调用 `firstFunction()` 时，微控制器会创建一个新的栈帧，其中包含局部变量 `a`、函数参数以及返回地址。在 `firstFunction()` 仍在运行期间，它又调用了 `secondFunction()`，导致另一个栈帧被放置在原有栈帧之上。这个新栈帧存储了局部变量 `b` 以及 `secondFunction()` 执行所需的信息。

![](./attachments/stack_memory.png)

一旦 `secondFunction()` 执行完毕，其栈帧会被自动移除，程序返回到 `firstFunction()`。当 `firstFunction()` 执行完毕后，其栈帧也会被释放，执行流程回到 `app_main()`。

![](./attachments/stack_free.png)

这一过程说明了为什么局部变量仅在其所属函数或嵌套函数中存在，以及为什么在函数执行完毕后无法访问其变量。函数返回后，其栈内存会被释放，并可能被其他函数后续重用。

#### 堆

与函数调用时自动分配和释放内存的栈不同，堆用于动态内存分配。堆中的内存由程序员手动管理，这意味着我们需要在需要时显式请求内存，并在不再需要时释放它。

与栈的后进先出结构不同，堆是一个大型内存池，其中的内存块可以按任意顺序分配和释放。然而，这种灵活性是有代价的。在堆上分配内存比在栈上要慢。

当编译时无法确定所需内存大小时，堆尤其有用。我们可以不在程序中创建固定大小的局部变量，而是在程序执行期间动态分配内存。这在嵌入式系统中处理缓冲区、通信数据包、传感器数据、图像处理或动态大小数组时非常常见。

在 ESP32-S3 上，堆内存由 ESP-IDF 内存分配器管理。常用的动态内存管理函数包括 `malloc()`、`calloc()`、`realloc()` 和 `free()`。ESP-IDF 还提供高级堆管理功能，允许开发者从不同内存区域（如内部 RAM、支持 DMA 的内存或外部 PSRAM）分配内存。

与栈不同，堆内存在函数执行完毕后不会自动释放。已分配的内存会一直保留，直到通过 `free()` 显式释放。未能释放未使用的堆内存会导致内存泄漏，这会逐渐减少可用内存，最终可能导致系统崩溃。

#### 操作堆内存

让我们创建一个简单示例，了解如何在 ESP32-S3 上使用 ESP-IDF 操作堆内存。

```c
#include <stdio.h>
#include <stdlib.h>

void app_main(void) {
// 为5个整数分配内存
int *numbers = (int *)malloc(5 * sizeof(int));
// 检查分配是否成功
if (numbers == NULL){
printf("内存分配失败！\n");
返回；
	}

// 将值存储到已分配的内存中
for (int i = 0; i < 5; i++){
numbers[i] = (i + 1) * 10;
    }

// 打印存储的值
for (int i = 0; i < 5; i++){
printf("numbers[%d] = %d\n", i, numbers[i]);
    }

// 释放已分配的内存
free(numbers);

printf("堆内存已释放\n");
}
```

在这个例子中，`malloc()` 函数从堆中动态分配了足够存储五个整数的内存空间。返回的指针 `numbers` 包含了所分配内存块的地址。

```c
int *numbers = (int *)malloc(5 * sizeof(int));
```

与栈变量不同，只要未使用 `free()` 释放，即使在另一个函数内部分配，这块内存仍然有效。

分配内存后，我们通过指针索引正常在数组中存储值。

```c
numbers[i] = (i + 1) * 10;
```

当不再需要这块内存时，我们使用 `free()` 释放它。

```
free(numbers);
```

调用 `free()` 后，堆管理器会将该内存区域标记为可用，以便程序的其他部分后续可以重新使用。

#### 在 ESP-IDF 中监控堆内存

ESP-IDF 提供了内置函数，用于在运行时监控堆内存使用情况。这在调试内存泄漏或检查可用空闲内存时非常有用。

```c
#include <stdio.h>
#include "esp_heap_caps.h"

void app_main(void){

printf("分配前的空闲堆内存：%d 字节\n", heap_caps_get_free_size(MALLOC_CAP_8BIT));

int *data = malloc(100 * sizeof(int));
printf("分配后的空闲堆内存：%d 字节\n", heap_caps_get_free_size(MALLOC_CAP_8BIT));

free(data);
printf("释放后的空闲堆内存：%d 字节\n", heap_caps_get_free_size(MALLOC_CAP_8BIT));

}
```

函数 `heap_caps_get_free_size()` 允许我们实时检查可用的堆内存。这在内存资源有限的嵌入式系统中尤为重要。

#### 控制堆内存分配

虽然像 `malloc()` 这样的标准 C 函数易于使用，但在高级嵌入式系统中存在一个重大限制：它们无法让我们指定内存应在微控制器的硬件中的哪个位置进行分配。ESP32-S3 拥有复杂的内存映射，包括快速的内部 SRAM（分为 IRAM 和 DRAM），以及通常速度较慢但容量更大的外部 PSRAM（SPI RAM）。

为了解决这一限制，ESP-IDF 提供了一个名为 `heap_caps_malloc()` 的强大函数。该函数允许我们通过应用特定的能力标志，明确指定数据应放置在哪个内存区域。

```c
#include "esp_heap_caps.h"

void *heap_caps_malloc(size_t size, uint32_t caps);
```

*   **`size`**：你想要分配的字节数。
*   **`caps`**：一个内存能力标志的位掩码，用于指定内存的分配位置和方式。

让我们探讨在 ESP32-S3 上控制堆分配的最常见用例。

**1\. 仅从内部 RAM 分配** 有时我们拥有对性能要求极高的数据，必须尽可能快地访问。默认情况下，如果启用了外部 PSRAM，`malloc()` 可能会将数据分配到那里。我们可以强制分配器仅使用快速的内部 DRAM：

```c
#include <stdio.h>
#include "esp_heap_caps.h"

void app_main(void) {
// 强制在快速内部 DRAM 中分配内存
int *data = heap_caps_malloc(100 * sizeof(int), MALLOC_CAP_INTERNAL);

if (data == NULL) {
printf("内部 RAM 分配失败！\n");
return;
    }

// ... 使用数据 ...

free(data);
}
```

**2\. 从外部 PSRAM 分配内存** ESP32-S3 的内部内存非常宝贵，通常保留给 FreeRTOS 任务和快速操作使用。如果我们需要大块内存，应将其分配到外部 PSRAM 中。这在以下场景中极为有用：

*   大数据缓冲区
*   图像处理或相机帧缓冲区
*   音频数据处理
*   大型网络数据包

```c
// 在外部 PSRAM 中分配一个巨大的缓冲区
int *buffer = heap_caps_malloc(10000 * sizeof(int), MALLOC_CAP_SPIRAM);

if (buffer == NULL) {
printf("PSRAM 分配失败！\n");
} else {
printf("成功在 PSRAM 中分配了大缓冲区。\n");
free(buffer);
}
```

**3\. DMA 安全分配** 当使用 SPI、I2S（音频）或 LCD 接口等硬件外设时，硬件通常会使用直接内存访问（DMA）。DMA 允许外设在无需 CPU 参与的情况下读写内存，从而大幅提升数据传输速度。然而，DMA 硬件只能访问特定的物理内存区域。

如果我们将标准内存传递给 DMA 控制器，可能会导致崩溃。我们必须显式请求支持 DMA 的内存：

```c
// 分配 DMA 硬件可物理访问的内存
uint8_t *dma_buffer = heap_caps_malloc(2048, MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL);
```

注意我们使用了按位或运算符（`|`）来组合标志位。这确保了内存既支持 DMA 访问，又位于内部 RAM 中。

**4\. 组合多个约束条件** 按位或运算符（`|`）允许我们根据应用需求叠加任意数量的约束条件。例如，若需要保证可字节访问（8 位）且严格位于内部 RAM 中的内存：

```c
// 必须可作为普通 8 位内存访问，且来自内部 RAM
void *p = heap_caps_malloc(512, MALLOC_CAP_8BIT | MALLOC_CAP_INTERNAL);
```

#### 堆碎片化

动态内存分配的一个重要问题是堆碎片化。当大量不同大小的内存分配与释放操作在内存中留下细小的未使用间隙时，就会产生碎片化。即使总空闲内存看似充足，也可能无法找到足够大的连续内存块来满足新的分配需求。

在长时间运行的 ESP32-S3 应用中，过度碎片化最终可能导致分配失败。为减少碎片化，可采取以下措施：

*   避免在快速循环中反复分配和释放内存。
*   尽可能复用缓冲区。
*   对于可预测的系统，优先使用固定大小的内存池。
*   尽快释放不再使用的内存。
*   在运行时监控堆内存使用情况。

在嵌入式系统中，高效的堆管理至关重要，因为与台式计算机相比，其内存资源十分有限。糟糕的内存管理可能导致系统崩溃、不稳定、意外重启或性能下降。

### 任务与内存

现在，我们需要将内存知识与之前学过的 FreeRTOS 概念联系起来。在裸机系统中，整个程序只有一个主栈。但在 RTOS 中，每个独立的任务都拥有专属的栈空间。

当我们调用 `xTaskCreatePinnedToCore()` 时，新任务栈所需的内存从何而来？它从堆中分配。如果我们指定某个任务需要 4096 字节的栈空间，RTOS 会在后台调用 `malloc`，从堆内存中专门为该任务的局部变量预留 4096 字节。

由于栈溢出是 ESP-IDF 开发中最常见的错误，FreeRTOS 提供了一个名为 `uxTaskGetStackHighWaterMark()` 的函数。该函数能告知我们任务自启动以来，其栈空间曾出现过的最小剩余量。如果这个数值接近零，我们就需要增加任务的栈大小。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include <stdio.h>

void memory_monitor_task(void *pvParameters) {
int local_array[100]; // 在此任务的栈上占用 400 字节

while (1) {
// 传入 NULL 以检查当前运行任务的栈
UBaseType_t high_water_mark = uxTaskGetStackHighWaterMark(NULL);

// 打印剩余栈空间大小（单位：字节，具体取决于架构）
printf("任务栈高水位标记：%u\n", high_water_mark);

vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

void app_main(void) {
// 我们从堆中分配 2048 字节作为此任务的栈空间
xTaskCreatePinnedToCore(memory_monitor_task, "MemTask", 2048, NULL, 1, NULL, 1);
}
```

### 处理闪存与持久数据存储

现在我们已经了解了 RAM 的工作原理，以及程序运行时它如何临时保存数据，接下来让我们探讨 ESP32-S3 如何处理需要断电或重启后依然保留的数据。与易失性且断电后数据会清空的 RAM 不同，闪存是非易失性的，这意味着即使没有电源，它也能无限期地保留数据。

ESP32-S3 使用外部 SPI 闪存，有时也使用内部嵌入式闪存来存储编译后的应用程序代码、引导加载程序和持久化用户数据。然而，我们不能简单地将数据随意写入闪存，必须进行高度有序的组织。

#### 分区表

由于闪存中存储着多个关键数据，它被划分为不同的区域或"分区"。可以将分区表视为一张地图或索引，它精确告知 ESP32-S3 每个数据的具体存储位置。

当我们编译 ESP-IDF 项目时，一个 `.csv` 文件定义了这种布局。典型的分区表将闪存划分为：

*   **引导加载程序：** 启动时立即运行以加载主应用程序的代码。
*   **出厂应用：** 我们程序的主要可执行编译代码。
*   **OTA 分区：** 为空中升级预留的空间，允许设备在旧应用仍在运行时下载新应用。
*   **NVS（非易失性存储）：** 用于保存配置设置的小型分区。
*   **数据分区：** 为 SPIFFS 或 LittleFS 等文件系统格式化的大空间。

通过以这种方式组织闪存，微控制器确保将文本文件写入数据分区时不会意外覆盖应用程序代码。
以下是 `partitions.csv` 文件的一个示例：

```
# 名称,   类型, 子类型, 偏移量,  大小, 标志
nvs,      data, nvs,     ,        0x6000,
phy_init, data, phy,     ,        0x1000,
factory,  app,  factory, ,        1M,
storage,  data, spiffs,  ,        1M,
```

#### NVS（非易失性存储）

非易失性存储（NVS）库是 ESP-IDF 用于保存少量数据的工具。它的运作方式类似于字典，采用**键值对**的形式。我们保存一段数据（值），并为其指定一个唯一的字符串名称（键）。之后，我们可以向 NVS 请求获取与该特定键关联的值。

NVS 针对闪存进行了高度优化。由于闪存只能承受有限次数的擦除操作，否则会性能下降，NVS 采用了一种称为磨损均衡的技术，将读写操作均匀分布到闪存的各个扇区，从而延长硬件的使用寿命。NVS 非常适合保存 Wi-Fi 凭据、校准偏移量或状态变量（如重启计数器）。

#### 使用 NVS

让我们来看一个使用 NVS 构建小型项目的简单示例，该项目用于统计用户按键次数，并在设备重启后仍能保存计数。

在这个项目中，我们将创建一个包含两个按钮的简单电路：

*   **计数增加按钮**连接到 **GPIO 4**
*   **重置按钮**连接到 **GPIO 5**

每次按下“计数增加”按钮，计数器数值就会增加并存储到 NVS 中。“重置”按钮会清除已存储的计数，并将计数器归零。

![](./attachments/circuit_nvs.png)

现在让我们为应用程序编写程序

```c
#include <stdio.h>
#include "nvs_flash.h"
#include "nvs.h"
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void set_counter(int32_t val){
    nvs_handle_t my_handle;
    esp_err_t err = nvs_open("storage", NVS_READWRITE, &my_handle);
    if (err == ESP_OK) {
        nvs_set_i32(my_handle, "counter", val);
        nvs_commit(my_handle);
        nvs_close(my_handle);
    }
}


void app_main(void) {
    gpio_reset_pin(5);
    gpio_reset_pin(4);

    gpio_set_direction(4, GPIO_MODE_INPUT);
    gpio_set_direction(5, GPIO_MODE_INPUT);

    gpio_set_pull_mode(5,GPIO_PULLDOWN_ONLY);
    gpio_set_pull_mode(4,GPIO_PULLDOWN_ONLY);
    
    
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        nvs_flash_erase();
        err = nvs_flash_init();
    }

    nvs_handle_t my_handle;
    int32_t counter = 0;

    err = nvs_open("storage", NVS_READWRITE, &my_handle);
    if (err == ESP_OK) {
        nvs_get_i32(my_handle, "counter", &counter);
        printf("counter  = %ld \n", counter);
        nvs_close(my_handle);
    }
    
    while (1){
	    if(gpio_get_level(4)){
			counter++;
	        set_counter(counter);
	        printf("counter  is now %ld.\n", counter);
            vTaskDelay(500 / portTICK_PERIOD_MS);
        
	    }
		if(gpio_get_level(5)){
		    counter = 0;
	        set_counter(counter);
			printf("counter  reset to 0 \n count =  %ld.\n", counter);
	        vTaskDelay(500 / portTICK_PERIOD_MS);
        }
    }
}
```

要使用 NVS（非易失性存储），我们首先需要包含所需的库。在本示例中，我们添加了两个新的头文件：`nvs_flash.h` 和 `nvs.h`。这些库提供了初始化、读取、写入和管理 NVS 内存中存储数据所需的函数。

在 `app_main()` 函数内部，我们首先使用 `nvs_flash_init()` 初始化 NVS 闪存。这会准备好闪存存储器，使其可用于数据存储。我们还会检查可能的初始化错误，例如空闲页面不足或检测到旧版 NVS。如果出现任一问题，我们会擦除 NVS 分区并重新初始化。

接下来，我们创建一个 NVS 句柄（`my_handle`）和一个名为 `counter` 的变量，其默认值为 `0`。该句柄相当于与 NVS 存储的连接。接着，我们使用 `NVS_READWRITE` 标志打开一个名为 `"storage"` 的 NVS 命名空间，该标志允许我们同时读写数据。打开命名空间后，通过 `nvs_get_i32()` 函数获取已保存的计数器值，并将其存入 `counter` 变量中。该值会被打印到终端，随后关闭 NVS 句柄。

程序随后进入一个无限循环，持续检测两个按键的状态。当连接到 GPIO 4 的按键被按下时，计数器值增加 1。更新后的值通过 `set_counter()` 函数存储到 NVS 中，并将新的计数器值打印到终端。当连接到 GPIO 5 的按键被按下时，计数器重置为 `0`，再次保存到 NVS，并显示更新后的值。

`set_counter()` 函数负责将计数器值保存到 NVS 中。它接受一个参数 `val`，代表我们要存储的值。在函数内部，我们打开 `"storage"` 命名空间，使用 `nvs_set_i32()` 将整数值保存到键 `"counter"` 下，然后调用 `nvs_commit()` 将更改永久写入闪存。最后，我们使用 `nvs_close()` 关闭 NVS 句柄以释放资源。

NVS 也支持字符串，我们可以通过 `nvs_set_str() 来存储字符串`

```c
char ssid[] = "MyWiFi";
nvs_set_str(my_handle, "wifi_ssid", ssid);nvs_commit(my_handle);
```

要读取字符串，我们使用 `nvs_get_str` 函数。由于不确定存储字符串的大小，我们首先用该函数获取字符串的大小，然后创建相同大小的缓冲区并读取字符串。

```c
size_t required_size;
nvs_get_str(my_handle, "wifi_ssid", NULL, &required_size);
char *buffer = malloc(required_size);
nvs_get_str(my_handle, "wifi_ssid", buffer, &required_size);
printf("SSID: %s\n", buffer);
free(buffer);
```

#### SPIFFS 和 LittleFS

虽然 NVS 非常适合存储小型变量，但在处理大型数据集时表现糟糕。如果我们正在构建一个网络服务器，需要存储 HTML、CSS 和图像文件，或者需要将数千个温度读数记录到文本文件中，我们就需要一个真正的文件系统。

ESP-IDF 主要支持两种用于内部闪存的文件系统：

*   **SPIFFS（SPI 闪存文件系统）：** 一种较老且广泛使用的文件系统。它提供磨损均衡功能，但不支持真正的目录结构。如果我们在 `/data/logs/temp.txt` 路径下创建文件，SPIFFS 会将整个字符串 `"/data/logs/temp.txt"` 简单地视为文件名。
*   **LittleFS：** 一种专为微控制器设计的新型高可靠性文件系统。与 SPIFFS 不同，LittleFS 支持实际的目录结构，速度显著更快，并且在写入操作期间对断电具有极强的恢复能力。凭借其卓越的性能和安全性，在现代 ESP32-S3 项目中，通常推荐使用 LittleFS 而非 SPIFFS。

#### 使用 LittleFS

默认情况下，ESP-IDF 不会为自定义文件系统预留任何闪存空间。要使用 LittleFS，我们需要创建一个自定义分区表，为存储分配专用的闪存区域。

在项目根目录下（与 `CMakeLists.txt` 同级）创建一个名为 `partitions.csv` 的文件，内容如下：

```csv
# 名称, 类型, 子类型, 偏移量, 大小, 标志
nvs, data, nvs, , 0x6000,
phy_init, data, phy, , 0x1000,
factory, app, factory, , 1M,
storage, data, spiffs, , 1M,
```

在此表中，`storage` 分区定义了一个 1 MB 的闪存区域，供 LittleFS 使用。尽管其子类型被列为 `spiffs`，该分区仍可挂载并与 LittleFS 配合使用。

接下来，我们需要告知 ESP-IDF 使用自定义分区表，并将 `esp_littlefs` 组件包含到项目中。为此，可以在 `main/` 目录下创建一个名为 `idf_component.yml` 的文件，通过 ESP-IDF 组件管理器来实现。

在构建过程中，ESP-IDF 组件管理器会自动下载并集成 LittleFS 组件。

```
dependencies:
espressif/esp_littlefs: "^1.1.0"
```

之后，在终端中运行以下命令，启用自定义分区表配置：

```
idf.py menuconfig
```

在配置菜单中，导航至：

```
分区表 -> 分区表
```

然后将选项从：

```
单一工厂应用，无 OTA
```

更改为：

```
自定义分区表 CSV
```

最后，保存配置并退出菜单。

现在 LittleFS 存储已配置完成，我们可以在项目中开始使用它。ESP-IDF 将 LittleFS 集成到虚拟文件系统（VFS）层，这意味着我们可以像在桌面系统上一样，使用标准的 C 语言文件函数，例如 `fopen()`、`fprintf()`、`fgets()` 和 `fclose()`。

以下示例演示了完整的工作流程：

*   挂载 LittleFS 分区
*   创建并写入文件
*   读取文件内容
*   卸载文件系统

```c
#include <stdio.h>
#include <string.h>

#include "esp_littlefs.h"

void app_main(void) {

    // 1. Configure the LittleFS mount
    esp_vfs_littlefs_conf_t conf = {
        .base_path = "/littlefs",       // The root path we will use in code
        .partition_label = "storage",   // Must match the name in partitions.csv
        .format_if_mount_failed = true, // Format the partition if it's empty/corrupted
        .dont_mount = false,
    };
    // 2. Initialize and mount the file system
    esp_err_t ret = esp_vfs_littlefs_register(&conf);

    if (ret != ESP_OK) {
        printf("Failed to mount or format filesystem\n");
        return;
    }

    printf("LittleFS mounted successfully.\n");

    // 3. Write a file (standard C library functions)
    printf("Opening file for writing...\n");
    FILE *f = fopen("/littlefs/hello.txt", "w");

    if (f == NULL) {
        printf("Failed to open file for writing\n");
        return;
    }
    
    fprintf(f, "Hello World from LittleFS on ESP32-S3!\n");
    fclose(f);
    
    printf("File written.\n");
    // 4. Read the file back
    printf("Opening file for reading...\n");
    
    f = fopen("/littlefs/hello.txt", "r");
    if (f == NULL) {
        printf("Failed to open file for reading\n");
        return;
    }
    char line[64];
    fgets(line, sizeof(line), f);
    fclose(f);
    
    // Remove newline character if present
    char *pos = strchr(line, '\n');
    if (pos) {
        *pos = '\0';
    }
    printf("Read from file: '%s'\n", line);
    // 5. Unmount (optional, usually done before a system reboot)
    esp_vfs_littlefs_unregister(conf.partition_label);
}
```

第一步是创建一个 `esp_vfs_littlefs_conf_t` 配置结构体。该结构体用于告知 ESP-IDF 文件系统的挂载方式。其中 `base_path` 字段定义了文件系统在应用程序中显示的虚拟目录，而 `partition_label` 必须与 `partitions.csv` 中定义的分区名称保持一致。

`format_if_mount_failed` 选项在开发阶段尤为实用。当分区为空或损坏时，ESP-IDF 会自动格式化该分区，而非直接挂载失败。

接下来，我们通过 `esp_vfs_littlefs_register()` 挂载文件系统。若挂载成功，该分区将通过 VFS 层变得可访问，从而允许我们使用标准的 C 语言文件操作函数。

创建文件时，我们使用 `fopen()` 并指定写入模式（`"w"`）。文件路径以 `/littlefs/` 开头，因为这是我们之前配置的挂载点。随后通过 `fprintf()` 向文件写入文本，最后用 `fclose()` 关闭文件，确保数据被正确保存。

写入后，示例以读取模式（`"r"`）重新打开同一文件。`fgets()` 函数将文件内容读入字符缓冲区，随后我们将其打印到串行监视器。

最后，我们使用 `esp_vfs_littlefs_unregister()` 卸载文件系统。在许多嵌入式应用中，此步骤并非必需，但在重启设备或关闭系统前执行该操作是良好的编程习惯。

#### 原始闪存分区访问

尽管 LittleFS 和 SPIFFS 等文件系统在管理文件和目录方面表现出色，但它们会引入额外开销。若需记录大量原始传感器数据数组、高速传输二进制数据流或管理自定义加密数据块，完全绕过虚拟文件系统（VFS）将赋予我们绝对的控制权与最佳性能。

访问原始闪存使我们能够通过 ESP-IDF 分区 API 直接读写 SPI 闪存芯片的特定物理扇区。

#### 原始闪存的规则

操作原始闪存效率极高，但必须遵循 NOR 闪存的物理限制：

*   **写入前先擦除：** 闪存单元在写入操作中只能从逻辑 `1` 变为 `0`。要将 `0` 恢复为 `1`，必须擦除整个扇区。未经擦除扇区，我们无法直接覆盖现有字节。
*   **扇区对齐：** 擦除操作无法针对单个字节，必须严格以 **4 KB 扇区** （4096 字节）为单位执行。任何擦除操作的偏移量和大小都必须与 4 KB 边界完美对齐。
*   **磨损管理：** 与 LittleFS 不同，原始分区访问不会自动包含磨损均衡功能。持续向同一扇区写入数据会过早地导致闪存芯片该区域性能退化。

#### 配置原始分区

为防止应用程序意外覆盖系统代码或 NVS 数据，我们必须在自定义配置中分配一个专用的沙盒分区。

在 `partitions.csv` 文件中，我们添加了一个自定义数据条目：

```
# 名称, 类型, 子类型, 偏移量, 大小, 标志
nvs, data, nvs, , 0x6000,
phy_init, data, phy, , 0x1000,
工厂，应用，工厂，，1M，
raw，data，0x99，，256K，
```

在此设置中，我们定义了一个名为 `raw_data` 的分区。我们使用通用的 `data` 类型，并分配一个自定义的十六进制子类型（`0x99`），以明确将其定义为非文件系统的原始存储空间。其大小限制为 256 KB。

#### 读写原始闪存

要与该分区交互，需包含 `esp_partition.h` 头文件。操作流程包括：通过标签定位分区、擦除目标扇区、写入二进制数据，最后读取验证。

以下示例演示了安全的原始闪存事务：

```c
#include <stdio.h>  
#include <string.h>  
  
#include "esp_partition.h"  
  
void app_main(void) {  
  
	// 1. Find the custom partition  
	const esp_partition_t *partition =  esp_partition_find_first(  
		ESP_PARTITION_TYPE_DATA,  
		0x40,  
		"storage"  
	);  
  
	if (partition == NULL) {  
		printf("Partition not found\n");  
		return;  
	}  
  
	printf("Partition found: %s\n", partition->label);  
  
	// 2. Data to write  
	const char *message = "Hello from raw flash memory!";  
  
	// 3. Erase flash sector before writing  
	esp_err_t ret = esp_partition_erase_range(  
		partition,  
		0,  
		4096  
	);  
  
	if (ret != ESP_OK) {  
		printf("Failed to erase partition\n");  
		return;  
	}  
  
	// 4. Write data to flash  
	ret = esp_partition_write(  
		partition,  
		0,  
		message,  
		strlen(message) + 1  
	);  
  
	if (ret != ESP_OK) {  
		printf("Failed to write data\n");  
		return;  
	}  
  
	printf("Data written successfully.\n");  
  
	// 5. Read data back  
	char buffer[64];  
  
	ret = esp_partition_read(  
		partition,  
		0,  
		buffer,  
		sizeof(buffer)  
	);  
  
	if (ret != ESP_OK) {  
		printf("Failed to read data\n");  
		return;  
	}  
  
	printf("Read from flash: '%s'\n", buffer);  
}
```

第一步是使用 `esp_partition_find_first()` 定位分区。我们搜索类型为 `ESP_PARTITION_TYPE_DATA`、子类型为 `0x40`、标签为 `"storage"` 的分区。

若分区存在，ESP-IDF 将返回指向 `esp_partition_t` 结构体的指针，该结构体包含分区的地址、大小和标签等信息。

在写入闪存之前，我们必须使用 `esp_partition_erase_range()` 擦除目标区域。闪存无法直接覆盖现有数据，必须先擦除闪存扇区，通常以 4 KB 块为单位进行擦除。

擦除扇区后，我们使用 `esp_partition_write()` 将原始字节写入闪存。在此示例中，我们存储了一个简单的文本字符串，但同样的方法也可用于二进制结构、数组或序列化对象。

为了验证操作，我们使用 `esp_partition_read()` 回读数据。数据被复制到 RAM 缓冲区中，并打印到串行监视器上。

与 LittleFS 或 SPIFFS 不同，原始闪存访问不提供文件、文件夹或自动内存管理。所有操作都需要手动处理，这使得该方法极其强大，但如果偏移量和擦除操作管理不当，也更容易出错。

#### 闪存退化

正如堆碎片化对 RAM 构成威胁一样，闪存磨损是非易失性存储的主要风险。

闪存物理上会随时间逐渐退化。每次擦除和写入一个扇区时，硅片都会产生微小的物理损伤。典型的闪存芯片每个扇区额定擦写次数约为10万次。

如果我们每秒将温度读数写入完全相同的原始闪存地址，只需一天多时间就会损坏该闪存芯片的特定扇区。为防止这种情况，NVS、SPIFFS 和 LittleFS 都采用了**磨损均衡**技术。磨损均衡算法会在后台将数据秘密映射到不同的物理位置，确保写入操作均匀分布在整个芯片上，从而显著延长硬件的使用寿命。
