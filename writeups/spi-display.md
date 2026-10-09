## "No, this isn't possible", until it was: adding SPI display support to zeroplay

In August 2026, someone opened [an issue on zeroplay](https://github.com/HorseyofCoursey/zeroplay/issues/29)
asking if it could drive a small SPI TFT panel instead of HDMI. My first
instinct was that it couldn't, and I said so almost that bluntly: zeroplay's
whole display path is zero-copy — the hardware decoder hands a DMA buffer
straight to the display scanout with the CPU never touching a pixel. SPI
panels can't take a DMA buffer off a GPU plane like that; they need the CPU
to convert every frame to RGB565 and shift it out over the bus one byte at
a time. I called it "a significant piece of work rather than a quick fix"
and figured that was the end of it.

`6by9` from Raspberry Pi replied the same day and showed me I
was wrong about the hard part. The hardware decoder's own ISP stage can
output RGB565 directly — I just had to request that pixel format instead
of NV12 — and the resulting buffer is still DMA-able memory that an SPI DRM
driver can consume. The actual CPU work left over was comparatively light:
no YUV-to-RGB conversion, just moving already-RGB565 bytes. I told him
straight: "when you first raised this I was going to say, 'no, this is not
possible', but now I think it is, would just be lots of work." I put it on
the roadmap.

### Building it — and finding a real kernel bug along the way

Over the following week 6by9 kept digging into the decoder/ISP side while I
built the zeroplay side: requesting RGB565 output from the V4L2 decoder,
an aspect-fit scaling path for panels with a different aspect ratio than
the source video, and a `--spi-fill` crop mode for the zero-scaling case.

Testing against a real panel (a PiTFT 3.5") surfaced something underneath
all of this that had nothing to do with zeroplay: the kernel's MIPI DBI
display drivers rejected any framebuffer that wasn't *exactly* panel-sized —

```
ili9341 spi0.0: bad framebuffer width 480, should be >= 320 && <= 320
```

— which meant cropping a sub-region out of a larger buffer (what
`--spi-fill` needed) was impossible at the kernel level, not just unbuilt.
6by9 traced it to `mode_config.max_width`/`max_height` being pinned equal
to the panel's minimum, and to the plane-update code silently ignoring the
source x/y offset entirely. He also flagged that the whole SPI display
driver area was effectively orphaned upstream — its previous maintainer
had stepped away for health reasons.

I built the fix: pass the plane's source offset through the DBI transfer
path, and raise the max width/height limits (capped at
`DRM_SHADOW_PLANE_MAX_WIDTH/HEIGHT`, per the kerneldoc's own recommendation
for shadow-plane drivers) on every driver sharing that flush path. Tested
on an ILI9341 on a Pi Zero 2 W: oversized framebuffers were accepted and
correctly cropped, non-zero offsets worked, native-size playback showed no
regression, and 30fps full-frame video ran with no performance hit. Merged
into Raspberry Pi's `rpi-7.2.y` kernel:
[raspberrypi/linux#7589](https://github.com/raspberrypi/linux/pull/7589).

zeroplay's side merged five days later — RGB565 decode for SPI/DBI panels,
aspect-fit as the default display mode, `--spi-fill` for the old crop
behavior, and a fix so a slow panel drops frames instead of dragging
playback down with it. Tested on ILI9341 and ST7789V panels. Issue #29,
closed as completed.

### Finding a heap overflow in my own fix

A couple weeks later I found a real bug in the kernel patch above: the
plane's source origin is a 16.16 fixed-point value, and truncating a
fractional origin to whole pixels before computing the controller's window
address meant the far edge of the flushed rectangle could extend up to one
pixel past the panel's actual bounds — writing past the end of a transfer
buffer sized exactly for the panel. A heap buffer overflow, from a
subtraction that assumed its result was already panel-sized when the
truncation could break that assumption. A narrower related case could also
collapse the rectangle to zero width or height, producing an inverted
address window.

Fixed by clamping the rectangle to the panel's fixed mode before using it
for anything, and skipping the flush entirely if the clamped result comes
out empty. While in there I also caught that `st7735r` — which lives under
`drivers/gpu/drm/sitronix/` rather than `drivers/gpu/drm/tiny/` — had been
missed from the original driver sweep; confirmed it was the only omission
by grepping the whole tree for the shared flush-helper macro. Merged:
[raspberrypi/linux#7643](https://github.com/raspberrypi/linux/pull/7643).

### Upstreaming it

Both fixes started as downstream patches on Raspberry Pi's kernel fork, but
6by9 recommended sending them upstream to dri-devel rather than letting them
live only downstream — patches that never leave a vendor fork accumulate
maintenance burden over time. The upstream submission went through three
revisions, with automated review catching real issues in the first two
rounds; v3 came back clean. It's currently in review with the DRM subsystem
maintainers cc'd.

TLDR: What started as a speculative issue in my repo turned into kernel patch.
