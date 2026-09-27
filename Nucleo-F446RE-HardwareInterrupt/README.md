# HardwareInterrupt

Blinks the on-board green LED (LD2) once per second using a hardware timer (TIM2) and an interrupt, instead of blocking the CPU with `HAL_Delay`. This is a rework of the `blinky` project with a different, non-blocking approach.

## Hardware

| Item | Detail |
|------|--------|
| Board | NUCLEO-F446RE (STM32F446RE) |
| LED | LD2, green, on **PA5** |
| Timer | TIM2, general-purpose 32-bit timer, internal clock source |

No external components are needed.

## Tools

- STM32CubeIDE 2.2.0
- STM32CubeMX (standalone, used to configure pins and generate code)

## The idea

`HAL_Delay` blocks the CPU while it waits, so nothing else can run during the delay. A hardware timer runs independently of the CPU: it counts on its own clock, and when it overflows it raises an interrupt that the CPU responds to immediately, no matter what it was doing. This frees the main loop to do other work (or nothing) while timing is handled entirely by the timer peripheral.

## Clock math

- System clock (HCLK): 84 MHz
- APB1 prescaler: /2, giving PCLK1 = 42 MHz
- Because the APB1 prescaler isn't 1, the APB1 timer clock is doubled: 42 MHz x 2 = 84 MHz feeding TIM2
- Target: 1 second between interrupts

The timer's actual divisor is `PSC + 1`, and it counts `ARR + 1` ticks before triggering an update event, so both registers are one less than the "round number" you're solving for:

- **Prescaler (PSC): 83** -> divides 84 MHz by 84, giving 1 MHz (1 microsecond) ticks
- **Auto-reload (ARR): 999999** -> 1,000,000 ticks at 1 MHz = 1 second per update event

## What it does

1. CubeMX configures TIM2 with the PSC and ARR above and enables its NVIC interrupt.
2. `HAL_TIM_Base_Start_IT(&htim2)` is called once, after `MX_TIM2_Init()`, to start the timer counting and enable its interrupt.
3. Every time TIM2 overflows (once per second), the HAL calls `HAL_TIM_PeriodElapsedCallback`. The default implementation in the HAL driver does nothing (it's declared `__weak`); this project overrides it in `main.c` to toggle the LED, after checking that the interrupt came from TIM2.
4. The main loop is empty. All timing and the toggle itself happen in the interrupt.

Code added, inside the `USER CODE` markers in `Core/Src/main.c`:

```c
/* USER CODE BEGIN 2 */
HAL_TIM_Base_Start_IT(&htim2);
/* USER CODE END 2 */
```

```c
/* USER CODE BEGIN 4 */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  if (htim->Instance == TIM2)
  {
    HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
  }
}
/* USER CODE END 4 */
```

The `while (1)` loop itself is left empty. The CPU has nothing to actively do; it just waits to be interrupted by the timer.

## CubeMX configuration

- Board: NUCLEO-F446RE
- PA5: `GPIO_Output`, user label `LD2`
- Timers, TIM2: Clock Source = Internal Clock
- TIM2 Parameter Settings: Prescaler = 83, Counter Period (ARR) = 999999
- TIM2 NVIC Settings: TIM2 global interrupt enabled
- SYS, Debug: Serial Wire
- Toolchain/IDE: STM32CubeIDE

## Build and run

1. In STM32CubeIDE: *File, Import, General, Existing Projects into Workspace*.
2. Select this folder and leave "Copy projects into workspace" unchecked.
3. Build (hammer icon), then Run.

LD2 should blink once per second, with no calls to `HAL_Delay` anywhere in the active code path.

## What I learned

- The difference between a blocking delay and a hardware timer running independently of the CPU.
- What an interrupt and an ISR (interrupt service routine) are, and the role of the NVIC.
- How a timer's prescaler and auto-reload register set its period, including the `+1` on both (the hardware divides by `PSC + 1` and counts `ARR + 1` ticks).
- How the timer clock is derived from HCLK through the APB1 bus, including the rule that the timer clock doubles the peripheral clock when the APB1 prescaler isn't 1.
- What a `__weak` function is, and how defining a function with the same name and signature elsewhere in the project overrides the HAL's default (do-nothing) implementation.
- Using `htim->Instance` to check which timer triggered a shared callback, since the same callback fires for every timer with an interrupt enabled.
- That an empty `while(1)` loop is a valid and common pattern once work is handled by interrupts, and that the CPU is still fully active in the background, just idling until interrupted.

## Verification

Commenting out `HAL_TIM_Base_Start_IT(&htim2)` stops the LED from blinking entirely, confirming the toggle really is driven by the timer interrupt and not by anything else.

## Possible extensions

- Recalculate PSC/ARR for a different period (for example, 250 ms) and confirm the math holds up on hardware.
- Add the UART "Hello World!" transmit from the `UARTHello` project into the main loop, now that it's free, and observe whether it interferes with the timer-driven blink.
- Replace the empty loop with `__WFI()` (wait-for-interrupt) to actually let the CPU sleep between interrupts.