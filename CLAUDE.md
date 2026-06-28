# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An Arduino sketch that reads, decrypts, and republishes data from Wiener Netze smart meters (Landis+Gyr, Iskraemeco, Siemens) over MQTT, primarily for Home Assistant. It targets **both ESP32 and ESP8266** from a single sketch. There is no PlatformIO/CMake — this is a plain Arduino IDE project. See `README.md` for wiring/setup.

This repo is a fork of `aldadic/esp-smartmeter-reader` and is kept mergeable with upstream: prefer minimal, upstream-aligned changes; don't restructure it or migrate it to PlatformIO/ESPHome here.

## Build & test

There is no local unit test suite; "test" means a successful compile for both targets. CI (`.github/workflows/compile_esp{32,8266}.yml`) renames the platform config to `config.h`, then uses `arduino/compile-sketches` with FastCRC, ArduinoJson, PubSubClient (and Crypto for ESP8266).

To reproduce a build locally with `arduino-cli`:

```sh
# Pick the target config (this file is gitignored as */config.h)
cp esp-smartmeter-reader/config_esp32.h esp-smartmeter-reader/config.h

# ESP32
arduino-cli compile --fqbn esp32:esp32:esp32 esp-smartmeter-reader
# ESP8266
arduino-cli compile --fqbn esp8266:esp8266:generic esp-smartmeter-reader
```

Required libraries: `FastCRC`, `ArduinoJson`, `PubSubClient`; `Crypto` is ESP8266-only (ESP32 uses bundled `mbedtls`).

## Configuration

`config.h` is required to build and is **not committed** (gitignored). Before building, copy `config_esp32.h` or `config_esp8266.h` to `esp-smartmeter-reader/config.h` and fill in `KEY[]` (meter decryption key as a byte array), WiFi/MQTT settings, and the smart-meter framing constants. The two template files are the source of truth for available options — keep them in sync when adding config.

## Architecture (single file: `esp-smartmeter-reader/esp-smartmeter-reader.ino`)

The whole pipeline lives in one `.ino` with platform branches selected by `#if defined(ESP32)` / `#elif defined(ESP8266)`. The data flow is:

1. **Serial read** (`ReadSerialData`) — a byte-by-byte state machine on `smart_meter` (a `HardwareSerial*` defined in config). It detects the HDLC start sequence `7E A0`, buffers exactly `MESSAGE_LENGTH` bytes into `received_data`, then calls `ParseReceivedData`.
2. **CRC check** (`ValidateCRC`) — X.25 CRC16 over the frame (excluding flag + trailing CRC); bad frames are dropped silently.
3. **Decrypt** (`DecryptMessage`) — AES-128-CTR. The IV is assembled from the frame's system title (`HEADER_LENGTH` offset) + frame counter, with `iv[15]=0x02`. ESP32 uses `mbedtls_aes_crypt_ctr`; ESP8266 uses the `Crypto` library's `CTR<AES128>`.
4. **Parse + publish** (`ParseReceivedData`) — fields are read from the decrypted payload by **fixed offsets counted backward from the end** (`PAYLOAD_LENGTH - N`) via `BytesToInt`, assembled into a JSON doc (`+A`,`-A`,`+R`,`-R` energy in kWh; `+P`,`-P`,`+Q`,`-Q` instantaneous power/reactive; plus `timestamp`), and published to `MQTT_TOPIC`.

### Constants that drive everything

`MESSAGE_LENGTH` and `HEADER_LENGTH` (set per meter brand in config) determine frame buffering, the CRC window, the IV offsets, and `PAYLOAD_LENGTH = MESSAGE_LENGTH - HEADER_LENGTH - 17`. The backward-counted offsets in `ParseReceivedData` assume the standard payload layout — when adding a meter brand, you typically only change these two constants, but verify the field offsets still align.

### Platform differences to preserve when editing

- **Serial mapping**: ESP32 uses `Serial2` for the meter and `Serial` for logging; ESP8266 uses `Serial` (UART0) for the meter and `Serial1` for logging. ESP8266 config defines `SWAP_SERIAL`, which calls `Serial.swap()` to remap UART0 to GPIO13/15 so it doesn't clash with USB.
- **RX/TX pins**: if `RX_PIN`/`TX_PIN` are defined (ESP32 config, for Arduino core v3.0+), `begin()` is called with explicit pins.
- **Crypto API**: the mbedtls vs. Crypto branches must stay behaviorally equivalent — any change to the decryption path needs both sides updated.

Any change should keep **both** ESP32 and ESP8266 compiling and behaving identically; CI compiles both on every push/PR.
