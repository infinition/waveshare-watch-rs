# waveshare-watch-rs

[![oosmetrics](https://api.oosmetrics.com/api/v1/badge/achievement/d90d81a8-b5be-4e00-8418-c1e0b9321f57.svg)](https://oosmetrics.com/repo/infinition/waveshare-watch-rs)

##  Featured On
* **Hackaday:** [Rust-y Firmware For Waveshare Smartwatch](https://hackaday.com/2026/04/11/rust-y-firmware-for-waveshare-smartwatch/)
  
<img width="400" height="270" alt="image" src="https://github.com/user-attachments/assets/331e6778-bf67-4f5f-a753-b65d6e0f1d57" />


100% Rust `no_std` smartwatch firmware for the **Waveshare ESP32-S3-Touch-AMOLED-2.06**.

Complete conversion of the original C/C++ project (ESP-IDF + Arduino GFX + LVGL) to a single-binary Rust codebase relying on `esp-hal` 1.0, `esp-rtos`, Embassy, and custom drivers for each of the board's peripherals.

The firmware handles the 410×502 QSPI display in 80 MHz DMA, the I2S audio codec, 2.4 GHz WiFi with NTP sync, SD card, capacitive touch, gyroscope, hardware RTC, AXP2101 power management, a launcher with 5 mini-games, a T9 keyboard, an MP3 player (UI), a Smart Home app (HTTP), a sleep/wake mode with an Apple Watch style Always-On Display, and an event-driven main loop based on GPIO interrupts to leave the CPU asleep >99% of the time on the watchface.

---

## Hardware target

| Component        | Reference                                        | Bus            |
|------------------|--------------------------------------------------|----------------|
| SoC              | ESP32-S3R8 (Xtensa LX7 dual-core, 8 MB PSRAM)    | —              |
| Display          | CO5300 AMOLED 410×502, rounded edge              | QSPI 80 MHz    |
| PMIC             | AXP2101 (charging, power rails)                  | I2C 400 kHz    |
| Touch            | FT3168                                           | I2C + INT GPIO |
| IMU              | QMI8658 (accelerometer + gyroscope + temp)       | I2C            |
| RTC              | PCF85063A                                        | I2C            |
| Audio codec      | ES8311 + amp                                     | I2C + I2S      |
| Memory           | 32 MB flash, 8 MB octal PSRAM                    | —              |
| SD Card          | SDHC SPI                                         | SPI3           |
| WiFi + BLE       | Integrated 2.4 GHz                               | —              |
| Tearing Effect   | GPIO13                                           | IRQ            |

Waveshare Wiki reference: [https://www.waveshare.com/wiki/ESP32-S3-Touch-AMOLED-2.06](https://www.waveshare.com/wiki/ESP32-S3-Touch-AMOLED-2.06)

Full pinout in `src/board.rs`.

---

## Software stack

| Layer        | Crate                           | Role                                                   |
|--------------|---------------------------------|--------------------------------------------------------|
| HAL          | `esp-hal` 1.0                   | Peripherals (GPIO, I2C, SPI, I2S, DMA, timers, PSRAM)  |
| Runtime      | `esp-rtos` 0.2                  | Boot, executor, radio integration                      |
| Async        | `embassy-executor` 0.9          | Cooperative scheduler, tasks, timers                   |
|              | `embassy-time`, `embassy-futures` | `select`, `Timer::after`, `Duration`                   |
|              | `embassy-net` 0.9               | smoltcp TCP/IP stack (DHCPv4, TCP, UDP, DNS)           |
| Radio        | `esp-radio` 0.17                | WiFi driver + Embassy interface                        |
| Graphics     | `embedded-graphics` 0.8         | 2D primitives, fonts, text layout                      |
| Storage      | `embedded-sdmmc` 0.8            | FAT32 on SD card                                       |
| Codec        | `nanomp3` 0.1                   | `no_std` MP3 decoder (prepared, not wired)             |
| Allocator    | `esp-alloc`                     | 64 KB SRAM heap + 8 MB PSRAM heap                      |
| Panic        | `esp-backtrace`                 | Symbolic backtrace via `esp-println`                   |

Zero C code, zero ESP-IDF components compiled into the final target. The bootloader is `esp-bootloader-esp-idf` on the loader side only.

---

## Architecture

```
main.rs
│
├── esp_hal::init(CpuClock::_160MHz)
├── PSRAM allocator init
├── esp_rtos::start(timer)         ← Embassy executor
│
├── [Peripherals init] ──── drivers/ + peripherals/
│   ├── Shared I2C bus (RefCell + RefCellDevice)
│   ├── AXP2101 power rails + battery monitor
│   ├── QSPI SPI2 80 MHz DMA 8 KB → Co5300Display
│   ├── PSRAM Framebuffer double buffer (2 × 402 KB)
│   ├── FT3168 touch + GPIO38 INT
│   ├── PCF85063A RTC
│   ├── QMI8658 IMU (power_down by default)
│   ├── SD SPI3 4 MHz
│   ├── ES8311 codec + I2S0 DMA
│   └── WiFi esp-radio + embassy-net DHCPv4 + NTP sync
│
├── [Event-driven loop]
│   │
│   ├── select3(
│   │       Timer::after(adaptive_tick),
│   │       touch_int.wait_for_falling_edge(),
│   │       boot_button.wait_for_falling_edge(),
│   │     ).await
│   │
│   ├── Sensors I/O (gated by screen_state + business need)
│   ├── Touch poll I2C (only if finger is pressed or just lifted)
│   ├── State machine sleep/wake (4 levels + AOD)
│   ├── WiFi auto-disconnect idle >5min
│   └── App state machine
│       ├── Watchface (3 pages: Clock / Sensors / System)
│       ├── Launcher
│       ├── Snake, 2048, Tetris, Flappy, Maze
│       ├── MP3 Player (UI)
│       ├── Settings + T9 keyboard
│       └── SmartHome (buttons → HTTP)
```

---

## Module organization

```
src/
├── main.rs                 Hardware init + async main loop
├── board.rs                Pinout + display dimensions
│
├── drivers/                Low-level drivers (direct hardware)
│   ├── qspi_bus.rs         QSPI bus quad-mode half-duplex, begin/stream/end
│   ├── co5300.rs           CO5300 init sequence, addr window, set_brightness,
│   │                       display_on/off (MIPI DCS), TEARON
│   └── framebuffer.rs      410×502 RGB565 PSRAM FB, double buffer, flush_vsync
│
├── peripherals/            High-level I2C / SPI / I2S drivers
│   ├── power.rs            AXP2101: battery %, voltage, is_charging, power rails
│   ├── touch.rs            FT3168: read, tracking swipe/tap, SwipeDirection
│   ├── rtc.rs              PCF85063A: get_time, set_time, DateTime
│   ├── imu.rs              QMI8658: read_accel/gyro/temp, power_up/down
│   ├── audio.rs            ES8311: Waveshare registers init, mute/unmute, beep
│   ├── sdcard.rs           Stub wrapper around embedded-sdmmc
│   ├── wifi.rs             WiFi types (scan stub)
│   └── http.rs             HTTP GET/POST client via embassy-net TCP
│
├── ui/                     UI components rendered on DrawTarget<Rgb565>
│   ├── watchface.rs        Clock + gyro ball + battery + FR date + AOD
│   ├── segments.rs         7-segment digits for time
│   ├── pages.rs            Clock / Sensors / System pages
│   ├── launcher.rs         App list, interpolated scroll
│   └── t9_keyboard.rs      Alphanumeric T9 keyboard
│
└── apps/                   Applications (implement the App trait)
    ├── snake.rs            Snake with I2S beep on consume
    ├── game2048.rs         2048 swipe merge
    ├── tetris.rs           Tetris gyro + touch
    ├── flappy.rs           Flappy Bird (direct rendering + framebuffer)
    ├── maze.rs             Maze with IMU ball
    ├── settings.rs         WiFi SSID/password + T9
    ├── mp3player.rs        MP3 player UI (decoding to be wired)
    └── smarthome.rs        Button grid → HTTP GET/POST
```

---

## Power management

The firmware is designed to leave the CPU parked most of the time.
The Embassy executor only wakes the core on:
- the GPIO38 touch interrupt (FT3168 in monitor mode)
- the GPIO0 button interrupt
- a periodic timer whose period depends on the current state

### Screen states (4 levels + AOD)

| State | Brightness | Idle trigger | Behavior                                                                         |
|-------|------------|--------------|----------------------------------------------------------------------------------|
| 3     | 0xD0       | —            | Normal interactive, full bright                                                  |
| 2     | 0x40       | 20 s         | Dimming (transition), still interactive                                          |
| 1     | 0x18       | 40 s         | **AOD**: minimal HH:MM, pure black background (AMOLED pixels OFF), 1 update/min  |
| 0     | DISPOFF    | 10 min in AOD| SLPIN panel, QSPI idle, only GPIO IRQ for wake                                   |

On wake via touch/button: immediate return to state 3, framebuffer forced into full redraw.

### Adaptive main loop ticks

| Context                            | Tick     | Effective frequency |
|------------------------------------|----------|---------------------|
| Screen OFF (state 0)               | 30 s     | 0.033 Hz            |
| AOD (state 1)                      | 10 s     | 0.1 Hz              |
| Watchface clock, gyro off          | 1 s      | 1 Hz                |
| Watchface clock, gyro on           | 33 ms    | ~30 Hz              |
| Sensors page                       | 100 ms   | 10 Hz               |
| System page                        | 2 s      | 0.5 Hz              |
| Launcher / Settings / MP3 / SmartHome | 100 ms| 10 Hz               |
| Snake / 2048 / Tetris / Maze       | 16 ms    | ~60 Hz              |
| Flappy                             | 8 ms     | 125 Hz              |
| Finger held on the screen          | 16 ms    | 60 Hz (override)    |

### Extra optimizations

- **160 MHz CPU** by default (instead of 240 MHz), ~30% CPU power saving.
- **IMU power-down**: `CTRL7 = 0x00` at boot, power-up only when requested by a consumer.
- **Touch I2C** polled only when the finger is placed (GPIO38 LOW) or just lifted.
- **RTC** polled at 1 Hz (instead of 5 Hz before optimization).
- **Battery** polled at 1/60 Hz (1/300 Hz when the screen is off).
- **Conditional Watchface flush**: the PSRAM FB is flushed only if `needs_render()` signals an actual change.
- **WiFi auto-disconnect** after 5 mins of inactivity: `wifi_controller.disconnect_async()`; the 2.4 GHz radio is the biggest constant consumer. Automatic reconnect on next wake.
- **TE VSync spin** limited to 400 iterations (instead of 2000) to avoid wasting cycles when TE doesn't pulse.
- **Blocking delays** replaced by `Timer::after(...).await` so the CPU stays parked during button debounces.
- **Audio PA amp** (GPIO46) held LOW at boot, codec muted (DAC power-down + HP drive off) immediately after init. The amp is pulled HIGH only while writing the beep via DMA, then pulled back down.
- **AOD Anti burn-in**: the position of the HH:MM block in AOD is shifted by `(minutes % 9) - 4` pixels in X and Y, like Apple Watch.

### Transition order

```
touch/button       interaction +0s    state 3 (full)
   └─ +20s idle    ────────────────→  state 2 (dim)
   └─ +40s idle    ────────────────→  state 1 (AOD, if Clock page) or state 0 (otherwise)
   └─ +300s idle   ────────────────→  WiFi disconnect
   └─ +600s idle   ────────────────→  state 0 (full OFF)

touch or button GPIO IRQ             → immediate state 3, WiFi reconnect follows
```

---

## Display pipeline

```
Embedded-graphics draw calls
         │
         ▼
410×502 u16 RGB565 PSRAM Framebuffer  (402 KB back buffer)
         │
         │  fb.flush() OR fb.flush_vsync(te_pin)
         ▼
Co5300Display::set_addr_window(...)
         │
         ▼
QspiBus::write_pixels()
         │
         ▼
esp-hal SPI2 half_duplex_write(
    DataMode::Quad,             ← 4-bit QSPI mode
    Command::_8Bit(0x12),       ← write memory
    Address::_24Bit(0x003C00),
    dummy = 0,
    buffer,                      ← pixel data in quad mode
)
         │
         ▼
DMA_CH0 → GPIO SIO0..SIO3 @ 80 MHz
```

- `swap_and_flush()`: double buffer, for games (Flappy); zero tearing.
- `flush_vsync()`: single buffer, waits for a TE pulse (GPIO13) before sending pixels.
- `flush_region(x, y, w, h)`: partial update, used by watchface partial updates.

---

## Build setup

### Prerequisites

- **Xtensa ESP Rust toolchain** (installed via [espup](https://github.com/esp-rs/espup)):
  ```bash
  cargo install espup
  espup install
  ```
  `rust-toolchain.toml` pins `channel = "esp"`.

- **MSVC linker** (Windows): needed for host build scripts. Install "Desktop development with C++" via Visual Studio Installer. The `link.exe` must be in the PATH when running cargo. On this project we typically have:
  ```bash
  export PATH="/c/Program Files/Microsoft Visual Studio/18/Community/VC/Tools/MSVC/14.50.35717/bin/Hostx64/x64:$PATH"
  ```

- **espflash**:
  ```bash
  cargo install espflash
  ```

### WiFi credentials

SSID and password are read at compile-time via `env!()`. They must be defined before building:

```bash
# Linux / macOS / Git Bash (Windows)
export WIFI_SSID="MyNetwork"
export WIFI_PASS="MyPassword"
```

```powershell
# PowerShell
$env:WIFI_SSID = "MyNetwork"
$env:WIFI_PASS = "MyPassword"
```

### Build

```bash
WIFI_SSID="MyNetwork" WIFI_PASS="MyPassword" cargo build --release
```

The final binary is around **579 KB** (full firmware with WiFi stack + games + UI).

### Flash + serial monitor

```bash
espflash flash --port COM7 --monitor target/xtensa-esp32s3-none-elf/release/waveshare-watch-rs
```

On Linux: `/dev/ttyACM0` or `/dev/ttyUSB0` depending on the USB bridge.

### Build config

- `opt-level = "s"` in dev AND release (size optimized).
- `lto = true` in release (global inlining, reduces size by ~20%).
- 64 KB SRAM heap (for the WiFi stack), 8 MB PSRAM heap (framebuffers + Vecs).

---

## Features

### Integrated and working
- 410×502 QSPI 80 MHz DMA double-buffer display
- PSRAM framebuffer + DMA flush
- Tearing Effect VSync anti-tearing
- FT3168 touch + iOS-like swipe detection + tap + diagonal rejection
- QMI8658 IMU accel + gyro + temperature with power management
- PCF85063A RTC + NTP sync via embassy-net UDP
- WiFi STA: DHCPv4, NTP, HTTP GET/POST, auto-disconnect idle
- ES8311 audio + I2S DMA (Snake beep)
- SD Card 4 GB detection (FAT32 /mp3 scan in place, stable mount if MBR is valid)
- Watchface 3 pages: Clock (7-segment time + battery + FR date + gyro ball), Sensors, System
- Launcher app list with smooth scroll
- Games: Snake, 2048, Tetris, Flappy Bird, Maze (gyro)
- Settings: WiFi SSID/password fields with T9 keyboard
- SmartHome: configurable 6-button HTTP grid
- MP3 Player: UI (play/pause, prev/next, progress bar)
- Screen sleep/wake 4 levels with minute-by-minute Always-On Display
- Boot button = launcher, swipe up = launcher

### Partially wired / stubbed
- **MP3 decoding**: `nanomp3` compiled and as a dependency, UI ready, SD → I2S stream to be wired.
- **BLE**: `esp-radio` compiled with stub feature, init disabled due to a panic `btdm_controller_init -4` in coex with WiFi (requires additional `coex` config).
- **WiFi scan list**: `ScanResult` types ready, Settings UI shows the field but without scan.
- **USB Mass Storage**: not wired (copy-from-PC would require `usb-device` + `usbd-storage`).
- **ESP deep sleep**: no `esp_hal::system::Sleep`; we stay in light sleep via the Embassy executor, sufficient for watch usage.

---

## Detailed custom drivers

### `drivers/qspi_bus.rs`: QspiBus

Half-duplex quad-SPI bus for the CO5300. API:
```rust
fn write_command(&mut self, cmd: u8)
fn write_c8d8(&mut self, cmd: u8, data: u8)
fn write_pixels(&mut self, pixels: &[u16])
fn begin_pixels(&mut self)
fn stream_pixels(&mut self, pixels: &[u16])
fn end_pixels(&mut self)
```

Uses esp-hal `Spi::half_duplex_write()` with `Command::_8Bit` + `Address::_24Bit`. `DataMode::Single` is used for commands, `DataMode::Quad` for pixels.

### `drivers/co5300.rs`: Co5300Display

Init sequence faithful to the C Arduino Waveshare driver `Arduino_CO5300.cpp`:
- Hardware reset (10ms low, 120ms high)
- SLPOUT (0x11) + delay 120 ms
- 0xFE 0x00 (vendor register access)
- 0xC4 0x80 (SPI mode control)
- 0x3A 0x55 (RGB565 pixel format)
- 0x53 0x20 (write CTRL display)
- 0x63 0xFF (HBM brightness)
- DISPON (0x29)
- 0x51 0xD0 (brightness)
- 0x35 0x00 (TEARON VBlank only)

Functions: `init`, `set_addr_window`, `set_brightness`, `display_on`, `display_off`, `bus_mut`.

### `peripherals/audio.rs`: Es8311

ES8311 init based on the Waveshare C driver. Critical registers missing in my first attempt:
- `0x00 = 0x1F` (reset) → `0x00 = 0x00` → **`0x00 = 0x80`** (power-on command, initially forgotten)
- Clock coefficients for 4.096 MHz MCLK @ 16 kHz sample rate
- `0x0D = 0x01`, `0x0E = 0x02` (power up analog)
- `0x12 = 0x00` (DAC power up), `0x13 = 0x10` (HP drive)
- `0x32 = 0xD9` (volume 85%)

API: `init`, `mute` (DAC power-down + HP off + vol 0), `unmute`, `set_volume`.

### `peripherals/power.rs`: Axp2101Power

Wrapper around `axp2101-embedded` for battery monitoring + power rails.

### `peripherals/touch.rs`: Ft3168Touch

**Monitor** mode (`REG_POWER_MODE = 0x01`): the chip asserts GPIO38 only on a touch event. Internal state machine to distinguish tap / swipe up/down/left/right with:
- minimum 30 px threshold to qualify a swipe
- 1.5× ratio on the dominant axis to reject diagonal swipes
- tracking start/end coordinates

### `peripherals/imu.rs`: Qmi8658Imu

Init accel ±2g @ 500 Hz, gyro ±512 dps @ 119 Hz, LPF enabled.
```rust
fn read_accel() -> AccelData    // m.x, y, z in g
fn read_gyro() -> GyroData      // °/s
fn read_temperature() -> f32    // °C
fn power_up() / power_down()    // CTRL7 0x03 / 0x00
```

### `peripherals/rtc.rs`: Pcf85063aRtc

BCD read/write of registers 0x04..0x0A. Auto conversion to `DateTime { year, month, day, hours, minutes, seconds }`.

### `peripherals/http.rs`: http_get / http_post

Minimal HTTP client without external crate: parse URL, `TcpSocket::connect`, format request manually, custom `write_all` (handling partial writes), read until close, parse status code + body truncated to 128 bytes.

---

## Runtime data flow

### On boot
1. `esp_hal::init(CpuClock::_160MHz)`
2. `esp_alloc::psram_allocator!`: 8 MB PSRAM heap
3. `esp_rtos::start(timg0.timer0)`: Embassy executor
4. Sequential init of all I2C/SPI/I2S drivers
5. `wifi_controller.connect_async().await`
6. `embassy_net::Stack` + `StackResources<3>`, spawn `net_task` task
7. Wait for DHCP IP
8. `ntp_sync()`: UDP to 216.239.35.0:123, parse timestamp, `rtc.set_time()`
9. Initial watchface render
10. Enter main loop

### In the loop
```rust
loop {
    // Choose tick based on state
    let tick = match (screen_state, app_state, current_page, gyro_enabled) { ... };

    // Sleep until next event
    select3(
        Timer::after(tick),
        touch_int.wait_for_falling_edge(),
        boot_button.wait_for_falling_edge(),
    ).await;

    // Sensors throttled by need + screen_state
    if need_imu { imu.read_accel(); ... }
    if screen_state >= 2 && now >= next_rtc { rtc.get_time(); ... }
    if now >= next_battery { power.get_battery_percent(); ... }

    // Conditional touch poll (finger placed or just lifted)
    if touch_active { touch.poll(); ... }

    // Sleep/wake state machine → transitions 3→2→1→0
    // WiFi auto-disconnect
    // AOD render path (1x/min) → continue
    // Screen OFF → continue
    // App state machine → render + conditional flush
}
```

---

## Project metrics

| Metric                 | Value      |
|------------------------|------------|
| Lines of Rust          | 5 545      |
| Source files           | 23         |
| Release binary         | 579 KB     |
| Dependency crates      | ~35        |
| Lines of C/C++         | 0          |
| SRAM Heap              | 64 KB      |
| Allocated PSRAM        | ~1.2 MB    |
| Framebuffers           | 2 x 402 KB |
| Handwritten drivers    | 8 (QSPI, CO5300, AXP2101, FT3168, QMI8658, PCF85063A, ES8311, HTTP) |

---

## C++ vs Rust comparison

| Aspect                     | C++ (ESP-IDF + Arduino)              | Rust (esp-hal + Embassy)                           |
|----------------------------|--------------------------------------|----------------------------------------------------|
| Runtime                    | FreeRTOS (preemptive, ~20 KB RAM)    | Embassy async (cooperative, ~0 KB overhead)        |
| UI Stack                   | LVGL (C, ~100 KB RAM)               | embedded-graphics (Rust, zero alloc)               |
| Display driver             | Arduino GFX (Arduino_CO5300.cpp)     | Custom driver qspi_bus.rs + co5300.rs              |
| SPI Bus                    | ESP-IDF spi_device, polling          | esp-hal half_duplex_write, DMA 8 KB                |
| Power management           | XPowersLib (C++)                     | Custom driver power.rs on embedded-hal I2C         |
| Audio                      | ES8311 Arduino driver                | Custom driver audio.rs (registers faithful to C)   |
| WiFi                       | ESP-IDF wifi_init + lwIP             | esp-radio + embassy-net (smoltcp)                  |
| Sleep                      | Not implemented                      | 4 levels + AOD, event-driven select3               |
| Build system               | PlatformIO / Arduino IDE             | Cargo, Xtensa cross-compile via espup              |
| Safety                     | Raw pointers, buffer overflows       | Ownership, borrow checker, no UB                   |
| Firmware size              | ~1.2 MB (ESP-IDF + LVGL + WiFi)      | 579 KB (all included)                              |

---

## Conversion history (C++ to Rust)

The original C++ project used:
- ESP-IDF + FreeRTOS
- Arduino GFX for the CO5300
- LVGL for the UI
- ES8311 codec via Arduino driver
- XPowersLib for the AXP2101

Major steps of the rewrite:

1. **QSPI bus + CO5300**. The hardest part: discovering that esp-hal `half_duplex_write` supports `DataMode::Quad` via the `Command` + `Address` machinery. Initial bug: using `with_miso` (input) instead of `with_sio1` (output) caused SIO1 to float → all blacks appeared green.

2. **AXP2101**. Activating the DC1 (3.3 V main) and ALDO1 (panel) rails via registers 0x80 and 0x92, otherwise the screen stays black even with the CO5300 correctly initialized.

3. **PSRAM Framebuffer**. Alignment issue: the CO5300 is strict on even widths for partial writes. Added even-rounding logic in `flush_region`. The PSRAM allocator requires `features = ["psram"]` on `esp-hal` + `esp_alloc::psram_allocator!` macro after `esp_hal::init`.

4. **ES8311 Audio**. 4 attempts before getting sound: the correct public method to play via I2S is **`write_dma()`** (not `write()` which is private). The init must exactly match the C sequence, particularly the `write_reg(0x00, 0x80)` after the reset, otherwise the codec stays in power-down.

5. **Event-driven loop**. Converted from `loop { Timer::after(5ms).await; ... }` to `select3(Timer, touch_edge, button_edge)`. Gain: CPU wake-ups reduced by ~6000× in screen OFF and ~200× in idle watchface.

6. **BLE**. Init attempt with `esp_radio::ble::BleConnector::new` → panic `btdm_controller_init returned -4`. BLE disabled in `Cargo.toml` features pending a correct coex configuration.

7. **Sleep/wake**. Initial bug: the `display_on()` sequence did DISPON then SLPOUT (incorrect order), so DISPON happened while the panel was still in SLPIN. Fixed to SLPOUT (120 ms) → DISPON (20 ms), standard MIPI DCS order.

---

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT License ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

Hardware drivers were written from scratch, informed by Waveshare's C/C++ examples and the [esp-hal](https://github.com/esp-rs/esp-hal) ecosystem.

---

## Resources

- Waveshare Wiki: [https://www.waveshare.com/wiki/ESP32-S3-Touch-AMOLED-2.06](https://www.waveshare.com/wiki/ESP32-S3-Touch-AMOLED-2.06)
- esp-hal: [https://github.com/esp-rs/esp-hal](https://github.com/esp-rs/esp-hal)
- esp-rtos: [https://github.com/esp-rs/esp-rtos](https://github.com/esp-rs/esp-rtos)
- Embassy: [https://embassy.dev](https://embassy.dev)
- embedded-graphics: [https://docs.rs/embedded-graphics](https://docs.rs/embedded-graphics)
- CO5300 datasheet: provided by Waveshare on the wiki
- AXP2101 datasheet: X-Powers
- ES8311 datasheet: Everest Semiconductor
- QMI8658 datasheet: QST Corporation


## Star History

<a href="https://www.star-history.com/?repos=infinition%2Fwaveshare-watch-rs&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=infinition/waveshare-watch-rs&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=infinition/waveshare-watch-rs&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=infinition/waveshare-watch-rs&type=date&legend=top-left" />
 </picture>
</a>
