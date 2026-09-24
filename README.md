# 🎛️ Dual-STM32 Potentiometer to Servo Controller (DMA & UART IDLE Line)

[![STM32](https://img.shields.io/badge/Hardware-STM32F446RE%20%7C%20STM32F072RB-002B49?logo=stmicroelectronics)](https://www.st.com/)
[![Framework](https://img.shields.io/badge/Framework-STM32%20HAL-blue.svg)](https://www.st.com/)
[![Language](https://img.shields.io/badge/Language-C11-green.svg)](https://en.wikipedia.org/wiki/C11_(C_standard_revision))
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Master-Slave embedded control system between two STM32 microcontrollers. The Master reads a potentiometer using **ADC + DMA (Circular Mode)** and sends the value over UART. The Slave receives it using **UART DMA + IDLE Line detection** and drives an RC servo with **hardware PWM (TIM2)**.

---

## 📌 System Architecture & Overview

* **Master (STM32F446RE):** Continuously samples the potentiometer in the background using **ADC1 + DMA (Circular Mode)**. Every 50 ms, the main loop reads the latest sample from the DMA buffer, maps the 12-bit value (0 – 4095) to a servo pulse width (500 – 2500 µs), and transmits it over **USART1** as an ASCII string terminated by `\n` (e.g. `1500\n`).
* **Slave (STM32F072RB):** Receives packets using **USART1 DMA + IDLE Line detection** (`HAL_UARTEx_ReceiveToIdle_DMA`). When the line goes idle after a packet, the callback parses the ASCII value with `atoi()`, validates it, and writes it directly to the **TIM2 Channel 1** compare register (`CCR1`) to update the PWM pulse width.

### Data Flow

```text
Potentiometer → ADC1 → DMA (circular) → main loop (every 50 ms)
    → map 0-4095 to 500-2500 µs → ASCII "1500\n" → USART1 TX
        ─────────────── UART (115200 8N1) ───────────────
    → USART1 RX + DMA → IDLE Line event → atoi() → range check
    → TIM2->CCR1 → 50 Hz PWM → Servo
```

---

## 🚀 Key Features

- 🔄 **ADC DMA Circular Mode (Master):** The ADC samples continuously into SRAM without CPU involvement; the latest value is always available in `adc_dma_buffer[0]`.
- 📡 **UART IDLE Line Detection (Slave):** `HAL_UARTEx_ReceiveToIdle_DMA` handles variable-length packets and triggers a callback as soon as the transmission ends. No fixed packet length and no polling.
- ⚡ **Interrupt-Driven Slave:** The Slave's main loop is empty. Reception is handled by DMA and the IDLE interrupt, and PWM generation is handled entirely by the timer hardware.
- 🛡️ **Software Clamping:** The Slave discards any value outside the 500 – 2500 µs range, protecting the servo from noise and corrupted packets.
- 🔁 **Error Recovery:** `HAL_UART_ErrorCallback` restarts the DMA reception after overrun/framing errors so the receiver never stays stuck.
- 💡 **Visual Diagnostics:** On the Master, `LD2` is on while a packet is being transmitted. On the Slave, `LD2` toggles on every valid packet received.

> **Note:** The Master's main loop uses `HAL_Delay(50)` for timing and `HAL_UART_Transmit()` in polling mode. The ADC acquisition itself is non-blocking (DMA), but the transmission loop is not. See [Possible Improvements](#-possible-improvements).

---

## 🔌 Hardware Wiring & Pinout

```text
  +----------------------+                     +----------------------+
  |   MASTER (F446RE)    |                     |    SLAVE (F072RB)    |
  |                      |                     |                      |
  |  PA0 <--- Pot. Wiper |                     |  PA10 <--------------+-- (Data from Master PA9 TX)
  |  PA9 (USART1_TX) ----+-------------------->|                      |
  |  GND ----------------+-------------------->|  GND (Common Ground) |
  +----------------------+       |             |  PA0  (TIM2_CH1) ----+-- (Servo Signal)
                                 |             +----------------------+
                                 |
                                === Common GND
```

### Pin Configuration

| Board | Pin | Function | Description |
| :--- | :--- | :--- | :--- |
| **Master (F446RE)** | `PA0` | `ADC1_IN0` | Potentiometer analog input |
| | `PA9` | `USART1_TX` | Serial data output |
| | `PA5` | `GPIO_Output` | On-board LED (LD2), transmit indicator |
| **Slave (F072RB)** | `PA10` | `USART1_RX` | Serial data input |
| | `PA0` | `TIM2_CH1` | Servo PWM signal (50 Hz) |
| | `PA5` | `GPIO_Output` | On-board LED (LD2), packet received indicator |

> ### ⚠️ CRITICAL HARDWARE NOTES
>
> - **Common Ground:** The `GND` pins of both boards must be connected together. A floating ground will corrupt UART packets.
> - **Servo Power Supply:** Power the servo's `+5V` and `GND` lines from an **external regulated 5V supply**, not from the STM32 board. High-current servos can cause voltage drops and MCU brownout resets. The external supply must share a common ground with both boards.
> - **Potentiometer:** Connect the outer pins to `3.3V` and `GND`, and the wiper to `PA0` on the Master. Do not use 5V.

---

## ⚙️ Peripheral Configuration (CubeMX)

### 1. Master: STM32F446RE

| Peripheral | Setting | Value |
| :--- | :--- | :--- |
| **Clock** | Source | HSI (16 MHz) → PLL (M=8, N=96, P=2) |
| | SYSCLK | 96 MHz |
| | AHB prescaler | /2 → HCLK = 48 MHz |
| | APB1 / APB2 | 24 MHz / 48 MHz |
| **ADC1** | Channel | `IN0` (PA0) |
| | Resolution | 12-bit, right aligned |
| | Continuous Conversion Mode | **Enabled** |
| | DMA Continuous Requests | **Enabled** |
| | Sampling Time | 3 cycles |
| **DMA2 Stream 0** | Request | `ADC1` |
| | Mode | **Circular** |
| | Data Width | Half Word / Half Word |
| **USART1** | Baud Rate | 115200, 8N1, TX/RX |

### 2. Slave: STM32F072RB

| Peripheral | Setting | Value |
| :--- | :--- | :--- |
| **Clock** | Source | HSI48, no PLL |
| | SYSCLK / HCLK / APB1 | 48 MHz |
| **TIM2** | Channel 1 | PWM Generation (`TIM_OCMODE_PWM1`) |
| | Prescaler (PSC) | `47` → 48 MHz / 48 = **1 MHz** (1 µs per tick) |
| | Period (ARR) | `19999` → 20 ms = **50 Hz** |
| | Pulse (CCR1) | `1500` (center position at startup) |
| | Auto-reload preload | Enabled |
| **USART1** | Baud Rate | 115200, 8N1, TX/RX |
| | Global interrupt (NVIC) | **Enabled** (required for IDLE line detection) |
| **DMA1 Channel 3** | Request | `USART1_RX` |
| | Mode | **Normal** |
| | Data Width | Byte / Byte |
| | Interrupt | `DMA1_Channel2_3_IRQn` enabled |

---

## 📡 Communication Protocol

| Property | Value |
| :--- | :--- |
| Baud rate | 115200 bps, 8N1 |
| Format | ASCII decimal number followed by `\n` |
| Value range | 500 – 2500 (servo pulse width in µs) |
| Send interval | 50 ms (20 packets per second) |
| Example packet | `1500\n` |

**Slave-side validation:**

1. The IDLE line event delivers the number of received bytes (`Size`).
2. A null terminator is appended and the string is converted with `atoi()`.
3. If the value is within `500 – 2500`, it is written to `TIM2->CCR1` and `LD2` toggles.
4. Otherwise the packet is silently discarded.
5. DMA reception is restarted immediately for the next packet.

---

## 📂 Project Structure

```bash
├── Master_F446RE/
│   ├── Core/
│   │   ├── Inc/
│   │   └── Src/
│   │       ├── main.c           # ADC DMA sampling, mapping & USART transmission
│   │       ├── stm32f4xx_it.c
│   │       └── stm32f4xx_hal_msp.c
│   └── Master_F446RE.ioc        # STM32CubeMX configuration file
│
└── Slave_F072RB/
    ├── Core/
    │   ├── Inc/
    │   └── Src/
    │       ├── main.c           # UART IDLE DMA callback & hardware PWM control
    │       ├── stm32f0xx_it.c
    │       └── stm32f0xx_hal_msp.c
    └── Slave_F072RB.ioc         # STM32CubeMX configuration file
```

---

## 🛠️ Build & Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/muhammedeminatasever-hash/-Dual-STM32-Potentiometer-to-Servo-Controller-DMA-UART-IDLE-Line-.git
   ```

2. Open **STM32CubeIDE**.
3. Select **File → Open Projects from File System...** and import both `Master_F446RE` and `Slave_F072RB`.
4. Connect the wiring as shown in the schematic above (including the common ground and the external servo supply).
5. Connect each Nucleo board via USB.
6. **Build** (`Ctrl + B`) and **flash** (`Run / F11`) each board with its own project.
7. Turn the potentiometer on the Master. The servo on the Slave should follow smoothly.

---

## 🧪 Troubleshooting

| Symptom | Likely Cause | Fix |
| :--- | :--- | :--- |
| Servo does not move at all | No common ground, or TX/RX not connected | Connect both `GND` pins; verify Master `PA9` → Slave `PA10` |
| Servo stuck at one end | ADC DMA not in Circular mode, or DMA Continuous Requests disabled | Check the ADC1 and DMA2 Stream 0 settings in CubeMX |
| Slave LED never toggles | IDLE interrupt not firing | Enable the USART1 global interrupt in NVIC; check DMA1 Channel 3 config |
| Servo jitters | ADC noise from the potentiometer | Add a 100 nF capacitor across the wiper and GND, or filter in software |
| MCU resets when servo moves | Servo powered from the board | Use an external 5V supply with common ground |
| Servo moves in the wrong range | Servo has different pulse limits | Adjust `SERVO_MIN_PULSE` / `SERVO_MAX_PULSE` in both `main.c` files |

---

## 🔧 Possible Improvements

- Replace `HAL_Delay(50)` on the Master with a `HAL_GetTick()` based scheduler, and use `HAL_UART_Transmit_DMA()` to make the transmit path fully non-blocking.
- Add a moving-average or low-pass filter on the ADC value to reduce servo jitter.
- Add a checksum or start/end markers to the packet format for stronger error detection.
- Use `snprintf()` instead of `sprintf()` on the Master to avoid any risk of buffer overflow.
- Call `HAL_UART_AbortReceive()` inside `HAL_UART_ErrorCallback()` on the Slave before restarting reception, so a still-active transfer cannot make the restart fail with `HAL_BUSY`.


