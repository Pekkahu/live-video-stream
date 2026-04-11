| LED blinking and it's light intensity control is remaining to be programmed |


# ESP32-CAM Live Video Stream
### Error Documentation & Fix Log
> PlatformIO + Arduino IDE — AI Thinker ESP32-CAM + OV3660

---

## Environment

| Property | Value |
|---|---|
| **Board** | AI Thinker ESP32-CAM |
| **Camera Sensor** | OV3660 |
| **IDE** | VS Code + PlatformIO / Arduino IDE |
| **Platform** | `espressif32 @ 6.13.0` |
| **Framework** | Arduino |
| **Framework Package** | `framework-arduinoespressif32 @ 3.20017.241212` |

---

## Errors & Fixes

### Error #1 — `cannot open source file "camera_pins.h"`

| | |
|---|---|
| **Error** | `cannot open source file "camera_pins.h"` |
| **File** | `main.cpp` |
| **Line** | `5` |
| **Cause** | `camera_pins.h` is not included in the Arduino/PlatformIO project. It must be created manually as it contains board-specific GPIO pin definitions. |
| **Fix** | Create `camera_pins.h` inside the `src/` folder with the AI Thinker ESP32-CAM pin definitions (`PWDN`, `RESET`, `XCLK`, `SIOD`, `SIOC`, `Y2–Y9`, `VSYNC`, `HREF`, `PCLK`). |

<details>
<summary>📄 camera_pins.h content</summary>

```cpp
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27

#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22
```

</details>

---

### Error #2 — `i2c probe timeout / ESP_ERR_NOT_SUPPORTED (0x106)`

| | |
|---|---|
| **Error** | `E (xxxx) i2c.master: probe device timeout` / `ESP_ERR_NOT_SUPPORTED (0x106)` |
| **File** | Serial Monitor output |
| **Line** | Runtime |
| **Cause** | Camera sensor OV3660 was not being detected. The camera config was set up for OV2640 (default AI Thinker). Also caused by loose ribbon cable or insufficient power. |
| **Fix** | Update camera config with OV3660-specific settings (see below). In sensor tuning, check for `OV3660_PID` and apply `vflip`, `hmirror`, `brightness` and `saturation` corrections. |

<details>
<summary>📄 OV3660 camera config fix</summary>

```cpp
// PSRAM block — change to:
if (psramFound()) {
    config.frame_size   = FRAMESIZE_UXGA;       // ← CHANGED
    config.jpeg_quality = 10;                   // ← CHANGED
    config.fb_count     = 2;                    // ← CHANGED
    config.fb_location  = CAMERA_FB_IN_PSRAM;   // ← NEW
} else {
    config.frame_size   = FRAMESIZE_SVGA;       // ← CHANGED
    config.jpeg_quality = 12;
    config.fb_count     = 1;
    config.fb_location  = CAMERA_FB_IN_DRAM;    // ← NEW
}

// Add after pixel_format:
config.grab_mode = CAMERA_GRAB_WHEN_EMPTY;      // ← NEW for OV3660

// Sensor tuning block — change to:
sensor_t * s = esp_camera_sensor_get();
if (s->id.PID == OV3660_PID) {
    s->set_vflip(s, 1);        // OV3660 image is upside down without this
    s->set_hmirror(s, 0);      // ← NEW
    s->set_brightness(s, 1);   // ← CHANGED
    s->set_saturation(s, -2);  // ← CHANGED
}
```

</details>

---

### Error #3 — `identifier "ledcAttach" is undefined`

| | |
|---|---|
| **Error** | `identifier "ledcAttach" is undefined  C/C++(20)` |
| **File** | `app_httpd.cpp` |
| **Line** | `844` |
| **Cause** | `ledcAttach(pin, freq, resolution)` is the new API introduced in `arduino-esp32 v3.0+`. The installed platform version was older and only supports the legacy 3-step API. |
| **Fix** | Replace the single `ledcAttach()` call with the old 3-step API. |

<details>
<summary>📄 Code fix</summary>

```cpp
// ❌ BEFORE (new API — not supported on older platform):
ledcAttach(LED_GPIO_NUM, 5000, 8);

// ✅ AFTER (old API — works on all versions):
ledcSetup(0, 5000, 8);
ledcAttachPin(LED_GPIO_NUM, 0);
ledcWrite(0, 0);
```

</details>

---

### Error #4 — `WiFiClass has no member "setSleep"`

| | |
|---|---|
| **Error** | `class "WiFiClass" has no member "setSleep"  C/C++(135)` |
| **File** | `main.cpp` |
| **Line** | `110` |
| **Cause** | `WiFi.setSleep()` was removed in newer versions of the ESP32 Arduino core. WiFi sleep is disabled by default when using `WIFI_STA` mode. |
| **Fix** | Delete or comment out the line. |

```cpp
// ❌ BEFORE:
WiFi.setSleep(false);

// ✅ AFTER:
// WiFi.setSleep(false);  // removed — not needed in newer core
```

---

### Error #5 — `undefined reference to setupLedFlash() / startCameraServer()`

| | |
|---|---|
| **Error** | `undefined reference to 'setupLedFlash()'` / `undefined reference to 'startCameraServer()'` |
| **File** | `main.cpp` (linker error) |
| **Line** | `88, 113` |
| **Cause** | `app_httpd.cpp` was placed inside the `include/` folder. PlatformIO **only compiles `.cpp` files found in `src/`** — files in `include/` are ignored by the compiler. |
| **Fix** | Move `app_httpd.cpp` from `include/` to `src/`. |

```
✅ Rule:
  .cpp files  →  src/
  .h files    →  include/
```

---

### Error #6 — Wrong WiFi library conflicts

| | |
|---|---|
| **Error** | `warning: converting to non-pointer type from NULL` / `integer overflow in DELAY_SPI` |
| **File** | `.pio/libdeps/esp32cam/WiFi/src/utility/wifi_drv.cpp` |
| **Line** | `451, 476` |
| **Cause** | The `platformio.ini` had `lib_deps` pointing to `https://github.com/arduino-libraries/WiFi.git` which is the Arduino WiFi Shield library — **not** the ESP32 WiFi. This conflicted with the built-in ESP32 WiFi that comes with the platform. |
| **Fix** | Remove wrong libraries from `lib_deps`. Delete cached `.pio/libdeps` and `.pio/build` folders. ESP32 WiFi is built into the platform and must not be added manually. |

<details>
<summary>📄 platformio.ini fix</summary>

```ini
; ❌ BEFORE:
lib_deps =
    https://github.com/arduino-libraries/WiFi.git
    https://github.com/espressif/arduino-esp32.git

; ✅ AFTER:
lib_deps =
    espressif/esp32-camera
```

Then delete these folders and rebuild:
```
.pio/libdeps/
.pio/build/
```

</details>

---

### Error #7 — `invalid header: 0xffffffff` (boot loop)

| | |
|---|---|
| **Error** | `invalid header: 0xffffffff` repeating in Serial Monitor |
| **File** | Serial Monitor output |
| **Line** | Runtime |
| **Cause** | Flash memory contained corrupt or old firmware. The bootloader could not find a valid firmware image and was stuck in a restart loop. |
| **Fix** | Erase flash completely, then re-upload. Add `board_build.flash_mode = dio` and `upload_speed = 115200` to `platformio.ini` for reliable flashing on ESP32-CAM. |

```bash
# Step 1 — Erase flash
pio run --target erase

# Step 2 — Re-upload
pio run --target upload
```

---

## Final Working `platformio.ini`

```ini
[env:esp32cam]
platform = espressif32@^6.5.0
board = esp32cam
framework = arduino
monitor_speed = 115200
upload_speed = 115200
board_build.flash_mode = dio

lib_deps =
    espressif/esp32-camera
```

---

## Final Project Structure

```
live video stream/
├── include/
│   ├── board_config.h
│   ├── camera_index.h
│   └── camera_pins.h          ← manually created
├── src/
│   ├── app_httpd.cpp          ← moved from include/
│   └── main.cpp
└── platformio.ini
```

---

*AI Thinker ESP32-CAM + OV3660 — PlatformIO Arduino Framework*
