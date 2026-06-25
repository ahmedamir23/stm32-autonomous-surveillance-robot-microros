# STM32 Autonomous Surveillance Robot — micro-ROS Low-Level Controller

> Low-level embedded firmware for an autonomous surveillance robot, running on the STM32F407VGT6 with FreeRTOS and micro-ROS over USB CDC. Handles motor control, encoder feedback, sensor acquisition, CAN communication, RC-based manual/autonomous mode switching, and RGB status indication — bridging the embedded world with a ROS 2 host system.

---

## Overview

This repository contains the STM32CubeIDE firmware project for the embedded low-level controller of a differential-drive surveillance robot. The firmware runs on the **STM32F407VGT6** (ARM Cortex-M4 @ 168 MHz) and integrates **micro-ROS** to expose the robot's actuators and sensors as standard ROS 2 topics and services — allowing a companion computer running ROS 2 (e.g. a Raspberry Pi or Jetson) to command the robot and receive telemetry over a USB serial link.

### Key Capabilities

- Quadrature encoder reading on 3 independent timer channels (up to 6 wheels)
- analog sensor acquisition via ADC with DMA
- RC receiver input for seamless switching between manual teleoperation and autonomous navigation
- RGB LED status indicators for visual robot state feedback (idle, RC mode, autonomous mode, fault, etc.)
- CAN bus interface at 500 kbps for inter-board communication
- FreeRTOS task scheduling with a dedicated micro-ROS task and a main application task
- Watchdog (IWDG) for fail-safe autonomous operation
- USB CDC transport for micro-ROS agent communication (no extra UART hardware required)

---

## Hardware

| Component | Details |
|---|---|
| MCU | STM32F407VGT6 (LQFP-100) |
| Core | ARM Cortex-M4 @ 168 MHz |
| Flash / RAM | 1 MB / 192 KB |
| Motor PWM | TIM5 CH3 — DMA-driven PWM output |
| Encoder inputs | TIM2 (4ch), TIM3 (4ch), TIM4 (2ch) — input capture, both edges |
| Analog sensor | ADC1 CH13 (PC3) — DMA circular mode |
| CAN bus | CAN2 @ 500 kbps (PB12/PB13) |
| micro-ROS transport | USB OTG FS — CDC virtual COM port |
| Clock source | 8 MHz HSE → PLL → 168 MHz SYSCLK |
| RTOS | FreeRTOS (CMSIS-V2), static allocation |

---

## Software Architecture

```
┌───────────────────────────────────────────┐
│              FreeRTOS Kernel               │
│                                           │
│  ┌────────────────┐  ┌─────────────────┐  │
│  │  microrosTask  │  │   mainTask      │  │
│  │  (stack 3 KB)  │  │  (stack 3 KB)  │  │
│  │                │  │                 │  │
│  │  micro-ROS     │  │  Motor control  │  │
│  │  executor      │  │  PID loops      │  │
│  │  pub/sub       │  │  Sensor reads   │  │
│  │                │  │  RC mode switch │  │
│  │                │  │  RGB LEDs       │  │
│  └───────┬────────┘  └──────┬──────────┘  │
│          │                  │             │
│  ┌───────▼──────────────────▼──────────┐  │
│  │        HAL Drivers / Peripherals    │  │
│  │  TIM2/3/4 encoders │ TIM5 PWM      │  │
│  │  ADC1 DMA          │ CAN2          │  │
│  │  RC input (GPIO)   │ RGB GPIOs     │  │
│  │  USB CDC (transport)│ IWDG          │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
         │  USB CDC (micro-ROS agent)
         ▼
  ┌──────────────────┐
  │  ROS 2 Host PC   │
  │  micro-ROS agent │
  │  Nav2 / SLAM     │
  └──────────────────┘
```

### FreeRTOS Tasks

| Task | Priority | Stack | Role |
|---|---|---|---|
| `microrosTask` | 24 | 3 KB | Runs the micro-ROS executor, handles ROS 2 pub/sub |
| `mainTask` | 32 | 3 KB | Application logic: motor PID, sensor processing |
| `normaloperation_timer` | — | — | Periodic timer for normal operation watchdog callback |

---

## Repository Structure

```
├── Core/
│   ├── Src/          # Application source (main.c, tasks, PID, drivers)
│   └── Inc/          # Application headers
├── Drivers/          # STM32 HAL + CMSIS drivers (auto-generated)
├── Middlewares/      # FreeRTOS middleware (auto-generated)
├── USB_DEVICE/       # USB CDC class (micro-ROS transport)
├── micro_ros_stm32cubemx_utils/  # micro-ROS build system integration (submodule)
├── microros.ioc      # STM32CubeMX project configuration
├── STM32F407VGTX_FLASH.ld        # Linker script (flash)
└── STM32F407VGTX_RAM.ld          # Linker script (RAM)
```

---

## Getting Started

### Prerequisites

- [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) v1.18.1 or later
- [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html) (optional, for pin reconfiguration)
- ST-Link v2 or compatible debugger/programmer
- ROS 2 (Humble or later) installed on the companion host
- `micro-ROS agent` running on the host over the USB serial port

### Clone

```bash
git clone --recurse-submodules https://github.com/ahmedamir23/stm32-autonomous-surveillance-robot-microros.git
```

> The `--recurse-submodules` flag is required to pull in `micro_ros_stm32cubemx_utils`.

### Build

1. Open STM32CubeIDE and import the project (`File → Import → Existing Projects into Workspace`).
2. Select the cloned directory.
3. Build with **Project → Build All** (or `Ctrl+B`).
4. Flash to the board via **Run → Debug** or using ST-Link Utility.

### Running the micro-ROS Agent (Host Side)

On the ROS 2 host (companion computer), start the micro-ROS agent over the USB CDC port:

```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyACM0 -b 115200
```

Once connected, the STM32 will appear as a ROS 2 node and its topics will be discoverable via `ros2 topic list`.

---

## Peripheral Configuration Summary

| Peripheral | Function | Pins |
|---|---|---|
| TIM2 CH1–CH4 | Encoder input capture (both edges) | PA15, PA1, PB10, PB11 |
| TIM3 CH1–CH4 | Encoder input capture (both edges) | PA6, PA7, PB0, PB1 |
| TIM4 CH1–CH2 | Encoder input capture (both edges) | PB6, PB7 |
| TIM5 CH3 | PWM output (DMA-driven) | PA2 |
| ADC1 CH13 | Analog sensor (DMA circular) | PC3 |
| CAN2 | Inter-board CAN @ 500 kbps | PB12 (RX), PB13 (TX) |
| USB OTG FS | micro-ROS CDC transport | PA11 (DM), PA12 (DP) |
| IWDG | Watchdog for fail-safe operation | — |

---

## RC Control & Operating Modes

The robot supports two operating modes, switchable in real time via an RC receiver:

| Mode | Description |
|---|---|
| **RC / Manual** | The operator drives the robot directly using an RC transmitter. Useful for teleoperation, inspection, and testing. |
| **Autonomous** | The STM32 hands off control to the ROS 2 host (Nav2 / SLAM). The RC channel acts as a safety override — switching back to RC instantly cuts autonomous commands. |

The RC input is read via a GPIO interrupt or timer capture pin. The active mode is reflected on the RGB status LEDs (see below).

---

## RGB Status LEDs

Onboard RGB LEDs provide at-a-glance robot state feedback without needing a monitor or serial terminal:

| Color | State |
|---|---|
| 🔵 Blue | System initializing / micro-ROS connecting |
| 🟢 Green | Autonomous mode active, ROS 2 agent connected |
| 🟡 Yellow | RC / manual mode active |
| 🔴 Red | Fault or watchdog warning |
| Off | System idle / powered down |

LED colors are driven via GPIO output pins on the STM32 and controlled from the `mainTask`.
