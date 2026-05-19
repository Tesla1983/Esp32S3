## 学习目标

*   探索通信协议

## 通信协议

在构建大型嵌入式系统项目时，我们经常需要与许多外部设备进行交互，例如传感器、显示器、存储模块和执行器。然而，像 ESP32-S3 这样的微控制器上可用的 GPIO 引脚数量可能有限。

为了克服这一限制，我们使用通信协议。这些协议使微控制器能够通过共享通信线路高效地发送和接收数据，从而与多个设备进行通信，而无需为每个设备分配专用引脚。这样，我们就能在最小化 GPIO 使用的同时控制多个外设。

此外，通信协议使得多个 ESP32 开发板能够相互连接。通过增加可用 I/O 引脚数量并在不同开发板之间分配任务，这扩展了整体系统能力。因此，系统变得更加模块化、可扩展，并能够处理复杂的应用。

ESP32-S3 支持多种通信协议，包括：

*   UART（通用异步收发传输器）
*   SPI（串行外设接口）
*   I²C（集成电路间总线）
*   I²S / PDM（音频通信协议）
*   USB OTG（On-The-Go，即插即用技术）
*   TWAI（双线汽车接口，兼容 CAN 总线）
*   RMT（用于精确信号生成与接收的远程控制外设）

### UART 协议

UART（通用异步收发器）是嵌入式系统中最广泛使用的通信协议之一。它通过串行通信提供了一种简单高效的方式在设备间交换数据。UART 主要是一种点对点通信协议，即设计用于直接连接两个设备，一个作为发送器，另一个作为接收器。与基于总线的协议不同，UART 不支持多个设备在没有额外硬件的情况下共享同一条通信线路。

UART 通信使用两个主要引脚：

*   **TX（发送）** 从设备发送数据
*   **RX（接收）** 从另一台设备接收数据

该协议是**异步**的，这意味着设备之间无需共享时钟信号。相反，两台设备必须约定一个共同的通信速度，即**波特率** 。数据通过 TX 线逐位发送，并通过 RX 线接收，通常以起始位和停止位作为帧结构，以确保正确同步。

#### UART 连接

要使用 UART 通信将两个设备连接在一起，我们需要将第一个设备的发送端（TX）连接到第二个设备的接收端（RX）。同样，第一个设备的接收端（RX）连接到第二个设备的发送端（TX）。换句话说，TX 连接 RX，RX 连接 TX。此外，我们还需要将两个设备的 GND 连接起来，使它们使用相同的电压参考基准。

![](./attachments/uart.png)

#### UART 工作原理

要使用 UART 协议，首先需要确保两个连接的设备能够相互理解。这需要通过配置四个主要的通信设置来实现：

*   **波特率** ：表示通信速度，以比特每秒（bps）为单位。它决定了比特传输的速度。两个设备必须使用相同的波特率，才能正确读取信号的时序。
*   **数据位长度** ：定义每个帧中用于表示实际数据的位数。最常见的值为 8 位，但根据应用场景也可设置为 5、6、7 或 9 位。
*   **校验位** ：用于基本的错误检测。校验方式可分为：
    *   **偶校验** ：当设置为偶校验时，若数据中 1 的总数为奇数，则校验位将被设为 1。
    *   **奇校验** ：当设置为奇校验时，若数据中 1 的总数为偶数，则校验位将被设为 1。
*   **停止位** ：表示数据帧的结束，同时为接收器准备下一次传输提供时间。常见配置使用 1 位或 2 位停止位。

将两个设备配置为相同的 UART 设置后，即可开始数据传输。

UART 传输线在空闲时保持高电平状态。要开始通信，发送器将 TX 线从高电平切换为低电平，这称为起始位，标志着数据帧的开始。

起始位之后，发送器按照配置的波特率时序逐位发送数据位。数据位通常从最低有效位（LSB）开始传输。

接下来，如果启用了奇偶校验，发送器会发送奇偶校验位用于错误检查。

最后，发送器通过将 TX 线重新设置为**高电平**来发送停止位。这表示当前数据帧的传输已结束。

![](./attachments/uart_signal.png)

#### 使用 UART 协议的 ESP32

ESP32-S3 拥有三个硬件 UART 接口：UART0、UART1 和 UART2。默认情况下，常用的引脚分配如下：

*   **UART0** 通常用于串行通信、固件烧录和调试
    *   **TX：** GPIO 43
    *   **RX：** GPIO 44
*   **UART1**
    *   **TX：** GPIO 17
    *   **RX：** GPIO 18

![](./attachments/UART_PINS.png)

得益于 GPIO 矩阵，ESP32-S3 上的 UART 引脚具有高度灵活性，因此如有需要，可在软件中将其重新映射到其他兼容的 GPIO 引脚。

#### UART 接口编程

现在我们已经了解了 UART 协议的工作原理，接下来让我们构建一个简单的项目，通过 UART 通信连接两块 ESP32-S3 开发板。在这个项目中，第一块 ESP32-S3 将连接一个按钮。当按钮被按下时，该开发板会通过 UART 协议向第二块 ESP32-S3 发送"ON"或"OFF"指令。第二块 ESP32-S3 随后读取接收到的数据，并相应地控制 LED 的亮灭。

让我们先搭建电路。为了实现两块 ESP32-S3 开发板之间的 UART 通信，我们将使用 GPIO 17 和 GPIO 18 作为 TX 和 RX 引脚。在第一块 ESP32-S3 上，GPIO 1 将用于读取按钮的状态；而在第二块 ESP32-S3 上，GPIO 1 将用于控制 LED。

![](./attachments/circuit1.png)

完成电路搭建后，我们可以开始为第一块 ESP32-S3 编写程序。首先导入 UART 通信所需的库文件，然后定义 UART 端口、TX 和 RX 引脚以及缓冲区大小的常量。

```C
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"  
#include "driver/uart.h"  
#include "driver/gpio.h"
#include <stdbool.h>

#define UART_PORT UART_NUM_1  
#define UART_TX_PIN 17  
#define UART_RX_PIN 18  
#define BUF_SIZE 1024

```

接下来，在 `app_main()` 函数中，我们将 GPIO 1 配置为连接到按钮的输入引脚。同时启用内部下拉电阻，确保按钮未按下时引脚保持低电平状态。这有助于防止因引脚悬空或噪声信号导致的错误读数。随后，我们创建一个名为 `ledState` 的布尔变量，并将其初始化为 `false`。该变量将用于存储后续发送至第二个 ESP32-S3 的当前状态。

```c
void app_main(void) {  
gpio_set_direction(1, GPIO_MODE_INPUT);
gpio_set_pull_mode(1,GPIO_PULLDOWN_ONLY);
bool ledState = false;
```

配置好 GPIO 引脚后，现在需要配置 UART 通信设置。首先，创建一个 `uart_config_t` 结构体，并设置 UART 参数，如波特率、数据位、校验位和停止位。在本项目中，我们使用 115200 的波特率、8 个数据位、无校验位和 1 个停止位，这是最常见的 UART 配置之一。同时，我们禁用硬件流控制，并使用默认时钟源。

```c
uart_config_t uart_config = {  
.baud_rate = 115200,
.data_bits = UART_DATA_8_BITS,  
.parity = UART_PARITY_DISABLE,  
.stop_bits = UART_STOP_BITS_1,  
.flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
.source_clk = UART_SCLK_DEFAULT,  
	};  
```

接下来，我们使用 `uart_driver_install()` 函数安装 UART 驱动。该函数会初始化 UART 驱动并分配 UART 通信所需的资源，此函数需要五个参数。
第一个参数 `UART_PORT` 指定了我们要使用的 UART 外设。之前我们将其定义为 `UART_NUM_1`，这意味着 ESP32-S3 上的 UART1 将用于通信。
第二个参数 `BUF_SIZE` 定义了接收缓冲区的大小（以字节为单位）。在我们的项目中，将其设置为 `1024`，这意味着 UART 驱动程序最多可以存储 1024 字节的传入数据，然后才会进行处理。
第三个参数表示发送缓冲区的大小。在本示例中，我们将其设置为 `0`，这会禁用专用的 TX 缓冲区，因为我们的应用程序很简单，不需要缓冲传输。
第四个参数定义了 UART 事件队列的大小。UART 驱动程序可以生成诸如接收数据或检测错误等事件，这些事件可以存储在队列中。由于本项目未使用 UART 事件，我们将此值设置为 `0`。
第五个参数是指向事件队列句柄的指针。通常情况下，如果启用了 UART 事件队列，此参数将存储所创建队列的地址。由于我们未使用事件，因此传入 `NULL`。
最后一个参数指定了 UART 驱动程序使用的中断分配标志。这些标志可用于自定义 ESP32-S3 系统中中断的分配方式。

安装驱动程序后，我们通过传递 UART 端口和配置结构体的地址，使用 `uart_param_config()` 函数将配置设置应用到选定的 UART 端口。

```c
uart_driver_install(UART_PORT, BUF_SIZE, 0, 0, NULL, 0);  
uart_param_config(UART_PORT, &uart_config);
```

最后，我们使用 `uart_set_pin()` 函数分配用于 UART 通信的 GPIO 引脚。在本例中，GPIO 17 被配置为 TX 引脚，GPIO 18 被配置为 RX 引脚。最后两个参数设置为 `UART_PIN_NO_CHANGE`，因为我们未使用 RTS 或 CTS 硬件流控制引脚。

```c
uart_set_pin(UART_PORT, UART_TX_PIN, UART_RX_PIN,  
UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);  
```

完成 UART 配置后，我们现在可以开始将数据从第一个 ESP32-S3 发送到第二个。在无限 `while` 循环中，程序通过 `gpio_get_level()` 持续读取 GPIO 1 的状态，以检测按钮是否被按下。

当按钮被按下时，我们切换 `ledState` 变量的值，接着通过 UART 使用 `uart_write_bytes()` 函数发送相应的消息。如果 `ledState` 为 `true`，ESP32-S3 发送 `"ON\r\n"`；否则发送 `"OFF\r\n"`。最后一个参数指定要发送的字节数，其中 `"ON\r\n"` 为 4 字节，`"OFF\r\n"` 为 5 字节。

最后，我们使用 `vTaskDelay()` 暂停任务 1 秒，然后重复循环。

```c

while (1) {  

if(gpio_get_level(1)){

ledState ^= 1

  }

uart_write_bytes(UART_PORT, ledState ? "ON\r\n" : "OFF\r\n", ledState ? 4 : 5);


vTaskDelay(pdMS_TO_TICKS(1000));

}  

}
```

我们已完成第一个 ESP32-S3 的程序。接下来，我们将为第二个 ESP32-S3 编写程序，该程序将接收 UART 数据并根据接收到的消息控制 LED。

与第一个程序类似，我们首先包含 FreeRTOS、GPIO 控制和 UART 通信所需的库。然后，我们定义 UART 端口、TX 和 RX 引脚以及 UART 缓冲区大小。

在 `app_main()` 函数中，我们将 GPIO 1 配置为输出引脚，因为它连接到了 LED。之后，我们使用与第一个 ESP32-S3 相同的配置来设置 UART 参数，以确保两块开发板能够正常通信。

```c
#include "freertos/FreeRTOS.h"  
#include "freertos/task.h"  
#include "driver/gpio.h"
#include "driver/uart.h"  

#define UART_PORT UART_NUM_1  
#define UART_TX_PIN 17  
#define UART_RX_PIN 18
#define BUF_SIZE 1024  


void app_main(void)  {  

gpio_set_direction(1, GPIO_MODE_OUTPUT);
uart_config_t uart_config = {
.baud_rate = 115200,  
.data_bits = UART_DATA_8_BITS,  
.parity = UART_PARITY_DISABLE,  
.stop_bits = UART_STOP_BITS_1,
.flow_ctrl = UART_HW_FLOWCTRL_DISABLE,  
.source_clk = UART_SCLK_DEFAULT,  
	};  

uart_driver_install(UART_PORT, BUF_SIZE, 0, 0, NULL, 0);
uart_param_config(UART_PORT, &uart_config);
uart_set_pin(UART_PORT, UART_TX_PIN, UART_RX_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
```

最后，我们创建一个大小为 `BUF_SIZE` 的数据数组，用于存储接收到的 UART 数据。在无限 `while` 循环中，我们使用 `uart_read_bytes()` 函数读取从第一个 ESP32-S3 发送过来的数据。

该函数接收四个参数：要读取的 UART 端口、用于存储接收数据的数组、要读取的字节数，以及超时值，该值指定 ESP32-S3 在停止读取操作前应等待传入数据的时间。

收到数据后，我们将接收到的消息与 `"ON\r\n"` 和 `"OFF\r\n"` 进行比较。如果接收到的数据是 `"ON\r\n"`，我们将 GPIO 1 设置为高电平以点亮 LED。否则，如果接收到的数据是 `"OFF\r\n"`，我们将 GPIO 1 设置为低电平以熄灭 LED。

```C
uint8_t data[BUF_SIZE];  
while (1) {  
uart_read_bytes( UART_PORT, data, BUF_SIZE, pdMS_TO_TICKS(1000) );

if (data[0] == 'O' && data[1] == 'N'){
gpio_set_level(1,1);
}else if(data[0] == 'O' && data[1] == 'F' && data[2] == 'F'){
gpio_set_level(1,0);
		}
	}  
}
```

### I2C 协议

UART 协议工作完美，但有一个小限制：它是一种点对点通信协议。这意味着它通常只能连接两个设备。如果要添加更多设备，就需要更多引脚和独立的串行端口。随着传感器和模块数量的增加，可用引脚很快就会被耗尽。为了解决这个问题，一种新协议应运而生：I2C。

**I2C**，即集成电路间总线（Inter-Integrated Circuit），是一种串行通信协议，仅需两根线即可实现多个设备之间的通信。
I2C 协议中使用的两条通信线路是：

*   **SDA（串行数据线）** 用于设备间传输数据
*   **SCL（串行时钟线）** 用于通过时钟信号同步通信

#### I2C 连接

该协议是同步的，这意味着通信通过共享的时钟信号进行协调。在 I²C 网络中，一个设备作为主设备控制通信，而一个或多个从设备响应主设备。每个从设备都有唯一的地址，从而允许多个设备在同一总线上运行，无需独立的通信线路。

*   主设备控制通信并在 SCL 线上生成时钟信号。
*   从设备在被寻址时响应主设备。

![](./attachments/I2C.png)

在 I²C 总线上连接设备时，SDA 和 SCL 两条线都需要连接上拉电阻至电源电压（VCC）。这是因为 I²C 设备采用开漏/开集电极配置，意味着设备只能将线路拉低，而无法直接驱动它们至高电平。上拉电阻确保当没有设备主动拉低线路时，线路能恢复至高电平状态。

#### I²C 工作原理

要使用 I²C 协议，所有连接的设备必须通过两条线共享同一通信总线：SDA 用于数据传输，SCL 用于时钟信号。通信由主设备控制，其他设备作为从设备运行。

在通信开始前，每个从设备必须拥有唯一的地址，以便主设备能够识别并与其单独通信。

I²C 通信遵循以下主要原则：

*   **主从通信** ：主设备控制整个通信过程。它在 SCL 线上生成时钟信号，并决定通信何时开始和结束。
*   **寻址** ：连接到总线的每个从设备都有一个唯一地址，通常为 7 位或 10 位长度。主设备在交换数据前发送该地址以选择目标从设备。
*   **同步通信** ：数据传输通过 SCL 时钟线进行同步。与 UART 不同，I²C 无需波特率匹配，因为主设备直接提供时钟时序。
*   **ACK/NACK 响应** ：每传输一个字节后，接收设备会发送一个应答位：
    *   **ACK（应答）**：表示数据已成功接收。
    *   **NACK（非应答）**：表示接收方未接受数据，或通信应终止。

当总线空闲时，由于上拉电阻的作用，SDA 和 SCL 两条线都保持高电平状态。

为了开始通信，主设备在 SCL 线保持高电平的同时将 SDA 线拉低，从而产生一个起始条件。这向所有连接的设备发出信号，表示传输即将开始。

接下来，主设备发送目标从设备的地址，随后是一个读/写（R/W）位：

*   **0** → 写操作（主设备向从设备发送数据）
*   **1** → 读取操作（主设备从从设备接收数据）

从设备在接收到地址后，会以应答位（ACK）作为响应。

随后，主设备与从设备通过 SDA 线逐字节交换数据。每个字节包含 8 位数据，并与 SCL 线上的时钟脉冲保持同步。每完成一个字节的传输，接收方会发送一个应答位（ACK）以确认成功接收。

当通信结束时，主设备通过将 SDA 线从低电平切换为高电平（同时 SCL 线保持高电平）来生成停止条件。此举会释放总线，使其恢复至空闲状态。

![](./attachments/i2c_signal.png)

#### 使用 UART 协议的 Esp32

ESP32-S3 包含两个硬件 I²C 控制器，负责管理 I²C 总线上的通信。这些控制器自动处理时钟信号的生成、数据的发送与接收、START 和 STOP 条件的检测，以及 ACK/NACK 响应的管理。

每个 I²C 控制器可以独立运行，作为以下任一角色：

*   **主设备** ：控制通信并生成时钟信号。
*   **从机** ：响应来自主设备的请求。

拥有两个独立的 I²C 控制器，使得 ESP32-S3 能够同时与多个 I²C 总线通信，任何 GPIO 引脚均可配置为 SDA 和 SCL 引脚。

#### I²C 接口编程

现在我们已经了解了 I²C 协议的工作原理，接下来让我们使用 I²C 通信替代 UART 来重建之前的示例。在此项目中，两块 ESP32-S3 开发板将通过 I²C 总线进行通信。第一块 ESP32-S3 将作为 I²C 主机，而第二块 ESP32-S3 将作为 I²C 从机。

和之前一样，第一个 ESP32-S3 上连接了一个按钮。当按下按钮时，主控板通过 I²C 协议向第二个 ESP32-S3 发送 `"ON"` 或 `"OFF"` 指令。从属 ESP32-S3 接收到消息后，会相应地打开或关闭 LED 灯。

要构建电路，两个 ESP32-S3 开发板必须共享 SDA 和 SCL 线路。在本示例中，我们将使用：

*   **GPIO 8** 作为 SDA
*   **GPIO 9** 作为 SCL

我们还在 SDA、SCL 与 3.3V 之间连接了上拉电阻。
最后，将主 ESP32-S3 的 GPIO 1 引脚连接到按钮，从 ESP32-S3 的 GPIO 1 引脚连接到 LED。

![](./attachments/i2c_circuit.png)

电路搭建完成后，我们可以开始为第一个 ESP32-S3 编写程序，该设备将作为 I²C 主设备运行。

首先，引入所需库并定义 I2C 配置常量。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/i2c_master.h"
#include "driver/gpio.h"
#include <stdbool.h>

#define I2C_MASTER_SCL_IO 9
#define I2C_MASTER_SDA_IO 8
#define I2C_PORT I2C_NUM_0
#define SLAVE_ADDR 0x28
```

在 `app_main()` 函数内部，我们首先将 GPIO 1 配置为连接到按键的输入引脚。同时启用内部下拉电阻以避免信号浮动。接着，我们创建一个名为 `ledState` 的布尔变量，用于存储当前 LED 状态，该状态后续将发送至从机 ESP32-S3。

```c
void app_main(void) {
gpio_set_direction(1, GPIO_MODE_INPUT);
gpio_set_pull_mode(1, GPIO_PULLDOWN_ONLY);
bool ledState = false;
```

接下来，我们使用 `i2c_master_bus_config_t` 结构体配置 I²C 主总线。在该结构体中，我们选择 I²C 控制器，分配 SDA 和 SCL 引脚，启用内部上拉电阻，并设置主设备忽略 I2C 线路上持续时间少于 7 个时钟周期的任何脉冲。

```c
i2c_master_bus_config_t i2c_mst_config = {
.clk_source = I2C_CLK_SRC_DEFAULT
.i2c_port = I2C_PORT
.scl_io_num = I2C_MASTER_SCL_IO
.sda_io_num = I2C_MASTER_SDA_IO
.glitch_ignore_cnt = 7,
.flags.enable_internal_pullup = true,
        };
```

创建配置结构后，我们使用 `i2c_new_master_bus()` 函数安装 I²C 主总线。该函数会初始化选定的 I²C 控制器，并返回一个代表 I²C 总线的句柄。

```c
i2c_master_bus_handle_t bus_handle;    
i2c_new_master_bus(&i2c_mst_config, &bus_handle);
```

接下来，我们使用 `i2c_device_config_t` 结构体配置从设备。在此，我们指定从设备地址、设置地址长度，并将 I²C 时钟频率设为 100 kHz。

```c
i2c_device_config_t dev_cfg = {
.dev_addr_length = I2C_ADDR_BIT_LEN_7,        
.device_address = SLAVE_ADDR,        
.scl_speed_hz = 100000,
    };
```

然后，我们使用 `i2c_master_bus_add_device()` 函数将从设备添加到 I²C 主总线上。

```c
i2c_master_dev_handle_t dev_handle;    
i2c_master_bus_add_device(bus_handle, &dev_cfg, &dev_handle);
```

完成 I²C 配置后，我们现在可以开始向第二个 ESP32-S3 发送数据。

在无限的 `while` 循环中，程序使用 `gpio_get_level()` 持续检测按键是否被按下。当按键被按下时，我们切换 `ledState` 的值。

接着，我们使用 `i2c_master_transmit()` 函数向从设备 ESP32-S3 发送 `"ON"` 或 `"OFF"`。该函数的参数包括：I2C 主设备的句柄、指向将通过 I2C 总线发送的数据字节的指针、从写入缓冲区传输的字节数，以及传输超时时间（以毫秒为单位）。我们将超时时间设置为 -1，这意味着它会阻塞并等待，直到数据传输完成。

```c
while (1) {        
if (gpio_get_level(1)) {
ledState ^= 1;        
		}        
if (ledState) {            
i2c_master_transmit(dev_handle, (uint8_t *)"ON", 2, -1);
} else {            
i2c_master_transmit(dev_handle, (uint8_t *)"OFF", 3, -1);        
		}        
vTaskDelay(pdMS_TO_TICKS(1000));
		}
	}
```

现在我们已经完成了第一个 ESP32-S3 的程序。接下来，我们将为第二个 ESP32-S3 创建程序，该设备将作为 I²C 从机运行，并根据接收到的数据控制 LED。

与之前一样，我们首先包含所需的库并定义 I²C 配置参数。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/i2c_slave.h"
#include "driver/gpio.h"

#define I2C_SLAVE_SCL_IO 9
#define I2C_SLAVE_SDA_IO 8
#define I2C_PORT I2C_NUM_0
#define SLAVE_ADDR 0x28
```

之后，我们创建一个新的结构体类型，并用它来定义一个上下文变量。该变量将用于读取和存储主 ESP32-S3 通过 I²C 发送的数据。

```c
typedef struct {
uint8_t data[10];
} i2c_slave_context_t;

i2c_slave_context_t context;
```

I2C 从设备不像 I2C 主设备那样主动，主设备知道何时发送数据、何时接收数据。I2C 从设备在大多数情况下非常被动，这意味着 I2C 从设备发送和接收数据的能力在很大程度上取决于主设备的动作。因此，我们在驱动中实现了回调函数来处理来自 I2C 主设备的写入请求。

回调函数应包含三个参数：I2C 从设备的句柄、I2C 接收事件数据（由驱动提供，保存主设备发送的数据），以及表示附加数据的参数。

在我们的案例中，对于回调函数，我们只需读取主设备 Esp32s3 发送的数据，并修改 context.data 以存储该消息。

```c
static bool i2c_slave_receive_cb(i2c_slave_dev_handle_t i2c_slave, const i2c_slave_rx_done_event_data_t *evt_data, void *arg){

context.data[0] = evt_data->buffer[0];
context.data[1] = evt_data->buffer[1];
context.data[2] = evt_data->buffer[2];
返回 0；
}
```

在 `app_main()` 函数内部，我们首先将 GPIO 1 配置为输出引脚，因为它连接到了 LED。

```c
void app_main(void) {
gpio_set_direction(1, GPIO_MODE_OUTPUT);
```

接下来，我们使用 `i2c_slave_config_t` 结构体配置 I²C 从设备。在此，我们选择 I²C 控制器，分配 SDA 和 SCL 引脚，配置从设备地址，并定义软件缓冲区大小。

```c
i2c_slave_config_t i2c_slv_config = {        
.i2c_port = I2C_PORT,
.clk_source = I2C_CLK_SRC_DEFAULT,        
.scl_io_num = I2C_SLAVE_SCL_IO,        
.sda_io_num = I2C_SLAVE_SDA_IO,        
.slave_addr = SLAVE_ADDR,
.send_buf_depth = 100,
.receive_buf_depth = 100,
.flags.enable_internal_pullup = true,
    };
```

配置完从机设置后，我们使用 `i2c_new_slave_device()` 函数初始化 I²C 从设备。

```c
i2c_slave_dev_handle_t slave_handle;    
i2c_new_slave_device(&i2c_slv_config, &slave_handle);
```

我们创建事件回调配置结构体，并将其注册到 I²C 从机驱动中。这样，当 ESP32-S3 接收到来自主设备的数据时，即可执行回调函数。

```c
i2c_slave_event_callbacks_t cbs = {
.on_receive = i2c_slave_receive_cb,
    };
i2c_slave_register_event_callbacks(slave_handle, &cbs, &context);
```

最后，我们创建一个缓冲区来存储接收到的数据。在无限 `while` 循环中，从设备持续检查接收到的数据，并相应地更新 LED 状态。

```c

while (1) {        

if (context.data[0] == 'O' && context.data[1] == 'N') {            

gpio_set_level(1, 1);

} else if (context.data[0] == 'O' && context.data[1] == 'F' && context.data[2] == 'F') {

gpio_set_level(1, 0);

		}       

vTaskDelay(pdMS_TO_TICKS(100));

	}

}
```

### SPI 协议

I²C 协议解决了仅用两根线连接多个设备的问题，但它有一个小局限：它使用单条数据线进行发送和接收，以半双工模式运行，无法同时发送和接收数据。与多个设备共享总线需要寻址，这会增加开销。为了实现更快、更简单的通信，另一种协议被广泛使用：**SPI**。

**SPI**，全称为串行外设接口，是一种同步串行通信协议，允许设备以全双工模式进行高速通信。

与仅使用两根线不同，SPI 通常采用四条通信线路：

*   **MOSI（主设备输出从设备输入）** 用于将数据从主设备发送至从设备
*   **MISO（主设备输入从设备输出）** 用于将数据从从设备发送至主设备
*   **SCLK（串行时钟）** 用于通过时钟信号同步通信
*   **CS（片选）/ SS（从机选择）** 由主机用于选择特定的从设备

#### SPI 连接

该协议是同步的，意味着通信通过共享的时钟信号进行协调。在 SPI 网络中，一个设备作为主机控制通信，而一个或多个从设备响应主机。与 I2C 不同，SPI 不使用地址。相反，主机为每个从设备使用专用的片选（CS）线来选择要与哪个设备通信。

*   主设备控制通信，在 SCLK 线上生成时钟信号，并管理 CS 线。
*   从设备监听 MOSI 线，当各自的 CS 线被激活时，通过 MISO 线回传数据。

在 SPI 总线上连接设备时，MOSI、MISO 和 SCLK 线由所有设备共享。然而，每个从设备都需要独立的 CS 线连接到主设备。SPI 使用推挽驱动方式，即线路会被主动驱动为高电平和低电平，这使得无需像 I2C 那样使用外部上拉电阻即可实现更高速度。

![](./attachments/spi_linking.png)

#### SPI 工作原理

使用 SPI 协议时，所有连接的设备必须共享公共总线线路（MOSI、MISO、SCLK），且主设备需为每个从设备配备独立的 CS 线路。通信始终由主设备发起并控制。

当总线空闲时，SCLK 线路处于静止状态（根据配置模式保持高电平或低电平），CS 线路保持高电平以使从设备处于非活动状态。

要开始通信，主设备将目标从设备的 CS 线路拉至低电平。这向特定设备发出信号，表明传输即将开始。

随后，主设备开始在 SCLK 线路上生成时钟脉冲。每个时钟周期，主设备通过 MOSI 线路推送一位数据，同时从设备通过 MISO 线路推送一位数据。由于移位寄存器架构，数据始终处于交换状态。

当通信结束时，主设备停止时钟信号并将 CS 线拉回高电平。这将释放从设备，使总线恢复到空闲状态。

![](./attachments/spi_signal.png)

#### 使用 SPI 协议的 ESP32

ESP32-S3 包含四个 SPI 控制器。

*   **SPI0** 和 **SPI1** 主要用于与外部闪存和 PSRAM 进行内部通信。
*   **SPI2** 和 **SPI3** 可用于通用应用。

每个 SPI 控制器可独立运行于以下两种模式之一：

*   **主模式** ：ESP32-S3 控制通信，生成串行时钟（**SCLK**），并管理片选（**CS**）信号。
*   **从模式** ：ESP32-S3 响应由外部 SPI 主机发起的事务。

SPI2 和 SPI3 可通过 ESP32-S3 的 GPIO 矩阵路由至几乎任意可用的 GPIO 引脚，为 MOSI、MISO、SCLK 和 CS 等 SPI 信号提供灵活的引脚分配方案。

#### SPI 接口编程

在理解 SPI 协议的工作原理后，让我们通过 SPI 通信来构建之前的示例。本项目中，两块 ESP32-S3 开发板将通过 SPI 总线进行通信。第一块 ESP32-S3 将作为 SPI 主机，第二块 ESP32-S3 则作为 SPI 从机。

与之前相同，第一块 ESP32-S3 连接了一个按键。当按键按下时，主控板通过 SPI 协议向第二块 ESP32-S3 发送 `"ON"` 或 `"OFF"` 指令。从机 ESP32-S3 接收到消息后，将相应地控制 LED 的亮灭。

要构建该电路，两块 ESP32-S3 开发板必须共用 MOSI、MISO、SCLK 和 CS 线路。本示例中，我们将使用：

*   **GPIO 11** 作为 MOSI
*   **GPIO 12** 作为 MISO
*   **GPIO 13** 作为 SCLK
*   **GPIO 10** 作为片选（CS）

最后，我们将主设备 ESP32-S3 的 GPIO 1 引脚连接到按钮，从设备 ESP32-S3 的 GPIO 1 引脚连接到 LED。

![](./attachments/circuit_spi.png)

搭建好电路后，我们可以开始为第一个 ESP32-S3 编写程序，它将作为 SPI 主设备运行。

首先，我们包含所需的库并定义 SPI 配置常量。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/spi_master.h"
#include "driver/gpio.h"
#include <string.h>
#include <stdbool.h>

#define SPI_MASTER_MOSI_IO 11
#define SPI_MASTER_MISO_IO 12
#define SPI_MASTER_SCLK_IO 13
#define SPI_MASTER_CS_IO 10
#define SPI_HOST SPI2_HOST
```

在 `app_main()` 函数内部，我们首先将 GPIO 1 配置为连接到按键的输入引脚。同时启用内部下拉电阻以避免信号浮动。接着，创建一个名为 `ledState` 的布尔变量，用于存储稍后将发送到从机 ESP32-S3 的当前 LED 状态。

```c
void app_main(void) {
gpio_set_direction(1, GPIO_MODE_INPUT);
gpio_set_pull_mode(1, GPIO_PULLDOWN_ONLY);    
bool ledState = false;
```

接下来，我们使用 `spi_bus_config_t` 结构体配置 SPI 总线。在该结构体中，我们分配 MOSI、MISO 和 SCLK 引脚，并定义最大传输大小。

```c
spi_bus_config_t buscfg = {
.mosi_io_num = SPI_MASTER_MOSI_IO,
.miso_io_num = SPI_MASTER_MISO_IO,
.sclk_io_num = SPI_MASTER_SCLK_IO,
.quadwp_io_num = -1,
.quadhd_io_num = -1,
.max_transfer_sz = 32,
    };
```

创建总线配置后，我们使用 `spi_bus_initialize()` 函数安装 SPI 主总线。

```c
    spi_bus_initialize(SPI_HOST, &buscfg, SPI_DMA_CH_AUTO);
```

接下来，我们使用 `spi_device_interface_config_t` 结构体配置从设备参数。在此指定时钟速度（1 MHz）`clock_speed_hz`、SPI 模式 `mode`，并分配 CS 引脚 `spics_io_num`。最后设置事务队列大小，该参数决定了可同时进行的事务数量。

然后，我们使用 `spi_bus_add_device()` 将从设备添加到 SPI 总线上。

```c
spi_device_interface_config_t devcfg = {
.clock_speed_hz = 1000000,           
.mode = 0,
.spics_io_num = SPI_MASTER_CS_IO,    
.queue_size = 1,
    };

spi_device_handle_t spi_handle;
spi_bus_add_device(SPI_HOST, &devcfg, &spi_handle);
```

完成 SPI 配置后，我们现在可以开始向第二个 ESP32-S3 发送数据。

在无限 `while` 循环中，程序持续检测按键是否被按下。当按键被按下时，我们切换 `ledState` 的值。

为了发送数据，我们使用 `spi_transaction_t` 结构体。我们以位为单位定义传输长度，并将数据分配给发送缓冲区，然后通过 `spi_device_transmit()` 发送数据。

```c
while (1) {        
if (gpio_get_level(1)) {            
ledState ^= 1;
        }        

spi_transaction_t t;
memset(&t, 0, sizeof(t));
t.length = 8 * 4; // 4 字节

if (ledState) {
t.tx_buffer = "ON\0\0";
} else {
t.tx_buffer = "OFF\0";
        } 

spi_device_transmit(spi_handle, &t);
vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

现在我们已经完成了第一个 ESP32-S3 的程序。接下来，我们将为第二个 ESP32-S3 创建程序，该设备将作为 SPI 从机运行，并根据接收到的数据控制 LED。

和之前一样，我们首先包含所需的库并定义 SPI 配置参数。

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/spi_slave.h"
#include "driver/gpio.h"
#include <string.h>

#define SPI_SLAVE_MOSI_IO 11
#define SPI_SLAVE_MISO_IO 12
#define SPI_SLAVE_SCLK_IO 13
#define SPI_SLAVE_CS_IO 10
#define SPI_HOST SPI2_HOST
```

在 `app_main()` 函数中，我们首先将 GPIO 1 配置为输出引脚，因为它连接到了 LED。

```c
void app_main(void) {    
gpio_set_direction(1, GPIO_MODE_OUTPUT);
```

接下来，我们配置 SPI 总线和 SPI 从机接口。我们使用相同的 `spi_bus_config_t` 结构体来分配引脚，然后使用 `spi_slave_interface_config_t` 配置从机模式和 CS 引脚。

```c
spi_bus_config_t buscfg = {
.mosi_io_num = SPI_SLAVE_MOSI_IO,
.miso_io_num = SPI_SLAVE_MISO_IO,
.sclk_io_num = SPI_SLAVE_SCLK_IO,
.quadwp_io_num = -1,
.quadhd_io_num = -1,
    };

spi_slave_interface_config_t slvcfg = {
.mode = 0,
.spics_io_num = SPI_SLAVE_CS_IO,
.queue_size = 3,
.flags = 0,
    };
```

配置完设置后，我们使用 `spi_slave_initialize()` 函数初始化 SPI 从设备。

```c
    spi_slave_initialize(SPI_HOST, &buscfg, &slvcfg, SPI_DMA_CH_AUTO);
```

最后，我们创建一个缓冲区来存储将从 SPI 主设备接收的数据。同时创建一个事务结构体，设置数据长度，并提供指向存储接收数据缓冲区的指针。在无限循环中，我们准备事务并通过 `spi_slave_transmit()` 将其传递给 SPI 驱动程序。

`spi_slave_transmit()` 函数会阻塞并被动等待，直到 SPI 主机发送数据。接收到数据后，我们检查接收到的命令，并根据主机 ESP32-S3 发送的是 `"ON"` 还是 `"OFF"` 来打开或关闭 LED。

```c
char recvbuf[4] = {0};
spi_slave_transaction_t t;
memset(&t, 0, sizeof(t));

t.length = 8 * 4; // 4 字节
t.rx_buffer = recvbuf;

while (1) {        

spi_slave_transmit(SPI_HOST, &t, portMAX_DELAY);

如果 (recvbuf[0] == 'O' && recvbuf[1] == 'N') {
gpio_set_level(1, 1);
} else if (recvbuf[0] == 'O' && recvbuf[1] == 'F' && recvbuf[2] == 'F') {
gpio_set_level(1, 0);
        }       
vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```