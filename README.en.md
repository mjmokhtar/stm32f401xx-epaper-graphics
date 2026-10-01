[🇮🇩 Bahasa Indonesia](README.md) | 🇬🇧 English

# STM32F401 Environment Monitor — E-paper SSD1680 + XY-MD02 (Modbus RTU)

FreeRTOS firmware for the STM32F401CCU6 (Blackpill): reads temperature/humidity from the
XY-MD02 Modbus RTU sensor every 10 seconds and shows it on an SSD1680 e-paper display (WeAct 2.13") every 60
seconds. **Status: TESTED AND WORKING ON HARDWARE** — see the [Test Results](#test-results) section below.

## Test Results

### Physical setup

![Physical setup](images/rangkaian.jpeg)

The STM32F401CCU6 Blackpill, XY-MD02 sensor (RS485), RS485-to-UART module, and SSD1680 e-paper are wired on a breadboard. The left USB-TTL is for USART1 debug; the blue wire to the e-paper carries power from a separate supply.

### E-paper display (final result)

![E-paper result](images/epaper.jpeg)

Final layout: "STM32 Env Monitor" header, Temp/Humi lines, uptime, and a credit footer — as designed in `RenderSensorData()`.

### Serial debug log (USART1, 115200 8N1, via Hercules Setup Utility)

![Serial output](images/serial.png)

The normal pattern is visible: `[Sensor] OK` every ~10 seconds, `[Display] rendering/refresh done` every ~60 seconds, and an occasional `[Sensor] TIMEOUT` (brief RS485 noise, not a bug — see the [Known Issues](#known-issues--final-notes) section).

## Features

- Reads temperature & humidity sensor (XY-MD02) over Modbus RTU RS485, 10-second interval
- Displays on the 2.13" e-paper (SSD1680), fixed 60-second refresh interval (not on every new reading — to conserve the panel's refresh cycles)
- E-paper panel sleeps (`epd_sleep()`) between refreshes to save power
- Two separate FreeRTOS tasks (Sensor + Display) — the blocking e-paper refresh (2–8 seconds) never delays the next sensor reading
- Debug log via USART1 (115200 8N1) — status of every sensor read cycle & display refresh
- No network (WiFi/4G LTE) — standalone sensor-to-display

## IO / Pins Used (Final, Confirmed Working)

MCU: **STM32F401CCU6** (Blackpill, UFQFPN48) — SYSCLK 84 MHz (HSE 25MHz → PLLM=25, PLLN=168, PLLP=2), APB1 42MHz, APB2 84MHz.
![CLK](images/image_clk.png)

![Final pinout](images/image_io.png)

### E-paper SSD1680 (WeAct 2.13") — SPI1

| MCU Pin | Function | Notes |
|---|---|---|
| PA5 | SCK  | SPI1 hardware AF |
| PA6 | MISO | SPI1 hardware AF, not used by the e-paper (write-only) but still configured |
| PA7 | MOSI | SPI1 hardware AF |
| PA4 | CS   | Manual GPIO output, High when idle (**software NSS**, not hardware NSS) |
| PA0 | DC   | GPIO output (Data/Command) |
| PA1 | RST  | GPIO output |
| PA8 | BUSY | GPIO input, no pull |
| 3.3V | VCC | — |
| GND | GND | — |

![Final pinout](images/image1.png)
![Final pinout](images/image2.png)
![Final pinout](images/image3.png)
![Final pinout](images/image4.png)
![Final pinout](images/image5.png)
![Final pinout](images/image6.png)
![Final pinout](images/image7.png)
![Final pinout](images/image8.png)
![Final pinout](images/image9.png)
![Final pinout](images/image10.png)



**Final SPI1 prescaler: `/2` → 42 MBit/s** (not `/32` as in the initial conservative recommendation — on the hardware the SSD1680 panel turned out to work fine at this speed, so this is the value used and proven to work, not just theory).

### XY-MD02 Sensor (Modbus RTU / RS485) — USART2

| MCU Pin | Function |
|---|---|
| PA2 | TX |
| PA3 | RX |

Baud rate: **9600 8N1** (confirmed). Modbus device address: `0x01` (XY-MD02 default).

![USART2 pinout](images/image12.png)


**Note:** there is no manual GPIO control for the DE/RE direction of the RS485 transceiver module in this code — suitable for auto-direction-sensing modules (e.g. MAX13487-based). If your RS485 module is a manual DE/RE type (e.g. a plain MAX485), you need to add a GPIO toggle before `HAL_UART_Transmit`.

Normal behavior: an occasional `TIMEOUT`/`CRC_ERROR` appears in the log (brief RS485 noise, also visible in the serial screenshot above) — this is **not treated as a fatal error**, the old data is kept until the next read succeeds (see `StartSensorTask` in `main.c`).

### Debug Print — USART1

| MCU Pin | Function |
|---|---|
| PA9  | TX |
| PA10 | RX |

![USART1 pinout](images/image11.png)

Baud rate: **115200 8N1** (confirmed, see the Hercules screenshot above). `printf()` is retargeted here through `__io_putchar()` (see `main.c`, `USER CODE BEGIN 4`).

## FreeRTOS Task Design

| Task | Priority | Stack | Behavior |
|---|---|---|---|
| `SensorTask` | `osPriorityNormal` | 512 words | `sensor_read()` every 10 seconds (`osDelayUntil`), writes to `g_latest_data` (mutex-protected) |
| `DisplayTask` | `osPriorityLow` | 1024 words | Reads `g_latest_data` (mutex) every 60 seconds, `epd_wake()` → render → `epd_display()` → `epd_sleep()` |

There is no shared bus between SPI1 (e-paper) and USART2 (sensor) → no risk of priority inversion from bus contention. No mediator/protocol-layer pattern is used — agreed to be overkill for this standalone sensor-to-display scope.

![RTOS](images/image13.png)
![RTOS](images/image14.png)


FreeRTOS heap: 15360 bytes (CubeMX default), Interface: CMSIS_V2, `USE_NEWLIB_REENTRANT` **Enabled** — important because `printf`/`snprintf` are called from two different tasks (Sensor & Display); this setting makes newlib thread-safe under FreeRTOS.

## Folder Structure

```
Core/
├── Inc/
│   ├── main.h
│   ├── epaper.h          # pin mapping + e-paper API (ported from ESP32)
│   ├── epaper_gfx.h       # drawing primitives (font, lines, shapes) — pure reuse
│   └── sensor.h           # Modbus RTU XY-MD02
└── Src/
    ├── main.c             # FreeRTOS setup, SensorTask, DisplayTask, RenderSensorData
    ├── epaper.c            # SSD1680 driver, ported from ESP-IDF → STM32 HAL
    ├── epaper_gfx.c        # reused directly, unchanged
    └── sensor.c            # Modbus RTU (CRC16, request/response parsing)
images/                 # CubeMX screenshots + test result photos
```

`Drivers/`, `Middlewares/Third_Party/FreeRTOS/`, `.ioc`, `.project`, `.cproject` are generated by STM32CubeIDE/CubeMX — not included here; overwrite/merge the `Core/` above into your own CubeMX project.

## Build & Flash

1. An STM32CubeIDE project for the STM32F401CCU6, with an `.ioc` that has FreeRTOS (CMSIS_V2), SPI1, USART1, USART2 enabled according to the pin table above.
2. Generate code from CubeMX.
3. Overwrite/merge this project's `Core/Inc/` and `Core/Src/` into the generated `Core/` folder.
4. Make sure the includes `epaper.h`, `epaper_gfx.h`, `sensor.h`, `<stdio.h>` are inside `USER CODE BEGIN Includes` (main.c) — so they are not lost when you regenerate.
5. Build (Clean + Build Project), flash to the board.
6. Open a serial monitor on USART1 (115200 8N1) to see the sensor/display status log.
