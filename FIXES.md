# ESP-IDF 5.4.2 Compatibility Issues

## Problems Found

1. **RMT Driver (components/t547/rmt_pulse.c)**
   - Uses legacy RMT API that was removed in ESP-IDF 5.4.2
   - Causes boot crash with RMT driver conflict error

2. **LCD Clock Source (components/t547/i2s_data_bus.c)**
   - Missing explicit clock source specification
   - ESP-IDF 5.4.2 no longer auto-detects clock source
   - Causes LCD bus initialization failure

3. **PSRAM Initialization (YAML configuration)**
   - Missing sdkconfig options to initialize PSRAM at boot
   - Results in 0 bytes PSRAM available instead of 8 MB
   - Display buffer allocation fails (needs 253 KB)




**Issue:** ESP-IDF 5.4.2 removed the legacy RMT driver API, causing boot crashes with error:
```
CONFLICT! driver_ng is not allowed to be used with the legacy driver
```

**Solution:** Migrated from legacy RMT API to new driver API.

**Changes:**
- **Headers:**
  ```c
  // Old:
  #include <driver/rmt.h>
  
  // New:
  #include <driver/rmt_tx.h>
  #include <driver/rmt_encoder.h>
  #include <esp_check.h>
  ```

- **Data structures:**
  ```c
  // Old:
  static rmt_config_t row_rmt_config;
  static rmt_item32_t rmt_items[RMT_BUFFER_LEN];
  
  // New:
  static rmt_channel_handle_t row_rmt_channel = NULL;
  static rmt_encoder_handle_t copy_encoder = NULL;
  static rmt_symbol_word_t rmt_items[RMT_BUFFER_LEN];
  ```

- **Initialization (`rmt_pulse_init()`):**
  ```c
  // Old:
  row_rmt_config.rmt_mode = RMT_MODE_TX;
  rmt_config(&row_rmt_config);
  rmt_driver_install(row_rmt_config.channel, 0, 0);
  
  // New:
  rmt_tx_channel_config_t tx_config = {
      .gpio_num = cfg->gpio_num,
      .clk_src = RMT_CLK_SRC_DEFAULT,
      .resolution_hz = 10000000,  // 10 MHz
      .mem_block_symbols = 64,
      .trans_queue_depth = 4,
  };
  rmt_new_tx_channel(&tx_config, &row_rmt_channel);
  rmt_enable(row_rmt_channel);
  
  rmt_copy_encoder_config_t encoder_config = {};
  rmt_new_copy_encoder(&encoder_config, &copy_encoder);
  ```

- **Transmission (`pulse_ckv_ticks()`):**
  ```c
  // Old:
  rmt_items[i].level0 = 1;
  rmt_items[i].duration0 = high_time_ticks;
  rmt_write_items(cfg->channel, rmt_items, i, false);
  rmt_wait_tx_done(cfg->channel, portMAX_DELAY);
  
  // New:
  rmt_items[i].level0 = 1;
  rmt_items[i].duration0 = high_time_ticks;
  rmt_transmit_config_t tx_cfg = {
      .loop_count = 0,
      .flags.eot_level = 0,
  };
  rmt_transmit(row_rmt_channel, copy_encoder, rmt_items, i * sizeof(rmt_symbol_word_t), &tx_cfg);
  rmt_tx_wait_all_done(row_rmt_channel, portMAX_DELAY);
  ```

### 2. LCD i80 Bus Clock Source (components/t547/i2s_data_bus.c)

**Issue:** ESP-IDF 5.4.2 requires explicit clock source specification, causing error:
```
esp_clk_tree_src_get_freq_hz(21): unknown clk src
```

**Solution:** Added explicit clock source to LCD i80 bus configuration.

**Changes:**
```c
// Added header:
#include "hal/lcd_types.h"

// Modified bus configuration:
esp_lcd_i80_bus_config_t bus_config = {
    .dc_gpio_num = CONFIG_DC,
    .wr_gpio_num = -1,
    // ... other config ...
    .clk_src = LCD_CLK_SRC_DEFAULT,  // ADDED: Explicit clock source
};
```

### 3. PSRAM Initialization (YAML configuration)

**Issue:** PSRAM was not being initialized at boot, showing:
```
Embedded PSRAM    : No
Free PSRAM: 0 bytes
```

This caused display buffer allocation to fail (requires ~253 KB).

**Solution:** Added ESP-IDF sdkconfig options to enable and initialize PSRAM.

**Changes in YAML:**
```yaml
esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: arduino
    sdkconfig_options:
      # Enable PSRAM support
      CONFIG_SPIRAM: "y"
      CONFIG_SPIRAM_MODE_OCT: "y"
      CONFIG_SPIRAM_SPEED_80M: "y"
      CONFIG_SPIRAM_USE_MALLOC: "y"
      CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL: "16384"
      CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP: "y"
      # Ensure PSRAM is initialized at boot
      CONFIG_ESP32S3_SPIRAM_SUPPORT: "y"
      CONFIG_SPIRAM_BOOT_INIT: "y"
      CONFIG_SPIRAM_IGNORE_NOTFOUND: "n"
```

**Additional platformio_options:**
```yaml
platformio_options:
  board_build.flash_size: 4MB
  build_flags:
    - "-DBOARD_HAS_PSRAM"
    - "-DARDUINO_USB_CDC_ON_BOOT=1"
    - "-mfix-esp32-psram-cache-issue"
```

### 4. Enhanced Error Handling (components/t547/t547.cpp)

**Added:** Diagnostic logging and fallback allocation strategy.

**Changes:**
```cpp
void T547::setup() {
  // Log memory before initialization
  ESP_LOGI(TAG, "=== Memory Before Display Init ===");
  ESP_LOGI(TAG, "Free PSRAM: %d bytes", heap_caps_get_free_size(MALLOC_CAP_SPIRAM));
  ESP_LOGI(TAG, "Total PSRAM: %d bytes", heap_caps_get_total_size(MALLOC_CAP_SPIRAM));
  ESP_LOGI(TAG, "Largest free PSRAM block: %d bytes", heap_caps_get_largest_free_block(MALLOC_CAP_SPIRAM));
  
  uint32_t buffer_size = this->get_buffer_length_();
  ESP_LOGI(TAG, "Required buffer size: %d bytes (%.2f KB)", buffer_size, buffer_size / 1024.0);
  
  this->buffer_ = (uint8_t *) ps_malloc(buffer_size);
  
  if (this->buffer_ == nullptr) {
    ESP_LOGE(TAG, "Could not allocate buffer for display!");
    
    // Try fallback allocation with heap_caps_malloc
    ESP_LOGW(TAG, "Trying fallback allocation with heap_caps_malloc...");
    this->buffer_ = (uint8_t *) heap_caps_malloc(buffer_size, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);
    
    if (this->buffer_ == nullptr) {
      ESP_LOGE(TAG, "Fallback allocation also failed!");
      this->mark_failed();
      return;
    }
  }
  
  ESP_LOGI(TAG, "Buffer allocated successfully at address: %p", this->buffer_);
}
```

## Results

### Before Fixes:
- ❌ Boot crash loop (RMT conflict)
- ❌ LCD bus initialization failure
- ❌ PSRAM: 0 bytes available
- ❌ Display buffer allocation failed
- ❌ Display marked as FAILED

### After Fixes:
- ✅ Clean boot without crashes
- ✅ LCD bus initialized successfully
- ✅ PSRAM: 8,388,608 bytes (8 MB) available
- ✅ Display buffer allocated (259,200 bytes)
- ✅ Display working and rendering graphics

## Memory Usage

```
SPIRAM Memory Info:
  Total Size        :  8388608 B (8192.0 KB)
  Free Bytes        :  8056268 B (7867.4 KB)
  Allocated Bytes   :   329692 B ( 322.0 KB)
  Largest Free Block:  7995392 B (7808.0 KB)
```

Display buffer (253 KB) successfully allocated in PSRAM.

## Compatibility

- **ESP-IDF:** 5.4.2 ✅
- **Arduino Framework:** 3.2.1 ✅
- **ESPHome:** 2025.10.5 ✅
- **Hardware:** LilyGo T5 4.7" Plus (ESP32-S3, 4.7" E-paper, OPI PSRAM) ✅

## Repository

Fixed version available at: [hbast/esphome-lilygo-t547plus-fixed](https://github.com/hbast/esphome-lilygo-t547plus-fixed)

## Usage Example

```yaml
external_components:
  - source: github://hbast/esphome-lilygo-t547plus-fixed
    components: ["t547"]
    refresh: 0s

display:
  - platform: t547
    update_interval: 30s
    lambda: |-
      it.line(0, 0, 960, 540);
```

## Key Takeaways

1. **ESP-IDF 5.x broke backward compatibility** with legacy peripheral drivers (RMT, I2S/LCD)
2. **PSRAM requires explicit initialization** via sdkconfig options in ESP-IDF 5.4.2
3. **Clock sources must be specified explicitly** for peripherals (was auto-detected in ESP-IDF 4.x)
4. **Memory diagnostics are essential** for debugging PSRAM-related issues

## Commits

- `cf62ea3` - RMT legacy API migration fix
- `f86ee6a` - LCD clock source fix
- `557de00` - PSRAM initialization via sdkconfig_options

---

**Date:** November 17, 2025  
**Author:** GitHub Copilot & hbast  
**Original Repository:** dabalroman/esphome-lilygo-t547plus
