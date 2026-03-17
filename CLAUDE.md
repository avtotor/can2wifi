# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ESP32-S3 CAN-to-WiFi bridge using ESP-IDF. Reads CAN bus frames via the built-in TWAI peripheral (with external TJA1050 transceiver) and exposes them over WiFi (Telnet on port 23) and UART serial (115200 baud) using the GVRET protocol. Compatible with SavvyCAN.

## Build Commands

Requires ESP-IDF 5.3.1 with `IDF_PATH` set.

```bash
idf.py build                    # compile
idf.py flash                    # flash to device (auto-detects COM port)
idf.py monitor                  # serial monitor at 115200 baud
idf.py build flash monitor      # all-in-one
idf.py -p COM3 flash monitor    # specify port explicitly
idf.py fullclean                # wipe build/ for clean rebuild
idf.py menuconfig               # interactive sdkconfig editor
```

No test framework is configured — there are no unit tests.

## Architecture

**Entry point:** `main/main.cpp` → `app_main()` initializes UART, GPIO, NVS, loads settings, sets up CAN and WiFi, then runs a 1ms-tick main loop.

**Data flow:** CAN frames → `CANManager` → `GVRET_Comm_Handler` (formats to GVRET binary/ASCII) → Telnet client or UART serial. Reverse path: incoming GVRET commands from Telnet/serial → protocol parser → CAN transmit.

**Key components:**

| File | Role |
|------|------|
| `esp32_can_idf.cpp` | TWAI driver wrapper (`ESP32CAN` class, global `CAN0`) |
| `can_manager.cpp` | CAN frame read/route/send, bus load calc |
| `gvret_comm.cpp` | GVRET protocol state machine (binary + ASCII modes), 2KB TX buffer |
| `wifi_manager_idf.cpp` | WiFi AP setup, Telnet server (port 23), UDP broadcast (port 17222) |
| `uart_serial.cpp` | UART0 serial I/O |
| `config_idf.h` | `EEPROMSettings` and `SystemSettings` structs — all runtime config |
| `serial_console_idf.cpp` | Serial command-line menu |
| `logger_idf.cpp` | Log levels: DEBUG/INFO/WARN/ERROR via `Logger_*()` functions |
| `sys_io_idf.cpp` | GPIO LED control |

**Two GVRET handler instances** exist: one for serial, one for WiFi — each with its own TX buffer.

## Hardware Pins

- **CAN TX:** GPIO4 → TJA1050 TX (3.3V direct)
- **CAN RX:** GPIO5 ← TJA1050 RX (via 1k/2k voltage divider 5V→3.3V)
- **Status LED:** GPIO2
- **UART TX/RX:** GPIO43/44 (ESP32-S3 defaults)

## Configuration

- `sdkconfig.defaults` is the source of truth for IDF config (target: ESP32-S3, 4MB flash, single-app partition). Do not edit `sdkconfig` directly.
- Runtime defaults are in `load_settings()` in `main.cpp`: CAN0 at 500kbps, WiFi AP SSID "AVTOTOR_CAN", password "mypassword", WiFi channel 1, max 4 connections (1 active Telnet client).

## Language

Source is C/C++ mixed. `config_idf.c` is C; everything else is C++. The `main/CMakeLists.txt` registers all sources via `idf_component_register()`.
