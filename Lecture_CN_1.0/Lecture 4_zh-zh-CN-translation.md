## 学习目标

*   使用显示设备

## 显示设备

在嵌入式系统项目中，我们常常需要一种方式来观察微控制器内部正在发生的事情。
这时显示设备就派上了用场。显示设备是重要的输出组件，能让微控制器直接与外部世界进行通信。它们帮助我们获取信息、显示传感器数据、呈现警报，并创建用户界面。与仅通过点亮单个 LED 来指示状态不同，显示器能让我们展示数字、文字，甚至复杂的图形。

### LED 矩阵

最简单的显示设备是 LED 矩阵，它是一个由 LED 按行和列排列而成的二维阵列。最常见的规格是 8x8 LED 矩阵，其中包含 64 个独立 LED，集成在一个模块中。
如果尝试单独控制 64 个 LED，就需要 64 个数字引脚，而大多数微控制器并不具备这么多引脚。为解决此问题，LED 采用了一种称为"多路复用"的技术进行布线。每行中 LED 的阳极（正极）相互连接，每列中 LED 的阴极（负极）也相互连接。

![](./attachments/led_matrix.png)

从示意图可以看出，一个简单的 8×8 LED 矩阵通常需要 16 个引脚来控制（8 行和 8 列）。例如，向第 1 列发送高电平信号即可激活该列。

![](./attachments/led_matrix_1.png)

然后，通过控制行，我们可以决定该列中的哪些 LED 灯亮起或熄灭。假设我们只想点亮第 1 列中的前两个 LED 灯。在这种情况下，我们会向第 1 列施加高电压，同时为了选择行信号，我们向除第一行和第二行之外的所有行发送高电平信号，这样只有这两个 LED 灯被激活，而其他 LED 灯保持熄灭。

![](./attachments/led_matrix_2.png)

然而，当我们尝试绘制更复杂的图形时，可能会遇到一个问题。我们可能会遇到冲突，即点亮特定 LED 灯会导致其他不需要的 LED 灯也亮起。结果，显示的图像可能不完整或不正确。为了解决这个问题，我们可以将绘制过程拆分为多个帧。例如，要绘制一个笑脸，第一帧可以显示眼睛，第二帧可以显示嘴巴。通过在这些帧之间快速切换，我们制造出两个部分同时显示的错觉。由于视觉暂留效应，观察者会感知到完整的笑脸，而不是分开的图像。

![](./attachments/led_matrix_3.png)

按照这种方法，我们可以绘制出大部分想要的形状，但使用全部 16 个引脚会占用微控制器上大部分可用引脚。为了简化 LED 矩阵的操作，我们采用 MAX7219 这类驱动芯片。该芯片内部处理所有多路复用操作，因此无需单独控制每个 LED。最终，我们只需要三个数字引脚：数据、时钟和加载（CS）。

#### 使用 MAX7219

MAX7219 驱动芯片帮助我们控制 LED 矩阵，同时将控制引脚数量减少到仅三个。为了管理所有 LED，它内置了一个 8×8 存储器（类似 SRAM），用于存储哪些 LED 应被点亮。其内部电路随后会解析这些数据并将其应用于 LED 矩阵。我们通过 SPI 通信与该驱动芯片交互，数据流由三个引脚控制：

*   **数据引脚（DIN）**：实际信息通过此引脚流入。我们逐位发送数据，前 8 位表示寄存器地址，后 8 位表示数值或图案。
*   **时钟引脚（CLK）**：提供时序信号。在每个时钟信号的上升沿，MAX7219 从 DIN 线采样一位数据并将其移入内部移位寄存器。
*   **CS / LOAD 引脚** ：用作锁存或片选信号（低电平有效）。当 CS 保持**低电平**时，芯片在每个 CLK 脉冲下接收并移入数据位。在精确的 16 个时钟脉冲（即发送 16 位数据）后，我们将 CS 拉高。此上升沿告知 MAX7219：“完整指令已就绪，现在解码并应用于显示屏或寄存器。”

简而言之，要与 MAX7219 通信，我们首先将 CS（片选）引脚拉低，表示数据传输开始。随后，在 16 个时钟周期内，我们在 DIN（数据）引脚上设置所需的位，并让 CLK（时钟）引脚先拉高再拉低。每个时钟脉冲向芯片发送一个位。在第 16 个时钟脉冲之后，我们将 CS 拉高。此时 MAX7219 锁存这 16 位数据，将其解析为地址和数据，并更新显示存储器或亮度、扫描限制等配置设置。完成后，MAX7219 会自动处理 LED 矩阵的逐行连续扫描。下图展示了这一操作过程。

![](./attachments/diagram.png)

让我们创建一个简单的项目，在 LED 矩阵上显示一颗心。首先，我们需要搭建电路。我们从将 ESP32-S3 的 5V 和 GND 引脚连接到 MAX7219 模块的 5V 和 GND 引脚开始。接下来，我们按如下方式连接命令引脚：

*   CLK 连接到 ESP32-S3 的 GPIO 19
*   CS 连接到 ESP32-S3 的 GPIO 20
*   DIN 到 Esp32s3 GPIO 21

![](./attachments/circuit_4_1.png)

现在让我们创建程序，首先包含所需的库

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "rom/ets_sys.h"
```

之后，我们创建代表引脚编号的常量

```c
#define DIN 21
#define CLK 19
#define CS  20
```

之后，我们创建一个处理数据发送逻辑的函数。由于硬件接口每次只发送一位数据，因此在传输前必须将字节分解为单独的位。我们通过使用位运算的掩码技术来实现这一点。在函数内部，我们定义了一个掩码数组：

```c
{0x80, 0x40, 0x20, 0x10, 0x08, 0x04, 0x02, 0x01}。
```

每个掩码对应字节中的一个位位置，从最高有效位（MSB）到最低有效位（LSB）。例如，`0x80`（二进制 `10000000`）用于隔离第一个位，而 `0x01`（二进制 `00000001`）用于隔离最后一个位。

在每次循环迭代中，我们使用按位与运算 `(data & bits[i])` 从字节中提取特定位。其原理很简单：与运算仅保留与掩码匹配的位，并清除所有其他位。如果结果不为零，则表示 `data` 中的该位为 `1`；否则为 `0`。

提取位值后，我们通过数据引脚（`DIN`）发送该值。如果该位为 `1`，我们将引脚置为高电平；如果为 `0`，则置为低电平。同时，我们控制时钟信号（`CLK`）以同步传输。时钟首先被置为低电平，然后将数据位放置到数据线上，最后将时钟置为高电平。
通过对所有8个比特重复此过程，该函数利用对数据线和时钟线的精确控制，成功地将整个字节逐位传输完毕。

```c
void send_byte(uint8_t data){
const uint8_t bits[8] = {0x80,0x40,0x20,0x10,0x08,0x04,0x02,0x01};
for (int i = 0; i < 8; i++) {
gpio_set_level(CLK, 0);
if ((data & bits[i]) != 0)
gpio_set_level(DIN, 1);
否则
gpio_set_level(DIN, 0);

ets_delay_us(2);
gpio_set_level(CLK, 1);
ets_delay_us(2);
    }
}
```

完成发送单个字节的函数后，下一步是构建一个更高级别的函数，用于处理与 MAX7219 驱动器的通信。

MAX7219 期望数据采用 16 位格式，该格式分为两个字节。第一个字节代表寄存器地址，指示我们要访问哪个内部寄存器（例如，某个数字位或控制寄存器）。第二个字节包含将写入该寄存器的实际数据。

为处理此操作，函数需接收两个参数：`reg`（寄存器地址）和 `data`（待发送数值）。通信开始时，先将片选（`CS`）引脚拉低，以此向 MAX7219 发送传输启动信号，告知其需关注即将传入的数据。

通信启动后，首先通过先前定义的 `send_byte` 函数发送寄存器地址，确保驱动器知晓数据应存储的位置。紧接着，再次使用相同的字节发送函数发送数据字节。

待两个字节均传输完毕后，将 `CS` 引脚拉高。此上升沿信号将通知 MAX7219 将接收到的 16 位数据锁存至其内部寄存器，从而完成整个操作。

```c
void max7219_send(uint8_t reg, uint8_t data){

gpio_set_level(CS, 0);

send_byte(reg);
send_byte(data);

gpio_set_level(CS, 1);

}
```

通信功能就绪后，下一步是初始化 MAX7219，使其按预期模式运行。这需要通过我们之前创建的 `max7219_send` 函数，向其内部寄存器发送一系列配置命令来实现。

每次调用 `max7219_send(reg, data)` 都会向驱动器内的特定寄存器写入一个值。初始化函数首先配置显示测试寄存器（`0x0F`），写入 `0x00` 以禁用测试模式。这一步至关重要，因为测试模式会强制所有 LED 点亮。

接着，将关机寄存器（`0x0C`）设置为 `0x01`。尽管名为"关机"，但此操作实际上会使设备退出关机模式并点亮显示屏。若缺少此步骤，MAX7219 将保持非活动状态。

接着，扫描限制寄存器（`0x0B`）被设置为 `0x07`。这告诉驱动器使用全部 8 个数字（从 0 到 7）。如果使用较小的值，则只有部分显示屏会被激活。

之后，亮度寄存器（`0x0A`）被设置为 `0x08`，用于控制 LED 的亮度。该值通常可在 `0x00`（最暗）到 `0x0F`（最亮）之间调整，因此这里设置的是中等亮度。

接着，解码模式寄存器（`0x09`）被设置为 `0x00`，以禁用 BCD 解码。这意味着我们工作在原始模式下，每个位直接控制一个 LED 段。

最后，一个循环遍历所有 8 行（从 1 到 8），并通过发送 `0x00` 来清除它们。这确保了显示屏以所有 LED 关闭的干净状态启动。

```c
void max7219_init(){
max7219_send(0x0F, 0x00);
max7219_send(0x0C, 0x01);
max7219_send(0x0B, 0x07);
max7219_send(0x0A, 0x08);
max7219_send(0x09, 0x00);


for (int i = 1; i <= 8; i++)
max7219_send(i, 0x00);

}
```

所有构建模块准备就绪后，我们现在可以在 `app_main` 中实现程序的主要逻辑。

在函数开头，我们将连接到 MAX7219 的 `DIN`、`CLK` 和 `CS` 引脚的 GPIO 配置为输出。然后将 `CS`（片选）线置为高电平，这意味着设备处于空闲状态，当前未接收数据。

接下来，我们调用 `max7219_init()` 函数，使用之前定义的配置来初始化驱动程序。

初始化后，我们定义了一个名为 `heart` 的数组，其中包含 8 个字节。每个字节代表 LED 矩阵的一行，而字节中的每一位对应那一行中的一个 LED。`1` 表示 LED 亮起，`0` 则表示熄灭。该数组中的二进制值模式在矩阵上显示时，会形成一个心形图案。

在无限循环中，我们首先遍历 `heart` 数组。对于每一行，我们调用 `max7219_send(i + 1, heart[i])`。其中 `i + 1` 代表行号，而 `heart[i]` 则是该行的图案。

显示心形图案后，我们使用 `vTaskDelay` 引入 500 毫秒的延迟。这能让心形图案短暂保持可见。

接着，我们运行另一个循环，通过向所有行发送 `0` 来清除显示。这实际上会关闭所有 LED。随后再延迟 500 毫秒，使显示屏保持空白相同的时间。

通过不断重复显示心形图案然后清除的这两个步骤，我们就能在 LED 矩阵上实现心形闪烁效果。

```c
void app_main(void) {

gpio_set_direction(DIN, GPIO_MODE_OUTPUT);
gpio_set_direction(CLK, GPIO_MODE_OUTPUT);
gpio_set_direction(CS, GPIO_MODE_OUTPUT);

gpio_set_level(CS, 1);

max7219_init();

uint8_t heart[8] = {
0B00000000,
0B01100110,
0B11111111,
0B11111111,
0B11111111,
0B01111110,
0B00111100,
0B00011000
  };
while (1) {
for (int i = 0; i < 8; i++) {
max7219_send(i + 1, heart[i]);
        }

vTaskDelay(pdMS_TO_TICKS(500));

for (int i = 0; i < 8; i++) {
max7219_send(i + 1, 0);
        }
vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

### 七段数码管

七段数码管是由七个 LED 段组成的数字显示器，可排列形成数字 0-9 及部分字母。这些段用字母 A 到 G 标记。通过点亮特定段的组合，我们可以显示任何数字。
例如，要显示数字"1"，我们点亮 B 段和 C 段。

![](./attachments/7segments.png)

七段数码管主要有两种类型：

*   **共阴极：** 所有 LED 的负极引脚（GND）连接在一起。我们向某个段的引脚发送高电平信号以点亮它。
*   **共阳极：** 所有 LED 的正极引脚（VCC）连接在一起。我们向某个段的引脚发送低电平信号以点亮它。

如果尝试单独控制每个段，则需要 7 个 GPIO 引脚。这对于单个显示器来说尚可，但使用多个七段数码管时，可用的 GPIO 引脚很快就会耗尽。

为解决此问题，许多七段显示器会搭配一个 BCD 转七段译码器。这种方法将所需的 GPIO 引脚数量从 7 个减少到仅 4 个控制引脚。我们无需直接驱动每个段，而是使用 BCD（二进制编码十进制）编码发送数字值，其中每个十进制数字由其对应的二进制数表示。

解码器随后将此二进制输入转换为适当的信号，以点亮正确的段。相应的映射关系如下表所示：

| 十进制数字 | BCD 码 (a,b,c,d) |
| --- | --- |
| 0 | 0000 |
| 1 | 0001 |
| 2 | 0010 |
| 3 | 0011 |
| 4 | 0100 |
| 5 | 0101 |
| 6 | 0110 |
| 7 | 0111 |
| 8 | 1000 |
| 9 | 1001 |

例如，对于共阴极七段数码管，我们可以通过发送（低电平、低电平、高电平、高电平）的 BCD 信号来显示数字"3"，如下所示：

![](./attachments/7segment_bcd.png)

### 四位7段数码管

在许多应用中，我们需要显示超过一位的数字，例如两位、四位或更大的数值。四位7段数码管是将四个独立的7段数码管组合成一个模块而成。其实现方式是将所有对应的段引脚连接在一起，同时为每个数码管添加一个独立的控制（选通）引脚，用于确定当前哪个数码管处于激活状态。

![](./attachments/4digit_7segments.png)

如果我们尝试手动控制所有四个数码管，则需要 12 个引脚：8 个引脚用于段控制，4 个引脚用于数码管选择。为了减少引脚数量，我们可以使用 BCD 编码器，这样仅需 9 个引脚即可驱动所有四个数码管（4 个 BCD 输入引脚 + 5 个段引脚，含小数点），若去掉小数点则可减少至 8 个引脚。

![](./attachments/4digit_7segments_bcd.png)

#### 使用 TM1637 芯片

TM1637 是一款专用于四位七段数码管的 LED 驱动控制电路。与 MAX7219 类似，其主要目标是大幅减少驱动显示屏所需的 GPIO 引脚数量。标准四位显示屏可能需要多达 12 个引脚，而 TM1637 仅需两个通信引脚（外加电源和接地）即可实现控制。

我们通过一种与 I2C 非常相似的 2 线串行协议与 TM1637 通信，不过该协议缺少设备寻址功能。数据流通过以下两个引脚进行管理：

*   **数据引脚（DIO）：** 这个双向引脚用于向芯片传输数据或从芯片读取数据。我们通过该线路逐位发送命令和显示数据。
*   **时钟引脚（CLK）：** 该引脚为数据传输提供时序。当时钟信号为高电平时，DIO 引脚上的数据必须保持稳定；只有当时钟信号为低电平时，数据状态才能发生变化。

与 MAX7219 依赖片选（CS）引脚指示传输开始和结束不同，TM1637 依靠数据线和时钟线上的特定信号条件：

*   起始条件：当 DIO 线被拉低且 CLK 线保持高电平时，传输开始。
*   停止条件：当 DIO 线被拉高且 CLK 线保持高电平时，传输结束。
*   **数据传输：** 在起始和停止条件之间，先发送 8 位数据（最低有效位优先）。发送完 8 位数据后，会生成第 9 个时钟脉冲，允许 TM1637 通过将 DIO 线拉低来发送应答（ACK）信号。之后，我们可以设置停止条件来终止数据传输，也可以继续发送另一组数据。

![](./attachments/diagram2.png)

让我们创建一个简单的项目，在 4 位 7 段数码管上显示数字"1234"。首先将 ESP32-S3 的 5V 和 GND 引脚连接到 TM1637 模块的 VCC 和 GND 引脚。接着按以下方式连接通信引脚：

*   CLK 连接至 ESP32-S3 的 GPIO 19
*   DIO 连接至 ESP32-S3 的 GPIO 21

![](./attachments/4_7segments_circuit.png)

现在开始编写程序。首先引入必要的库。

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "rom/ets_sys.h"
```

之后，我们创建代表引脚的常量。

```
#define CLK 19
#define DIO 21
```

由于 TM1637 协议依赖起始和停止条件而非 CS 引脚，我们需要创建两个小型辅助函数来处理这些特定时序。对于起始条件，DIO 在 CLK 为高电平时变为低电平；对于停止条件，DIO 在 CLK 为高电平时变为高电平。

```c
void tm1637_start(void) {
gpio_set_level(CLK, 1);
gpio_set_level(DIO, 1);
ets_delay_us(2);
gpio_set_level(DIO, 0);
ets_delay_us(2);
gpio_set_level(CLK, 0);
}

void tm1637_stop(void) {
gpio_set_level(CLK, 0);
gpio_set_level(DIO, 0);
ets_delay_us(2);
gpio_set_level(CLK, 1);
ets_delay_us(2);
gpio_set_level(DIO, 1);
}
```

接下来，我们实现负责发送单个字节的函数。与 MAX7219 要求数据以最高有效位（MSB）优先传输不同，TM1637 要求先发送最低有效位（LSB）。为适应这一要求，我们反转了之前使用的位提取逻辑。不再按从首到尾的顺序遍历掩码数组，而是以相反方向循环处理。
在全部 8 位数据传输完毕后，TM1637 需要进入应答（ACK）阶段。在此阶段，设备会将 `DIO` 线拉低，以表示成功接收该字节。为处理此过程，我们生成一个额外的（第 9 个）时钟脉冲，并将数据线置低以完成该周期，但无需显式检查 ACK 信号。

```c
void tm1637_write_byte(uint8_t data) {
const uint8_t bits[8] = {0x80,0x40,0x20,0x10,0x08,0x04,0x02,0x01};
for (int i = 7; i >= 0; i--) {
gpio_set_level(CLK, 0);
if (data & bits[i]) {
gpio_set_level(DIO, 1);
} 否则 {
gpio_set_level(DIO, 0);
        }
ets_delay_us(2);
gpio_set_level(CLK, 1);
ets_delay_us(2);
    }
// 应答周期 1 脉冲
gpio_set_level(CLK, 0);
gpio_set_level(DIO, 0);
ets_delay_us(2);
gpio_set_level(CLK, 1);
ets_delay_us(2);
gpio_set_level(CLK, 0);
}
```

要在七段数码管上显示数字，我们需要知道每个数字（0-9）需要点亮哪些段。为此，我们可以定义一个数组，其中索引对应数字，值是需要点亮的段的十六进制表示。

```c
const uint8_t segment_map[] = {
0x3F, // 0: 段 A、B、C、D、E、F
0x06, // 1: 段 B、C
0x5B, // 2: 段 A、B、D、E、G
0x4F, // 3: 段 A、B、C、D、G
0x66, // 4: 段 B、C、F、G
0x6D, // 5: 段 A、C、D、F、G
0x7D, // 6: 段 A、C、D、E、F、G
0x07, // 7: 段 A、B、C
0x7F, // 8: 段 A、B、C、D、E、F、G
0x6F  // 9: 段 A、B、C、D、F、G
};
```

现在我们可以将所有内容整合起来，实现 TM1637 显示屏的主要应用逻辑。

在 `app_main` 函数开始时，我们将 `CLK` 和 `DIO` 引脚配置为输出模式。这两行代码构成了与 TM1637 的通信接口。随后，我们将两条线路都设置为高电平，这代表总线的空闲状态。这一点很重要，因为 TM1637 协议要求在没有数据传输时，时钟线和数据线都保持高电平。

通信过程分为三个主要命令，每个命令都通过 `tm1637_start()` 和 `tm1637_stop()` 函数包裹在起始和停止条件之间。这些函数定义了每次传输的开始和结束。

第一个命令发送 `0x40`，将 TM1637 配置为自动地址递增模式。这意味着在写入一个字节的数据后，内部地址指针会自动移动到下一个位置，从而便于连续发送多个数字。

第二条指令控制显示状态和亮度。数值 `0x8F` 结合了两项设置：开启显示（`0x88`）并将亮度调至最高（`0x07`）。这确保了数字清晰可见且亮度充足。

在无限循环中，第三条指令首先发送 `0xC0`，用于设置显示数据的起始地址（即第一个数码管的位置）。随后，我们发送四个字节，每个字节代表一个要显示的数字。这些数值来自 `segment_map` 数组。

发送完所有指令后，我们使用 `vTaskDelay` 添加 1 秒的延时。这能使显示的数字保持稳定，随后循环重复并再次刷新显示。

```c
void app_main(void) {

// 将引脚配置为输出
gpio_set_direction(CLK, GPIO_MODE_OUTPUT);
gpio_set_direction(DIO, GPIO_MODE_OUTPUT);

// 设置初始状态为高电平（空闲状态）
gpio_set_level(CLK, 1);
gpio_set_level(DIO, 1);

// 命令1：写入数据，自动递增地址
tm1637_start();
tm1637_write_byte(0x40);
tm1637_stop();

// 命令2：开启显示，设置亮度
tm1637_start();
tm1637_write_byte(0x8F); 
tm1637_stop();

while (1) {

// 命令 3：设置起始地址（0xC0）并发送 4 位数字
tm1637_start();
tm1637_write_byte(0xC0); 
tm1637_write_byte(segment_map[1]); // 发送数字"1"的段码
tm1637_write_byte(segment_map[2]); // 发送数字"2"的段码
tm1637_write_byte(segment_map[3]); // 发送数字"3"的段码
tm1637_write_byte(segment_map[4]); // 发送数字"4"的段码
tm1637_stop();

vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

为了让项目更有趣，我们修改主逻辑，使显示屏显示从59到0的倒计时，而非固定数字。程序整体结构保持不变——初始化和通信设置完全沿用之前的代码，仅更新显示逻辑。

在无限循环中，我们引入一个从 59 递减至 0 的 `for` 循环。该循环控制着要在 4 位数码管上显示的值。每次循环迭代时，我们都会向显示屏发送一个新数值。

显示更新从通常的起始条件开始，并发送 `0xC0` 命令来设置起始地址。然后我们发送四个字节，每个字节代表一个数字：

*   前两位数字设置为 `0`，因此显示为 `00`。这样可以将注意力集中在最后两位数字上。
*   第三位数字使用 `i / 10` 设置。这会提取数字的十位数。例如，如果 `i = 47`，那么 `47 / 10 = 4`。
*   第四位数字使用 `i % 10` 设置。这会提取个位数。例如，`47 % 10 = 7`。

通过结合这两个操作，我们将数字拆分为十进制位，并正确显示在七段数码管上。

发送数字后，我们停止传输并使用 `vTaskDelay` 等待 1 秒。这会产生可见的倒计时效果，每秒减少一个数字。

```c
void app_main(void) {

gpio_set_direction(CLK, GPIO_MODE_OUTPUT);
gpio_set_direction(DIO, GPIO_MODE_OUTPUT);

gpio_set_level(CLK, 1);
gpio_set_level(DIO, 1);

tm1637_start();
tm1637_write_byte(0x40);
tm1637_stop();

tm1637_start();
tm1637_write_byte(0x8F);
tm1637_stop();

while (1) {
for (int i = 59; i >= 0; i--){
tm1637_start();
tm1637_write_byte(0xC0);
tm1637_write_byte(segment_map[0]);
tm1637_write_byte(segment_map[0]);
tm1637_write_byte(segment_map[i/10]);
tm1637_write_byte(segment_map[i%10]);
tm1637_stop();
vTaskDelay(pdMS_TO_TICKS(1000));
        }   
    }
}
```

### LCD 显示屏

当我们需要显示数字或简单形状时，LED 矩阵和七段数码管非常实用。然而，它们存在一个重要的局限性。七段数码管主要针对数字设计，而 LED 矩阵若要显示可读文本，则需要大量 LED 灯珠和复杂的控制逻辑。为解决这一问题，我们采用 LCD 显示屏（液晶显示屏）。这类显示器能够显示文本字符，有时还能呈现小型符号，非常适合用于构建用户界面。

LCD（液晶显示器）的工作原理依赖于一种名为液晶的特殊材料。这类材料兼具液体和固态晶体的特性。

在 LCD 模块内部，包含多个层：

1.  两片偏光片
2.  一层液晶
3.  带有透明电极的玻璃板
4.  背光源

![](./attachments/lcd.png)

与 LED 显示屏不同，LCD 像素本身并不发光。相反，它们充当微小的快门，控制光线如何穿过屏幕。当内部控制器对特定像素区域施加电压时，液晶分子会发生物理扭转。这种扭转会改变光的偏振方向。在未通电时，光线自然穿过液晶分子；而扭转后的液晶分子会使第二层（前）偏振滤光片阻挡光线。结果，该像素呈现暗色，而周围区域保持明亮。这种被阻挡的光线形成了构成文字、数字和符号的暗色"像素"。
LCD 有多种模块类型，但市场上最常见的模块包括：

*   **16 × 2 LCD** 每行显示 16 个字符，共 2 行。
*   **20 × 4 LCD** 每行显示 20 个字符，共 4 行。
*   **40 × 2 LCD** 每行显示 40 个字符，共 2 行。
*   **40 × 4 LCD** 每行显示 40 个字符，共 4 行。

标准的 16×2 LCD 模块通常有 16 个引脚。这些引脚用于电源、控制信号、数据通信和背光控制。

![](./attachments/lcd_16_2.png)

要与 LCD 直接通信，我们有两种可能的模式：

*   8位模式
*   4位模式

在 8 位模式下，我们使用 LCD 的全部八条数据引脚（D0-D7）一次性发送一个完整的字节数据。这种方法速度更快，因为整个字符可以在单次操作中传输完成。但该模式需要占用大量 GPIO 引脚：

*   8个数据引脚
*   2 个控制引脚（RS、使能）

在 4 位模式下，LCD 分两步接收数据而非一步完成。首先发送高 4 位数据，随后发送同一字节的低 4 位数据。LCD 内部会将这两部分组合起来，重新构建完整的 8 位数值。
这种方法减少了所需的引脚数量：

*   4 个数据引脚（D4–D7）
*   2 个控制引脚（RS 和 Enable）

让我们创建一个简单的“Hello World”示例，使用 4 位模式的 16×2 LCD。首先，将 LCD 连接到 ESP32-S3：将 RS 连接到 GPIO 20，EN 连接到 GPIO 19，D4 连接到 GPIO 1，D5 连接到 GPIO 2，D6 连接到 GPIO 42，D7 连接到 GPIO 41。对于电源连接，将 VSS、RW 和 K 连接到 GND，将 VDD 和 A 连接到 5V。

![](./attachments/lcd_16_2_circuit.png)

现在我们来创建 LCD 组件，首先在项目根目录下添加一个 `components` 目录。在该目录中，创建一个名为 `lcd` 的文件夹，用于存放所有与 LCD 驱动相关的文件。
在 `include`lcd`lcd.h` 文件夹中，我们添加一个 `CMakeLists.txt` 文件来定义构建规则，一个 `lcd.c` 文件用于实现功能，以及一个 `目录用于存放` 头文件。

```
├── CMakeLists.txt 
├── main/
│  ├── CMakeLists.txt 
│  └── main.c 
└── components/ 
└── lcd/
├── CMakeLists.txt 
├── lcd.c 
└── include/ 
└── lcd.h
```

在 `components/lcd/CMakeLists.txt` 文件中，我们使用 `idf_component_register` 函数来告知构建系统需要编译哪些源文件以及公共头文件的查找路径：

```
idf_component_register(SRCS "lcd.c"  
INCLUDE_DIRS "include")
```

最后，我们更新 `main` 文件夹内的 `CMakeLists.txt` 文件。通过添加 `REQUIRES` 关键字，我们确保构建系统将 `lcd` 组件链接到主应用程序中：

```
idf_component_register(SRCS "main.c"  
INCLUDE_DIRS "."
REQUIRES lcd)
```

完成项目重构后，我们可以开始实现 LCD 组件逻辑。首先，我们需要阅读数据手册，了解如何对该设备进行编程和控制。
快速检查可知，RS 和 RW 引脚控制着 LCD 的工作模式：

*   RS=0 且 RW=0 时，LCD 处于监听配置命令状态。
*   RS=1 且 RW=0 时，LCD 接收显示指令。

此外，从数据手册可知，命令通过数据引脚 D0...D7 发送，LCD 接受以下命令：

![](./attachments/config_table.png)

对于发送的每条命令，我们需要在 E（使能）引脚上发送一个脉冲来验证数据，下图展示了信号应如何发送。

![](./attachments/time_diagram.png)

最后，在发送任何命令或尝试在 LCD 上显示任何字符之前，我们需要先等待 15 毫秒，并通过将 D5 和 D4 设置为 1 并发送三个使能脉冲来初始化它。之后，我们需要发送的第一个命令是显示模式，我们将其设置为 4 位模式，然后就可以通过发送其余命令来配置 LCD。下图展示了工作流程。

![](./attachments/chart.png)

现在我们知道 LCD 的工作原理了，让我们开始创建程序。首先包含我们将要使用的库。

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "rom/ets_sys.h"
```

现在让我们创建函数，首先从 `lcd_pulse_enable` 开始，该函数用于处理 E 引脚上的使能脉冲。

```c
void lcd_pulse_enable() {
gpio_set_level(E, 1);
ets_delay_us(1);
gpio_set_level(E, 0);
ets_delay_us(20);
}
```

由于我们计划使用 4 位模式，让我们创建一个负责向 LCD 发送 4 位数据的函数。该函数接受一个字节类型的数据作为参数，由于我们一次只能向引脚发送 1 位数据，因此我们使用按位与运算和掩码来提取所需的位。

```c
void lcd_send_bits(uint8_t bits){
gpio_set_level(D4, bits & 0b1);
gpio_set_level(D5, bits & 0b10);
gpio_set_level(D6, bits & 0b100);
gpio_set_level(D7, bits & 0b1000);
lcd_pulse_enable();
}
```

现在我们来实现负责发送命令的函数。在 4 位模式下，LCD 需要分两步接收数据：先发送高 4 位（高半字节），再发送低 4 位（低半字节）。要提取高 4 位，我们将其右移 4 位。

```c
void send_command(uint8_t command){
lcd_send_bits(command >> 4);
lcd_send_bits(command);
}
```

我们创建一个函数来初始化我们的 GPIO 引脚

```c
void lcd_gpio_init(){
// 1. 设置 GPIO 引脚
gpio_set_direction(D4, GPIO_MODE_OUTPUT);
gpio_set_direction(D5, GPIO_MODE_OUTPUT);
gpio_set_direction(D6, GPIO_MODE_OUTPUT);
gpio_set_direction(D7, GPIO_MODE_OUTPUT);
gpio_set_direction(RS, GPIO_MODE_OUTPUT);
gpio_set_direction(E, GPIO_MODE_OUTPUT);

// 初始状态
gpio_set_level(RS, 0);
gpio_set_level(E, 0);
}
```

现在我们已经搭建好了 LCD 通信的基础框架，接下来可以实现初始化函数。该函数用于将 LCD 正确配置为 4 位模式。

当 LCD 上电时，默认处于 8 位模式。要将其切换为 4 位模式，必须遵循控制器数据手册中定义的特定初始化序列。首先，我们等待约 50 毫秒，确保 LCD 已完全上电。然后，使用（D4 和 D5 引脚）发送三次 `0x03` 值，每次发送后跟随一个使能脉冲。这一步强制 LCD 进入已知的 8 位状态，无论其之前处于何种状态。
之后，我们发送 `0x02` 将 LCD 切换为 4 位模式。
一旦 LCD 处于 4 位模式，我们就可以发送完整命令来配置显示屏。首先使用功能设置命令选择 4 位接口和双行显示模式。接着，开启显示屏同时禁用光标和闪烁功能。最后，清除显示屏并等待操作完成。

```c
void initialize_lcd(){ 
lcd_gpio_init();
// 等待 LCD 上电
vTaskDelay(pdMS_TO_TICKS(50));

// 将 D4 和 D5 设置为高电平以产生三个使能脉冲
lcd_send_bits(0x03);
vTaskDelay(pdMS_TO_TICKS(5));

lcd_pulse_enable();
ets_delay_us(100);

lcd_pulse_enable();
ets_delay_us(10);


// 将命令模式设置为4位
lcd_send_bits(0x02);

// 功能设置：我们将数据长度设为4位，显示设为2行
//Dl=0
//N = 1
send_command(0x28);
//现在我们将显示设置为开启，并将光标和闪烁光标设置为关闭
send_command(0x0C);
//最后我们清除显示并返回至位置0 0
send_command(0x01);
vTaskDelay(pdMS_TO_TICKS(2));

  }
```

在实现用于显示文本和字符的函数之前，我们首先定义两个辅助函数。
第一个函数清除显示并将光标返回到起始位置。这是通过发送清除显示命令（ `0x01`），然后进行短暂延迟以允许 LCD 完成操作来实现的。

```C
void clear_display(){
send_command(0x01);
vTaskDelay(pdMS_TO_TICKS(2));
}
```

第二个辅助函数将光标移动到由行和列指定的特定位置。其实现方式是将最高位（D7）设置为 1，并与目标 DDRAM 地址组合，如下表所示。

对于大多数 16×2 LCD：

*   第 1 行起始地址为 `0x00` → 命令基址为 `0x80`
*   第 2 行起始地址为 `0x40` → 命令基址 `0xC0`

![](./attachments/address.png)

```c
void move_to(uint8_t line, uint8_t column){
if (line == 1){
send_command(0x80 | column);
}else{
send_command(0xC0 | column);
	}
}
```

现在我们可以实现负责显示单个字符的函数。该函数接收一个字符作为参数，将 RS（寄存器选择）引脚置为高电平以表示正在发送数据（而非命令），将字符发送到 LCD，然后将 RS 重新置为低电平。

```c
void write_character(char c){
gpio_set_level(RS, 1);
send_command(c);
gpio_set_level(RS, 0);
}
```

我们还可以创建一个函数来显示完整的消息（字符串）。该函数接收一个字符数组，并使用循环遍历每个字符，直到遇到空终止符（`'\0'`）。然后，每个字符通过 `write_character` 函数发送到 LCD。

```c
void write_word(char message[]){
for (int i=0;message[i]!='\0';i++){
write_character(message[i]);
  }
}
```

最后一步是创建 `lcd.h` 头文件。该文件定义了 LCD 驱动程序的公共接口，允许程序的其他部分与 LCD 进行交互。

我们首先使用包含守卫来防止头文件被多次包含，这可能会导致错误。这是通过在文件顶部使用 `#ifndef`、`#define`，并在底部使用 `#endif` 来实现的。
在守卫之间，我们编写计划共享的函数，并定义代表引脚常量的值。

```c
#ifndef LCD_H
#define LCD_H  

#include <stdint.h>  

#define D4 1
#define D5 2
#define D6 42
#define D7 41
#define RS 20
#define E 19

// 初始化
void initialize_lcd(void);

// 基本控制
void clear_display(void);
void move_to(uint8_t line, uint8_t column);  

void write_character(char c);  
void write_word(char message[]);

#endif
```

现在，在 `main.c` 文件中，我们可以包含 LCD 组件并显示一条简单的消息。

初始化 LCD 后，我们发送一个字符串以显示在屏幕上。

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "lcd.h"

void app_main(void) {
initialize_lcd();
write_word("你好，世界");

}
```