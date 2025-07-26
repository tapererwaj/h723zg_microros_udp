# STM32H723ZG Micro-ROS over Ethernet (UDP)

This repository contains the STM32CubeIDE project for running **Micro-ROS over UDP** on the **STM32H723ZG** Nucleo board using **LwIP** and **FreeRTOS**. The goal of this project is to establish real-time communication between the microcontroller and a ROS 2 agent via Ethernet, useful in robotics applications such as RoboCup and autonomous systems.

---

## 📦 Project Overview

- **Board**: STM32H723ZG (Nucleo-144)
- **Transport**: UDP over Ethernet (RMII + LAN8742A PHY)
- **Middleware**: Micro-ROS Client (via LwIP/FreeRTOS)
- **ROS 2 Agent**: Typically running on a host PC with ROS 2 Humble
- **Tools**:
  - STM32CubeIDE
  - STM32CubeMX (with LwIP and FreeRTOS middleware)
  - `micro_ros_setup` for building the agent

---

## 📁 Repository Structure

```
STM32H723_MicroROS_UDP/
├── Core/                  # STM32 main source and Micro-ROS transport setup
├── Drivers/               # HAL and BSP drivers
├── Middlewares/           # FreeRTOS, LwIP, Micro-ROS
├── micro_ros_config/      # DDS-XRCE client config (custom transport over LwIP)
├── .ioc                   # STM32CubeMX project file
├── FLASH.ld               # Linker script with custom LwIP buffer placement
├── README.md              # This file
```

---

## ⚙️ Setup Instructions

1. **Micro-ROS Agent (on Host)**:

   - Install Micro-ROS agent via `micro_ros_setup`:
     ```bash
     ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
     ```

2. **STM32 Board**:

   - Flash this firmware using STM32CubeIDE.
   - Ensure Ethernet is connected.
   - On boot, the board will try to communicate with the Micro-ROS agent via UDP.

3. **Network Configuration**:

   - Static IP (e.g., 192.168.1.100) or DHCP (if enabled in LwIP config).
   - Make sure firewall does not block UDP port `8888`.

---

## ⚠️ Known Issues & Troubleshooting

### 1. **LwIP Heap Overlap with Rx Buffers**

- Issue: `LWIP_RAM_HEAP_POINTER` was initially overlapping with `RxBufferAddress`.
- Fix: Shift heap to a safe address (e.g., `0x30000400`) in `.ioc` and linker script.

### 2. **MPU Conflicts**

- Issue: Multiple overlapping MPU regions for `RAM_D2`.
- Fix: Consolidated into a single 32KB region for ETH descriptors + LwIP heap, marked as non-cacheable.

### 3. **UDP Packet Drop or No Communication**

- Potential Causes:
  - LwIP heap too small.
  - MPU misconfiguration (cacheable memory for DMA).

---

## 📚 References

- [Micro-ROS Docs](https://micro.ros.org)
- [STM32H723ZG Reference Manual](https://www.st.com/resource/en/reference_manual/dm00314099.pdf)
- [How to create a project for STM32H7 with Ethernet and LwIP stack working](https://community.st.com/t5/stm32-mcus/how-to-create-a-project-for-stm32h7-with-ethernet-and-lwip-stack/ta-p/49308)

---

## 🧛 Contributing

If you encounter issues or would like to contribute (e.g., IPv6 support, DHCP client, etc.), feel free to submit a pull request or open an issue.

