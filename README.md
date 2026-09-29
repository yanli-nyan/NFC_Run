# NFC Run

[中文文档](README_CN.md) | [English](README.md)

Firmware for an **ESP32-S3** NFC/button input device. It reads NFC tokens from a PN532 reader or button tokens from six GPIO inputs, then sends each token to a Home Assistant event endpoint. The firmware also includes a WS2812 status LED, UDP log forwarding, and browser-based OTA updates.

## Features

- Reads **Android HCE** cards/phones using a fixed APDU SELECT AID, then reports the returned text token.
- Reads **MIFARE Classic** cards by authenticating and reading Sector 1, Block 0, then reports its 16-byte text payload.
- Reports six active-low buttons as `K1`–`K6` tokens with 30 ms debounce.
- Sends JSON asynchronously to Home Assistant's `/api/events/nfc_scanned` endpoint.
- Provides a Web OTA page at `http://<device-ip>/` and accepts firmware binaries at `POST /update`.
- Mirrors ESP-IDF logs to the serial console and a configurable UDP receiver.
- Uses the onboard/external WS2812 LED to show startup, Wi-Fi, OTA, and error states.
- Uses an OTA partition layout with two 3 MiB OTA application slots.

## Hardware and wiring

The checked-in configuration targets **ESP32-S3**. Verify all GPIO assignments against your board before powering it.

| Function | GPIO / setting | Notes |
| --- | --- | --- |
| PN532 SDA | GPIO 8 | I²C, internal pull-up enabled |
| PN532 SCL | GPIO 9 | I²C at 100 kHz |
| PN532 address | `0x24` | 7-bit I²C address used by this firmware |
| Status LED | GPIO 48 | One WS2812/NeoPixel, brightness limited to 20/255 |
| Button K3 | GPIO 4 | Active low, internal pull-up |
| Button K2 | GPIO 13 | Active low, internal pull-up |
| Button K1 | GPIO 14 | Active low, internal pull-up |
| Button K6 | GPIO 16 | Active low, internal pull-up |
| Button K5 | GPIO 17 | Active low, internal pull-up |
| Button K4 | GPIO 18 | Active low, internal pull-up |

Each button should connect its GPIO to GND when pressed. Do not assume GPIO 48 is available on every ESP32-S3 board.

## Prerequisites

- ESP-IDF **v5.3.5** (the version locked by `dependencies.lock`)
- An ESP32-S3 development board
- A PN532 configured for I²C mode
- A 2.4 GHz Wi-Fi network reachable by the device
- A reachable Home Assistant instance with a Long-Lived Access Token
- Optional: a UDP listener for log collection

## Configure before flashing

This project currently stores environment-specific values directly in source code. Update them before building:

| File | Values to set |
| --- | --- |
| `main/main.c` | `WIFI_SSID` and `WIFI_PASS` |
| `components/ha_client/ha_client.c` | Home Assistant event URL and Bearer token |
| `components/udp_logger/udp_logger.c` | UDP log receiver IP address and port |
| `components/pn532_reader/pn532_reader.c` | PN532 pins/address, Android HCE AID, and MIFARE key if your hardware/cards differ |
| `components/button_reader/button_reader.c` | Button GPIOs and token mapping |
| `components/status_led/status_led.c` | WS2812 GPIO and brightness |

The device sends the following JSON shape:

```json
{
  "device": "esp32s3_n8r2",
  "token": "<NFC-or-button-token>"
}
```

Configure a Home Assistant automation to listen for the `nfc_scanned` event and act on `trigger.event.data.token`.

## Build, flash, and monitor

Open an ESP-IDF terminal, then run:

```bash
cd NFC_Run
idf.py set-target esp32s3
idf.py build
idf.py -p <serial-port> flash monitor
```

Use `Ctrl+]` to exit the serial monitor. The initial serial logs show the assigned IP address after Wi-Fi connects.

## Web OTA

1. Build the firmware with `idf.py build`.
2. Put the device and the browser on the same network.
3. Open `http://<device-ip>/`.
4. Select `build/NFC_Run.bin` and start the upload.
5. Keep power connected until the device reboots.

The partition table contains two OTA application slots, so a successful OTA is written to the inactive slot and selected for the next boot.

> Warning: the OTA server uses plain HTTP and has no authentication. Restrict it to a trusted network or add authentication and TLS before any broader deployment.

## Status LED

| State | Color |
| --- | --- |
| Startup / idle after a token report | Yellow |
| Connecting to Wi-Fi | Blue |
| Wi-Fi connected | Green |
| OTA update in progress | Purple |
| Initial Wi-Fi connection timed out | Red |

Wi-Fi reconnects automatically after a disconnect. The OTA server and UDP logger are started even if the initial 10-second Wi-Fi connection wait times out.

## Project layout

```text
main/
  main.c                         Wi-Fi setup and application startup
components/
  pn532_reader/                  PN532 polling, HCE, and MIFARE Classic support
  button_reader/                 Six-button polling and debounce
  ha_client/                     Home Assistant event client
  web_ota/                       HTTP firmware-update server
  udp_logger/                    UDP log forwarding
  status_led/                    WS2812 system-state indicator
partitions.csv                   Factory and dual-OTA partition layout
```

## Security notes

- Do not commit Wi-Fi passwords, Home Assistant tokens, card keys, or private IP addresses. Rotate any credential that has already been committed.
- Use a dedicated, least-privilege Home Assistant token.
- OTA traffic and Home Assistant communication are HTTP in the current implementation. Keep the device on a trusted LAN, or add HTTPS and authentication.
- NFC/card content and button tokens are logged and forwarded; treat them as sensitive identifiers.
- Connect only hardware and cards you are authorized to use.
