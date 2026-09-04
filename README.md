# GIF/PNG → OLED Bitmap Converter

## [Click here to access the tool online](https://modernwestern.github.io/gif-to-oled/)

A browser‑based tool that converts animated GIFs or static PNGs into **1‑bit per pixel** bitmaps for **128×64** OLED displays (e.g., SSD1306 / SSD1309).

It produces a compact binary file (.bin) that can be read from LittleFS, or a C header (.h) with PROGMEM arrays ready to compile directly into your Arduino firmware.

> **All processing runs client‑side** – no data is uploaded to any server.
> 

---

## Features

- **Drag & drop** GIF or PNG files – supports both static and animated content.
- **Smart fitting** – choose between *contain* (adds borders), *cover* (crops), or *stretch*.
- **Color quantization** – reduce the number of input levels per channel before thresholding (controls banding/detail).
- **Threshold** – adjust the black/white cutoff (1–254).
- **Floyd–Steinberg dithering** – preserves gradients and smooths edges (toggle on/off).
- **Invert** – swap black and white (useful for white‑background designs).
- **Frame rate optimisation** – resample GIF frames to a fixed FPS (e.g., 15 or 20) without changing the total animation duration, reducing memory usage.
- **Live preview** – see the monochrome output in real time, with a pixel‑grid overlay.
- **Export formats**:
    - **.bin** – a self‑describing binary file (magic `MOLD`, width, height, frame count, per‑frame delays, and frame data). Ideal for streaming from SD/LittleFS.
    - **.h** – a C header containing PROGMEM‑stored frame arrays and delays, ready to use with U8g2’s `drawXBMP()`.

---

## How to use

1. **Open the tool** in any modern web browser (Chrome, Firefox, Edge, Safari).
2. **Upload** a GIF or PNG via drag‑and‑drop or the file picker.
3. **Adjust** the conversion parameters (fit, threshold, dithering, etc.) – the preview updates instantly.
4. **Export** your animation:
    - Enter a **variable name** (used for the .h header and .bin filename).
    - Click **Download .bin** – a binary file ready for LittleFS.
    - Click **Download .h** – a C header with all frame data as `PROGMEM` constants.
5. **Use** the output in your Arduino project (see snippet below).

---

## Binary file format (`.bin`)

The `.bin` file is a compact, self‑describing format with the following structure (little‑endian):

| Offset | Size | Description |
| --- | --- | --- |
| 0 | 4 bytes | Magic bytes: `"MOLD"` (0x4D 0x4F 0x4C 0x44) |
| 4 | 2 bytes | Width (always 128) |
| 6 | 2 bytes | Height (always 64) |
| 8 | 2 bytes | Number of frames (uint16) |
| 10 | n×2 | Frame delays in milliseconds (uint16 per frame) |
| 10+n×2 | n×1024 | Frame data, each frame = 1024 bytes (128×64 / 8) in **XBM order** (LSB‑first, bit0 = leftmost pixel) |

**Reading from LittleFS with U8g2** (Arduino example):

```cpp
#include <Arduino.h>
#include <U8g2lib.h>
#include <LittleFS.h>

U8G2_SSD1306_128X64_NONAME_1_HW_I2C u8g2(...);

void playAnimation(const char* path) {
  File f = LittleFS.open(path, "r");
  if (!f) return;

  uint8_t header[10];
  f.read(header, 10);
  uint16_t w      = header[4] | (header[5] << 8);
  uint16_t h      = header[6] | (header[7] << 8);
  uint16_t frames = header[8] | (header[9] << 8);

  uint16_t delays[frames];
  f.read((uint8_t*)delays, frames * 2);

  const uint16_t BUF_SIZE = (w / 8) * h;  // 1024
  uint8_t frameBuf[BUF_SIZE];

  for (int i = 0; i < frames; i++) {
    f.read(frameBuf, BUF_SIZE);
    u8g2.clearBuffer();
    u8g2.drawXBMP(0, 0, w, h, frameBuf);
    u8g2.sendBuffer();
    delay(delays[i]);
  }
  f.close();
}

void setup() {
  LittleFS.begin();
  playAnimation("/myAnimation.bin");
}
```

## **C header (`.h`) usage**

The generated `.h` file defines:

- `#define YOURNAME_FRAME_COUNT`
- `const uint8_t YOURNAME_frame_0[] PROGMEM = { ... }` for each frame
- `const uint8_t* const YOURNAME_frames[] PROGMEM = { ... }`
- `const uint16_t YOURNAME_delays[] PROGMEM = { ... }`

**Example U8g2 usage**:

cpp

```
#include "myAnimation.h"

void loop() {
  for (int i = 0; i < MYANIMATION_FRAME_COUNT; i++) {
    u8g2.clearBuffer();
    u8g2.drawXBMP(0, 0, MYANIMATION_WIDTH, MYANIMATION_HEIGHT,
                  (const uint8_t*)pgm_read_ptr(&myAnimation_frames[i]));
    u8g2.sendBuffer();
    delay(pgm_read_word(&myAnimation_delays[i]));
  }
}
```

---

## **Why this tool?**

- **No online dependencies** – runs fully offline after loading the page.
- **Exact 128×64 output** – designed for small OLED screens commonly used in DIY projects.
- **Memory‑conscious** – the binary format and PROGMEM arrays keep frame data in flash, not RAM.
- **Flexible frame handling** – you can control the number of frames via the FPS optimisation, balancing quality and storage.

---

## **Technical notes**

- The tool supports Floyd–Steinberg dithering to smooth gradients. But keep in mind that the physical screen is just 128×64 pixels, so don't expect wonders. Fine details will always be blocky regardless of dithering.
- **Color quantisation** can be used to pre‑band the image before thresholding – this can help emphasise edges or reduce noise.
- For **GIFs**, the tool respects the original frame delays and disposal methods, preserving animation timing.
- For **PNGs**, the file is treated as a single static frame.

---

## **License**

MIT – use it freely in your own projects, commercial or otherwise.
