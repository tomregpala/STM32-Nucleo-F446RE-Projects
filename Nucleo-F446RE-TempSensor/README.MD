# Nucleo-F446RE_TempSensor

Reads the STM32F446RE's built-in temperature sensor through the ADC and prints the result over UART once per second, reusing the serial setup from `UARTHello`.

## Hardware

| Item | Detail |
|------|--------|
| Board | NUCLEO-F446RE (STM32F446RE) |
| Sensor | Internal die temperature sensor, routed to ADC1's dedicated temperature channel (not an external pin) |
| UART | USART2, TX on **PA2**, via the ST-LINK virtual COM port |
| Status LED | LD2 on **PA5** (configured, not actively used in this project) |

No external components are needed. The sensor is built into the chip and cannot be probed externally; it measures die temperature, not ambient air temperature.

## Tools

- STM32CubeIDE 2.2.0
- STM32CubeMX (standalone)
- A serial terminal on the PC (115200 8N1, same settings as `UARTHello`)

## The idea

The ADC converts the sensor's analog output into a 12-bit raw value (0-4095). ST factory-calibrates each chip by measuring this raw value at two known temperatures and burning the results into fixed memory addresses. Using these two calibration points to build a straight-line equation gives a more accurate result for the specific chip than the datasheet's generic typical-value formula, and avoids needing to know the board's exact reference voltage.

## Calibration points

| Constant | Address | Known temperature |
|----------|---------|---------------------|
| TS_CAL1 | 0x1FFF7A2C | 30 degC |
| TS_CAL2 | 0x1FFF7A2E | 110 degC |

Both were measured by ST at the factory with VDDA = 3.3V. Each is a 16-bit value, read by casting the address to a `uint16_t *` and dereferencing it:

```c
uint16_t TS_CAL1_reading = *(uint16_t *)0x1FFF7A2C;
uint16_t TS_CAL2_reading = *(uint16_t *)0x1FFF7A2E;
```

## Formula

Treating the two calibration points as a line (temperature as a function of raw ADC reading), anchored at TS_CAL1:

```
T = 30 + [(110 - 30) / (TS_CAL2_reading - TS_CAL1_reading)] * (live_reading - TS_CAL1_reading)
```

In code, written to force floating-point division (otherwise the integer division of two whole numbers would truncate to zero):

```c
temperature = 30 + (80.0 / (TS_CAL2_reading - TS_CAL1_reading)) * (live_reading - TS_CAL1_reading);
```

Using `80.0` (a floating-point literal) on one side of the division is enough to promote the entire expression to floating-point, since C promotes to the widest type present whenever either operand of an operation is a float.

## What it does

1. At startup, reads both calibration values once (they never change).
2. In the main loop, once per second:
   - Starts an ADC conversion (`HAL_ADC_Start`, single-conversion mode).
   - Polls for completion with a 10 ms timeout (`HAL_ADC_PollForConversion`).
   - On success, reads the raw value (`HAL_ADC_GetValue`), computes the temperature, formats it into a string with `snprintf`, and sends it over UART.
   - On a failed poll, that iteration is skipped entirely: nothing is sent, rather than resending a stale or garbage value.
3. Waits one second (`HAL_Delay(1000)`) and repeats, regardless of whether the read succeeded.

## CubeMX configuration

- Board: NUCLEO-F446RE
- Connectivity, USART2: Asynchronous, 115200 baud, 8N1 (same as `UARTHello`)
- Analog, ADC1:
  - Temperature Sensor Channel enabled (the IN5/PA5 channel that CubeMX initially also checked was removed, since it isn't needed here and would have added an unwanted second channel to the conversion sequence)
  - Resolution: 12-bit
  - Continuous Conversion Mode: Disabled (single-conversion, explicitly re-triggered each second)
  - Sampling Time: 480 cycles (ADC clock is 21 MHz from APB2/4; 480 cycles is about 22.9 microseconds, the smallest available option that clears the datasheet's 10 microsecond minimum for the temperature channel)
- SYS, Debug: Serial Wire
- Toolchain/IDE: STM32CubeIDE

## A toolchain gotcha worth noting

Printing the float value initially produced blank output ("Temp:   C" with no number). This turned out not to be a code bug: STM32CubeIDE's default GCC/newlib-nano C library ships with a reduced `printf`/`snprintf` that doesn't support `%f` by default, to save flash space. The fix is a linker setting (in Project Properties, under the linker's miscellaneous settings, enabling float support for printf/newlib-nano, equivalent to the `-u _printf_float` flag), followed by a full rebuild. Worth remembering for any future project that formats floating-point values for output on an embedded target.

## Build and run

1. In STM32CubeIDE: *File, Import, General, Existing Projects into Workspace*.
2. Select this folder and leave "Copy projects into workspace" unchecked.
3. If starting fresh from this source, confirm float support for printf is enabled in the linker settings (see above) before building.
4. Build (hammer icon), then Run.
5. Open a serial terminal on the board's COM port at 115200 8N1.

You should see a line like `Temp: 32.15 C` once per second. Touching the chip or breathing on the board briefly should raise the reading, then it should drift back down.

## What I learned

- How the ADC converts an analog voltage into a 12-bit digital value, and what role the reference voltage plays.
- Reading a value from a known, fixed memory address in C: casting an integer literal to a typed pointer, then dereferencing it.
- Deriving a two-point linear calibration from factory-measured reference values, rather than relying on generic typical-value constants.
- Why dividing two integers truncates in C, and how introducing a single floating-point literal promotes the whole expression.
- The ADC's blocking workflow (Start, PollForConversion, GetValue) compared to UART's single blocking call, and the difference between single-conversion and continuous conversion modes.
- Matching CubeMX's "Sampling Time" cycle-count options against a datasheet's microsecond requirement, using the ADC clock frequency to convert between the two.
- Variable scope mistakes: declaring something inside an `if` block that's needed outside it, and the difference between a value that needs to persist (declared once, outside the loop) and one that's recomputed fresh every pass (declared inside the loop or inside its `if`).
- `snprintf` for safely formatting a string with a variable in it, and the `-u _printf_float` newlib-nano limitation on embedded GCC toolchains.
- The importance of deciding what happens on a failure path (a timed-out poll) rather than letting it fall through silently.

## Known limitations

- Accuracy depends on VDDA actually being close to 3.3V, the voltage the factory calibration was performed at. No correction for a different VDDA is applied.
- The sensor itself is only accurate to within a few degrees Celsius per the datasheet; printing two decimal places implies more precision than the sensor actually provides.
- `HAL_ADC_Start`'s return value is captured but not checked.
- It measures die temperature, which runs warmer than ambient air, not room temperature.

## Possible extensions

- Add a correction term for VDDA if it's known to differ from 3.3V.
- Compare this calibrated formula's output against the generic `V25`/`Avg_Slope` datasheet formula on the same hardware.
- Average several readings per second to smooth out noise.
- Reduce the printed precision to one decimal place to better match the sensor's real accuracy.