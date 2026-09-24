# 🎛️ Dual-STM32 Potentiometer to Servo Controller (DMA & UART IDLE Line)
[![STM32](https://img.shields.io/badge/Hardware-STM32F446RE%20%7C%20STM32F072RB-002B49?logo=stmicroelectronics)](https://www.st.com/)
[![Framework](https://img.shields.io/badge/Framework-STM32%20HAL-blue.svg)](https://www.st.com/)
[![Language](https://img.shields.io/badge/Language-C11-green.svg)](https://en.wikipedia.org/wiki/C11_(C_standard_revision))
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
A real-time, zero-CPU-blocking Master-Slave embedded control system between two STM32 microcontrollers utilizing **Direct Memory Access (DMA)** and **UART IDLE Line** interrupt detection for remote RC servo motor actuation.
---
## 📌 System Architecture & Overview
* **Master (STM32F446RE):** Continuously samples analog voltage from a potentiometer in the background using **ADC1 + DMA (Circular Mode)**. The 12-bit ADC value ($0 - 4095$) is mapped into standard RC servo pulse widths ($500 - 2500\ \mu\text{s}$) and transmitted over **USART1** every 50 ms.
* **Slave (STM32F072RB):** Captures incoming serial packets using **USART1 DMA + IDLE Line Detection**. Without needing a fixed-length buffer or polling, the hardware triggers an interrupt as soon as the transmission line goes idle. The payload is parsed and written directly into the **TIM2 Channel 1 Hardware PWM** compare register (`CCR1`).
---
## 🚀 Key Features
- ⚡ **True Non-Blocking Execution:** No blocking delays or polling loops (`HAL_Delay()` on the Slave or `HAL_ADC_PollForConversion()` are completely eliminated).
- 🔄 **ADC DMA Circular Mode:** Continuous sampling occurs directly into SRAM with zero CPU intervention; values are available instantaneously.
- 📡 **UART IDLE Line Detection:** Utilizes `HAL_UARTEx_ReceiveToIdle_DMA` to process variable-length ASCII packets dynamically upon transmission completion, completely preventing buffer overflows.
- 🛡️ **Software Clamping (Mechanical Protection):** Rejects invalid packets or noise outside the $500 - 2500\ \mu\text{s}$ envelope to prevent the servo from hitting physical mechanical endstops.
- 💡 **Visual Diagnostics:** Built-in `LD2` LED pulses on the Master during transmission and toggles (heartbeat) on the Slave upon each valid packet receipt.
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
Pin Configuration Table
Board	Pin	Function	Description
Master (F446RE)	PA0	ADC1_IN0	Potentiometer Analog Input
PA9	USART1_TX	Serial Data Output
PA5	GPIO_Output	On-board LED (LD2) - Transmit Indicator
Slave (F072RB)	PA10	USART1_RX	Serial Data Input
PA0	TIM2_CH1	Servo Motor PWM Signal (50 Hz)
PA5	GPIO_Output	On-board LED (LD2) - Packet Received Heartbeat
⚠️ CRITICAL HARDWARE NOTES:

Common Ground: The GND pins of both boards must be tied together. Floating grounds will result in corrupted UART packets.
Servo Power Supply: Always power the servo's 
+
5
V
+5V and 
GND
GND lines from an external regulated 5V power supply. Powering high-draw servos directly from the STM32 board will cause voltage drops and MCU brownout resets. Ensure the external supply shares the common ground.
⚙️ Peripheral Configuration (CubeMX)
1. Master: STM32F446RE
Clock Tree: HSI 
→
→ PLL 
→
→ 
96
 MHz
96 MHz SYSCLK / 
48
 MHz
48 MHz HCLK
ADC1 Configuration:
Resolution: 12-bit
Data Alignment: Right Alignment
Continuous Conversion Mode: Enabled
DMA Continuous Requests: Enabled (Crucial for continuous DMA cycling)
DMA Request: ADC1 (DMA2 Stream 0), Mode: Circular, Data Width: Half Word / Half Word
USART1:
Baud Rate: 115200 bps, 8N1
2. Slave: STM32F072RB
Clock Tree: Internal HSI48 
→
→ 
48
 MHz
48 MHz HCLK / APB1
TIM2 (PWM Generation):
Prescaler (PSC): 47 
→
→ 
48
 MHz
47
+
1
=
1
 MHz
47+1
48 MHz
​
 =1 MHz (
1
 
μ
s
1 μs per tick resolution)
Period (ARR): 19999 
→
→ 
20
,
000
 
μ
s
=
20
 ms
20,000 μs=20 ms (
50
 Hz
50 Hz standard RC servo frame rate)
Pulse (CCR1): 1500 (Default 90° center position)
USART1:
Baud Rate: 115200 bps, 8N1
DMA Request: USART1_RX (DMA1 Channel 3), Mode: Normal, Data Width: Byte / Byte
NVIC: USART1 global interrupt ENABLED (Mandatory for IDLE line detection callback)
📂 Project Structure
bash


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
    │       ├── main.c           # UART IDLE DMA Callback & Hardware PWM control
    │       ├── stm32f0xx_it.c
    │       └── stm32f0xx_hal_msp.c
    └── Slave_F072RB.ioc         # STM32CubeMX configuration file
🛠️ Build & Installation
Clone the repository to your local machine:
bash


git clone https://github.com/your-username/stm32-dma-pot-servo.git
Open STM32CubeIDE.
Select File 
→
→ Open Projects from File System... and import both Master_F446RE and Slave_F072RB directories.
Connect the respective STM32 Nucleo boards via USB.
Build (Ctrl + B) and flash (Run / F11) each board with its corresponding project.
Verify hardware connections according to the schematic. Rotating the potentiometer on the Master will immediately and smoothly move the servo connected to the Slave.
📜 License
This project is licensed under the 
MIT License
 - feel free to use it for educational, academic, or commercial embedded applications.
