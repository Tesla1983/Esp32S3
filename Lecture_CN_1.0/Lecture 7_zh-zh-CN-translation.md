## 学习目标

*   探索 ESP32-S3 的 Wi-Fi 功能
*   配置接入点（AP）模式
*   配置工作站（STA）模式

## ESP32-S3 上的 Wi‑Fi

在现代嵌入式系统中，连接能力与处理能力同样重要。Wi-Fi 是一种无线网络技术，它允许设备通过 IEEE 802.11 标准在局域网内进行通信。通过集成 Wi-Fi，我们的微控制器从一个孤立的计算器转变为一个物联网（IoT）设备，能够与云服务器同步、提供基于网页的用户界面，并与其他智能设备通信。

ESP32-S3 配备了高度集成的 Wi-Fi 射频和 MAC（媒体访问控制）。在 ESP-IDF 框架中，Wi-Fi 驱动程序依赖于几个核心系统组件，主要是非易失性存储（NVS）用于保存网络凭证，以及事件循环用于管理异步网络事件，如连接、断开连接或接收 IP 地址。

根据项目需求，Wi-Fi 外设可以在不同模式下运行：

*   **站点（STA）模式：**ESP32-S3 作为客户端，连接到现有的路由器。
*   **接入点（AP）模式：**ESP32-S3 创建自己的 Wi-Fi 网络，允许智能手机或笔记本电脑等其他设备直接连接。
*   **AP+STA 模式：** 芯片同时扮演两种角色。

ESP32-S3 上的 Wi-Fi 通信通过 ESP-IDF 网络协议栈处理，该协议栈基于 TCP/IP 和 FreeRTOS 构建。

### Wi-Fi 协议

Wi-Fi 是一种基于 IEEE 802.11 标准的无线通信技术。设备无需使用物理以太网线缆，而是通过无线电波传输数据来进行通信。

一个典型的 Wi-Fi 网络包含：

*   一个管理网络的**路由器**或**接入点（AP）**
*   多个**客户端设备** ，例如智能手机、笔记本电脑、传感器或像 ESP32-S3 这样的微控制器

当设备连接到同一 Wi-Fi 网络时，它们可以通过网络协议和唯一的 IP 地址交换数据。当设备加入网络时：

1.  路由器对设备进行身份验证
2.  设备获取一个 IP 地址
3.  数据以数据包的形式进行交换

#### TCP/IP 和 HTTP

Wi-Fi 仅提供设备之间的无线连接。它就像一条道路，允许设备通过空气发送信号。然而，Wi-Fi 本身并不定义数据如何在设备之间组织、传输、交付或理解。
为了实现通信，ESP32-S3 使用更高级别的网络协议，主要包括：

*   **TCP/IP** 处理网络上的设备间通信
*   **HTTP** 处理网络客户端与网络服务器之间的通信

#### TCP/IP

TCP/IP 是允许设备在任何网络上相互通信的基础框架。它由两种不同协议组成，协同工作：

*   **互联网协议（IP）——邮件投递：** 专注于寻址和路由。它为每台设备分配一个数字地址（例如 `192.168.1.50`），并将数据分割成更小的数据包\*进行发送。但它不保证数据包能安全到达。
*   **传输控制协议（TCP）的质量控制：** 位于 IP 协议之上，用于确保可靠性。它在发送数据前建立稳固连接，追踪数据包，将其重新排列为正确顺序，并在数据丢失或损坏时要求重新发送。

在 ESP32-S3 中，TCP/IP 被集成到 ESP-IDF 网络协议栈中，该协议栈会自动处理所有底层网络通信。一旦设备连接到 Wi-Fi，它就能获取 IP 地址、建立 TCP 连接并与其他设备通信，无需手动控制数据包处理。

#### HTTP（网页通信层）

HTTP（超文本传输协议）建立在 TCP/IP 之上，专门用于基于网页的通信。TCP/IP 负责数据的传输方式，而 HTTP 则定义数据的含义及其在网页浏览器、服务器和物联网设备等系统间交换时的结构规范。

HTTP 遵循客户端-服务器模型，其中一个设备发送请求，另一个设备做出响应。客户端请求资源或发送数据，服务器处理该请求并返回响应。这种结构化的通信方式使设备能够以可预测且标准化的方式进行交互。

例如，当 ESP32-S3 作为 HTTP 客户端时，它可以通过 HTTP 请求将传感器数据发送到网络服务器。请求中包含数据以及关于应执行何种操作的指令，例如存储数据或分析数据。随后，服务器会响应一条状态消息，确认请求是否成功，或返回附加信息。

HTTP 使用不同的方法来定义客户端与服务器之间的数据交换方式。最常见的方法有 **GET**、**POST** 和 **PUT**。

*   **GET** 用于从服务器检索数据而不做任何更改，主要用于读取信息。
*   **POST** 用于向服务器发送数据，例如上传温度或湿度等传感器数值。
*   **PUT** 用于更新服务器上的现有数据。

### 接入点（AP）模式

当 ESP32-S3 配置为接入点（AP）模式时，它会创建自己的无线网络并广播服务集标识符（SSID）。附近的设备（如智能手机、笔记本电脑或平板电脑）可以扫描该 SSID，并以连接标准 Wi-Fi 路由器的方式直接连接到 ESP32-S3。

在此模式下，ESP32-S3 还可运行 HTTP 服务器。这使得微控制器能够充当小型网络服务器，托管网页、接收来自网页表单的数据，并与移动或 Web 应用程序通信。因此，用户无需外部互联网连接或路由器，即可通过浏览器界面与 ESP32-S3 进行交互。

#### 初始化与设置模式

在设备连接到 ESP32-S3 之前，必须初始化 Wi-Fi 驱动程序并将其配置为**接入点（AP）模式** 。以下示例展示了如何初始化 Wi-Fi 协议栈、将 ESP32-S3 设置为 AP 模式，并配置基本的网络凭证。

```c
#include "esp_wifi.h"  
#include "esp_event.h"  
#include "nvs_flash.h"  
#include <string.h>

void wifi_init_ap(void)  {  
	// Initialize NVS  
	nvs_flash_init();  
  
	// Initialize TCP/IP stack  
	esp_netif_init();  
  
	// Create default event loop  
	esp_event_loop_create_default();  
  
	// Create default Wi-Fi AP interface  
	esp_netif_create_default_wifi_ap();  
  
	// Initialize Wi-Fi driver  
	wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();  
	esp_wifi_init(&cfg);  
  
	// Configure Access Point settings  
	wifi_config_t ap_config = {  
		.ap = {  
		.ssid = "ESP32_AP",  
		.ssid_len = strlen("ESP32_AP"),  
		.password = "mypassword123",  
		.max_connection = 4,  
		.authmode = WIFI_AUTH_WPA_WPA2_PSK  
		},  
	};  
  
	// Set Wi-Fi mode to Access Point  
	esp_wifi_set_mode(WIFI_MODE_AP);  
  
	// Apply AP configuration  
	esp_wifi_set_config(WIFI_IF_AP, &ap_config);  
  
	// Start Wi-Fi  
	esp_wifi_start();  
}

```

在 Wi-Fi 无线电开启之前，代码会先初始化核心软件环境：

*   **`nvs_flash_init()`**：初始化非易失性存储（NVS）。ESP32 的 Wi-Fi 驱动在底层使用此持久化内存来存储校准数据和内部 Wi-Fi 配置。
*   **`esp_netif_init()`**：初始化底层的 TCP/IP 网络协议栈，使芯片能够管理 IP 地址和网络数据包。
*   **`esp_event_loop_create_default()`**：创建一个系统级的事件循环。由于 Wi-Fi 操作是异步进行的（例如设备连接或断开连接），此循环会发出警报，供你的代码后续响应。
*   **`esp_netif_create_default_wifi_ap()`**：创建一个专为接入点定制的默认网络接口，将 TCP/IP 协议栈绑定到 Wi-Fi 硬件上。
*   **`esp_wifi_init(&cfg)`**：该代码通过 `WIFI_INIT_CONFIG_DEFAULT()` 拉取一组标准且安全的内部配置，并将其传递给 Wi-Fi 初始化函数。这会分配必要的内存、启动 Wi-Fi 任务并唤醒 Wi-Fi 驱动程序。

接下来，代码使用 `wifi_config_t` 结构定义热点的特性：

*   **SSID（`"ESP32_AP"`）**：这是设备扫描时可见的 Wi-Fi 网络公共名称。
*   **密码（`"mypassword123"`）**：加入网络所需的安全密钥。
*   **最大连接数（`4`）**：将网络限制为最多同时连接 4 个客户端设备，从而节省内存和处理能力。
*   **认证模式（`WIFI_AUTH_WPA_WPA2_PSK`）**：强制使用标准的 WPA/WPA2 安全协议。随后的 `if` 语句作为安全检查：如果密码字符串完全为空，则自动将安全模式降级为 `WIFI_AUTH_OPEN`，允许任何人无需密码即可连接。

在定义接入点配置后，ESP32 被切换为 AP 模式，应用配置并启动 Wi-Fi 服务。

*   **`esp_wifi_set_mode(WIFI_MODE_AP)`**：将 ESP32 设置为仅以接入点模式运行，使其能够创建并广播自己的无线网络，而不是作为客户端连接到其他路由器。
*   **`esp_wifi_set_config(WIFI_IF_AP, &ap_config)`**：应用先前定义的接入点设置，包括 SSID、密码、认证模式以及最大连接客户端数量。
*   **`esp_wifi_start()`**：启动 Wi-Fi 驱动程序，并启用 ESP32 无线电硬件，开始广播已配置的网络并接受传入的客户端连接。

#### 创建并启动 HTTP 服务器

此时，设备可以成功连接到 ESP32-S3 Wi-Fi 网络。然而，仅加入网络并不足以实现通信。ESP32 仍需一种方式来接收请求并向连接的设备发送响应。这正是 HTTP 服务器发挥作用的地方。
HTTP 服务器使 ESP32-S3 能够像一个小型网络服务器一样运行。服务器启动后，连接到 ESP32 接入点的手机、笔记本电脑或浏览器可以发送 HTTP 请求，例如：

*   打开由 ESP32 托管的网页
*   发送命令控制 LED 或传感器
*   请求传感器读数或系统信息
*   通过 API 与微控制器交换数据

如果没有 HTTP 服务器，ESP32 虽然可以创建 Wi-Fi 网络，但连接的设备将无法以结构化方式与微控制器内部的固件进行通信。ESP-IDF 框架已内置了一个名为 `esp_http_server` 的轻量级 HTTP 服务器库，这使得创建 Web 界面和 REST API 变得更加容易。

以下是一个基础示例，展示如何在 ESP32-S3 上创建并启动 HTTP 服务器，以及注册一个基本的 Web 端点：

```c
#include "esp_http_server.h"

// 1. 定义处理传入请求的处理函数
esp_err_t hello_get_handler(httpd_req_t *req) {
const char* resp_str = "来自 ESP32-S3 的问候！";

// 将响应发送回客户端
httpd_resp_send(req, resp_str, HTTPD_RESP_USE_STRLEN);

return ESP_OK;
}

// 2. 定义将 URL 链接到处理程序的 URI 结构
httpd_uri_t hello_uri = {
.uri      = "/hello",
.method   = HTTP_GET,
.handler  = hello_get_handler,
.user_ctx = NULL
};

// 3. 初始化并启动服务器的函数
httpd_handle_t start_webserver(void) {
httpd_handle_t server = NULL;

// 拉取默认服务器配置
httpd_config_t config = HTTPD_DEFAULT_CONFIG();

// 启动 HTTP 服务器
if (httpd_start(&server, &config) == ESP_OK) {
// 如果服务器启动成功，则注册 URI 处理程序
httpd_register_uri_handler(server, &hello_uri);
return server;
    }

返回 NULL； // 如果服务器启动失败，则返回 NULL
}
```

在服务器开始监听传入连接之前，代码会初始化其核心设置并启动底层任务：

*   **`httpd_config_t config = HTTPD_DEFAULT_CONFIG()`**：获取一组标准、安全的 Web 服务器内部配置。该操作处理后台细节，例如分配默认 Web 流量端口（端口 80）、设置最大开放套接字以及定义任务优先级。
*   **`httpd_start(&server, &config)`**：分配必要内存，启动后台 HTTP 服务器任务，并在我们之前创建的 Wi-Fi 网络接口上打开监听端口。若成功，它将正在运行的服务器实例分配给我们的 `server` 变量。

接下来，代码使用 `httpd_uri_t` 结构定义 Web 端点的行为。这决定了服务器应如何路由传入流量：

*   **URI（`"/hello"`）**：客户端将请求的特定网络路径或端点（例如 `[http://192.168.4.1/hello](http://192.168.4.1/hello)`）。
*   **方法（`HTTP_GET`）**：预期的 HTTP 请求类型。`HTTP_GET` 通常用于获取数据（如加载网页），而其他方法如 `HTTP_POST` 则用于接收来自客户端的数据。
*   **处理函数（`hello_get_handler`）**：指向自定义 C 函数的指针。当客户端访问此特定 URI 时，ESP32-S3 将自动执行该函数。
*   **用户上下文（`NULL`）**：一个可选指针，允许您将自定义数据或变量传递到处理函数中。此处我们将其留空。

在服务器运行且路由规则定义完成后，代码会将它们关联起来，并处理实际的客户端响应：

*   **`httpd_register_uri_handler(server, &hello_uri)`**：将我们新创建的 URI 结构绑定到活动服务器上。这充当了服务器的目录，告诉它在收到匹配的请求时具体触发哪个函数。
*   **`httpd_resp_send(req, resp_str, HTTPD_RESP_USE_STRLEN)`**：位于我们的处理函数内部，它接收原始字符串（`"Hello from ESP32-S3!"`），使用 `HTTPD_RESP_USE_STRLEN` 自动计算其长度，并将其打包成标准的 HTTP 响应。然后，它通过 Wi-Fi 无线电将这一有效载荷传输回发起请求的浏览器或设备。

### 处理请求

现在 HTTP 服务器正在运行，下一步是学习 ESP32 如何实际处理来自已连接设备的传入请求。

当浏览器、移动应用或其他客户端与 ESP32 通信时，它会向一个称为 URI（统一资源标识符）的特定地址发送 HTTP 请求。URI 的作用类似于路由或端点，用于告知服务器应执行哪项功能。
例如：

*   `/` → 主页面
*   `/led/on` → 打开 LED 灯
*   `/sensor` → 读取传感器数据
*   `/api/data` → 返回 JSON 信息

ESP32 HTTP 服务器允许我们将处理函数附加到这些 URI 上。每当客户端访问特定路由时，对应的处理函数会自动执行。

```c
#include "esp_http_server.h"

// 处理函数
esp_err_t hello_handler(httpd_req_t *req){
const char* response = "来自 ESP32 的问候！";

httpd_resp_send(req, response, HTTPD_RESP_USE_STRLEN);

返回 ESP_OK；
}

// URI 配置
httpd_uri_t hello_uri = {
.uri      = "/hello",
.method   = HTTP_GET,
.handler  = hello_handler,
.user_ctx = NULL
};
```

`hello_handler()` 函数负责生成发送回客户端的响应。在该函数内部，`httpd_resp_send()` 通过 HTTP 连接传输数据。

`httpd_uri_t` 结构体定义了路由的行为方式：

*   `.uri` 指定了端点地址。
*   `.method` 定义了 HTTP 方法，例如 `GET` 或 `POST`。
*   `.handler` 指向请求到达时执行的函数。
*   `.user_ctx` 可存储可选的用户自定义数据。

然而，仅定义 URI 是不够的。处理程序还必须注册到正在运行的 HTTP 服务器中。

```c
httpd_register_uri_handler(server, &hello_uri);
```

注册完成后，通过浏览器访问：

```
http://192.168.4.1/hello
```

连接到 ESP32 接入点的浏览器将触发该处理程序并返回：

```
Hello from ESP32!
```

#### 返回 HTML 页面

发送纯文本对于测试很有用，但大多数实际应用需要包含按钮、样式和交互内容的完整网页。

除了返回简单字符串外，ESP32 还可以直接将完整的 HTML 文档发送到浏览器。

```c
esp_err_t webpage_handler(httpd_req_t *req){
const char* html_page =
"<!DOCTYPE html>"
"<html>"
"<head>"
"<title>ESP32 网页服务器 </title>"
"</head>"
"<body>"
"<h1>ESP32-S3 网页服务器 </h1>"
"<p> 来自 ESP32 的问候！</p>"
"</body>"
"</html>";

httpd_resp_set_type(req, "text/html");
httpd_resp_send(req, html_page, HTTPD_RESP_USE_STRLEN);
return ESP_OK;
}
```

在发送响应之前，使用以下方式更改内容类型：

```c
httpd_resp_set_type(req, "text/html");
```

这告诉浏览器，响应中包含的是 HTML 而非纯文本。如果没有这个头部信息，浏览器会直接将原始 HTML 代码显示为文本。
当页面在浏览器中加载后，ESP32 就像一个小型网站服务器，能够托管仪表盘、控制面板和监控界面。

#### 处理 GET 请求

`GET` 方法主要用于客户端希望从 ESP32 获取信息时。
例如：

*   读取传感器数值
*   加载网页
*   请求系统状态
*   获取配置信息

每当打开网页时，浏览器会自动发送一个 `GET` 请求。

```c
httpd_uri_t sensor_uri = {
.uri = "/sensor",
.method = HTTP_GET,
.handler = sensor_handler,
.user_ctx = NULL
};
```

当用户访问 `/sensor` 时，`sensor_handler()` 函数会执行，并返回传感器读数或设备信息。

GET 请求简单且轻量，因为它们通常不包含大量传入数据。

#### 处理 POST 请求

与主要用于从服务器请求或检索数据的 `GET` 方法不同，`POST` 方法用于客户端需要向 ESP32 发送数据时。通过使用 `POST`，网页界面可以直接与 ESP32 通信，并安全高效地传输用户输入、配置值或命令。

以下是一个简单的 POST 请求处理程序：

```c
esp_err_t post_handler(httpd_req_t *req){
char buffer[100];

int received = httpd_req_recv(req, buffer, req->content_len);

if (received <= 0) {
return ESP_FAIL;
    }

buffer[received] = '\0';

printf("接收到的数据: %s\n", buffer);

httpd_resp_send(req, "数据已接收", HTTPD_RESP_USE_STRLEN);

return ESP_OK;
}
```

这里的关键部分是：

```c
httpd_req_recv()
```

该函数用于读取并获取客户端发送的传入数据。例如，一个网页表单可能会发送：

```c
led=on&motor=off
```

ESP32 接收此数据，进行处理，并执行所需的动作。

要将路由注册为 POST 端点，我们使用 `HTTP_POST`：

```c
httpd_uri_t post_uri = {
.uri = "/submit",
.method = HTTP_POST,
.handler = post_handler,
.user_ctx = NULL
};
```

#### 返回 JSON 响应

到目前为止，我们探讨了 ESP32 作为传统 Web 服务器生成完整 HTML 页面并推送到浏览器的场景。虽然这种方式适用于基础配置，但现代 Web 开发通常不采用这种模式。设备不再传输臃肿的预格式化用户界面，而是通过传递轻量级的原始**数据** ，让移动应用、前端框架（如 React 或 Vue）或桌面应用程序来渲染用户界面。

为了实现高效的数据交互，我们使用 **JSON**。

JSON，全称 JavaScript 对象表示法，是一种轻量级的基于文本的格式，用于存储和传输数据。你可以把它想象成互联网的通用货币。无论你的服务器是用 C 语言（如 ESP32）编写的，你的移动应用是用 Swift 编写的，还是你的前端是用 JavaScript 编写的，它们都能读写 JSON，而不会在转换过程中丢失任何信息。

JSON 完全建立在两种结构之上：

*   **键值对：** 一个键（必须是用双引号括起来的字符串）后跟一个冒号，然后是它的值。
*   **对象：** 用花括号 `{}` 包裹的这些键值对的集合。

以下是表示传感器读数的标准 JSON 数据格式：

```json
{
"temperature": 24.5,
"humidity": 60,
"device_name": "ESP32_LivingRoom",
"status_ok": true
}
```

以下是我们如何使用 ESP-IDF HTTP 服务器组件实现一个干净的 JSON 响应：

```c
esp_err_t json_handler(httpd_req_t *req){
const char* json_response =
"{"
"\"temperature\":24,"
"humidity": 60
}

httpd_resp_set_type(req, "application/json");

httpd_resp_send(req, json_response, HTTPD_RESP_USE_STRLEN);

返回 ESP_OK；
}
```

*   **`application/json`**：这是关键转变。通过修改 `Content-Type` 头部，浏览器或 API 客户端会立即知道应将此响应解析为对象，而非将其显示为纯文本行。
*   **字符串转义（`\"`）**：由于 C 语言使用双引号标记字符串的起始和结束，我们必须使用反斜杠（`\`）来告诉编译器：“嘿，这个引号属于我们正在发送的实际文本包中的内容。”