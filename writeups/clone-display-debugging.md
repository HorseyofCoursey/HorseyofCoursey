## The display that lied about what chip it was

A 0.96" 160x80 SPI panel for a Pict-O-Vision Video unit was showing a
corrupted image — a solid band of random-looking static, hard-edged,
always in roughly the same place. Every datasheet, every reference driver,
and every value I tried made it worse or left it exactly the same.

### Ruling things out, one at a time

The obvious suspects went first: a typo in `config.txt`, wiring, SPI
speed, a bad physical panel. I swapped in a second, completely different
physical unit — same corruption. I confirmed the kernel's `mipi-dbi-spi`
overlay actually supported the `x-offset`/`y-offset` parameters I was
tuning (`dtoverlay -h mipi-dbi-spi`), confirmed via `dmesg` that the panel
driver was probing successfully, and — when varying the offset value
across several different numbers produced the *identical* corruption every
time — deliberately corrupted the firmware file on disk to confirm the
driver was genuinely reading it live and not something cached. It was.
None of it explained anything.

The panel was sold and documented as an ST7735S, so I pulled the
manufacturer's own wiki page and, when that got close but not quite right,
their actual shipped demo source code — exact register values, exact
`MADCTL` byte, exact GRAM offsets, straight from the vendor. Applying them
*exactly* got much closer (most of the image displayed correctly) but
still left a diagonally sheared, color-inverted mess layered on top. The
real breakthrough wasn't a value — it was what *didn't* change: swapping
in a completely different, independently-sourced ST7735 init sequence
produced the identical artifact, pixel for pixel. Different commands, same
exact corruption. That's not what a wrong command produces; a wrong
command produces *different* wrong output. This meant the problem wasn't
in the panel's init sequence at all, and no amount of further guessing at
register values was going to fix it.

### Taking the Pi out of the equation entirely

With the software and firmware side exhausted, I built an independent test
rig on a completely different platform — a $5 ESP32 microcontroller, wired
directly to the same physical panel, running a well-established Arduino
display library with zero relation to the Pi's Linux/V4L2/DRM stack. If
the same panel showed the same symptom on totally different hardware and
software, the problem was the panel itself or something fundamental about
how it was being driven, not a Linux driver or a zeroplay bug. If it came
out clean, I'd know exactly which variable mattered, free of a multi-second
Pi reboot cycle for every single test.

A labeled test pattern (colored blocks + readable text + a diagonal line)
on the ESP32 immediately showed something concrete: the RED/GREEN/BLUE
blocks were displaying as BLUE/GREEN/RED — a color channel swap, not
random noise. And raising the SPI clock from 8MHz to 20-40MHz reproduced
the *exact* diagonal shearing artifact that had been plaguing the Pi the
entire time. That was the first real, falsifiable signal in the whole
investigation.

### The actual root cause

Reading the controller's own ID register (command `0x04`, RDDID) on the
ESP32 returned `83 76 0F`. A genuine ST7735S returns `7C 89 F0` — the exact
*bitwise inverse*. The panel wasn't a real ST7735S at all. It was a clone
chip, correctly MIPI DBI-compatible but with its own MADCTL bit meaning,
its own GRAM offset, and its own inversion polarity — none of which any
official datasheet or vendor reference driver could ever have told me,
because none of them were describing this actual silicon.

With the real values confirmed independently on the ESP32 — correct
MADCTL byte, correct inversion setting, correct offsets, and SPI capped at
a speed the clone's logic could actually keep up with — I applied the same
recipe back on the Pi. It worked on the first try.

**Takeaway:** when a component is sold and documented as a well-known part
but behaves like it isn't, the datasheet stops being evidence. Isolating
the variable — in this case, by removing the entire software stack and
testing the hardware on its own — found in minutes what hours of
plausible-looking kernel/driver changes couldn't.
