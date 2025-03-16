# STM32H723ZG microROS UDP Integration
This repository provides an example project that demonstrates how to integrate microROS over UDP using lwIP and FreeRTOS on the STM32H723ZG Nucleo board.

## Features
- **Microcontroller:** STM32H723ZG Nucleo
- **Networking:** UDP transport using lwIP
- **RTOS:** FreeRTOS for task scheduling
- **microROS Integration:** Configured for microROS communication over UDP
- **Custom Memory Management:** MPU and linker script modifications tailored for the limited 32KB D2 SRAM

## Environment
- **STM32CubeIDE:** v1.16.1
- **Firmware:** STM32Cube FW H7 V1.11.2
- **Toolchain:** GNU Tools for STM32 (GCC 12.3.rel1)

## Project Structure
- **Core/Src/**: Source files including FreeRTOS tasks, lwIP configuration, and microROS interface.
- **Core/Inc/:** Header files and configuration definitions.
- **LinkerScript.ld:** Custom linker script with dedicated sections for Ethernet DMA descriptors and buffer relocation.
- **lwipopts.h:** lwIP configuration file with settings for MEM_SIZE, LWIP_RAM_HEAP_POINTER, and more.

## Getting Started
1. **Clone the Repository:**
  ```bash
  git clone https://github.com/tapererwaj/h723zg_microros_udp.git
  ```
2. **Open the Project:** <br>
  Open the project in STM32CubeIDE.
4. **Build and Flash:** <br>
  Build the project and flash the firmware onto your STM32H723ZG Nucleo board.
5. **Test TCP Communication:** <br>
  Try to ping the board on the configured IP (192.168.1.10).
6. **Integrate microROS:** <br>
  Use the provided UDP transport layer as a basis for microROS integration. Refer to the microROS documentation for further instructions.
