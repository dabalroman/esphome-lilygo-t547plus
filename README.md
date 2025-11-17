This repository contains a Display component for [ESPHome](https://esphome.io/) 2025.11+ 
to support the ESP32-S3 [LILYGO T5 4.7" Plus E-paper display](https://www.lilygo.cc/products/t5-4-7-inch-e-paper-v2-3).

**(Do not confuse it with the original ESP32-based Lilygo T5 4.7 board.)**

For more info on the display components, see the [ESPHome display documentation](https://esphome.io/#display-components)

## ESP-IDF 5.4.2+ Compatibility

**Important:** This version includes fixes for ESP-IDF 5.4.2+ compatibility. See [FIXES.md](FIXES.md) for details.

Key fixes:
- RMT driver migrated to new API (fixes boot crashes)
- LCD i80 bus requires explicit clock source
- **PSRAM must be initialized via `sdkconfig_options`** (required for display buffer)

I confirmed it works with esphome 2025.10.5 with the following components:
- time
- binary_sensor (homeassistant, gpio)
- text_sensor (homeassistant)
- online_image
- font

## Usage
To use the board with [ESPHome](https://esphome.io/), adjust your `.yaml` config:
```yaml
external_components:
  - source: github://dabalroman/esphome-lilygo-t547plus_esphome-2025.11
    components: ["t547"]
```

## Full example
Make note how `platformio_options`, `libraries` and `esp32` are set up, these are important for the compilation stage.

**Important for ESP-IDF 5.4.2+:** You must add `sdkconfig_options` to enable PSRAM initialization.

```yaml
esphome:
  name: lilygo
  friendly_name: lilygo e-ink display

  platformio_options:
    # Unless noted otherwise, based on https://github.com/Xinyuan-LilyGO/LilyGo-EPD47/blob/1eb6119fc31fcff7a6bafecb09f4225313859fc5/examples/demo/platformio.ini#L37
    upload_speed: 921600
    monitor_speed: 115200
    board_build.mcu: esp32s3
    board_build.f_cpu: 240000000L
    board_build.arduino.memory_type: qspi_opi
    board_build.flash_size: 4MB
    board_build.flash_mode: qio
    board_build.flash_type: qspi
    board_build.psram_type: opi
    board_build.memory_type: qspi_opi
    board_build.boot_freq: 80m
    build_flags:
      - "-DBOARD_HAS_PSRAM"
      - "-DARDUINO_USB_CDC_ON_BOOT=1"
      - "-mfix-esp32-psram-cache-issue"

  libraries:
    - SPI

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: arduino
    # CRITICAL for ESP-IDF 5.4.2+: Enable PSRAM initialization
    sdkconfig_options:
      CONFIG_SPIRAM: "y"
      CONFIG_SPIRAM_MODE_OCT: "y"
      CONFIG_SPIRAM_SPEED_80M: "y"
      CONFIG_SPIRAM_USE_MALLOC: "y"
      CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL: "16384"
      CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP: "y"
      CONFIG_ESP32S3_SPIRAM_SUPPORT: "y"
      CONFIG_SPIRAM_BOOT_INIT: "y"
      CONFIG_SPIRAM_IGNORE_NOTFOUND: "n"

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "uji41VdMORB5966gRlVYEbBaC5LU9NZwHStB8j2d/nU="

ota:
  - platform: esphome
    password: "4ab30fdd89bd61087359d5eebf85d296"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Lilygo Fallback Hotspot"
    password: "zYQIOsSr8spF"

captive_portal:

external_components:
  - source: github://dabalroman/esphome-lilygo-t547plus_esphome-2025.11
    components: ["t547"]


display:
  - platform: t547
    update_interval: 30s
    lambda: |-
      it.line(0, 0, 960, 540);
```

EspHome will fetch the lib before compilation. If it fails, try `Clean build files` before using `Install`.

This does not support z-lib compressed fonts.

## Discussion

https://github.com/esphome/feature-requests/issues/1960
