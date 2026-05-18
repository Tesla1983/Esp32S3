## Objectives

- Exploring Wi-Fi Capabilities on the ESP32-S3
- Configuring Access Point (AP) Mode 
- Configuring Station (STA) Mode 
## Wi-Fi on the ESP32-S3
In modern embedded systems, connectivity is just as important as processing power. Wi-Fi is a wireless networking technology that allows devices to communicate over local area networks using the IEEE 802.11 standards. By integrating Wi-Fi, our microcontroller transforms from an isolated calculator into an Internet of Things (IoT) device, capable of syncing with cloud servers, providing web-based user interfaces, and communicating with other smart devices.

The ESP32-S3 features a highly integrated Wi-Fi radio and MAC (Media Access Control). In the ESP-IDF framework, the Wi-Fi driver relies on several core system components, primarily Non-Volatile Storage (NVS) to save network credentials, and the Event Loop to manage asynchronous network events like connecting, disconnecting, or receiving an IP address.

The Wi-Fi peripheral can operate in different modes depending on the project's needs:
- **Station (STA) Mode:** The ESP32-S3 acts as a client and connects to an existing router.
- **Access Point (AP) Mode:** The ESP32-S3 creates its own Wi-Fi network, allowing other devices like smartphones or laptops to connect directly to it.
- **AP+STA Mode:** The chip performs both roles simultaneously.

Wi-Fi communication on the ESP32-S3 is handled through the ESP-IDF networking stack, which is built on top of TCP/IP and FreeRTOS.

### The Wi-Fi Protocol
Wi-Fi is a wireless communication technology based on the IEEE 802.11 standard. Instead of using physical Ethernet cables, devices communicate by transmitting data through radio waves.

A typical Wi-Fi network contains:

- A **router** or **access point (AP)** that manages the network
- Multiple **client devices** such as smartphones, laptops, sensors, or microcontrollers like the ESP32-S3

When devices connect to the same Wi-Fi network, they can exchange data using network protocols and unique IP addresses. When a device joins a network:
1. The router authenticates the device
2. The device receives an IP address
3. Data is exchanged in the form of packets
#### TCP/IP and HTTP
Wi-Fi only provides the wireless connection between devices. It acts like a road that allows devices to send signals through the air. However, Wi-Fi alone does not define how data is organized, transmitted, delivered, or understood between devices.  
To make communication possible, the ESP32-S3 uses higher-level networking protocols, mainly:
- **TCP/IP** handles device-to-device communication over a network
- **HTTP** handles communication between web clients and web servers
#### TCP/IP 
TCP/IP is the foundational framework that allows devices to talk to each other across any network. It is a tag-team of two distinct protocols:

- **Internet Protocol (IP) The Mail Delivery:** Focuses on addressing and routing. It assigns every device a digital address (e.g., `192.168.1.50`) and chops data into smaller packets* to ship them out. It does not guarantee they arrive safely.
- **Transmission Control Protocol (TCP) The Quality Control:** Sits on top of IP to ensure reliability. It establishes a solid connection before sending data, tracks packets, rearranges them into the correct order, and demands a re-send if anything gets lost or corrupted.

In the ESP32-S3, TCP/IP is integrated into the ESP-IDF networking stack, which handles all low-level network communication automatically. Once the device connects to Wi-Fi, it can obtain an IP address, establish TCP connections, and communicate with other devices without needing manual control of packet handling.

#### HTTP (Web Communication Layer)
HTTP, or HyperText Transfer Protocol, is built on top of TCP/IP and is used specifically for web-based communication. While TCP/IP handles how data is delivered, HTTP defines what the data means and how it should be structured when exchanged between systems such as web browsers, servers, and IoT devices.

HTTP follows a client-server model, where one device sends a request and another device responds. The client requests a resource or sends data, and the server processes that request and returns a response. This structured communication makes it possible for devices to interact in a predictable and standardized way.

For example, when the ESP32-S3 acts as an HTTP client, it can send sensor data to a web server using an HTTP request. The request contains both the data and instructions about what action should be performed, such as storing the data or analyzing it. The server then responds with a status message confirming whether the request was successful or returning additional information.

HTTP uses different methods to define how data is exchanged between a client and a server. The most common are **GET**, **POST**, and **PUT**.

- **GET** is used to retrieve data from a server without changing anything. It is mainly used for reading information.  
- **POST** is used to send data to a server, such as uploading sensor values like temperature or humidity.  
- **PUT** is used to update existing data on the server.

### Access Point (AP) Mode
When the ESP32-S3 is configured in Access Point (AP) mode, it creates its own wireless network and broadcasts a Service Set Identifier (SSID). Nearby devices such as smartphones, laptops, or tablets can scan for this SSID and connect directly to the ESP32-S3 in the same way they would connect to a standard Wi-Fi router.

In this mode, the ESP32-S3 can also run an HTTP server. This allows the microcontroller to function as a small web server capable of hosting web pages, receiving data from web forms, and communicating with mobile or web applications. As a result, users can interact with the ESP32-S3 through a browser interface without requiring an external Internet connection or router.

#### Initilizing and Setting the Mode
Before devices can connect to the ESP32-S3, the Wi-Fi driver must be initialized and configured in **Access Point (AP) mode**. Here example how we can initialize the Wi-Fi stack, sets the ESP32-S3 to AP mode, and configures basic network credentials.
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
Before the Wi-Fi radio can even turn on, the code initializes the core software environment:
- **`nvs_flash_init()`**: Sets up Non-Volatile Storage (NVS). The ESP32's Wi-Fi driver uses this persistent memory under the hood to store calibration data and internal Wi-Fi configurations.
- **`esp_netif_init()`**: Initializes the underlying TCP/IP networking stack, which allows the chip to manage IP addresses and network packets.
- **`esp_event_loop_create_default()`**: Creates a system-wide event loop. Because Wi-Fi operations happen asynchronously (like a device connecting or disconnecting), this loop sends out alerts that your code can react to later.
- **`esp_netif_create_default_wifi_ap()`**: Creates a default network interface specifically tailored for an Access Point, binding the TCP/IP stack to the Wi-Fi hardware.
- **`esp_wifi_init(&cfg)`**: The code pulls a set of standard, safe internal configurations using `WIFI_INIT_CONFIG_DEFAULT()` and passes them to the Wi-Fi init function. This allocates the necessary memory, starts the Wi-Fi task, and wakes up the Wi-Fi driver.

Next, the code defines the personality of your hotspot using the `wifi_config_t` structure:
- **SSID (`"ESP32_AP"`)**: This is the public name of the Wi-Fi network that devices will see when scanning.
- **Password (`"mypassword123"`)**: The security key required to join the network.
- **Max Connections (`4`)**: Restricts the network to a maximum of 4 simultaneously connected client devices, saving RAM and processing power.
- **Authentication Mode (`WIFI_AUTH_WPA_WPA2_PSK`)**: Enforces standard WPA/WPA2 security. The subsequent `if` statement acts as a safety check: if the password string is left completely empty, it automatically downgrades the security mode to `WIFI_AUTH_OPEN`, allowing anyone to connect without a password.

After defining the Access Point configuration, the ESP32 is switched into AP mode, the configuration is applied, and the Wi-Fi service is started.

- **`esp_wifi_set_mode(WIFI_MODE_AP)`**: Sets the ESP32 to operate exclusively in Access Point mode, allowing it to create and broadcast its own wireless network instead of connecting to another router as a client.
- **`esp_wifi_set_config(WIFI_IF_AP, &ap_config)`**: Applies the previously defined Access Point settings, including the SSID, password, authentication mode, and maximum number of connected clients.
- **`esp_wifi_start()`**: Starts the Wi-Fi driver and enables the ESP32 radio hardware to begin broadcasting the configured network and accepting incoming client connections.

#### Creating and Starting the HTTP Server
At this point, devices can successfully connect to the ESP32-S3 Wi-Fi network. However, simply joining the network is not enough for communication to happen. The ESP32 still needs a way to receive requests and send responses back to connected devices. This is where the HTTP server comes in.    
An HTTP server allows the ESP32-S3 to behave like a tiny web server. Once the server is running, phones, laptops, or browsers connected to the ESP32 Access Point can send HTTP requests such as:
- Opening a web page hosted by the ESP32
- Sending commands to control LEDs or sensors
- Requesting sensor readings or system information
- Exchanging data with the microcontroller through APIs

Without an HTTP server, the ESP32 would create a Wi-Fi network, but connected devices would have no structured way to communicate with the firmware running inside the microcontroller. The ESP-IDF framework already provides a lightweight built-in HTTP server library called `esp_http_server`, which makes creating web interfaces and REST APIs much easier.

Here is a basic example showing how to create and start an HTTP server, and register a basic web endpoint on the ESP32-S3:
```c
#include "esp_http_server.h"

// 1. Define the handler function for incoming requests
esp_err_t hello_get_handler(httpd_req_t *req) {
    const char* resp_str = "Hello from ESP32-S3!";
    
    // Send the response back to the client
    httpd_resp_send(req, resp_str, HTTPD_RESP_USE_STRLEN);
    
    return ESP_OK;
}

// 2. Define the URI structure linking the URL to the handler
httpd_uri_t hello_uri = {
    .uri      = "/hello",
    .method   = HTTP_GET,
    .handler  = hello_get_handler,
    .user_ctx = NULL
};

// 3. Function to initialize and start the server
httpd_handle_t start_webserver(void) {
    httpd_handle_t server = NULL;
    
    // Pull default server configuration
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();

    // Start the HTTP server
    if (httpd_start(&server, &config) == ESP_OK) {
        // Register the URI handler if the server starts successfully
        httpd_register_uri_handler(server, &hello_uri);
        return server;
    }

    return NULL; // Return NULL if server failed to start
}
```
Before the server can begin listening for incoming connections, the code initializes its core settings and starts the underlying task:
- **`httpd_config_t config = HTTPD_DEFAULT_CONFIG()`**: Pulls a set of standard, safe internal configurations for the web server. This handles background details like assigning the default web traffic port (Port 80), setting maximum open sockets, and defining task priorities.
- **`httpd_start(&server, &config)`**: Allocates the necessary memory, spawns the background HTTP server task, and opens the listening port on the Wi-Fi network interface we previously created. If successful, it assigns the running server instance to our `server` variable.

Next, the code defines the behavior of our web endpoint using the `httpd_uri_t` structure. This dictates how the server should route incoming traffic:
- **URI (`"/hello"`)**: The specific network path or endpoint that the client will request (e.g., `[http://192.168.4.1/hello](http://192.168.4.1/hello)`).
- **Method (`HTTP_GET`)**: The type of HTTP request expected. `HTTP_GET` is universally used for retrieving data (like loading a web page), whereas other methods like `HTTP_POST` are used for receiving data from a client.
- **Handler (`hello_get_handler`)**: A pointer to the custom C function that the ESP32-S3 will automatically execute whenever a client navigates to this exact URI.
- **User Context (`NULL`)**: An optional pointer allowing you to pass custom data or variables into the handler function. We leave it empty here.

After the server is running and the routing rules are defined, the code links them together and handles the actual client response:
- **`httpd_register_uri_handler(server, &hello_uri)`**: Binds our newly created URI structure to the active server. This acts as the server's directory, telling it exactly which function to trigger when a matching request comes in.
- **`httpd_resp_send(req, resp_str, HTTPD_RESP_USE_STRLEN)`**: Found inside our handler function, this takes our raw string (`"Hello from ESP32-S3!"`), automatically calculates its length using `HTTPD_RESP_USE_STRLEN`, and packages it into a standard HTTP response. It then transmits this payload back over the Wi-Fi radio to the browser or device that requested it.

### Handling Requests
Now that the HTTP server is running, the next step is learning how the ESP32 actually handles incoming requests from connected devices.

When a browser, mobile application, or another client communicates with the ESP32, it sends an HTTP request to a specific address called a URI (Uniform Resource Identifier). A URI works like a route or endpoint that tells the server which functionality should be executed.     
For example:
- `/` → Main homepage
- `/led/on` → Turn on LED
- `/sensor` → Read sensor data
- `/api/data` → Return JSON information

The ESP32 HTTP server allows us to attach handler functions to these URIs. Whenever a client accesses a specific route, the corresponding handler function is automatically executed.
```c
#include "esp_http_server.h"

// Handler function
esp_err_t hello_handler(httpd_req_t *req){
    const char* response = "Hello from ESP32!";
    
    httpd_resp_send(req, response, HTTPD_RESP_USE_STRLEN);

    return ESP_OK;
}

// URI configuration
httpd_uri_t hello_uri = {
    .uri      = "/hello",
    .method   = HTTP_GET,
    .handler  = hello_handler,
    .user_ctx = NULL
};
```
The `hello_handler()` function is responsible for generating the response sent back to the client. Inside the function, `httpd_resp_send()` transmits data through the HTTP connection.

The `httpd_uri_t` structure defines how the route behaves:
- `.uri` specifies the endpoint address.
- `.method` defines the HTTP method such as `GET` or `POST`.
- `.handler` points to the function executed when the request arrives.
- `.user_ctx` can store optional custom user data.

However, defining the URI alone is not enough. The handler must also be registered with the running HTTP server.
```c
httpd_register_uri_handler(server, &hello_uri);
```
Once registered, visiting:
```
http://192.168.4.1/hello
```
from a browser connected to the ESP32 Access Point will trigger the handler and return:
```
Hello from ESP32!
```
#### Returning HTML Pages
Sending plain text is useful for testing, but most real applications need proper web pages with buttons, styles, and interactive content.

Instead of returning simple strings, the ESP32 can also send full HTML documents directly to the browser.
```c
esp_err_t webpage_handler(httpd_req_t *req){
    const char* html_page =
        "<!DOCTYPE html>"
        "<html>"
        "<head>"
        "<title>ESP32 Web Server</title>"
        "</head>"
        "<body>"
        "<h1>ESP32-S3 Web Server</h1>"
        "<p>Hello from the ESP32!</p>"
        "</body>"
        "</html>";

    httpd_resp_set_type(req, "text/html");
    httpd_resp_send(req, html_page, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}
```
Before sending the response, the content type is changed using:
```c
httpd_resp_set_type(req, "text/html");
```
This tells the browser that the response contains HTML instead of plain text. Without this header, the browser would simply display the raw HTML code as text.   
Once the page is loaded in a browser, the ESP32 behaves like a miniature website server capable of hosting dashboards, control panels, and monitoring interfaces.

#### Working with GET Requests
The `GET` method is mainly used when the client wants to retrieve information from the ESP32.   
For example:   
- Reading sensor values
- Loading web pages
- Requesting system status
- Fetching configuration information

A browser automatically sends a `GET` request whenever a webpage is opened.
```c
httpd_uri_t sensor_uri = {
    .uri = "/sensor",
    .method = HTTP_GET,
    .handler = sensor_handler,
    .user_ctx = NULL
};
```
When the user accesses `/sensor`, the `sensor_handler()` function executes and can return sensor readings or device information.

GET requests are simple and lightweight because they usually do not contain large amounts of incoming data.

#### Working with POST Requests
Unlike the `GET` method, which is mainly used to request or retrieve data from a server, the `POST` method is used when the client needs to send data to the ESP32. By using `POST`, a web interface can communicate directly with the ESP32 and transfer user input, configuration values, or commands securely and efficiently.

Here is a simple POST request handler:
```c
esp_err_t post_handler(httpd_req_t *req){
    char buffer[100];

    int received = httpd_req_recv(req, buffer, req->content_len);

    if (received <= 0) {
        return ESP_FAIL;
    }

    buffer[received] = '\0';

    printf("Received Data: %s\n", buffer);

    httpd_resp_send(req, "Data Received", HTTPD_RESP_USE_STRLEN);

    return ESP_OK;
}
```
The important part here is:
```c
httpd_req_recv()
```
This function reads and retrive the incoming data sent by the client.  For example, a webpage could form send:
```c
led=on&motor=off
```
The ESP32 receives this data, processes it, and performs the required action.

To register the route as a POST endpoint we use `HTTP_POST`:
```c
httpd_uri_t post_uri = {
    .uri = "/submit",
    .method = HTTP_POST,
    .handler = post_handler,
    .user_ctx = NULL
};
```
#### Returning JSON Responses
Up to this point, we have looked at the ESP32 acting as a traditional web server generating whole HTML pages and pushing them to a browser. While that works for basic setups, modern web development doesn't usually work this way. Instead of sending bulky, pre-formatted user interfaces, devices pass raw, lightweight **data** back and forth, leaving the user interface to be rendered by a mobile app, a frontend framework (like React or Vue), or a desktop application.

To do this efficiently, we use **JSON**.

JSON, which stands for JavaScript Object Notation, is a lightweight, text-based format used for storing and transporting data. Think of it as the universal currency of the internet. Whether your server is written in C (like the ESP32), your mobile app is written in Swift, or your frontend is written in JavaScript, they can all read and write JSON without losing anything in translation.

JSON is built entirely on two structures:
- **Key-Value Pairs:** A key which must be a string in double quotes followed by a colon, and then its value.
- **Objects:** Collections of these pairs wrapped in curly braces `{}`.

Here is what a standard JSON payload looks like representing sensor readings:
```json
{
  "temperature": 24.5,
  "humidity": 60,
  "device_name": "ESP32_LivingRoom",
  "status_ok": true
}
```
Here is how we implement a clean JSON response using the ESP-IDF HTTP server component:

```c
esp_err_t json_handler(httpd_req_t *req){
    const char* json_response =
        "{"
        "\"temperature\":24,"
        "\"humidity\":60"
        "}";

    httpd_resp_set_type(req, "application/json");

    httpd_resp_send(req, json_response, HTTPD_RESP_USE_STRLEN);

    return ESP_OK;
}
```
- **`application/json`**: This is the crucial shift. By altering the `Content-Type` header, a browser or API client immediately knows to parse this response as an object rather than displaying it as a flat line of text.
- **String Escaping (`\"`)**: Because C uses double quotes to mark the beginning and end of strings, we have to use a backslash (`\`) to tell the compiler, _"Hey, this quote belongs inside the actual text package we are sending."_
