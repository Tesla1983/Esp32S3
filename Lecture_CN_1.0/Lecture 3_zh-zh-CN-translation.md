## 学习目标

*   理解通用输入输出（GPIO）引脚
*   使用电机

## 通用输入输出（GPIO）引脚

通用输入输出（GPIO）引脚使 ESP32-S3 能够连接并与外部世界通信。这些引脚可根据项目需求配置为输入或输出模式。

ESP32-S3 配备了高度灵活的 GPIO 矩阵，提供多达 45 个物理 GPIO 引脚（具体可用数量取决于我们使用的特定 DevKit）。这些引脚主要分为两大功能用途：

*   **数字引脚：** 用于读取或写入数字信号 HIGH（1）或 LOW（0）。
*   **模拟引脚：** 用于读取模拟信号，通过内部 ADC（模数转换器）捕获一定范围的电压值。

![](./attachments/Esp32_Gpio.png)

### 数字 GPIO 引脚

与更简单的微控制器不同，ESP32-S3 几乎允许将任何引脚配置为数字输入或输出功能。这些引脚仅处理两种状态：高电平（3.3V）和低电平（0V）。这意味着它们可以发送或接收数字信号，例如打开或关闭 LED，或读取按钮的状态。

每个数字引脚可根据电路需求配置为输入或输出。当设置为输入时，引脚读取来自外部组件的信号；当设置为输出时，它发送信号以控制设备。

#### ESP-IDF 中的数字 GPIO 功能

在使用 ESP-IDF 框架进行 C 语言编程时，我们使用特定的驱动函数来控制数字引脚。要访问这些函数，必须包含 `#include "driver/gpio.h"` 头文件。

**`gpio_set_direction()`：** 此函数用于配置数字引脚的模式。它允许我们将引脚设置为输入或输出，接受两个参数：

*   引脚编号（使用 `GPIO_NUM_x` 宏，例如 `GPIO_NUM_13`）
*   我们想要设置的**模式** （`GPIO_MODE_INPUT` 或 `GPIO_MODE_OUTPUT`）

**``gpio_set_pull_mode()： 此函数用于设置内部电阻，以避免不稳定的读数（例如使用按钮时）。它接受两个参数：引脚编号和状态，如 `GPIO_PULLUP_ONLY` 或 `GPIO_PULLDOWN_ONLY`。``**

**`gpio_set_level()`:** 此函数用于向引脚发送数字信号。它允许我们将引脚设置为高电平或低电平，接受两个参数：

*   引脚编号
*   信号状态（`1` 表示高电平，`0` 表示低电平）

**`gpio_get_level()`:** 最后，此函数用于读取数字引脚的状态。根据输入信号，它返回 `1`（高电平）或 `0`（低电平），接受一个参数：

*   引脚编号

### 数字输出与输入设备

现在我们已经掌握了处理数字 GPIO 引脚的基本功能，接下来让我们探索一些常见且最常用的数字传感器和设备。

#### LED 灯与按钮开关

LED（发光二极管）是一种数字输出组件，当电流通过时会发光。它具有极性，这意味着电流只能单向流动。LED 有两种状态：开启和关闭。我们可以通过向数字引脚发送高电平信号来点亮 LED，发送低电平信号来关闭它。LED 常被用作系统状态指示器。

按钮开关用作输入设备。它们也有两种状态：按下和释放。当按钮被按下时，它会向 ESP32-S3 发送一个信号（高电平或低电平，具体取决于电路设计）。微控制器读取该信号，并根据按钮状态执行相应操作。

让我们构建一个简单的电路，通过按钮开关控制 LED 的亮灭。按下按钮时 LED 点亮，再次按下时 LED 熄灭。

本项目将使用 GPIO 4 控制 LED，GPIO 1 连接按钮开关。我们将为 LED 串联一个 220 欧姆电阻，并为按钮开关配置 GPIO 引脚的下拉电阻。

![](./attachments/circuit1.png)

现在，让我们创建程序，首先包含必要的库。接着，我们声明一个变量来追踪 LED 的状态。

在 `app_main` 函数中，我们首先重置引脚。然后，将 GPIO 2 配置为 LED 的输出，GPIO 4 配置为按键的输入，同时为 GPIO 4 启用下拉电阻，使其默认状态为低电平（0）。这样能确保按键未按下时引脚读取到稳定的 0 值。
之后，我们创建一个无限循环，持续读取 GPIO 4 的状态。如果引脚为高电平，说明用户按下了按键。此时我们切换 `LedState` 变量，并相应更新 LED 状态。

最后，我们添加500毫秒的延迟以防止按键抖动效应，确保单次按压不会被多次检测到。

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include <stdbool.h>

bool LedState = false;

void app_main(void){

gpio_reset_pin(4);
gpio_reset_pin(2);
gpio_set_direction(2, GPIO_MODE_OUTPUT);
gpio_set_direction(4, GPIO_MODE_INPUT);
gpio_set_pull_mode(4, GPIO_PULLDOWN_ONLY);

while (1) {
if(gpio_get_level(4)){
LedState ^=1;
gpio_set_level(2, LedState);
vTaskDelay(500 / portTICK_PERIOD_MS);
        }  
    }
}
```

#### 蜂鸣器

蜂鸣器是一种特殊组件，当我们向其发送高电平信号时，它会发出声音。在电子项目中，它常被用作警报或通知装置，例如闹钟、计时器和警告系统。蜂鸣器主要分为两种类型：

*   **有源蜂鸣器：** 此类蜂鸣器内置振荡器，能以预设频率发出声音。我们只需发送高电平或低电平信号即可控制其开关。它使用简单，但对声音的控制能力有限。
*   **无源蜂鸣器：** 此类蜂鸣器没有内置振荡器。相反，它允许我们通过 ESP32-S3 发送 PWM（脉冲宽度调制）信号来生成不同频率的声音。这让我们对音调拥有更强的控制能力，从而可以播放旋律或不同的警报音。

让我们构建一个简单的项目：当按下按钮时，LED 闪烁且有源蜂鸣器发出声音。我们将 LED 连接到 GPIO 2，蜂鸣器连接到 GPIO 20，按钮连接到 GPIO 5。

![](./attachments/circuit2.png)

现在，我们使用 ESP-IDF 创建 C 程序。首先包含必要的库。在这个示例中，我们不需要跟踪任何状态，因此直接进入 `app_main` 函数。首先，重置将要使用的引脚，然后将 GPIO 4 配置为输入，GPIO 2 和 GPIO 20 配置为输出。接着，将 GPIO 5 设置为下拉模式，这样在未按下按钮时默认读取为低电平（0）。

最后，创建一个无限循环。在这个循环中，使用一个嵌套循环，当按钮被按下（GPIO 5 为高电平）时运行。此时，将 GPIO 20 设为高电平以激活蜂鸣器，同时通过点亮 LED（连接至 GPIO 2）、等待 500 毫秒、熄灭 LED、再等待 500 毫秒的方式，使 LED 产生闪烁效果。

在内层循环外部，将 GPIO 20 设为低电平，这样当按钮未被按下时，蜂鸣器将停止工作。

```C
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"

void app_main(void){
gpio_reset_pin(5);
gpio_reset_pin(2);
gpio_reset_pin(20);

gpio_set_direction(2, GPIO_MODE_OUTPUT);
gpio_set_direction(20, GPIO_MODE_OUTPUT);
gpio_set_direction(5, GPIO_MODE_INPUT);

gpio_set_pull_mode(5, GPIO_PULLDOWN_ONLY);
while (1) {
while(gpio_get_level(5)){
gpio_set_level(20, 1);
gpio_set_level(2, 1);
vTaskDelay(500 / portTICK_PERIOD_MS);
gpio_set_level(2, 0);
vTaskDelay(500 / portTICK_PERIOD_MS);
        }  
gpio_set_level(20, 0);
    }
}
```

#### 障碍物传感器

另一个重要的传感器是障碍物传感器。它使我们能够通过红外信号检测障碍物的存在。该传感器有一个发射红外光的 LED 和一个接收反射红外信号的接收器。当物体位于传感器前方时，红外光会反射回接收器，从而使传感器能够检测到障碍物。

该传感器还包含一个内部可变电阻（电位器），使我们能够调节检测距离。障碍物传感器有三个引脚：VCC、GND 和 OUT（输出）。当检测到障碍物时，输出引脚变为低电平（0），当没有障碍物时，输出引脚变为高电平（1）。

我们使用内置电位器来调节障碍物传感器的检测距离。

当电位器顺时针旋转时，检测距离变大，传感器灵敏度提高；当逆时针旋转时，检测距离变短，传感器灵敏度降低。

![](./attachments/obstacle.png)

让我们创建一个简单的项目：当障碍物传感器检测到物体时，蜂鸣器发出声音。我们将使用 GPIO 20 控制蜂鸣器，并使用 GPIO 4 读取障碍物传感器的信号。

![](./attachments/circuit3.png)

现在让我们创建程序。和之前一样，我们首先包含程序所需的库。在 `app_main` 函数中，我们重置 GPIO 引脚 4 和 20。然后，将 GPIO 4 配置为输入，GPIO 20 配置为输出。同时将 GPIO 4 设置为下拉模式。

接下来，我们创建一个无限循环让程序持续运行。在这个循环中，我们检查 GPIO 4 的状态。如果为低电平（0），表示检测到物体，于是将 GPIO 20 设置为高电平（1）以开启蜂鸣器。否则，将 GPIO 20 设置为低电平（0）以关闭蜂鸣器。

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void app_main(void){

gpio_reset_pin(4);
gpio_reset_pin(20);

gpio_set_direction(4, GPIO_MODE_INPUT);
gpio_set_direction(20, GPIO_MODE_OUTPUT);

gpio_set_pull_mode(4, GPIO_PULLDOWN_ONLY);

while (1) {
if(gpio_get_level(4) == 0){
gpio_set_level(20, 1);

} else{
gpio_set_level(20, 0);

        }
vTaskDelay(200 / portTICK_PERIOD_MS);
    }
}
```

#### 继电器

ESP32-S3 的数字引脚严格采用 3.3V 逻辑电平，每个引脚最大可提供约 40mA 电流。这足以控制 LED、蜂鸣器等小型电子元件，以及为其他模块提供逻辑输入信号。

然而，当我们需要构建大型项目时，可能需要控制需要更高电压（如 12V 直流或 220V 交流）或更大电流的设备，例如灯具、风扇、水泵或电机。ESP32-S3 的引脚无法直接为这些设备供电，若未加保护直接连接，将永久损坏微控制器。

为了解决这个问题，我们使用继电器。继电器是一种电控开关，它允许 ESP32-S3 发出的低功率 3.3V 信号安全地控制高功率设备。继电器内部有一个线圈，起着关键作用。当 ESP32-S3 发送控制信号时，小电流流过线圈，产生磁场。这个磁场拉动金属衔铁，使内部开关闭合，从而允许电流在高功率电路中流通。当微控制器停止发送信号时，线圈中的电流停止，磁场消失，弹簧将衔铁推回原位，开关再次断开。

![](./attachments/relay.png)

让我们用继电器和物体传感器构建一个简单的项目。在这个项目中，我们将使用障碍物传感器，当用户将手放在传感器前时打开灯，当用户再次将手放在传感器前时关闭灯。

首先，连接障碍物传感器：

*   **VCC → 3.3V** 连接到开发板
*   **GND → GND** 在开发板上
*   **输出引脚 → GPIO 10**

接下来，我们连接继电器模块，请确保您的继电器模块能够被 3.3V 逻辑信号触发，大多数标准光耦隔离继电器模块都支持。控制侧，低压部分：

*   **VCC → 3.3V**
*   **GND → GND**
*   **IN（控制引脚）→ GPIO 1**

**高功率侧：** 在高功率侧，继电器有三个端子：COM（公共端）、NO（常开端）和 NC（常闭端）。要控制一盏灯：

*   将电源的**火线（相线）** 连接到 COM 端。
*   将常开端连接到灯的一根线上。
*   将灯的另一根线接回零线。

![](./attachments/circuit4.png)

现在我们来为项目编写 C 程序。

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

bool lightState = false;

void app_main(void){

gpio_reset_pin(1);
gpio_reset_pin(10);


gpio_set_direction(1, GPIO_MODE_OUTPUT);
gpio_set_direction(10, GPIO_MODE_INPUT);

gpio_set_pull_mode(10,GPIO_PULLDOWN_ONLY);

while (1) {
if(gpio_get_level(10) == 0){
lightState ^=1;
gpio_set_level(1, lightState);
vTaskDelay(1000 / portTICK_PERIOD_MS);
        } 

    }
}
```

#### 超声波传感器

超声波传感器是与微控制器配合使用的最实用的传感器之一。它们通过声波帮助我们测量距离。当我们需要计算井的深度或洞穴的长度时，常常依赖声音。例如，过去人们会向井中扔一块石头，倾听石头撞击地面时发出的声音。通过测量声音返回所需的时间，他们可以估算井的深度。

探索洞穴时也适用同样的原理。探险者可能会大喊一声，然后倾听回声。回声返回所需的时间能让人了解洞穴岩壁有多远。

类似地，像 HC-SR04 这样的超声波传感器会发出高频声波，并测量回声返回所需的时间。ESP32-S3 利用这个时间计算出到某个对象的距离。

![](./attachments/ultrasonic.png)

HC-SR04 超声波传感器有四个引脚：VCC 和 GND 用于为传感器供电。TRIG 引脚用于发送一个短暂的高电平脉冲（约 10 微秒），使传感器发射超声波。随后 ECHO 引脚变为高电平，并保持高电平，直到反射的声波返回传感器。这个高电平信号的持续时间代表了声音传播到对象并返回所需的时间。

利用这一点，我们可以通过将持续时间乘以声速，再除以二来计算距离。之所以除以二，是因为测量的持续时间代表声波传播到对象并返回的时间，这相当于实际距离的两倍。

让我们创建一个项目，使用三个 LED 来指示距离：

*   当物体距离达到 60 厘米或更远时，红色 LED 灯亮起。
*   当物体距离至少 30 厘米时，黄色 LED 灯亮起。
*   当物体距离小于 30 厘米时，绿色 LED 灯亮起。

每个 LED 灯均通过 220Ω电阻连接到对应的 GPIO 引脚。

*   **GPIO 13** → 红色 LED
*   **GPIO 12** → 黄色 LED
*   **GPIO 11** → 绿色 LED

HC-SR04 超声波传感器有四个引脚：

1.  **VCC** 连接至 5V 引脚。
2.  **GND** 连接至开发板上的 GND 引脚。
3.  **Trig（扳机）** 用于发送超声波脉冲，我们将其连接至 GPIO 20。
4.  **Echo** 用于接收反射信号，我们将其连接至 GPIO 19。

Echo 引脚输出 5V 电压，可能会损坏工作在 3.3V 逻辑电平的微控制器。
为了安全连接，请使用分压电路：

*   一个220Ω电阻
*   一个330Ω电阻

![](./attachments/circuit5.png)

完成电路连接后，接下来需要编写程序。我们将使用 `esp_timer_get_time()` 函数来测量微秒级时间。

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_timer.h"
#include "esp_rom_sys.h" 

void app_main(void) {
    gpio_set_direction(13, GPIO_MODE_OUTPUT);
    gpio_set_direction(12, GPIO_MODE_OUTPUT);
    gpio_set_direction(11, GPIO_MODE_OUTPUT);
    gpio_set_direction(20, GPIO_MODE_OUTPUT);
    gpio_set_direction(19, GPIO_MODE_INPUT);

    float duration, distance;

    while (1) {
        
        gpio_set_level(20, 0);
        esp_rom_delay_us(2);
        gpio_set_level(20, 1);
        esp_rom_delay_us(10);
        gpio_set_level(20, 0);

        uint32_t start_time = 0;
        uint32_t end_time = 0;

        while(gpio_get_level(19) == 0) {
            start_time = esp_timer_get_time();
        }
        
        while(gpio_get_level(19) == 1) {
            end_time = esp_timer_get_time();
        }

        duration = (float)(end_time - start_time);
        distance = (duration * 0.0343) / 2.0;

        if (distance >= 60) {  
            gpio_set_level(13, 1); 
            gpio_set_level(12, 0); 
            gpio_set_level(11, 0);
        } else if (distance >= 30) {  
            gpio_set_level(13, 0); 
            gpio_set_level(12, 1); 
            gpio_set_level(11, 0); 
        } else {  
            gpio_set_level(13, 0); 
            gpio_set_level(12, 0);  
            gpio_set_level(11, 1); 
        }
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

首先引入必要的库：一个用于控制 GPIO 引脚，其他用于处理任务和定时功能。在 `app_main` 函数中，我们将 GPIO 引脚（11、12、13、20）配置为输出模式（用于 LED 和触发引脚），将 GPIO 19 配置为输入模式（用于回波引脚）。同时声明两个变量来存储信号持续时间和计算出的距离。

接着进入无限循环。首先从 GPIO 20 发送一个短脉冲：先将其设为低电平，然后保持高电平 10 微秒，再恢复为低电平。这个脉冲会触发 HC-SR04 超声波传感器发射声波。随后，我们测量回波信号返回所需的时间。当回波引脚变为高电平时记录 `start_time`，当它再次变为低电平时记录 `end_time`，这表示声波已返回传感器。

利用这个时间差，我们计算回波脉冲的持续时间，然后根据声速计算出距离。最后，我们使用条件语句控制三个 LED：当物体距离较远（≥ 60 厘米）时点亮一个，中等距离（≥ 30 厘米）时点亮另一个，近距离时点亮最后一个。每次循环迭代结束时，会添加一个短暂的延迟，然后重复整个过程。

### 脉冲宽度调制

脉冲宽度调制（PWM）是一种利用数字信号模拟模拟输出的技术。PWM 的原理是在恒定时间周期 T 内，反复将输出信号打开和关闭。

这个固定周期 T 被称为 PWM 周期，在此周期内，信号会保持开启一段时间，并在剩余时间内保持关闭。信号在一个周期内保持开启的时间百分比称为占空比。

输出电压由信号在一个完整周期内的平均值决定。例如：

*   如果输出在整个周期内保持开启（100%占空比），平均电压将为 **3.3V**。
*   如果输出从未开启（0%占空比），平均电压将为 **0V**。
*   如果输出在半个周期内保持开启（50%占空比），平均电压将为 **1.65V**。

$$
V_{average}​=\frac{Ton}{T}​​×Vmax​=DutyCycle×Vmax​ 
$$

 ![](./attachments/PWM.png)

ESP32-S3 具备高度灵活的 GPIO 矩阵。与那些将 PWM 限制在特定硬件引脚上的传统微控制器不同，ESP32-S3 允许我们利用 LED 控制（LEDC）外设，将 PWM 信号路由到所有可作为输出的引脚上。

在 ESP-IDF 中使用 PWM 时，我们需要用到 LEDC（LED 控制器）外设。该硬件模块能够高效生成 PWM 信号，且不会过度占用 CPU 资源。配置过程主要分为两步：首先设置一个定时器，用于定义 PWM 信号的频率和分辨率；然后配置一个通道，将该定时器连接到指定的 GPIO 引脚。配置完成后，我们便可通过调整占空比来控制信号——占空比决定了在每个 PWM 周期内信号保持高电平的时间长度。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"

void app_main() {

// 1. 配置 LEDC 定时器
ledc_timer_config_t ledc_timer = {
.speed_mode       = LEDC_LOW_SPEED_MODE,
.timer_num        = LEDC_TIMER_0,
.duty_resolution  = LEDC_TIMER_12_BIT, // 12 位分辨率
.freq_hz          = 5000, // 5 千赫兹
.clk_cfg          = LEDC_AUTO_CLK
    };
ledc_timer_config(&ledc_timer);

// 2. 配置 LEDC 通道
ledc_channel_config_t ledc_channel = {
.speed_mode     = LEDC_LOW_SPEED_MODE,
.channel        = LEDC_CHANNEL_0,
.timer_sel      = LEDC_TIMER_0,
.gpio_num       = 2,
.duty           = 0,   // 从 0% 占空比开始
.hpoint         = 0
    };
ledc_channel_config(&ledc_channel);

// 3. 在运行时更改占空比
while (1) {
ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 2047); // 约 50%
ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
vTaskDelay(pdMS_TO_TICKS(1000));

ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0); // 0%
ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

定时器配置负责定义 PWM 信号的行为。`freq_hz` 字段设置信号重复的速度；在此示例中，5000 Hz 表示信号每秒完成 5000 个周期。`duty_resolution` 决定占空比可用的离散步数。采用 12 位分辨率时，计数器范围从 0 到 4095，可对信号进行精细控制。这两个参数相互关联：提高频率会降低最大可实现的分辨率，而提高分辨率则会限制最大频率。时钟源（`clk_cfg`）通常设置为自动选择，允许驱动程序选择最合适的时钟。

通道配置将定时器连接到物理 GPIO 引脚。`channel` 字段用于选择可用的 LEDC 通道之一，而 `timer_sel` 将该通道链接到先前配置的定时器。`gpio_num` 决定哪个引脚输出 PWM 信号。`duty` 字段设置初始占空比，`hpoint` 定义信号在 PWM 周期内变为高电平的时刻（通常保持为 0 以实现标准行为）。在 ESP32-S3 上，所有通道均以低速模式运行，对 PWM 参数的更改必须由软件显式应用。

一旦 PWM 开始运行，我们通过修改占空比来控制它。占空比表示在一个周期内信号为高电平的时间比例。在 12 位分辨率下，有效值范围为 0 到 4095。值为 0 对应 0%（始终关闭），而 4095 对应接近 100%（始终开启）。例如，值为 2047 时会产生约 50%的占空比。要应用新的占空比值，我们首先调用 `ledc_set_duty()` 更新内部值，然后调用 `ledc_update_duty()` 将更改推送到硬件。如果不调用更新函数，新的占空比将不会生效。通过持续调整占空比，我们可以控制亮度、电机速度或任何其他由 PWM 驱动的行为。

### 模拟 GPIO 引脚

我们了解了数字信号及其仅处理高电平和低电平（0V 和 3.3V）两种值的方式，也了解了如何利用 PWM（脉冲宽度调制）来模拟模拟输出。然而，在处理电子设备和传感器时，我们常常需要读取模拟信号——不仅仅是 0V 或 3.3V，而是介于两者之间的任意值。

这就是模数转换器（ADC）的作用所在。ADC 是一种将连续模拟电压转换为微控制器能理解的数字信号的组件。简单来说，它能把电压值（比如 1.2 伏或 2.7 伏）转换成数值，让 ESP32-S3 能够通过代码进行处理。

ESP32-S3 拥有两个独立的模数转换器单元：**ADC1** 和 **ADC2**。这两个 ADC 可以读取 0 伏至 3.3 伏连续范围内的电压值，非常适合配合温度传感器、光敏传感器和电位计等产生变化信号的传感器使用。

每个模拟引脚都连接着内置 ADC，负责将输入的电压转换为数字值。默认情况下，ESP32-S3 的 ADC 采用 **12 位分辨率** ，这意味着它会将输入电压映射到 **0 至 4095** 的范围内：

*   0伏 → 0
*   3.3 V → 4095

并非所有 GPIO 引脚都连接到 ADC。在 ESP32-S3 上，只有特定引脚可用于读取模拟信号，总共 20 个支持模拟功能的引脚。

*   **ADC1 引脚（推荐）：GPIO1 至 GPIO10**
*   **ADC2 引脚：GPIO11 至 GPIO20**（这些引脚可能与 Wi-Fi 冲突，在此情况下可靠性较低）

![](./attachments/esp32s3_analogue.png)

### 模拟传感器

#### 电位器

电位器是一种具有三个端子的可变电阻器，可让我们手动改变输入到模拟引脚的电压。

*   两个外侧端子分别连接至 VCC（3.3V）和 GND。
*   中间的端子（滑动端）连接至支持 ADC 功能的 GPIO 引脚，例如 GPIO 1。

当我们旋转电位器时，中间引脚上的电压会从 0V 平滑变化至 3.3V。ESP32-S3 读取该电压值后，会输出一个介于 0 到 4095 之间的数值。

让我们构建一个通过电位器控制 LED 亮度的项目。本项目需要准备一个电位器、一个 LED 和一个 220Ω电阻。首先将电位器的两个外侧端子分别连接至 GND 和 3.3V，然后将中间端子连接至 GPIO 1。接着将 LED 的短脚（阴极）连接至 GND，长脚（阳极）通过 220Ω电阻连接至 GPIO 9。

![](./attachments/circuit6.png)

我们首先引入本项目所需的库文件。由于目标是控制 LED 亮度，我们将使用 PWM（脉冲宽度调制）。在 ESP32 上，PWM 功能由 LEDC 驱动管理，因此需要包含 LED 控制、FreeRTOS 定时和 ADC 功能的相关头文件。同时，由于需要读取模拟信号（例如来自电位器的信号），还需包含 ADC 单次触发驱动。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"
#include "esp_adc/adc_oneshot.h"
```

接下来，在 `app_main()` 函数中，我们配置 PWM 定时器。该定时器定义了 PWM 信号的行为——其频率和分辨率。本例中，我们选择 5 kHz 的频率（足以避免可见闪烁）和 12 位分辨率（可精细控制亮度，数值范围为 0 至 4095）。

接着我们配置一个 PWM 通道，并将其连接到连接 LED 的 GPIO 9 引脚上。初始占空比设为 0，即 LED 处于熄灭状态。

```c
void app_main() {
ledc_timer_config_t ledc_timer = {
.speed_mode = LEDC_LOW_SPEED_MODE, 
.timer_num = LEDC_TIMER_0,
.duty_resolution = LEDC_TIMER_12_BIT, 
.freq_hz = 5000,
.clk_cfg = LEDC_AUTO_CLK
    };
ledc_timer_config(&ledc_timer);

ledc_channel_config_t ledc_channel = {
.speed_mode = LEDC_LOW_SPEED_MODE, 
.channel = LEDC_CHANNEL_0,
.timer_sel = LEDC_TIMER_0, 
.gpio_num = 9,
.duty = 0, 
.hpoint = 0
    };
ledc_channel_config(&ledc_channel);
```

现在我们来配置 ADC（模数转换器）。ADC 允许 ESP32 读取模拟电压，并将其转换为程序可以处理的数字值。
我们使用单次触发模式，这意味着每次需要读取数据时手动触发转换，而不是在后台持续运行。

首先定义保存其配置的变量：

*   `adc_oneshot_unit_handle_t adc1_handle;` 存储 ADC 单元初始化后的句柄（引用），后续我们将用它来访问 ADC。
*   `adc_oneshot_unit_init_cfg_t init_config1 = { .unit_id = ADC_UNIT_1 };` 该结构体用于配置我们要使用的 ADC 单元。ESP32 拥有多个 ADC 单元，此处我们选择 ADC 单元 1。
*   `adc_oneshot_chan_cfg_t config = { ... };` 该结构体定义了特定 ADC 通道的行为方式，包含以下内容：
    *   **位宽** （`.bitwidth = ADC_BITWIDTH_DEFAULT`）：
        此项设置 ADC 的分辨率。默认情况下为 12 位，即输出值范围为 0 至 4095。
    *   **衰减** （`.atten = ADC_ATTEN_DB_12`）：
        这决定了可测量的电压范围。在 12 dB 衰减下，ADC 可读取约 0 V 至 3.3 V 的电压，这与典型的 ESP32 输入电平相匹配。

接下来，我们使用 `adc_oneshot_new_unit` 函数通过配置来创建（初始化）ADC 单元，该函数会设置 ADC 硬件，并将生成的句柄存储在 `adc1_handle` 中。

```c
adc_oneshot_new_unit(&init_config1, &adc1_handle);
```

最后，我们配置要读取的具体通道：

```c
adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_0, &config);
```

我们选择 `ADC_CHANNEL_0`（对应 GPIO 1），并应用之前定义的通道配置。

以下是完整程序：

```c

adc_oneshot_unit_handle_t adc1_handle;

adc_oneshot_unit_init_cfg_t init_config1 = { .unit_id = ADC_UNIT_1 };

adc_oneshot_chan_cfg_t config = {

.bitwidth = ADC_BITWIDTH_DEFAULT,

.atten = ADC_ATTEN_DB_12 

    };

adc_oneshot_new_unit(&init_config1, &adc1_handle);



adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_0, &config);


int potValue = 0;
```

最后，我们进入程序的主循环。该循环持续运行，并执行三个简单的步骤：

1.  使用 ADC 读取电位器的模拟值。
2.  直接将该值用作 PWM 占空比。
3.  更新 PWM 输出，使 LED 亮度随之变化。

由于 ADC 和 PWM 均使用 12 位范围（0–4095），我们可以直接将 ADC 值映射到 LED 亮度，无需额外缩放。

添加一小段延时，以使系统稳定并避免 CPU 过度占用。

```c

while (1) {

adc_oneshot_read(adc1_handle, ADC_CHANNEL_0, &potValue);

ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, potValue);

ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);

vTaskDelay(pdMS_TO_TICKS(10));

    }

}
```

#### 光敏电阻

LDR（光敏电阻），又称光敏电阻器，是一种特殊类型的电阻器，其阻值会根据所接收的光照强度而变化。

*   在**黑暗**中，它的电阻非常高。
*   在**强光**下，它的电阻变得非常低。

为了读取其变化的电阻值，我们将其与一个固定电阻（通常为 10kΩ）串联，构成分压电路。输出电压取自两个电阻之间的连接点，其数值取决于电阻的比值。

$$
V_{1}​=V_{in}​×\frac{R_1}{R_2​+R_2}​​ 
$$

 

$$
V_{2}​=V_{in}​×\frac{R_2}{R_2​+R_2}​​ 
$$

以下是电路描述：

![](./attachments/voltage_divider.png)

让我们构建一个自动夜灯，当环境变暗时它会点亮 LED。将光敏电阻（LDR）的一个引脚连接到 3.3V，另一个引脚连接到 GPIO 1（ADC1\_CH0）以及 10kΩ电阻的一端。将 10kΩ电阻的另一端连接到 GND。最后，通过 220Ω电阻将 LED 连接到 GPIO 9。

![](./attachments/circuit7.png)

由于我们只需要标准的数字输出来控制 LED，因此将使用标准的 GPIO 驱动（`driver/gpio.h`）。 现在开始编写程序，首先包含所需的库，然后将 GPIO 7 配置为数字输出以控制 LED。ADC 的设置与之前相同，用于从传感器读取模拟值。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_adc/adc_oneshot.h"

void app_main() {

gpio_reset_pin(7);
gpio_set_direction(7, GPIO_MODE_OUTPUT);


adc_oneshot_unit_handle_t adc1_handle;
adc_oneshot_unit_init_cfg_t init_config1 = { .unit_id = ADC_UNIT_1 };
adc_oneshot_new_unit(&init_config1, &adc1_handle);
adc_oneshot_chan_cfg_t config = {
.bitwidth = ADC_BITWIDTH_DEFAULT, .atten = ADC_ATTEN_DB_12
    };
adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_0, &config);

int ldrValue = 0;
```

在主循环中，我们持续读取光照强度，并通过一个简单的条件判断来控制 LED。若测量值高于设定的阈值（1600），则表示环境光线充足，此时点亮 LED；反之则熄灭 LED。这展示了如何利用模拟输入，根据条件触发数字动作。

```c
while (1) {
adc_oneshot_read(adc1_handle, ADC_CHANNEL_0, &ldrValue);


if (ldrValue > 1600) {
gpio_set_level(7, 1); 
} else {
gpio_set_level(7, 0);
        }   
vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

#### 火焰传感器

火焰传感器通过检测火焰发出的红外线（IR）来探测火情。由于火焰在可见光和红外线波长范围内都会产生辐射，该传感器专门用于检测红外光，通常覆盖 **760–1100 纳米**波段。与通常需要外接分压电阻的光敏电阻电路不同，火焰传感器模块已内置必要的电阻和信号调理电路。该模块通常有四个引脚：VCC、GND、DO（数字输出）和 AO（模拟输出）。

*   **DO 引脚：** 根据模块上电位器调节的内置比较器阈值，输出高电平或低电平信号，用于指示是否检测到火焰。
*   **AO 引脚：** 根据火焰强度输出连续电压。

模块中的传感元件是一个红外光电二极管。当火焰的红外辐射到达光电二极管时，会产生一个微弱的电信号。该信号的强度取决于火焰的强度。此信号被发送到模块的两个不同部分：

**DO 引脚**提供简单的高电平或低电平信号，用于指示是否检测到火焰。模块内部有一个比较器电路，将传感器信号与参考电压进行比较。该参考电压可通过内置电位器进行调节。顺时针旋转电位器可提高传感器灵敏度，使其能够检测更小或更弱的火焰；逆时针旋转则会降低灵敏度，使其仅能检测更强或更大的火焰。

*   当检测到火焰时，DO 引脚变为低电平。
*   当没有火焰时，DO 引脚输出高电平。

AO 引脚输出的电压会根据检测到的红外辐射强度连续变化。

*   如果火焰较弱或距离较远，输出电压较低。
*   如果火焰较强或距离较近，输出电压较高。

![](./attachments/flame_sensor.png)

让我们构建一个项目，当检测到火焰时蜂鸣器会发出警报。将传感器的 GND 和 VCC（3.3V）连接到 ESP32-S3。将 AO 引脚连接到 GPIO 1（ADC1\_CH0）。将有源蜂鸣器的正极引脚连接到 GPIO 7，其 GND 连接到系统地线。

![](attachments/circuit8.png)

现在让我们创建程序。首先包含必要的库，然后将 GPIO 7 配置为输出。ADC 的初始化方式与之前相同，用于读取火焰传感器的模拟值。

在主循环中，我们持续读取传感器数值。与光敏电阻不同，火焰传感器通常呈反向特性：较低的 ADC 值表示更强的红外辐射，这暗示着火焰的存在。因此，我们不是检查高数值，而是判断读数是否低于某个阈值（1200）。如果低于阈值，则激活蜂鸣器；否则保持关闭状态。这个简单的条件判断将系统转变为基本的火灾警报机制。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_adc/adc_oneshot.h"

void app_main() {
gpio_reset_pin(7);
gpio_set_direction(7, GPIO_MODE_OUTPUT);

adc_oneshot_unit_handle_t adc1_handle;
adc_oneshot_unit_init_cfg_t init_config1 = { .unit_id = ADC_UNIT_1 };
adc_oneshot_new_unit(&init_config1, &adc1_handle);
adc_oneshot_chan_cfg_t config = {
.bitwidth = ADC_BITWIDTH_DEFAULT, .atten = ADC_ATTEN_DB_12
    };
adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_0, &config);

int flameValue = 0;

while (1) {
adc_oneshot_read(adc1_handle, ADC_CHANNEL_0, &flameValue);

// ADC 值越低表示红外检测越强
if (flameValue < 1200) {
gpio_set_level(7, 1); // 蜂鸣器鸣响
} else {
gpio_set_level(7, 0); // 关闭蜂鸣器
        }

vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

#### 土壤湿度传感器

土壤湿度传感器用于测量土壤中的水分含量。它们主要分为两大类：

*   **电阻式传感器：** 采用两个裸露电极。随着湿度增加，导电性增强，电阻值降低。
*   **基于电容式：** 测量土壤的介电常数。不易受腐蚀影响。

![](./attachments/soil_sensor.png)

电容式传感器测量土壤的介电特性。因此，它受腐蚀影响较小，通常使用寿命更长。该传感器有三个引脚：VCC、GND 和 OUT，输出引脚会根据土壤湿度水平释放变化的电压：

*   当土壤非常湿润时，传感器输出较低的电压。
*   当土壤变得干燥时，传感器输出较高的电压。

基于电阻的传感器有两个裸露的电极，可插入土壤中。当土壤含水量较高时，其导电性增强，这意味着电极间的电阻减小。当土壤干燥时，电阻增大。这类传感器是最常见且成本最低的选择。然而，探头本身通常只有两个引脚，因此无法直接连接到 ESP32S3，需要借助一个外部模块来处理信号。该模块充当探头与 ESP32S3 之间的接口，通常有四个引脚：VCC、GND、AO（模拟输出）和 DO（数字输出）。AO 引脚提供与测量电阻值对应的连续模拟电压。

*   当土壤**非常湿润**时，电阻较低，传感器输出**较低的电压** 。
*   当土壤变得**更干燥**时，电阻较高，传感器输出**较高的电压** 。

DO 引脚根据预设阈值输出数字信号（高电平或低电平），该阈值可通过板载电位器调节；当测量值超过设定阈值时输出高电平，否则输出低电平。

让我们来构建一个自动植物浇水系统。我们需要传感器模块、一个 12V 水泵、一个 12V 电源以及一个 3.3V 继电器模块。

将湿度传感器的 AO 引脚连接到 GPIO 1。将继电器的 IN 引脚连接到 GPIO 8。将高压侧与 12V 水泵连接，通过继电器的常开（NO）和公共（COM）端子为水泵供电。

![](./attachments/circuit9.png)

现在我们来创建程序。首先导入所需的库，然后配置 ADC 以读取土壤湿度水平，并将 GPIO 8 设置为数字输出来控制水泵。

在循环中，系统持续检测土壤状况。当土壤变干时，水泵自动开启为植物浇水；一旦检测到足够湿度，水泵便关闭。这样，植物仅在需要时得到浇水，构建了一个直接响应环境的简单自动植物浇水系统。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_adc/adc_oneshot.h"

void app_main()
gpio_reset_pin(8);
gpio_set_direction(8, GPIO_MODE_OUTPUT);

adc_oneshot_unit_handle_t adc1_handle;
adc_oneshot_unit_init_cfg_t init_config1 = { .unit_id = ADC_UNIT_1 };
adc_oneshot_new_unit(&init_config1, &adc1_handle);
adc_oneshot_chan_cfg_t config = {
.bitwidth = ADC_BITWIDTH_DEFAULT, .atten = ADC_ATTEN_DB_12
    };
adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_0, &config);

int moistureValue = 0;

while (1) {
adc_oneshot_read(adc1_handle, ADC_CHANNEL_0, &moistureValue);


if (moistureValue > 2400) {
gpio_set_level(8, 1); 
} else {
gpio_set_level(8, 0);
        }
vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

## 电机

电机是机电一体化与机器人系统中至关重要的装置，它们构成了这些系统中执行器的主要部分。在典型的机电一体化系统中，传感器首先从外部环境采集信息，随后由控制器对这些信息进行处理。当决策完成后，控制器向执行器发送指令，电机便执行所需的运动或动作。
电机通过将电能转换为机械能来工作。这一过程基于磁场与电流之间的相互作用，从而在电机转子上产生运动。电机并非只有单一形式，而是存在多种不同类型。每种电机都根据所需的速度、精度、扭矩和控制要求，针对特定应用场景进行设计。

### 直流电机

标准的直流电机是我们最常见的一种电机类型。它能提供持续的旋转运动。
直流电机的工作原理基于电磁学（洛伦兹力原理）。电机内部有一个线圈（电枢），放置在永磁体的两极之间。当电流通过线圈时，会产生一个磁场，该磁场与永磁体相互作用，推动线圈旋转。一个名为"换向器"的机械部件会不断改变电流方向，使电机能够持续朝一个方向旋转。电机的转速取决于供电电压，而旋转方向则取决于施加在电机端子上的电压极性。

![](./attachments/dc_motor.png)

Esp32s3 只能提供约 20 mA 的电流，无法直接驱动大多数直流电机。为了解决这个问题，我们需要使用电机驱动器（如 L298N 或 L293D）或晶体管电路。

该驱动器包含一个"H 桥"电路的集成电路，这使我们能够安全地从外部电池提供大电流，并轻松反转电机方向。

![](./attachments/driver_dc.png)

让我们创建一个使用 L298N 电机驱动器控制直流电机的简单项目。首先，将电机连接到驱动器的输出端子。之后，将驱动器的电源引脚连接到电池和地线。然后，将驱动器的控制引脚连接到 Esp32S3。

*   **IN1 和 IN2** 控制旋转方向。我们将其中一个设为**高电平** ，另一个设为**低电平**来选择方向。我们将 IN1 连接到 GPIO 20，IN2 连接到 GPIO 19。
*   **ENA** 通过 PWM 控制电机速度，我们使用 GPIO 21 引脚来实现这一点。

![](./attachments/motor_circuit.png)

对于这个程序，我们首先引入必要的库。接着创建并配置一个 PWM 定时器，将其连接到 21 号引脚以控制信号。然后，将 19 号和 20 号引脚设置为输出引脚。一切准备就绪后，进入无限循环，交替改变这两个引脚的状态，同时调整 21 号引脚的 PWM 占空比。这样电机就能以半速向一个方向旋转一秒，再以四分之一速向相反方向旋转一秒。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"
#include "driver/gpio.h"

void app_main() {

    ledc_timer_config_t ledc_timer = {
        .speed_mode       = LEDC_LOW_SPEED_MODE,
        .timer_num        = LEDC_TIMER_0,
        .duty_resolution  = LEDC_TIMER_12_BIT, // 12-bit resolution
        .freq_hz          = 5000, // 5 kHz
        .clk_cfg          = LEDC_AUTO_CLK

    };

    ledc_timer_config(&ledc_timer);

    ledc_channel_config_t ledc_channel = {

        .speed_mode     = LEDC_LOW_SPEED_MODE,
        .channel        = LEDC_CHANNEL_0,
        .timer_sel      = LEDC_TIMER_0,
        .gpio_num       = 21,
        .duty           = 0,   
        .hpoint         = 0

    };
    ledc_channel_config(&ledc_channel);
    
    gpio_reset_pin(19);
    gpio_reset_pin(20);
    gpio_set_direction(19, GPIO_MODE_OUTPUT);
    gpio_set_direction(20, GPIO_MODE_OUTPUT);
    gpio_set_level(19, 0);
    gpio_set_level(20, 0);
     
    while (1) {
        gpio_set_level(19,0);
        gpio_set_level(20,1);
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 2047);
        ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
        vTaskDelay(pdMS_TO_TICKS(1000));

        gpio_set_level(19,1);
        gpio_set_level(20,0);
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 1023);
        ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
        vTaskDelay(pdMS_TO_TICKS(1000));

    }
}
```

### 舵机

舵机不像普通直流电机那样持续旋转，而是专为精确定位设计。标准航模舵机可旋转至特定角度，通常在0到180度之间。它们非常适合转向机构、机械臂和相机云台。

舵机是一种“闭环”系统。在其外壳内部，包含一个小型直流电机、用于减速并增加扭矩的齿轮箱，以及一个与输出轴相连的电位器（可变电阻）。当电机转动时，电位器随之旋转，持续测量当前角度并将该信息反馈给内部控制电路。控制电路接收 PWM 控制信号，并将期望位置与电位器测量的当前位置进行比较。若位置存在差异，电机便会旋转，直至达到正确角度。

![](./attachments/servo_motor.png)

舵机通常有三个引脚：GND、VCC 和一个控制引脚。控制引脚接收 PWM（脉冲宽度调制）信号，该信号决定了输出轴的位置。

控制信号是频率为 50 Hz 的 PWM 波形，对应周期为 20 毫秒。舵机的位置由该周期内脉冲的宽度来控制。

对于标准舵机：

*   一个 **1 毫秒** 的脉冲可将转轴旋转至约 **−90°**
*   一个 **1.5 毫秒** 的脉冲可将转轴设定为 **0°（中心位置）**
*   一个 **2 毫秒** 的脉冲可将转轴旋转至约 **+90°**

由于脉冲宽度与角度之间的关系近似线性，我们可以将其表示为：

$$
Pms = 1.5 + \theta \times \frac{0.5}{90}​ 
$$

其中：

*   Pms P 毫秒 ​ 是脉冲宽度，单位为毫秒
*   $\theta$ 是角度，单位为度（范围从−90°到+90°）

如果 PWM 分辨率为 4095 步（例如 12 位定时器），且总周期为 20 毫秒，则给定脉冲宽度对应的数字值可计算如下：

$$
N = \frac{4095}{20} \times P_{width} 
$$

其中：

*   N 是加载到 PWM 寄存器的数字计数值
*   Pwidth P 宽度 ​ 是脉冲宽度，单位为毫秒

让我们创建一个简单的项目，让舵机反复旋转至 **90°保持 1 秒** ，再旋转至 **−90°保持 1 秒** 。
首先，我们从搭建电路开始。连接：

*   **GND**舵机的 **GND** 连接到开发板的
*   **VCC**舵机的 **5V 引脚** 连接到 ESP32-S3 的
*   舵机的 **控制引脚** 连接到 **GPIO 21**

![](./attachments/servo_circuit.png)

现在，让我们创建程序。首先，我们包含 FreeRTOS 和 LEDC PWM 驱动所需的库。之后，我们将 GPIO 21 配置为生成 50 Hz 频率的 PWM 信号，这是控制舵机所必需的。

在主循环中，我们在两个占空比值之间交替切换，这两个值分别对应 **+90°** 和 **−90°**，每个位置保持 1 秒后再切换。
我们通过之前讨论的数学关系来获取占空比值。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"

void app_main() {

ledc_timer_config_t ledc_timer = {
.speed_mode       = LEDC_LOW_SPEED_MODE,
.timer_num        = LEDC_TIMER_0,
.duty_resolution  = LEDC_TIMER_12_BIT,
.freq_hz          = 50, // 5 kHz
.clk_cfg = LEDC_AUTO_CLK
    };

ledc_timer_config(&ledc_timer);

ledc_channel_config_t ledc_channel = {
.speed_mode     = LEDC_LOW_SPEED_MODE,
.channel        = LEDC_CHANNEL_0,
.timer_sel      = LEDC_TIMER_0,
.gpio_num       = 21,
.duty = 0,
.hpoint = 0
    };

ledc_channel_config(&ledc_channel);

while (1) {
ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 511);
ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
vTaskDelay(pdMS_TO_TICKS(1000));

ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 102);
ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 无刷直流电机（BLDC）

无刷电机，常被称为 BLDC 电机，其设计旨在像普通直流电机一样实现持续旋转，但摒弃了传统电机中的电刷结构。它采用电子换向替代机械换向，因此效率更高、噪音更低且使用寿命更长。

在标准直流电机中，永磁体位于外部（定子），而电磁线圈在内部（转子）旋转。BLDC 电机则颠覆了这一设计：线圈固定于外部保持静止，永磁体则在内部旋转。

由于没有机械换向器来切换线圈中的电流，它依赖电子速度控制器（ESC）来快速按精确顺序切换线圈的供电，从而带动磁转子旋转。这种切换会产生旋转磁场，使转子转动。
控制信号以 PWM 信号形式呈现，决定了这一顺序发生的速度。占空比越高，转速越快；占空比越低，转速越慢。

无刷直流电机通常有三根输入线，对应定子的三相（常标记为 A、B、C）。每根线连接电机内部的一组线圈。

![](./attachments/bldc.png)

让我们构建一个使用 ESP32s3 控制无刷电机的简单项目。该项目需要无刷直流电机、作为驱动器的电子速度控制器（ESC），以及能为电机提供足够电力的合适电池。

首先，将电调控制线中的 GND 和 VCC（5V）线连接到 ESP32S3 的 GND 和 5V 引脚。然后将电调的信号（控制）线连接到 GPIO 9，该引脚将发送用于控制电机转速的 PWM 信号。接下来，将电池连接到电调的电源输入端，以提供所需电力。最后，将无刷电机的三根线连接到电调的三根输出线上。这些线对应电机的三个相位，电调通过切换它们之间的电流来驱动电机。

![](./attachments/bldc_circuit.png)

与舵机类似，标准电调的控制信号是频率为 50 Hz 的 PWM 波形，对应周期为 20 ms。电机的转速由该周期内脉冲的宽度控制。

对于标准电调：

*   **1 ms** 的脉冲将电机设置为 **0%油门（停止状态）**
*   一个 **1.5 毫秒** 的脉冲将电机设置为 **50% 油门**
*   一个 **2 毫秒** 的脉冲将电机设置为 **100% 油门（最大速度）**

由于脉冲宽度与油门百分比之间的关系是线性的，我们可以将其表示为：

$$
P_{ms} = 1.0 + T \times \frac{1.0}{100} 
$$

其中：

*   Pms P ms ​ 是脉冲宽度，单位为毫秒
    
*   $T$ 是油门百分比（范围从0到100）
    

如果 PWM 分辨率为 4095 步（例如 12 位定时器），且总周期为 20 毫秒，则给定脉冲宽度对应的数字值可以像舵机一样精确计算：

$$
N = \frac{4095}{20} \times P_{width} 
$$

其中：

*   $N$ 是加载到 PWM 寄存器的数字计数值
*   Pwidth P 宽度 ​ 是脉冲宽度，单位为毫秒

现在，让我们利用上述信息来创建程序。首先，我们包含 FreeRTOS 和 LEDC PWM 驱动所需的库。之后，我们将 GPIO 9 配置为生成 50 Hz 频率的 PWM 信号，这是标准电调所要求的。

需要注意的是，电调出于安全原因需要"解锁序列"。当它们首次通电时，必须接收 0%油门信号（1 毫秒脉冲）持续几秒钟，之后才会响应速度指令。我们在 `app_main` 中的循环开始前处理这一过程。在主循环内部，我们在对应 50%油门和 0%油门的占空比值之间交替切换。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"

void app_main() {

    ledc_timer_config_t ledc_timer = {
        .speed_mode       = LEDC_LOW_SPEED_MODE,
        .timer_num        = LEDC_TIMER_0,
        .duty_resolution  = LEDC_TIMER_12_BIT, 
        .freq_hz          = 50, // 50 Hz
        .clk_cfg          = LEDC_AUTO_CLK
    };

    ledc_timer_config(&ledc_timer);

    ledc_channel_config_t ledc_channel = {
        .speed_mode     = LEDC_LOW_SPEED_MODE,
        .channel        = LEDC_CHANNEL_0,
        .timer_sel      = LEDC_TIMER_0,
        .gpio_num       = 9,
        .duty           = 0,  
        .hpoint         = 0
    };

    ledc_channel_config(&ledc_channel);

    // ESC Arming Sequence
    // Send a 1 ms pulse (~205 out of 4095) to arm the ESC
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 205);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
    vTaskDelay(pdMS_TO_TICKS(3000));

    while (1) {
        // 50% Throttle (1.5 ms pulse -> ~307 out of 4095)
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 307);
        ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
        vTaskDelay(pdMS_TO_TICKS(2000));

        // 0% Throttle / Stop (1 ms pulse -> ~205 out of 4095)
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 205);
        ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

### 步进电机

步进电机是一种以微小精确步进旋转而非连续旋转的电机，每个电脉冲会使电机转动一个固定角度。如果电机的步进角为 1.8 度，则恰好需要 200 步才能完成一整圈旋转。它们无需反馈即可实现精确定位，因此成为 3D 打印机和数控机床的核心部件。
步进电机包含一个中央带齿的齿轮状磁性转子，周围环绕着按"相"组织的多个电磁线圈（定子）。通过按特定顺序依次给这些线圈通电，转子齿会被磁力拉向通电线圈，从而使电机向前转动几分之一度。

![](./attachments/stepper.png)

步进电机与无刷电机类似，需要专用驱动器才能正常运行。驱动器作为控制系统（Esp32S3）与电机之间的"大脑"，接收通常以步进脉冲和方向指令形式呈现的控制信号，并将其转换为精确的电气信号序列，按正确顺序为电机线圈供电。驱动器控制每个时刻哪些线圈通电，确保转子以增量方式精确移动到目标位置。部分驱动器还提供限流、微步进（将步进细分为更小增量以实现更平滑运动）和过热保护等功能。常见的步进电机驱动器包括 A4988、DRV8825 和 ULN2003。

![](./attachments/stepper_driver.png)

A4988 驱动器共有 16 个引脚，通常排列为两排：一排用于电机电源，一排用于控制逻辑。各引脚功能如下：

*   **VMOT** 连接电机电源（通常为 8-35V），为步进电机线圈供电。
*   **GND（VMOT 旁）** 连接电机电源的负极。
*   **VDD** 连接至 Arduino 5 V（逻辑电源），为驱动器的内部逻辑电路供电。
*   **GND（逻辑侧）** 连接至 Arduino GND，必须与 Arduino 共地。
*   **A1 与 A2** 连接至步进电机的一个线圈。
*   **B1 与 B2** 连接至步进电机的另一个线圈。
*   **STEP** 脉冲输入。每个脉冲使电机移动**一步** 。
*   **DIR** 方向输入。高电平或低电平选择顺时针或逆时针旋转。
*   **ENABLE** 低电平启用驱动器，高电平禁用电机输出。若始终启用，可接低电平。
*   **RESET** 复位驱动器逻辑。若不使用，通常接高电平。
*   **睡眠**  使驱动器进入低功耗睡眠模式。连接高电平可唤醒驱动器。
*   **MS1、MS2、MS3** 控制微步进分辨率。

| MS1 | MS2 | MS3 | 微步进模式 |
| --- | --- | --- | --- |
| LOW | LOW | LOW | 整步 |
| HIGH | LOW | LOW | 半音阶 |
| LOW | HIGH | LOW | 四分之一音阶 |
| HIGH | HIGH | LOW | 八分之一音阶 |
| HIGH | HIGH | HIGH | 十六分之一音阶 |

让我们用一个 12V 步进电机和 A4988 步进电机驱动器来做一个简单示例。首先，将步进电机的线圈 1 连接到驱动器的 A1 和 B1 引脚，线圈 2 连接到 B1 和 B2 引脚。接着，将 VMOT 连接到 12V 电池的正极，GND 连接到电池的负极。
之后，开始将驱动器连接到 Esp32s3。将驱动器底部的 GND 引脚连接到 Esp32s3 的 GND，VDD 连接到 Esp32s3 的 5V，最后将 DIR 引脚和 STEP 引脚分别连接到 Esp32s3 的 19 号和 20 号引脚。

![](./attachments/stepper_circuit.png)

控制信号主要通过一个 `STEP` 脉冲和一个 `DIR`（方向）逻辑电平来表示。在 `STEP` 引脚上施加一个脉冲，电机就会移动一步，而脉冲的频率决定了速度——频率越高速度越快，频率越低速度越慢。

我们来编写一个简单的程序，让电机先朝一个方向移动 100 步，等待 3 秒，再朝相反方向移动 100 步。首先，引入所需的库。然后，将 GPIO 19 和 GPIO 20 配置为输出引脚。GPIO 19 用于控制电机的方向，而 GPIO 20 用于生成步进信号。

在程序中，我们首先将 GPIO 19 设置为低电平以选择一个方向，然后通过 GPIO 20 生成 100 个步进脉冲来驱动电机。短暂延迟后，我们将 GPIO 19 切换为高电平以反转电机方向，并再次发送 100 个步进脉冲。此序列不断重复，使电机在正转和反转之间交替运动。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"

void app_main() {

gpio_reset_pin(20);
gpio_reset_pin(19);
gpio_set_direction(20, GPIO_MODE_OUTPUT);
gpio_set_direction(19, GPIO_MODE_OUTPUT);

while (1) {
gpio_set_level(19, 0);
for (int i=0;i<100;i++){
gpio_set_level(20, 1);
vTaskDelay(pdMS_TO_TICKS(50));
gpio_set_level(20, 0);
vTaskDelay(pdMS_TO_TICKS(50));
        }
vTaskDelay(pdMS_TO_TICKS(3000));
gpio_set_level(19, 1);
for (int i=0;i<100;i++){
gpio_set_level(20, 1);
vTaskDelay(pdMS_TO_TICKS(50));
gpio_set_level(20, 0);
vTaskDelay(pdMS_TO_TICKS(50));
        }
vTaskDelay(pdMS_TO_TICKS(3000));
    }
}
```