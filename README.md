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

## ⚠️ Known Issues and Debug Summary

This section documents the key issues encountered when attempting to establish a stable micro-ROS communication over UDP on the STM32H723ZG using FreeRTOS and LwIP. It includes detailed debug logs, source references, and troubleshooting steps.

---

### 📌 1. UDP Packets Not Received on Host

- **Symptoms**:
  - `tcpdump` on host shows **no incoming UDP traffic** from STM32.
  - LwIP logs show packets being **buffered and queued**, but not transmitted.

- **Relevant Logs (SWV ITM Console)**:
  ```
  udp_send: sending datagram of length 32
  ip4_output_if: call netif->output()
  etharp_request: sending ARP request.
  etharp_query: queued packet 0x3000024c on ARP entry 0
  ```

- **Root Cause**: ARP resolution incomplete. STM32 keeps queuing packets while waiting for host ARP reply.

- **Key Source Files**:
  - `src/core/ipv4/etharp.c` → `etharp_request()`, `etharp_query()`
  - `src/core/udp.c` → `udp_send()`

- **Fix Suggestions**:
  - Ensure the host replies to ARP requests.

---

### 📌 2. UDP Payload Corruption or Memory Reuse

- **Symptoms**:
  - `pbuf_copy()` sometimes called on invalid or freed memory.
  - Payloads missing or replaced with garbage data.

- **Relevant Logs**:
  ```
  pbuf_copy: end of chain reached.
  pbuf_free(0x3000024c)
  ```

- **Root Cause**: Reuse of `pbuf` after freeing. Possible misalignment of memory regions or incorrect `pbuf_alloced_custom()` use.

- **Key Source Files**:
  - `src/core/pbuf.c` → `pbuf_alloc()`, `pbuf_chain()`, `pbuf_free()`
  - `lwipopts.h` → `MEM_SIZE`, `PBUF_POOL_SIZE`, `TCP_SND_BUF`

- **Fix Suggestions**:
  - Avoid reusing `pbufs` after `pbuf_free()`.
  - Validate no overlaps in memory buffers with descriptors (check `TxDescAddress`/`RxBufferAddress`).

---

### 📌 3. Socket API Succeeds, But Packets Not Sent

- **Symptoms**:
  - `lwip_socket()`, `lwip_bind()`, `lwip_sendto()` all succeed.
  - No data seen on wire.

- **Logs**:
  ```
  lwip_socket(PF_INET, SOCK_DGRAM, 0) = 0
  lwip_bind(0, addr=0.0.0.0 port=8888)
  lwip_sendto(...) [to 192.168.1.1:8888]
  ```

- **Possible Causes**:
  - `netif` not fully up (`netif_set_link_up()` not called).
  - ARP or PHY link not initialized.

- **Key Source**:
  - `ethernetif.c` (user file) → `low_level_init()`
  - `src/api/sockets.c`

- **Fix**:
  - Ensure `netif_set_up()` and `netif_set_link_up()` are called in `MX_LWIP_Init()`.

---

### 📌 4. MPU Conflicts with Ethernet Buffers

- **Symptoms**:
  - LwIP logs show:
    ```
    pbuf_alloced_custom(length=0)
    ```
  - Unexpected zero-length packets or buffer errors.

- **Cause**: Ethernet buffers located in cache-enabled or protected regions, causing data coherency issues.

- **Memory Settings** (from .ioc):
  - `RxBufferAddress = 0x30000200`
  - `TxDescAddress = 0x30000100`
  - MPU Region 1 & 2 use base `0x30000000` but may have limited or incorrect size attributes.

- **Fix**:
  - Use **non-cacheable memory** for Ethernet buffers (RAM_D2).
  - Double-check MPU config in `system_stm32h7xx.c`.
  - Disable DCache or manually clean/invalidate it on each transmission.

---

### 📌 5. FreeRTOS Static Memory Configuration Limits Available Heap

- **Context**:
  - `defaultTask` stack size = 4000
  - `configTOTAL_HEAP_SIZE = 30720`
  - Heap and stack placed close to LwIP RAM heap at `0x30000200`.

- **Risk**: Overlap of FreeRTOS heap with DMA or LwIP pbuf pools.

- **Fix Suggestions**:
  - Use separate memory regions for FreeRTOS and LwIP (e.g., place heap in `DTCM` or `RAM_D3`).
  - Review `.ld` linker script and `.ioc` memory settings.

---

### 🧪 Debug Tools & Methodology

| Tool        | Purpose |
|-------------|---------|
| **SWV ITM Console** | Real-time log of `printf`-like messages using `ITM_SendChar()` |
| **tcpdump** | To verify UDP traffic on host side |
| **STM32CubeMX** | Hardware configuration and MPU/memory planning |
| **LwIP Debug Macros** | All enabled via CubeMX `.ioc` file for granular logs |

---

## 📚 References

- [Micro-ROS Docs](https://micro.ros.org)
- [STM32H723ZG Reference Manual](https://www.st.com/resource/en/reference_manual/dm00314099.pdf)
- [How to create a project for STM32H7 with Ethernet and LwIP stack working](https://community.st.com/t5/stm32-mcus/how-to-create-a-project-for-stm32h7-with-ethernet-and-lwip-stack/ta-p/49308)

---

## 🧛 Contributing

If you encounter issues or would like to contribute (e.g., IPv6 support, DHCP client, etc.), feel free to submit a pull request or open an issue.

