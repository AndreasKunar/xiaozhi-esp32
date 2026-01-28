# Custom Development Board Guide - custom-board.md

This guide explains how to create a new board‑initialization program for the **Xiaozhi AI Voice Chatbot** project. Xiaozhi AI supports more than 70 ESP‑32 series development boards, and each board’s initialization code is stored in its own directory.

## Important Notice  

> **Warning**: When the IO pin configuration of a custom board differs from the original board, **do not overwrite** the original board’s configuration when compiling firmware. You must either create a **new board type** or distinguish it through the `name` and `sdkconfig` macros in the `config.json` file. Use  

```
python scripts/release.py [board_directory_name]
```  

to compile and package the firmware.  

If you simply overwrite the original configuration, future OTA updates may replace your custom firmware with the standard one, causing the device to stop working. Each board has a unique identifier and an OTA channel that matches it; keeping the board identifier unique is crucial.

## Directory Structure  

Each board’s directory typically contains the following files:

- `xxx_board.cc` – Main board‑level initialization code; implements board‑specific initialization and functionality  
- `config.h` – Board‑level configuration file; defines hardware pin mappings and other constants  
- `config.json` – Build configuration; specifies the target chip and special compile options  
- `README.md` – Documentation for the board  

## Custom Board Steps  

### 1. Create a New Board Directory  

First, create a new directory under `boards/` with a naming convention of **[brand]-[board_type]**. For example, `m5stack-tab5`:

```bash
mkdir main/boards/my-custom-board
```

### 2. Create Configuration Files  

#### config.h  

In `config.h` define all hardware configurations, such as:

- Audio sampling rate and I2S pin configuration  
- Audio codec address and I2C pin configuration  
- Button and LED pin configuration  
- Display parameters and pin configuration  

Example taken from `lichuang-c3-dev`:

```c
#ifndef _BOARD_CONFIG_H_
#define _BOARD_CONFIG_H_

#include <driver/gpio.h>

// Audio configuration
#define AUDIO_INPUT_SAMPLE_RATE  24000
#define AUDIO_OUTPUT_SAMPLE_RATE 24000

#define AUDIO_I2S_GPIO_MCLK GPIO_NUM_10
#define AUDIO_I2S_GPIO_WS   GPIO_NUM_12
#define AUDIO_I2S_GPIO_BCLK GPIO_NUM_8
#define AUDIO_I2S_GPIO_DIN  GPIO_NUM_7
#define AUDIO_I2S_GPIO_DOUT GPIO_NUM_11

#define AUDIO_CODEC_PA_PIN       GPIO_NUM_13
#define AUDIO_CODEC_I2C_SDA_PIN  GPIO_NUM_0
#define AUDIO_CODEC_I2C_SCL_PIN  GPIO_NUM_1
#define AUDIO_CODEC_ES8311_ADDR  ES8311_CODEC_DEFAULT_ADDR

// Button configuration
#define BOOT_BUTTON_GPIO        GPIO_NUM_9

// Display configuration
#define DISPLAY_SPI_SCK_PIN     GPIO_NUM_3
#define DISPLAY_SPI_MOSI_PIN    GPIO_NUM_5
#define DISPLAY_DC_PIN          GPIO_NUM_6
#define DISPLAY_SPI_CS_PIN      GPIO_NUM_4

#define DISPLAY_WIDTH   320
#define DISPLAY_HEIGHT  240
#define DISPLAY_MIRROR_X true
#define DISPLAY_MIRROR_Y false
#define DISPLAY_SWAP_XY true

#define DISPLAY_OFFSET_X  0
#define DISPLAY_OFFSET_Y  0

#define DISPLAY_BACKLIGHT_PIN GPIO_NUM_2
#define DISPLAY_BACKLIGHT_OUTPUT_INVERT true

#endif // _BOARD_CONFIG_H_
```

#### config.json  

In `config.json` define the build configuration; this file is read by `scripts/release.py` for automated compilation:

```json
{
    "target": "esp32s3",  // target chip: esp32, esp32s3, esp32c3, esp32c6, esp32p4, etc.
    "builds": [
        {
            "name": "my-custom-board",  // board name used for firmware package naming
            "sdkconfig_append": [
                // Flash size configuration
                "CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y",
                // Custom partition table configuration
                "CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/8m.csv\""
            ]
        }
    ]
}
```

**Configuration item details:**  

- `target` – target chip model, must match the hardware.  
- `name` – name of the compiled output firmware package; it is recommended to keep it consistent with the directory name.  
- `sdkconfig_append` – an array of extra `sdkconfig` options that will be appended to the default configuration.  

**Common `sdkconfig_append` options:**  

```json
// Flash size
"CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y"   // 4 MB Flash
"CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y"   // 8 MB Flash
"CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y"  // 16 MB Flash

// Partition table
"CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/4m.csv\""  // 4 MB partition table
"CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/8m.csv\""  // 8 MB partition table
"CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/16m.csv\"" // 16 MB partition table

// Language settings
"CONFIG_LANGUAGE_EN_US=y"  // English
"CONFIG_LANGUAGE_ZH_CN=y"  // Simplified Chinese

// Wake‑word settings
"CONFIG_USE_DEVICE_AEC=y"          // Enable device‑side AEC
"CONFIG_WAKE_WORD_DISABLED=y"      // Disable wake‑word
```

### 3. Write Board Initialization Code  

Create a `my_custom_board.cc` file and implement the board’s entire initialization logic.  

A typical board class definition includes the following parts:

1. **Class definition** – inherit from `WifiBoard` (or `Ml307Board`)  
2. **Initialization functions** – initialize I2C, display, buttons, IoT peripherals, etc.  
3. **Virtual function overrides** – such as `GetAudioCodec()`, `GetDisplay()`, `GetBacklight()`, etc.  
4. **Board registration** – use `DECLARE_BOARD` macro to register the board  

```cpp
#include "wifi_board.h"
#include "codecs/es8311_audio_codec.h"
#include "display/lcd_display.h"
#include "application.h"
#include "button.h"
#include "config.h"
#include "mcp_server.h"

#include <esp_log.h>
#include <driver/gpio.h>
#include <driver/i2c_master.h>
#include <driver/spi_common.h>

#define TAG "MyCustomBoard"

class MyCustomBoard : public WifiBoard {
private:
    i2c_master_bus_handle_t codec_i2c_bus_;
    Button boot_button_;
    LcdDisplay* display_;

    // I2C initialization
    void InitializeI2c() {
        i2c_master_bus_config_t i2c_bus_cfg = {
            .i2c_port = I2C_NUM_0,
            .sda_io_num = AUDIO_CODEC_I2C_SDA_PIN,
            .scl_io_num = AUDIO_CODEC_I2C_SCL_PIN,
            .clk_source = I2C_CLK_SRC_DEFAULT,
            .glitch_ignore_cnt = 7,
            .intr_priority = 0,
            .trans_queue_depth = 0,
            .flags = {
                .enable_internal_pullup = 1,
            },
        };
        ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_bus_cfg, &codec_i2c_bus_));
    }

    // SPI initialization (used for the display)
    void InitializeSpi() {
        spi_bus_config_t buscfg = {};
        buscfg.mosi_io_num = DISPLAY_SPI_MOSI_PIN;
        buscfg.miso_io_num = GPIO_NUM_NC;
        buscfg.sclk_io_num = DISPLAY_SPI_SCK_PIN;
        buscfg.quadwp_io_num = GPIO_NUM_NC;
        buscfg.quadhd_io_num = GPIO_NUM_NC;
        buscfg.max_transfer_sz = DISPLAY_WIDTH * DISPLAY_HEIGHT * sizeof(uint16_t);
        ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));
    }

    // Button initialization
    void InitializeButtons() {
        boot_button_.OnClick([this]() {
            auto& app = Application::GetInstance();
            if (app.GetDeviceState() == kDeviceStateStarting) {
                EnterWifiConfigMode();
                return;
            }
            app.ToggleChatState();
        });
    }

    // Display initialization (example uses ST7789)
    void InitializeDisplay() {
        esp_lcd_panel_io_handle_t panel_io = nullptr;
        esp_lcd_panel_handle_t panel = nullptr;
        
        esp_lcd_panel_io_spi_config_t io_config = {};
        io_config.cs_gpio_num = DISPLAY_SPI_CS_PIN;
        io_config.dc_gpio_num = DISPLAY_DC_PIN;
        io_config.spi_mode = 2;
        io_config.pclk_hz = 80 * 1000 * 1000;
        io_config.trans_queue_depth = 10;
        io_config.lcd_cmd_bits = 8;
        io_config.lcd_param_bits = 8;
        ESP_ERROR_CHECK(esp_lcd_new_panel_io_spi(SPI2_HOST, &io_config, &panel_io));

        esp_lcd_panel_dev_config_t panel_config = {};
        panel_config.reset_gpio_num = GPIO_NUM_NC;
        panel_config.rgb_ele_order = LCD_RGB_ELEMENT_ORDER_RGB;
        panel_config.bits_per_pixel = 16;
        ESP_ERROR_CHECK(esp_lcd_new_panel_st7789(panel_io, &panel_config, &panel));
        
        esp_lcd_panel_reset(panel);
        esp_lcd_panel_init(panel);
        esp_lcd_panel_invert_color(panel, true);
        esp_lcd_panel_swap_xy(panel, DISPLAY_SWAP_XY);
        esp_lcd_panel_mirror(panel, DISPLAY_MIRROR_X, DISPLAY_MIRROR_Y);
        
        // Create the display object
        display_ = new SpiLcdDisplay(panel_io, panel,
                                    DISPLAY_WIDTH, DISPLAY_HEIGHT, 
                                    DISPLAY_OFFSET_X, DISPLAY_OFFSET_Y, 
                                    DISPLAY_MIRROR_X, DISPLAY_MIRROR_Y, DISPLAY_SWAP_XY);
    }

    // MCP Tools initialization
    void InitializeTools() {
        // See MCP documentation for details
    }

public:
    // Constructor
    MyCustomBoard() : boot_button_(BOOT_BUTTON_GPIO) {
        InitializeI2c();
        InitializeSpi();
        InitializeDisplay();
        InitializeButtons();
        InitializeTools();
        GetBacklight()->SetBrightness(100);
    }

    // Get the audio codec
    virtual AudioCodec* GetAudioCodec() override {
        static Es8311AudioCodec audio_codec(
            codec_i2c_bus_, 
            I2C_NUM_0, 
            AUDIO_INPUT_SAMPLE_RATE, 
            AUDIO_OUTPUT_SAMPLE_RATE,
            AUDIO_I2S_GPIO_MCLK, 
            AUDIO_I2S_GPIO_BCLK, 
            AUDIO_I2S_GPIO_WS, 
            AUDIO_I2S_GPIO_DOUT, 
            AUDIO_I2S_GPIO_DIN,
            AUDIO_CODEC_PA_PIN, 
            AUDIO_CODEC_ES8311_ADDR);
        return &audio_codec;
    }

    // Get the display
    virtual Display* GetDisplay() override {
        return display_;
    }
    
    // Get the back‑light controller
    virtual Backlight* GetBacklight() override {
        static PwmBacklight backlight(DISPLAY_BACKLIGHT_PIN, DISPLAY_BACKLIGHT_OUTPUT_INVERT);
        return &backlight;
    }
};

// Register the board
DECLARE_BOARD(MyCustomBoard);
```

### 4. Add Build‑System Configuration  

#### Add a board option in `Kconfig.projbuild`  

Open `main/Kconfig.projbuild` and, inside the `choice BOARD_TYPE` block, add a new option for your board:

```kconfig
choice BOARD_TYPE
    prompt "Board Type"
    default BOARD_TYPE_BREAD_COMPACT_WIFI
    help
        Board type. 开发板类型
    
    # ... other board options ...

    config BOARD_TYPE_MY_CUSTOM_BOARD
        bool "My Custom Board (自定义开发板)"
        depends on IDF_TARGET_ESP32S3   # adjust according to your target chip
    endchoice
```

**Notes:**  

- `BOARD_TYPE_MY_CUSTOM_BOARD` must be all uppercase with underscores separating words.  
- `depends on` restricts the option to a specific target chip (e.g., `IDF_TARGET_ESP32S3`, `IDF_TARGET_ESP32C3`, …).  
- The description can be bilingual.

#### Add board configuration in `CMakeLists.txt`  

Open `main/CMakeLists.txt` and, inside the `elseif` chain that selects board types, add a clause for your board:

```cmake
# Add the clause inside the existing elseif list
elseif(CONFIG_BOARD_TYPE_MY_CUSTOM_BOARD)
    set(BOARD_TYPE "my-custom-board")  # must match the directory name
    set(BUILTIN_TEXT_FONT font_puhui_basic_20_4)   # choose a font size that fits the screen
    set(BUILTIN_ICON_FONT font_awesome_20_4)
    set(DEFAULT_EMOJI_COLLECTION twemoji_64)   # optional; use a different set for larger screens
endif()
```

**Font & Emoji selection guidance:**  

| Screen size | Recommended text font | Recommended icon font |
|-------------|----------------------|-----------------------|
| 128 × 64 OLED | `font_puhui_basic_14_1` / `font_awesome_14_1` | `twemoji_32` (32 px) |
| 240 × 240 | `font_puhui_basic_16_4` / `font_awesome_16_4` | `twemoji_32` |
| 240 × 320 | `font_puhui_basic_20_4` / `font_awesome_20_4` | `twemoji_64` |
| 480 × 320 or larger | `font_puhui_basic_30_4` / `font_awesome_30_4` | `twemoji_64` |

### 5. Build & Flash  

#### Method 1: Use `idf.py` interactively  

1. **Select the target chip** (first time or when changing chips):  

   ```bash
   # For ESP32‑S3
   idf.py set-target esp32s3
   
   # For ESP32‑C3
   idf.py set-target esp32c3
   
   # For ESP32
   idf.py set-target esp32
   ```

2. **Clean old configurations:**  

   ```bash
   idf.py fullclean
   ```

3. **Enter the configuration menu:**  

   ```bash
   idf.py menuconfig
   ```

   Navigate to `Xiaozhi Assistant` → `Board Type` and select your custom board.

4. **Compile, flash, and monitor:**  

   ```bash
   idf.py build
   idf.py flash monitor
   ```

#### Method 2: Use `release.py` (recommended if you have a `config.json`)  

```bash
python scripts/release.py my-custom-board
```

The script automatically:

- Reads the `target` entry from `config.json` and sets the appropriate chip.  
- Applies all entries listed under `sdkconfig_append`.  
- Compiles the firmware and creates a packaged binary.

### 6. Create a README.md  

Place a `README.md` in the board’s directory to document its specifics, hardware requirements, build steps, and usage tips.

```markdown
# My Custom Board Documentation  

## Common Board Components  

### 1. Display  

The project supports a variety of displays, including:  

- ST7789 (SPI)  
- ILI9341 (SPI)  
- SH8601 (QSPI)  
- …  

### 2. Audio Codec  

Supported codecs include:  

- ES8311 (common)  
- ES7210 (microphone array)  
- AW88298 (power amplifier)  
- …  

### 3. Power Management  

Some boards use a power‑management IC such as:  

- AXP2101  
- …  

### 4. MCP Device Control  

You can add various MCP tools so the AI can control them:  

- Speaker (volume control)  
- Screen (brightness adjustment)  
- Battery (read battery level)  
- Light (LED switching)  
- …  

## Board Class Inheritance  

- `Board` – base board class  
  - `WifiBoard` – boards with Wi‑Fi connectivity  
  - `Ml307Board` – boards that use a 4G module  
  - `DualNetworkBoard` – boards that can switch between Wi‑Fi and 4G networks  

## Development Tips  

1. **Reference similar boards** – If your board shares hardware with an existing one, copy and adapt its code.  
2. **Step‑wise debugging** – First get basic peripherals (e.g., display) working, then add advanced features (e.g., audio).  
3. **Pin mapping** – Ensure every pin is correctly defined in `config.h`.  
4. **Hardware compatibility** – Verify that all chips and drivers are supported by the ESP‑IDF version you are using.  

## Frequently Encountered Issues  

1. **Display not showing anything** – Check SPI configuration, mirroring, and color inversion settings.  
2. **No audio output** – Verify I2S pins, PA‑enable pin, and codec address.  
3. **Unable to connect to Wi‑Fi** – Double‑check Wi‑Fi credentials and network settings.  
4. **Cannot communicate with server** – Verify MQTT/WebSocket configuration.  

## Reference Materials  

- ESP‑IDF Documentation: https://docs.espressif.com/projects/esp-idf/  
- LVGL Documentation: https://docs.lvgl.io/  
- ESP‑SR Documentation: https://github.com/espressif/esp-sr  

