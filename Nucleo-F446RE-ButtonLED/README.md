# ButtonLED

Toggles the on-board green LED (LD2) each time the blue user button (B1) is pressed on the NUCLEO-F446RE.

## Hardware

| Item | Detail |
|------|--------|
| Board | NUCLEO-F446RE (STM32F446RE) |
| LED | LD2 on pin **PA5**, configured as output |
| Button | B1 on pin **PC13**, configured as input, **active low** (reads high when idle, low when pressed) |
| Connection | USB cable to the ST-LINK port on the board |

No external components are needed.

## Tools

- STM32CubeIDE 2.2.0
- STM32CubeMX (standalone, used to configure pins and generate code)

## What it does

Each pass through the main loop, the program reads the button and compares the reading to the previous pass. A press is detected only on the **falling edge** (high to low), so holding the button down toggles the LED once, not repeatedly. After detecting an edge, it waits 20 ms and checks the pin again, which filters out contact bounce.

The code I added, inside the `USER CODE` markers in `Core/Src/main.c`:

```c
/* USER CODE BEGIN 2 */
GPIO_PinState last_state = GPIO_PIN_SET;  // button idles high
/* USER CODE END 2 */

while (1)
{
  /* USER CODE BEGIN 3 */
  GPIO_PinState now = HAL_GPIO_ReadPin(B1_GPIO_Port, B1_Pin);

  if (last_state == GPIO_PIN_SET && now == GPIO_PIN_RESET)
  {
    HAL_Delay(20);  // debounce
    if (HAL_GPIO_ReadPin(B1_GPIO_Port, B1_Pin) == GPIO_PIN_RESET)
    {
      HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
    }
  }
  last_state = now;
}
/* USER CODE END 3 */
```

## CubeMX configuration

- Board: NUCLEO-F446RE
- PA5: `GPIO_Output`, user label `LD2`
- PC13: `GPIO_Input`, user label `B1`
- SYS -> Debug: Serial Wire
- Toolchain/IDE: STM32CubeIDE

## Build and run

1. In STM32CubeIDE: *File -> Import... -> General -> Existing Projects into Workspace*.
2. Select this folder and leave "Copy projects into workspace" unchecked.
3. Build (hammer icon), then Run.
4. Press the blue button (B1). The green LED should toggle on each press.

## What I learned

- Reading a GPIO input with `HAL_GPIO_ReadPin`.
- Edge detection: remembering the previous reading (`last_state`) so one press causes one action.
- Software debouncing with a short delay and a confirmation read.
- Active-low inputs.

## Troubleshooting notes

- "Target is not responding" or "Unknown MCU found on target" errors were fixed by a power cycle of the board. If they persist, check that SYS -> Debug is set to Serial Wire, and try *Connect under reset* in the debug configuration.
- Accidentally assigned B1 to PA13 instead of PC13, which overrode the debugger pin and caused some issues. Assigned pins properly and the board worked afterward! 

## Possible extensions

- Remove the debounce delay and see if the LED ever misbehaves.
- Make the LED light only while the button is held.
- Replace polling with an external interrupt (EXTI).