
# ESP32-S3 Fundamentals
This course provides a structured introduction to embedded systems, electronics, and low-level microcontroller programming using the ESP32-S3 platform. It is designed for beginners who want to understand both the theoretical foundations of electronics and the practical development of real-world embedded applications with ESP-IDF.

The course is divided into 8 lectures, progressing from basic electrical concepts to multitasking, networking, and wireless communication on the ESP32-S3.

![Lectures](https://img.shields.io/badge/Lectures-8-00c896?style=flat-square)  
![Language](https://img.shields.io/badge/Language-C%20%2F%20C%2B%2B-c77dff?style=flat-square)  
![Level](https://img.shields.io/badge/Level-Beginner%20%E2%86%92%20Advanced-ffd166?style=flat-square)

---

# 📋 Table of Contents

- [Course Overview](#-course-overview)
- [Curriculum](#-curriculum)
    - Lecture 1 — Foundations
    - Lecture 2 — ESP-IDF Platform
    - Lecture 3 — GPIO, Sensors & Motors 
    - Lecture 4 — Displays & Devices
    - Lecture 5 — Communication Protocols
    - Lecture 6 — FreeRTOS & Memory
    - Lecture 7 — Wi-Fi Communication
    - Lecture 8 — Bluetooth & BLE
- [Prerequisites](#-prerequisites)
- [Learning Path](#-learning-path)

---

# 📚 Course Overview

|Category|Details|
|---|---|
|📚 **Lectures**|8 structured modules|
|🔌 **Hardware**|ESP32-S3 Development Board|
|💻 **Framework**|ESP-IDF|
|🎯 **Level**|Absolute Beginner → Advanced Embedded|
|⚙️ **Languages**|C / C++|
|📡 **Protocols**|UART · SPI · I²C · Wi-Fi · Bluetooth|
|🧠 **Topics**|GPIO · RTOS · Networking · Drivers · Memory|

---

# 📖 Curriculum

## Lecture 1 — Foundations: Electricity, Electronics & Microcontrollers

This lecture starts with the fundamentals of electricity and electronics, then gradually introduces semiconductor devices and microcontrollers.

### Topics Covered

- Static electricity and electric charge
- Voltage, current, resistance, and power
- Ohm’s Law and circuit analysis
- Series and parallel circuits
- Conductors, insulators, and semiconductors
- Diodes and transistors
- Digital vs analogue signals
- Introduction to embedded systems
- What is a microcontroller?
- ESP32-S3 architecture overview

---

## Lecture 2 — ESP-IDF Platform: Setup, IDE & Programming Basics

Learn how to configure the ESP-IDF development environment and write your first ESP32-S3 applications.

### Topics Covered

- Exploring the ESP32 family and ESP32-S3 features
- Installing and configuring ESP-IDF
- Flashing firmware to the ESP32-S3
- Project structure in ESP-IDF
- `app_main()` entry point
- Variables and data types
    - `int`
    - `float`
    - `bool`
    - `char`
- Operators and expressions
- Control flow
    - `if / else`
    - `switch`
    - `for`
    - `while`

---

## Lecture 3 — GPIO, Sensors, Actuators & Motors

Learn how to interface the ESP32-S3 with external hardware using GPIO, ADC, PWM, and drivers.

### GPIO & Analogue Features

- GPIO input and output
- `gpio_set_level()` and `gpio_get_level()`
- Pull-up and pull-down resistors
- ADC analogue input
- PWM using LEDC
- PWM frequency and duty cycle

### Sensors & Output Devices
- LEDs and brightness control
- Buzzers and tone generation
- Push buttons
- LDR (light sensor)
- Flame sensor
- Obstacle avoidance sensor
- Ultrasonic distance sensor (HC-SR04)

### Motors
- DC motors with L298N driver
- PWM speed control
- Direction control
- Servo motor control
- Stepper motor basics
- Brushless DC motors (ESC control)

---

## Lecture 4 — Displays & Devices

Learn how to interface common display modules with the ESP32-S3.

### Topics Covered

- 7-segment displays
    - Common anode and common cathode
    - Multiplexing
- MAX7219 LED dot matrix
- LCD 16×2 displays

---

## Lecture 5 — Communication Protocols

Understand how embedded systems communicate with peripherals and other devices.

### Communication Protocols

|Protocol|Speed|Wires|Common Use|
|---|---|---|---|
|UART|Up to 115200+ baud|TX/RX|Debugging, modules|
|I²C|100–400 kHz|SDA/SCL|Sensors, displays|
|SPI|Up to 10+ MHz|MOSI/MISO/SCK/CS|Displays, SD cards|

### Topics Covered

- UART communication
- Serial debugging
- I²C master communication
- SPI communication


---

## Lecture 6 — FreeRTOS & Memory

Learn multitasking, task scheduling, and memory management on the ESP32-S3.

### Topics Covered

### FreeRTOS

- Tasks and multitasking
- Task priorities
- Delays and timing
- Queues
- Semaphores
- Mutexes
- Dual-core processing basics

### Memory Management

- Stack vs heap
- SRAM and Flash memory
- Memory regions on ESP32-S3
- Dynamic memory allocation
- PSRAM basics

---

## Lecture 7 — Wi-Fi Communication

Build network-enabled embedded applications using the ESP32-S3 Wi-Fi capabilities.

### Topics Covered

- Wi-Fi station mode
- Access Point (AP) mode
- Connecting to a router
- TCP/IP basics
- HTTP server implementation
- HTTP request methods
    - `GET`
    - `POST`
- JSON data handling
- REST APIs
- Wi-Fi event handling

---

## Lecture 8 — Bluetooth & BLE

Learn wireless short-range communication using Bluetooth Classic and BLE.

### Topics Covered

- Bluetooth Classic overview
- Bluetooth Low Energy (BLE)
- BLE services and characteristics
- ESP32-S3 BLE server
- ESP32-S3 BLE client
- Sending sensor data over BLE
- Mobile app communication
- Wireless device control

---

# 🛠 Prerequisites

- No prior programming experience required
- ESP32-S3 development board
- USB-C cable
- Breadboard and jumper wires
- Basic electronics components
    - LEDs
    - Resistors
    - Push buttons
    - Buzzers
- Sensors and modules used throughout the course
- VS Code installed
- ESP-IDF installed and configured

---

# 🗺 Learning Path

Each lecture builds directly on the previous one.

```
[L1] Electricity & Electronics          ↓[L2] ESP-IDF & C Programming Basics          ↓[L3] GPIO · Sensors · Motors          ↓[L4] Displays & External Devices          ↓[L5] Communication Protocols          ↓[L6] FreeRTOS & Memory Management          ↓[L7] Wi-Fi Networking          ↓[L8] Bluetooth & BLE
```

- **Lecture 1** builds the electronics foundation needed for embedded systems.
- **Lecture 2** introduces ESP-IDF and low-level programming concepts.
- **Lecture 3** focuses on hardware interfacing and control.
- **Lecture 4** introduces display technologies and visual output.
- **Lecture 5** teaches device communication protocols.
- **Lecture 6** explains multitasking and memory management.
- **Lecture 7** enables internet-connected embedded applications.
- **Lecture 8** introduces wireless communication using Bluetooth and BLE.

---

# 🚀 Goals of This Course

By the end of this course, students will be able to:

- Understand electronics fundamentals
- Program the ESP32-S3 using ESP-IDF
- Interface sensors, displays, and motors
- Use communication protocols like UART, SPI, and I²C
- Build multitasking applications with FreeRTOS
- Create Wi-Fi and BLE-enabled embedded systems
- Develop real-world IoT applications

---

## 💖 Support This Project
If you find this course helpful and would like to support its development, you can contribute via PayPal:

PayPal: alitighiouart2001@gmail.com

Your support helps improve the content, add more projects, and continue building free educational resources for ESP32-S3 and embedded systems learners.

_© 2026 ESP32-S3 Fundamentals — All rights reserved_