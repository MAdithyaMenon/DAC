# ESP32 smart desk DAC — build guide

*A Bluetooth-receiving DAC with a live display, weather, and a cloud-AI assistant, built around one ESP32.*

## Quick summary

- **Core idea:** the ESP32 receives music over Bluetooth (like any Bluetooth speaker), converts it to clean analog audio through a real DAC chip, and drives a small display showing time, weather, and what's playing.
- **The "AI" layer:** the ESP32 calls a cloud LLM API over WiFi and shows the answer on-screen — text first, voice later if you want to push further. There's no way to run a real LLM on the chip itself, and that's normal, not a compromise.
- **Board:** original ESP32 (WROOM-32 / WROVER) — **not** S3, S2, C3, or C6. Explained in section 1, and it's the one decision that can quietly sink the whole project if you get it wrong at checkout.
- **Budget:** roughly ₹1,900–2,900 for the core build, up to ~₹4,300 with the nicer round display and standalone speakers. Comfortably inside your ₹3,000–8,000 range, with room left for mistakes.
- **Timeline:** about 4–6 weekends phase by phase, longer if you push into the voice-AI phase.

---

## 1. The one decision that makes or breaks this

Buy an **ESP32-WROOM-32** or **ESP32-WROVER** dev board — the "classic" ESP32 — not any of the newer S3 / S2 / C3 / C6 chips.

Streaming music from a phone over Bluetooth uses a profile called **A2DP**, which runs over **Bluetooth Classic**. The ESP32-S3, S2, C3, and C6 only have **Bluetooth Low Energy (BLE)** — there's no Classic radio on those chips at all, so they physically cannot receive streamed phone audio this way. Only the original ESP32 has WiFi and Classic Bluetooth on the same silicon, which is exactly the combination this project needs (Bluetooth for audio, WiFi for time/weather/AI).

When shopping, look for boards labeled "ESP32 DevKit," "ESP32-WROOM-32," "NodeMCU-32S," "ESP32 DevKitC," or "WeMos LOLIN32" — these are all the classic chip in different board shapes and are what you want. Skip anything with "S3," "S2," "C3," or "C6" in the name for this particular build.

## 2. How the pieces fit together

Four things happen around the ESP32 (this is the system you saw in the diagram above):

1. **Phone → ESP32, over Bluetooth.** You pair your phone with the ESP32 exactly like pairing a Bluetooth speaker. Music streams over as A2DP audio.
2. **ESP32 → DAC → speaker/headphones, over I2S.** I2S is a digital audio protocol — a dedicated few-wire path for audio data, separate from general-purpose GPIO. The DAC chip turns that digital stream into the analog signal your speakers or headphones actually need.
3. **ESP32 → display, over I2C or SPI.** Shows the clock, weather, and — once wired up — the track that's playing.
4. **ESP32 → WiFi → the internet.** Fetches accurate time (NTP), weather (OpenWeatherMap), and later, AI responses.

A rotary encoder rounds it out: twist for volume, click for menu/screen switching.

One nice technical detail: the standard Bluetooth library for this (`BluetoothA2DPSink`, by pschatzmann) doesn't just move audio — it also speaks AVRCP, the sub-protocol phones use to share track metadata. That's what lets you show song title and artist on the display later, instead of just playing audio blind.

## 3. Bill of materials

Prices are what I found across Indian sellers (Robu.in, Robocraze, Robokits India, HubTronics, ElectronicsComp, Zbotic) in early August 2026 — always cross-check current listings, since component pricing shifts and varies by seller.

| Component | What it does | Qty | Approx price (₹) | Notes |
|---|---|---|---|---|
| ESP32 DevKit (WROOM-32) | The brain — WiFi + Classic Bluetooth | 1 | 350–500 | Buy 2 if your budget allows — a burnt pin or bad solder joint on the first one is extremely common and not a sign you did anything wrong |
| PCM5102A I2S DAC module | Converts digital audio to clean stereo line-out | 1 | 250–450 | This is your actual "DAC" — 3.5mm jack feeds existing speakers, a headphone amp, or powered speakers |
| 0.96" I2C OLED (SSD1306, 128×64) | Phase 1–4 display: clock, weather, status | 1 | 150–250 | Start here — far simpler to code than a color display |
| GC9A01 1.28" round TFT (240×240, SPI) | Phase 5 display upgrade — the "smart hub" look | 1 | 400–650 | Optional; add once the OLED version is working end to end |
| DS3231 RTC module | Keeps accurate time even if WiFi/NTP drops | 1 | 100–150 | Shares the I2C bus with the OLED — no extra pins needed |
| KY-040 rotary encoder (with push button) | Volume control + menu navigation | 1 | 50–100 | |
| PAM8610 stereo amp module *(optional)* | Only needed for built-in speakers instead of external ones | 1 | 120–200 | Skip entirely if you'll plug into speakers/headphones you already own |
| 2× small 4Ω 3–5W speakers *(optional)* | Pairs with the PAM8610 for a standalone unit | 2 | 300–500 total | Skip if using external speakers |
| INMP441 I2S MEMS mic *(Phase 7, optional)* | Voice input for the "talk to it" upgrade | 1 | 150–250 | Only needed for the advanced voice phase |
| Perfboard/general PCB, jumper wires, headers, resistors, solder | Assembly | – | 300–500 | |
| USB cable + 5V/2A power adapter | Power | 1 | 150–250 | Use a proper wall adapter, not a laptop USB port — see troubleshooting |
| Enclosure (project box, 3D print, or laser-cut acrylic) | Housing | 1 | 300–1000 | Cardboard/acrylic DIY is completely fine for v1 |

**Core build total (Phases 1–4, no standalone speakers, OLED display):** ~₹1,900–2,900
**With round display + standalone speakers (Phases 1–5):** ~₹2,900–4,300

Either way, you're well inside ₹3,000–8,000, with room for spares or a nicer enclosure.

### PCM5102A vs. MAX98357A — which DAC to actually buy

- **PCM5102A (recommended default):** true stereo line-out DAC, no built-in amp. Feeds your existing speakers, headphones, or a headphone amp through a 3.5mm jack. This is the "proper DAC" experience and what most people picture when they say "desk DAC."
- **MAX98357A (simpler alternative):** combines a mono DAC *and* a small amplifier on one board, driving a speaker directly — no separate amp needed. Great if you want a fully standalone speaker box and don't mind mono sound (fine for spoken AI replies, less ideal for music, since you'd need two boards for real stereo).

If you're not sure, start with the PCM5102A — it's the more flexible and better-sounding starting point, and you can always add speakers later.

## 4. Software you'll need

- **Arduino IDE** (2.x) with the ESP32 board package added via Boards Manager — search "esp32" and install the one by **Espressif Systems**.
- Libraries, all installable via Library Manager:
  - **ESP32-A2DP** by pschatzmann — the Bluetooth audio receiver
  - **Arduino Audio Tools** by pschatzmann — I2S streaming helper that pairs with A2DP
  - **Adafruit SSD1306** + **Adafruit GFX** — for the OLED (Phases 1–4)
  - **Arduino_GFX** or **Adafruit GC9A01A** — for the round display (Phase 5)
  - **RTClib** by Adafruit — for the DS3231
  - **ArduinoJson** — for parsing weather/AI API responses
  - `WiFi.h` and `HTTPClient.h` — already built into the ESP32 core, nothing to install

### What you can (and can't) simulate

Checked this directly against Wokwi's current peripheral support table rather than assume, since it changes how you should spend the weeks before parts arrive:

- **Bluetooth is listed as not implemented, for every ESP32 variant Wokwi simulates** — not "partial," not "BLE only," just absent, with an open feature request tracking it. Nothing to test against today.
- **I2S is listed as partial / work in progress, and only on the classic ESP32** (S3/C3/C6 don't get it at all). In practice, your I2S driver code will compile and run without erroring in the simulator — genuinely useful for catching a typo or config mistake — but there's no virtual DAC or speaker rendering actual sound, so it can't confirm your wiring or audio path is correct.

Net effect: **Phase 1 needs real hardware, full stop.** Two things are still worth doing in Wokwi while your parts are in transit:

1. **Compile-check the Phase 1 sketch anyway.** Paste it into a blank Wokwi ESP32 project with nothing wired up and hit run. It won't pair or play anything, but it'll catch missing includes, typos, and library issues before you're debugging on real hardware.
2. **Fully build and test Phase 2 (the clock).** I2C and WiFi are both completely simulated (not partial), so an OLED + NTP clock will behave in Wokwi exactly like it will on the real board — genuinely worth finishing before your parts show up. Wokwi even has a ready-made NTP example project you can start from (search "NTP Client" among their project examples, or start from a blank ESP32 template and add an SSD1306 OLED part from the parts picker).

One detail that trips people up: Wokwi's simulated WiFi network is always called `Wokwi-GUEST`, open, no password. That's what goes in `WiFi.begin()` *only* while you're inside the simulator — swap it for your real home WiFi credentials the moment you flash actual hardware.

## 5. Suggested pin map

Designed so nothing collides, including the optional later phases:

| Function | Signal | GPIO |
|---|---|---|
| I2S → DAC | BCK | 26 |
| I2S → DAC | WS/LRC | 25 |
| I2S → DAC | DATA OUT | 27 |
| I2C → OLED + DS3231 (shared bus) | SDA | 21 |
| I2C → OLED + DS3231 (shared bus) | SCL | 22 |
| Rotary encoder | CLK | 32 |
| Rotary encoder | DT | 33 |
| Rotary encoder | SW (button) | 4 |
| SPI → round display *(Phase 5)* | SCK | 18 |
| SPI → round display *(Phase 5)* | MOSI | 23 |
| SPI → round display *(Phase 5)* | CS | 5 |
| SPI → round display *(Phase 5)* | DC | 2 |
| SPI → round display *(Phase 5)* | RST | 15 |
| I2S → mic, second peripheral *(Phase 7)* | BCK / WS / SD | 14 / 13 / 12 |
| Push-to-talk button *(Phase 7)* | — | 17 |

GPIO 2 and 12 are strapping pins on some boards — if you get boot weirdness after wiring the round display or mic, that's the first thing to suspect; move to a spare pin like 17 (if free) and retest.

---

## 6. Phased build plan

### Phase 1 — Get sound out (week 1)

**Goal:** pair your phone with the ESP32 and hear music through the DAC.

#### 1a. Wire the PCM5102A

| PCM5102A pin | Connects to | Why |
|---|---|---|
| VCC | ESP32 3.3V | Power. Many boards also tolerate 5V thanks to an onboard regulator, but 3.3V is always safe — check your specific board if unsure |
| GND | ESP32 GND | Ground |
| BCK | GPIO 26 | Bit clock |
| LCK (LRCK) | GPIO 25 | Word select / left-right clock |
| DIN | GPIO 27 | Audio data |
| SCK | GND | Tells the DAC to generate its own internal clock from BCK — no separate master clock needed |
| FMT | GND | Selects standard I2S format |
| DEMP | GND | De-emphasis off — leave off unless you specifically need it |
| XSMT | 3.3V | **Un-mutes the chip** — see the warning below |
| OUT_L / OUT_R / GND | Speakers, headphone amp, or a 3.5mm jack if not pre-soldered | Analog audio out |

**The one wiring mistake that causes 90% of "I did everything right but there's no sound" reports on this exact board:** XSMT has to sit HIGH (3.3V) to un-mute the chip — left on GND or floating, you get perfectly correct-looking wiring and dead silence. On many of these boards, SCK/FMT/DEMP/XSMT aren't loose header pins at all but tiny solder-jumper pads on the back labeled H1L–H4L, bridged to Low (GND) at the factory. Three of those four factory defaults (SCK, FMT, DEMP) are exactly what you want already — but the XSMT one needs to move to High, either with a jumper wire to 3.3V if it's a header pin on your board, or by re-flowing that one solder bridge if it's pad-only. Look at your specific board before assuming either way.

#### 1b. Install libraries and select your board

1. Library Manager → install **ESP32-A2DP** (by pschatzmann) and **Audio Tools** / **arduino-audio-tools** (also pschatzmann).
2. Tools → Board → ESP32 Arduino → **ESP32 Dev Module**.
3. Tools → Port → select the port that appears once the board is plugged in. If nothing shows up, install the CP2102 or CH340 USB driver, depending on which USB-serial chip your specific board uses.

#### 1c. Flash this

```cpp
#include "AudioTools.h"
#include "BluetoothA2DPSink.h"

I2SStream i2s;
BluetoothA2DPSink a2dp_sink(i2s);

void setup() {
  Serial.begin(115200);

  auto cfg = i2s.defaultConfig();
  cfg.pin_bck  = 26;   // -> PCM5102A BCK
  cfg.pin_ws   = 25;   // -> PCM5102A LCK
  cfg.pin_data = 27;   // -> PCM5102A DIN
  i2s.begin(cfg);

  a2dp_sink.start("Desk DAC");  // the name your phone will see in Bluetooth settings
  Serial.println("Ready — look for 'Desk DAC' in your phone's Bluetooth settings.");
}

void loop() {
  // A2DP runs in the background via its own tasks — nothing needed here yet
}
```

#### 1d. Pair and test

1. Upload, then open the Serial Monitor at 115200 baud — you should see the "Ready" message.
2. On your phone: Settings → Bluetooth → scan → tap **Desk DAC** to pair, same as pairing headphones. No PIN needed.
3. Play any audio on the phone. It should reach your speakers within a second or two of connecting.
4. Paired but silent → go straight back to XSMT.
5. Choppy or crackly → try a proper 5V/2A wall adapter instead of a laptop USB port; that alone fixes most of these.

This is genuinely the hardest technical leap in the whole project. Once this works, everything after it is comparatively easy.

### Phase 2 — Clock (week 1–2)

Wire the OLED to the I2C pins above (it'll share the bus with the DS3231 if you add that now too). Sync time over NTP:

```cpp
#include <WiFi.h>
#include "time.h"

const char* ssid = "your_wifi_name";
const char* password = "your_wifi_password";
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 19800;   // IST = UTC+5:30
const int   daylightOffset_sec = 0;

void setup() {
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
}

void loop() {
  struct tm timeinfo;
  if (getLocalTime(&timeinfo)) {
    // timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec — draw these to the OLED
  }
  delay(1000);
}
```

If you added the DS3231, use it to keep time through reboots and brief WiFi drops, syncing it from NTP whenever WiFi is available.

### Phase 3 — Weather (week 2)

Sign up for a free OpenWeatherMap API key (just needs an email). Their free tier is still active and gives roughly a thousand calls a day, which is enormous overkill for a device checking the forecast every 10–15 minutes — you'll never come close to the limit.

```cpp
#include <HTTPClient.h>
#include <ArduinoJson.h>

void fetchWeather() {
  HTTPClient http;
  String url = "http://api.openweathermap.org/data/2.5/weather?q=Thrissur,IN&appid=YOUR_API_KEY&units=metric";
  http.begin(url);

  if (http.GET() == 200) {
    String payload = http.getString();
    JsonDocument doc;  // ArduinoJson v7 syntax — use StaticJsonDocument<1024> doc; on v6
    deserializeJson(doc, payload);

    float temp = doc["main"]["temp"];
    const char* condition = doc["weather"][0]["main"];
    // update the OLED with temp and condition
  }
  http.end();
}
```

Call this every 10–15 minutes, not every loop — no need to hammer the API, and it saves power/WiFi airtime for the audio side.

### Phase 4 — Volume + now playing (week 2–3)

Wire the rotary encoder to the pins above. Reading it is a standard interrupt-driven quadrature decode — any ESP32 rotary encoder library (or a short interrupt handler) works fine here; this isn't specific to this project.

For "now playing," `BluetoothA2DPSink` exposes AVRCP metadata callbacks (track title, artist, play/pause state). The exact callback names shift slightly between library versions, so once it's installed, check its examples folder for the current metadata callback signature and wire it to update your display.

### Phase 5 — Round display upgrade (optional, week 3–4)

Swap the OLED for the GC9A01 using the SPI pins above. This is a bigger jump than it looks: you go from simple monochrome text (OLED) to a color framebuffer, which usually means adopting a UI library like **LVGL** on top of `Arduino_GFX` for anything beyond basic shapes and text. Budget extra time here — it's a real step up in complexity, not a drop-in swap.

### Phase 6 — Text AI assistant (optional, week 4+)

The realistic, achievable version of "AI" on this hardware:

1. The ESP32 hosts a tiny local web page (`WebServer.h`) with a text box.
2. From your phone or laptop, on the same WiFi, visit the ESP32's IP address, type a question, hit submit.
3. The ESP32 does an HTTPS POST to an LLM API — Claude, GPT, and Gemini all offer API access suitable for a hobby project like this; check current pricing and free-tier terms when you get to this phase, since they change often.
4. The response comes back as JSON; extract the text and show it on the display (scrolling if it's long).

No microphone needed yet — your phone's keyboard is the input device. This alone will feel like a genuinely "smart" device.

### Phase 7 — Voice AI (optional, advanced, week 5+)

The full "talk to it" version. Worth attempting only after phases 1–6 are solid:

1. Add the INMP441 mic on the second I2S peripheral (I2S1 — the DAC already occupies I2S0).
2. Use a push-to-talk button rather than always-listening wake-word detection — dramatically simpler, and wake-word spotting on an ESP32 is its own multi-week project.
3. Record a few seconds of audio on button press, send it to a speech-to-text service.
4. Send the transcribed text to the same LLM API from Phase 6.
5. Convert the reply to speech via a text-to-speech service, play it back through the existing DAC/speaker path.

Be honest with yourself about scope here — this chains together four separate cloud calls with real network latency at each hop. Most hobbyists spend a couple of weeks smoothing this out. It's a great phase 2 project once the core device is something you actually use daily.

---

## 7. Enclosure ideas

- **Cheapest:** acrylic sheet or a repurposed plastic project box, hand-cut holes for the display, speaker grille, and USB port.
- **Middle ground:** 3D printed enclosure — look for a local makerspace, college fab lab, or an online print-on-demand service if you don't have access to a printer.
- **Nicest:** laser-cut acrylic or plywood, stacked/layered design — common for "desk gadget" aesthetics and not too expensive per panel.

Don't let the enclosure block you from starting — breadboard the whole thing and enjoy it functioning before you spend time on housing it.

## 8. Problems you'll likely hit

- **Phone pairs fine, but there's total silence** — check XSMT on the PCM5102A first. It needs to be HIGH (3.3V) to un-mute; GND or floating gives you correct-looking wiring and no sound. See section 6, Phase 1.
- **Choppy/crackling Bluetooth audio** — usually WiFi+Bluetooth radio coexistence, insufficient I2S DMA buffer settings, or a noisy USB power source. Try a proper 5V/2A wall adapter instead of a laptop USB port first; it's the most common fix.
- **OLED shows nothing, or "SSD1306 allocation failed"** — wrong I2C address (try both `0x3C` and `0x3D` in the library init call), or a wiring/voltage mismatch. Double-check the OLED's rated voltage against what you're feeding it.
- **ESP32 won't accept new firmware, hangs at "Connecting..."** — hold the board's BOOT button during the upload, or install the correct USB driver for your board's chip (CP2102 or CH340, depending on model).
- **Phone can't find "Desk DAC"** — Bluetooth Classic devices show up in your phone's normal Bluetooth settings list (same place as headphones), not in a separate BLE scanner app.
- **Clock is wrong or won't update** — check the GMT offset math, confirm the ESP32 can actually reach `pool.ntp.org`, and if you're relying on the DS3231 standalone, check its CR2032 backup battery.

## 9. Where to buy (India)

Robu.in, Robocraze, Robokits India, HubTronics, and Amazon.in carry everything on this list. Prices vary somewhat by seller — worth a quick compare across two or three before ordering, especially for the ESP32 board itself where prices ranged widely across sellers.

## 10. Budget summary

| Build | Approx total |
|---|---|
| Core (Phases 1–4, OLED, no standalone speakers) | ₹1,900–2,900 |
| + round display (Phase 5) | +₹250–400 |
| + standalone speakers (PAM8610 + 2 speakers) | +₹400–700 |
| + mic for voice AI (Phase 7) | +₹150–250 |
| **Full build, everything included** | **~₹2,900–4,300** |

All comfortably inside your ₹3,000–8,000 budget, with room left over for a spare ESP32, a nicer enclosure, or better speakers if you want to spend the rest.

---

*Come back for full working code on any phase once your parts arrive — best built and debugged incrementally against real hardware rather than written all at once.*
