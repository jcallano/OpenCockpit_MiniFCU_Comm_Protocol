# OpenCockpit MiniFCU Communication Protocol

Reverse Engineering and Wireless Bridge Implementation for Minicockpit MiniFCU/MiniEFIS

**Author:** @jcallano
**Version:** 1.1
**Last Updated:** 2026-Jan-01
**Target:** Minicockpit MiniFCU + Cap EFIS for Airbus A320 (Fenix A320 - MSFS2020)

## Disclaimer

This document and code are the result of reverse engineering work using man-in-the-middle techniques. This is **NOT** official documentation and is not affiliated with any hardware manufacturer or aircraft add-on developer. May contain errors. **Use at your own risk.**

---

## Project Overview

This project provides:
1. **Protocol Documentation** - Complete reverse-engineered serial protocol for the MiniFCU/MiniEFIS hardware
2. **Wireless Bridge** - ESP-NOW based wireless bridge to replace the USB cable connection
3. **Hardware Emulator** - Python script to emulate the hardware for development/testing

### Architecture

```
┌─────────────────┐     ESP-NOW      ┌─────────────────┐     USB Native     ┌─────────┐
│   MiniFCU HW    │◄───(Wireless)───►│    ESP32-S3     │◄─────────────────►│   PC    │
│   (CH340 USB)   │                  │    (Device)     │                   │ (MSFS)  │
└────────┬────────┘                  └─────────────────┘                   └─────────┘
         │ USB Host
         ▼
┌─────────────────┐
│    ESP32-S3     │
│     (Host)      │
└─────────────────┘
```

---

## Arduino Code Files

### `arduino_code/esphost/esphost.ino`

**Purpose:** ESP32-S3 USB Host that connects directly to the MiniFCU hardware.

**Functionality:**
- Acts as USB Host to communicate with the CH340 USB-to-Serial chip inside the MiniFCU
- Initializes the CH340 with proper control commands (baud rate, UART enable)
- Receives serial data from MiniFCU and forwards it wirelessly via ESP-NOW
- Receives data from the Device ESP32 via ESP-NOW and writes it to the CH340

**Key Features:**
- USB Host driver for CH340 chipset
- ESP-NOW optimized for low latency:
  - WiFi power save disabled
  - Fixed channel (Channel 1)
  - 802.11b protocol disabled (forces OFDM 6Mbps+)
- Elastic buffer with 2ms flush timeout for optimal packet aggregation
- Compatible with ESP32 Arduino Core 3.x (updated callback signatures)

**Hardware:**
- WeAct ESP32-S3 Dev Board
- Pins: RXD0=44, TXD0=43

**Configuration Required:**
- Set `peerAddress[]` to the MAC address of the Device ESP32

---

### `arduino_code/espdev/espdev.ino`

**Purpose:** ESP32-S3 Device that connects to the PC via USB Native.

**Functionality:**
- Connects to PC as a USB CDC device (appears as COM port)
- Receives commands from PC (simulator software) and forwards via ESP-NOW to Host
- Receives data from Host ESP32 via ESP-NOW and writes to USB (for PC to read)

**Key Features:**
- Uses ESP32-S3 native USB CDC (no external USB-Serial chip)
- Same ESP-NOW optimization as Host (channel, power save, protocol)
- Identical elastic buffer implementation for symmetry
- Serial baud rate: 9600 (matching MiniFCU protocol)

**Hardware:**
- WeAct ESP32-S3 Dev Board with USB CDC On Boot enabled
- Native USB-C connection to PC

**Configuration Required:**
- Set `hostMacAddress[]` to the MAC address of the Host ESP32

---

## Python Tools

### `minifcu_emulator.py`

**Purpose:** Software emulator of the MiniFCU hardware for development and testing.

**Functionality:**
- Emulates the MiniFCU hardware behavior over a Virtual COM Port
- Responds to handshake commands (`C,` → `901;956;959;`)
- Responds to keep-alive polling (`6,` → status response)
- Allows manual input of simulated hardware events (button presses, encoder turns)

**Usage:**
1. Install a Virtual COM Port pair (e.g., com0com on Windows)
2. Configure `PORT_VIRTUAL` to one end of the pair
3. Connect the official Windows software to the other end
4. Run the emulator and interact via command line

**Requirements:**
- Python 3.x
- pyserial (`pip install pyserial`)

---

## Protocol Documentation

### Physical Layer & Connection

| Parameter | Value |
|-----------|-------|
| Interface | USB Serial (CH340 chipset) |
| Baud Rate | 9600 bps |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Flow Control | None |

**Critical:** DTR and RTS lines MUST be asserted HIGH for the microcontroller to power up.

### Protocol Characteristics

- ASCII-based, token-oriented, event-driven
- Panel is stateless - all logic runs on PC side
- **PC → Hardware:** Tokens end with comma `,` (e.g., `S100,`)
- **Hardware → PC:** Frames end with semicolon `;` (e.g., `17;`)

### Startup Sequence

1. **Handshake:** Send `C,` - Device responds with firmware IDs
2. **Initialization Burst:** Configure limits and scaling
3. **Repaint:** Set all displays, LEDs, and segments

### Input Events (Hardware → PC)

| Function | CW | CCW | Push | Pull |
|----------|-----|-----|------|------|
| Speed | 13; | 14; | 11; | 12; |
| Heading | 3; | 4; | 1; | 2; |
| Altitude | 17; | 18; | 15; | 16; |
| VS/FPA | 21; | 22; | 19; | 20; |
| BARO | 101,val; | 102,val; | 70; | 69; |

**FCU Buttons:** AP1(50), AP2(51), A/THR(52), LOC(53), EXPED(54), APPR(55)
**EFIS Buttons:** FD(62), LS(63), CSTR(64), WPT(65), VOR.D(66), NDB(67), ARPT(68)

### Output Commands (PC → Hardware)

| Command | Example | Description |
|---------|---------|-------------|
| `Sxxx,` | `S280,` | Speed display |
| `Hxxx,` | `H084,` | Heading display |
| `Axxxxx,` | `A35000,` | Altitude display |
| `Fxxxx,` | `F-700,` | VS/FPA display |
| `#xxxx,` | `#1013,` | QNH/BARO display |
| `Bxxxx,` | `B1000,` | Backlight brightness |

**LED Control:** Uppercase = ON, Lowercase = OFF
- AP1(P/p), AP2(U/u), A/THR(T/t), LOC(L/l), EXPED(E/e), APPR(R/r)

---

## Installation & Setup

### ESP32-S3 Requirements

- Arduino IDE with ESP32 Arduino Core 3.x
- Board: WeAct ESP32-S3 (or compatible)
- USB Host capability required for Host unit

### Configuration Steps

1. Flash `esphost.ino` to the Host ESP32-S3 (connects to MiniFCU)
2. Flash `espdev.ino` to the Device ESP32-S3 (connects to PC)
3. Update MAC addresses in both sketches
4. Ensure both use the same WiFi channel

### Getting MAC Addresses

Upload a simple sketch to print the MAC:
```cpp
#include <esp_mac.h>
void setup() {
  Serial.begin(115200);
  uint8_t mac[6];
  esp_read_mac(mac, ESP_MAC_WIFI_STA);
  Serial.printf("MAC: %02X:%02X:%02X:%02X:%02X:%02X\n",
    mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
}
void loop() {}
```

---

## Files

| File | Description |
|------|-------------|
| `arduino_code/esphost/esphost.ino` | USB Host bridge - connects to MiniFCU hardware |
| `arduino_code/espdev/espdev.ino` | USB Device bridge - connects to PC |
| `minifcu_emulator.py` | Python hardware emulator for testing |
| `dump.txt` | Raw protocol capture from man-in-the-middle session |
| `requirements.txt` | Python dependencies |

---

## License

This project is provided for educational and hobbyist purposes. Not affiliated with Minicockpit or any simulator software vendor.
