# 01 · 从零搭建 HAL 工程（CubeMX → 移植 → 跑通）

> **目标**：读完能用 STM32CubeMX 从零生成一个 HAL + CMake 工程，把业务代码移植进去，并在板上跑起来。
> **示例工程**：STM32 数字钟（STM32F103C8T6）。流程对任意 STM32F1 + CubeMX + CMake 工程通用。
> **工具链**：STM32CubeMX 6.18 + FW_F1 V1.8.7 ／ CMake + arm-none-eabi-gcc ／ VS Code。

> ⚠️ **先决前提：时钟树 = HSI 8MHz（PLL 不倍频），APB1 不分频 → TIM3 定时器时钟 = 8MHz。**
> 本文所有定时器参数（ARR / CCR）**仅在此前提下成立**。
> CubeMX 默认是 72MHz（PLL 拉满，APB1 = 36MHz → TIM3 = 72MHz）；若照默认走还填 `ARR = 3905`，
> 实际频率会变成 72000000 / 3906 ≈ **18.4kHz** —— 超出可听范围，蜂鸣器不响或怪响。
> **改了主频就必须重算 ARR / CCR。**

---

## 一、MX 图形化配置

**1. 先配 System Core 里的 SYS（Debug 挡位）**

```
# System Core
- DMA
- ✅ GPIO
- IWDG
- ✅ NVIC
- RCC
- ✅ SYS——————————————————————先点击这个
- WWDG
```

```
# SYS Mode and Configuration
## Mode
- Debug：
  - No Debug————————————————————把这个默认的改成 Serial Wire——————SWD调试
  - Serial Wire
  - JTAG (4 pins)
  - JTAG (5 pins)
  - Trace Asynchronous Sw
- Sys：☐ 未勾选
- Timeba：文字截断
```

此时芯片引脚图中 4 个引脚发生变化（SWD 调试被启用）：

- BOOT0 和 NRST 变色 —— 说明被调用
- PA13 和 PA14 被设置为 SWDIO 和 SWCLK

**2. 先配"难"的外设，再配 GPIO（先难后易比较快，个人习惯）**

> UART1 和 TIM3_CH3 配好后不是绿色，因为具体参数还没配置，是黄色 🟡。

**(1) 配置 USART1**

```
# Categories（A->Z）
- System Core
- Analog
- Timers
- Connectivity
  - CAN
  - I2C1
  - I2C2
  - SPI1
  - SPI2
  - ✅ USART1——————————————————点击这个
  - USART2
  - USART3
  - USB
- Computing
```

```
# USART1 Mode and Configuration
## Mode
Mode下拉选项：
- Disable————————————————————把默认的这个改为Asynchronous
- Asynchronous
- Synchronous
- Single Wire (Half-Duplex)
- Multiprocessor Communication
- IrDA
- LIN
- SmartCard
```

下面的保持默认即可，已经十分标准：

```
## Basic Parameters
- Baud Rate: 115200 Bits/s
- Word Length: 8 Bits (including Parity)
- Parity: None
- Stop Bits: 1

## Advanced Parameters
```

**(2) 配置定时器（ARR + CCR 两个值就够，其他按默认）**

```
# Timers
- System Core
- Analog
- Timers
  - RTC
  - ⚠️ TIM1
  - TIM2
  - ✅ TIM3————————点击这个，因为引脚对应的功能是TIM3_CH3
  - TIM4
```

```
## Counter Settings
- Prescaler (PSC - 16 bits value): 0
- Counter Mode: Up
- Counter Period (AutoReload Register): 3905————————这个看蜂鸣器参数改
- Internal Clock Division (CKD): No Division
- auto-reload preload: Disable

## Trigger Output (TRGO) Parameters
- Master/Slave Mode (MSM bit): Disable (Trigger input effect not delayed)
- Trigger Event Selection: Reset (UG bit from TIMx_EGR)

## PWM Generation Channel 3
- Mode: PWM mode 1
- Pulse (16 bits value): 1953————————————上面的3905➗2————50%占空比
- Output compare preload: Enable
- Fast Mode: Disable
- CH Polarity: High
```

**ARR / CCR 怎么算（以蜂鸣器为例）**

#### ① 已知条件

* **蜂鸣器谐振频率**：$f = 2048 \text{ Hz}$
* **TIM3 输入时钟**：$8 \text{ MHz}$
* **PSC**：$0$
* **驱动方式**：无源蜂鸣器采用 $50\%$ 占空比驱动

#### ② 计算 ARR

PWM 频率公式：

$$f_{\mathrm{PWM}} = \frac{f_{\mathrm{TIM}}}{(PSC+1)(ARR+1)}$$

反推：

$$ARR+1 = \frac{8\,000\,000}{2048} \approx 3906.25$$

考虑寄存器只能设置整数，取 $ARR+1 = 3906$，所以：

$$\boxed{ARR = 3905}$$

实际 PWM 频率约为 $2048.13 \text{ Hz}$。

#### ③ 计算 Pulse（CCR）

按照 $50\%$ 占空比：

$$\text{CCR} \approx 3906 \times 50\% = 1953$$

因此：

$$\boxed{\text{Pulse} = 1953}$$

**(3) 配置 GPIO** —— 别漏了就行。

---

## 二、Generate Code 后的代码移植

先移植 8 个 `.c` 和 9 个 `.h` 文件。

CubeMX CMake 项目的**标准设计**：

| 文件                                 | 谁管的        | 作用                    |
| ---------------------------------- | ---------- | --------------------- |
| 根目录 `CMakeLists.txt`               | **我**      | 主入口，只生成一次，CubeMX 不再动它 |
| `cmake/stm32cubemx/CMakeLists.txt` | **CubeMX** | 每次重新生成代码都会覆盖          |

根目录 `CMakeLists.txt` 头部明确写了（MX 修改生成不会被覆盖）：

> _"This file is generated only once, and is not re-generated if converter is called multiple times."_

根目录 `CMakeLists.txt` 包含 `cmake/stm32cubemx/CMakeLists.txt`：

```cmake
# Add STM32CubeMX generated sources
add_subdirectory(cmake/stm32cubemx)
```

只要修改根目录的就行：

### `CMakeLists.txt` — 1 处，8 行

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    Core/Src/app.c             ← 你写的
    Core/Src/buzzer.c          ← 你写的
    Core/Src/clock_time.c      ← 你写的
    Core/Src/hw_test.c         ← 你写的
    Core/Src/key.c             ← 你写的
    Core/Src/led.c             ← 你写的
    Core/Src/seg_display.c     ← 你写的
    Core/Src/uart_console.c    ← 你写的
)
```

### `stm32f1xx_it.c` — 2 处，2 行

```c
/* USER CODE BEGIN Includes */
#include "clock_time.h"        ← 你写的
/* USER CODE END Includes */
```

```c
void SysTick_Handler(void)
{
  /* USER CODE BEGIN SysTick_IRQn 0 */
  clock_time_isr_1ms();        ← 你写的
  /* USER CODE END SysTick_IRQn 0 */
  HAL_IncTick();
```

### `main.c` — 3 处，5 行

```c
/* USER CODE BEGIN Includes */
#include "app.h"               ← 你写的
#include "hw_test.h"           ← 你写的
/* USER CODE END Includes */
```

```c
  /* USER CODE BEGIN 2 */
  app_init();                  ← 你写的
  hw_test_run();               ← 你写的
  /* USER CODE END 2 */
```

```c
    /* USER CODE BEGIN 3 */
    app_task();                ← 你写的
    /* USER CODE END 3 */
```

---

## 三、运行代码

VS Code 内点击：

```
生成 + STM32Cube: Launch ST-Link GDB Server
```

上面就是从零开始建立 HAL 库 → 点击调试后板子正常运行的完整过程。
