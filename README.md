# OLED Not Working / "SSD1306 Allocation Failed" — Code & Checklist

I2C scanner sketch, RAM-saving fixes for `"SSD1306 allocation failed"`, and `SH1106` configuration for 1.3" OLED displays on Arduino Uno/Nano and ESP32.

---

### 1. I2C Address Scanner Sketch
Run this first to verify wiring and find your screen's true 7-bit I2C address (`0x3C` or `0x3D`). Note: silkscreen labels showing `0x78` or `0x7A` are 8-bit representations of `0x3C` and `0x3D`.

```cpp
#include <Wire.h>

void setup() {
  Serial.begin(115200);
  Wire.begin();   // ESP32 with custom pins: Wire.begin(SDA_PIN, SCL_PIN);
  for (uint8_t addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
      Serial.print("Found device at 0x");
      Serial.println(addr, HEX);
    }
  }
  Serial.println("Done");
}

void loop() {}
```

---

### 2. Standard 0.96" SSD1306 Minimal Test (`Adafruit_SSD1306`)
* For 4-pin I2C modules (without a reset pin), set `OLED_RESET` to `-1`.
* Always call `display.display()` after drawing, or the screen stays black!
* Use the `F()` macro on Arduino Uno/Nano so strings stay in Flash instead of eating up the 2 KB SRAM needed for the 1024-byte screen buffer.

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Serial.begin(115200);
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println(F("OLED is working!"));
  display.display(); // pushes buffer to screen
}

void loop() {}
```

---

### 3. Low-RAM Alternative (`U8g2` Page Mode — Only 128 Bytes of RAM)
If your Arduino Uno/Nano project uses an SD card or big libraries and still hits `"SSD1306 allocation failed"`, switch to `U8g2` in **page buffer mode** (`_1_` instead of `_F_`):

```cpp
#include <U8g2lib.h>
#include <Wire.h>

// _1_ = 128-byte page buffer (saves ~896 bytes of RAM compared to full buffer)
U8G2_SSD1306_128X64_NONAME_1_HW_I2C u8g2(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);

void setup() {
  u8g2.begin();
  u8g2.firstPage();
  do {
    u8g2.setFont(u8g2_font_ncenB08_tr);
    u8g2.drawStr(0, 15, "Low RAM mode works!");
  } while (u8g2.nextPage());
}

void loop() {}
```

---

### 4. Fix for 1.3" OLED with Noise Stripe (`SH1106` Driver)
Most 1.3" I2C OLEDs use the **SH1106** controller (132x64 internal RAM), not SSD1306:
* **Adafruit library:** Install `Adafruit SH110X` and use `Adafruit_SH1106G display(128, 64, &Wire, -1);` with `display.begin(0x3C, true);`
* **U8g2 library:** Use constructor `U8G2_SH1106_128X64_NONAME_1_HW_I2C`

---

### 5. Quick Pinout Reference
* **Arduino Uno / Nano:** `SDA -> A4` | `SCL -> A5` | `VCC -> 5V (or 3.3V)` | `GND -> GND`
* **ESP32 DevKit:** `SDA -> GPIO 21` | `SCL -> GPIO 22` | `VCC -> 3V3` | `GND -> GND`
* **Warning:** Always check the silkscreen on your OLED module—both `GND-VCC-SCL-SDA` and `VCC-GND-SCL-SDA` pin orders exist!
