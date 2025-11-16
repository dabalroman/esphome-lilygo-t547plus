This repository contains a Display component for [ESPHome](https://esphome.io/) 2025.11+ 
to support the ESP32-S3 [LILYGO T5 4.7" Plus E-paper display](https://www.lilygo.cc/products/t5-4-7-inch-e-paper-v2-3).

**(Do not confuse it with the original ESP32-based Lilygo T5 4.7 board.)**

For more info on the display components, see the [ESPHome display documentation](https://esphome.io/#display-components)

## Usage
To use the board with [ESPHome](https://esphome.io/), adjust your `.yaml` config:

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
    board_build.flash_size: 16MB
    board_build.flash_mode: qio
    board_build.flash_type: qspi
    board_build.psram_type: opi
    board_build.memory_type: qspi_opi
    board_build.boot_freq: 80m
    build_flags:
      - "-DBOARD_HAS_PSRAM"

  libraries:
    - SPI
      
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
