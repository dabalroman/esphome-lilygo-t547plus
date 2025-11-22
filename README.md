This repository contains a Display component for [ESPHome](https://esphome.io/) 2025.11+ 
to support the ESP32-S3 [LILYGO T5 4.7" Plus E-paper display](https://www.lilygo.cc/products/t5-4-7-inch-e-paper-v2-3).
For more info on the display components, see the [ESPHome display documentation](https://esphome.io/#display-components)

**(Do not confuse it with the original ESP32-based Lilygo T5 4.7 board.)**

![](readme-files/full_setup.jpg)
<p align='center'><b>Image of full setup config</b></p>


> [!TIP]
> The "dusty" marks on the image above are signs of e-ink screen fatigue.  
> This will also happen to your display, so avoid frequent updates and rendering the same stuff over and over.

## Contributions
- Huge thanks to [@hbast](https://github.com/hbast) for ESP-IDF 5.4.2+ Compatibility!


## Usage (TLDR)
To use the board with [ESPHome](https://esphome.io/), adjust your `.yaml` config:
Make note how `platformio_options`, `libraries`, `esp32`, `sdkconfig_options` and `external_components` are set up in [basic setup](#basic-setup) below, these are important for the compilation stage.
Make sure to `Clean build files` if you're editing your old `.yaml` configs. 

```yaml
external_components:
  - source: github://dabalroman/esphome-lilygo-t547plus_esphome-2025.11
    components: ["t547"]
```


## Usage (step by step)
1. Open the esphome dashboard.
2. Fill in the secrets with ssid and password of your Wi-Fi network.
3. Create new device.
4. In `New Device` wizard use `ESP32S3` board.
5. You may install the firmware now or skip and apply the config directly from [basic.yaml](basic.yaml).
6. Make sure to not override the `name`, `friendly_name`, `api` and `ota` sections with the example, cause the Wi-Fi update may might fail. The values there should be unique for every setup to make sure your device is secure!
7. Flash the [basic setup](basic.yaml) or [full setup](full_setup.yaml) via OTA to confirm the connection does work.
8. Now you can do whatever you want :)

> [!IMPORTANT]
> EspHome aggressively uses cache. 
> Try using `Clean build files` on device card before `Install` if compilation fails.

## Basic setup

See [`basic.yaml`](basic.yaml) for a complete, working reference configuration.

**Key configuration requirements:**
- Complete PSRAM initialization via `sdkconfig_options` (critical for ESP-IDF 5.4.2+)
- Correct build flags with hardware workarounds
- 16MB flash size (correct for hardware)
- External components using GitHub source

You can also use the [`full_setup.yaml`](full_setup.yaml) file to test most supported display features, including fetching time data from HomeAssistant.


## Z-lib
This branch does not support z-lib compressed fonts.


## Discussion
https://github.com/esphome/feature-requests/issues/1960
