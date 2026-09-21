# UARTHelloWorld

Sends "Hello World!" over UART once per second from the NUCLEO-F446RE to a terminal on the PC. The text travels through the board's built-in ST-LINK virtual COM port, so only the USB cable is needed.

## Hardware

| Item | Detail |
|------|--------|
| Board | NUCLEO-F446RE (STM32F446RE) |
| UART | USART2, TX on **PA2**, RX on **PA3** (RX unused in this project) |
| Link to PC | ST-LINK virtual COM port over the same USB cable used for flashing |
| Status LED | LD2 on **PA5** |

No external components or wiring are needed.

## Tools

- STM32CubeIDE 2.2.0
- STM32CubeMX (standalone, used to configure pins and generate code)
- A serial terminal on the PC (for example PuTTY)

## Serial settings

| Setting | Value |
|---------|-------|
| Baud rate | 115200 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |

These are often written as "115200 8N1". The terminal must use the same settings as the microcontroller, or the output will be garbage.

## What it does

1. Before the main loop, the program stores the message ("Hello World!" followed by a carriage return and line feed) and measures its length once.
2. In the main loop, it transmits the message with the HAL's blocking UART transmit function, using a 10 ms timeout.
3. It checks the value returned by the transmit call. On success, LD2 is turned on. On any other result, LD2 is turned off.
4. It waits one second and repeats.

The 10 ms timeout leaves comfortable margin: at 115200 baud each byte takes about 10 bit-times on the wire (8 data bits plus start and stop bits), so the 14-byte message takes roughly 1.2 ms to send.

## CubeMX configuration

- Board: NUCLEO-F446RE
- Board Project Options: Human Machine Interface (BSP) unchecked, so the pins are plain GPIO
- Connectivity, USART2: Asynchronous mode, 115200 baud, 8 data bits, no parity, 1 stop bit
- PA2 as `USART2_TX`, PA3 as `USART2_RX`
- PA5: `GPIO_Output`, user label `LD2`
- SYS, Debug: Serial Wire
- Toolchain/IDE: STM32CubeIDE

## Build and run

1. In STM32CubeIDE: *File, Import, General, Existing Projects into Workspace*.
2. Select this folder and leave "Copy projects into workspace" unchecked.
3. Build with the hammer icon.
4. Find the board's port (on Windows: Device Manager, Ports, "STMicroelectronics STLink Virtual COM Port").
5. Open the serial terminal on that port using the settings above.
6. Run the program from CubeIDE.

"Hello World!" should appear on its own line every second.

## What I learned

- What a UART is and why both sides must agree on baud rate and frame format (or else the thing spits out a bunch-a garbo).
- How the Nucleo's virtual COM port removes the need for a separate USB-serial adapter.
- What a HAL handle (`UART_HandleTypeDef`) is and why functions take a pointer to it.
- Refreshed on C strings (null terminator and strlen).
- Pointer types and casts: passing text to a function that expects a `uint8_t` pointer.
- What a blocking call and a timeout mean, and how to estimate a sensible timeout from baud rate.
- Checking `HAL_StatusTypeDef` return values.

## Troubleshooting notes

| Symptom | Likely cause |
|---------|--------------|
| Random symbols | Baud rate or frame format mismatch between terminal and MCU |
| Nothing appears | Wrong COM port, terminal not connected, or USART2 not enabled |
| Lines stair-step across the window | Using \r\n instead of just \n |
| "Target is not responding" or "Unknown MCU" when flashing | Power cycle the board; check SYS, Debug is set to Serial Wire |

## Notes

- The user button (B1) is left at its default interrupt-on-falling-edge configuration but is not used here.
- The transmit call blocks the CPU while sending, and `HAL_Delay` blocks it while waiting. Later projects replace these with interrupts, timers, and DMA.

## Possible extensions

- Print a counter that increases each second.
- Receive characters over USART2 and react to them.
- Make the LED blink on each transmission so it shows the loop is alive.