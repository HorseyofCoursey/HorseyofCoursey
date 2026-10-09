### Hi, I'm Jonathan. I build embedded Linux systems and firmware.

I like the parts of engineering that don't show up in tutorials: bringing up
unfamiliar hardware, porting drivers across kernel API changes nobody warned
you about, and getting real-time media and displays working on devices with
no CPU, RAM, or desktop environment to spare.

---

### Featured Projects

**[zeroplay](https://github.com/HorseyofCoursey/zeroplay)** A lightweight
H.264 video player for the Raspberry Pi, built as a modern replacement for
the discontinued omxplayer. Runs directly on V4L2 hardware decode, DRM/KMS
display, and ALSA audio. Zero CPU video decode, zero X11. Actively
maintained with outside contributors and real users.

📰 Covered by Adafruit: ["A lightweight H.264 video player for the Raspberry
Pi"](https://blog.adafruit.com/2026/03/10/a-lightweight-h-264-video-player-for-the-raspberry-pi/)

**[trixie-aiyprojects](https://github.com/HorseyofCoursey/trixie-aiyprojects)**
Ported Google's long-abandoned AIY Voice Kit v2 drivers to run on Debian
Trixie (kernel 6.12). The original drivers predate several kernel subsystem
changes, i.e. platform driver `remove()` signatures, the PWM subsystem's new
ownership model, device tree matching, so this meant fixing real kernel
module code across the sound driver, rt5645 codec, and gpio/pwm/adc
platform drivers, not just updating a package list.

**[stormshell](https://github.com/HorseyofCoursey/stormshell)** A
dependency-free ASCII-art weather display for the terminal, with 13
hand-drawn weather conditions, a night sky with calculated moon phase, and
a kiosk mode that takes over HDMI on a Pi's console (TTY) with no desktop
environment running. Pulls live data from Open-Meteo and OpenStreetMap so
no API keys required. 65 stars.

**[EMBER](https://github.com/HorseyofCoursey/EMBER)** Themeable
MP3/FLAC/WAV/AAC player firmware for the M5Stack Cardputer ADV (ESP32-S3,
no PSRAM). Ships as two separate builds to work around the chip's memory
limits, one with full-screen FFT spectrum/visualizer rendering, one with
Subsonic/Gonic network streaming, since both together don't fit in RAM
reliably at once. Drives the board's second, otherwise-unused I2S
peripheral for real external stereo DAC output alongside the internal
speaker. 400+ downloads via M5Launcher.

---

### Write-ups

**["No, this isn't possible", until it was](writeups/spi-display.md)** A
zeroplay feature request I almost turned down, a Raspberry Pi kernel
engineer showing me why it wasn't impossible after all, and the real
kernel bugs (a rejected-framebuffer limit, then a heap overflow) that
turned up along the way to building it — upstreamed to the Linux kernel's
DRM subsystem.

**[The story of the display that lied about what chip it was](writeups/clone-display-debugging.md)**
A "known" SPI panel that stayed corrupted no matter what I changed, until
an independent microcontroller test rig and a controller ID readback
revealed it wasn't the chip it was sold as — a bitwise-inverted ID next to
a genuine ST7735S's.

---
