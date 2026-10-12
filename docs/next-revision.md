# Next revision: Wi-Fi in place of LVDS

Plan for the board revision after the second production run (commit 0d7833f),
written October 2026. Nothing here is in the KiCad files yet.

## Goals

- Shows keep playing from the microSD card. Pixel data never crosses the network.
- Wi-Fi carries only uploads (shows, config, firmware) and timing.
- Be as compatible with FPP (Falcon Player) as practical, so a board can follow an
  FPP player in a show with music, and xLights/FPP can find and upload to it.
- Standalone shows with no music need no Pi: board 0 leads, the others follow.
- Simpler and cheaper than the LVDS board.

## How it runs

- **Storage:** every board keeps its own shows on SD, containing only the
  channels for the strings it drives. Each board has a file under the same show
  name.
- **Uploads over Wi-Fi:** HTTP upload of FSEQ files, config and a playlist
  (`playlist.csv`, one show per line, which replaces SD directory order). Write
  to a temporary name and rename when complete, so a dropped upload never leaves
  a half-written show. Pause playback during an upload: the card is one SPI bus
  and FatFs must not be used from both cores at once.
- **Timing:** FPP MultiSync (UDP): start show X, periodic sync at frame N, stop.
  The same packets whether the leader is board 0 or an FPP player, so followers
  have one code path.
  - **Board 0 leads** in standalone mode. It is the only board with a schedule
    and the only one that needs network time (NTP).
  - **An FPP player leads** in shows with music. FPP plays the audio, and the
    audio position is the clock.
  - A follower that stops hearing sync keeps playing on its own clock. A setting
    chooses what it does after a few seconds of silence: finish the show and go
    dark, or carry on the playlist alone.
- **Live preview:** DDP input over Wi-Fi, for designing shows from xLights. Fine
  at small sizes (about 1.4 Mbps for 1,415 pixels at 40 fps), not for full shows.
- **FPP compatibility, in order of value:** MultiSync follower, answering FPP's
  discovery ping (so the board shows up in FPP and xLights), and FPP Connect
  uploads. ESPixelStick implements all three on a small chip with an SD card.
  Copy its packet formats and endpoints from source rather than from memory.

Board ID is set in software (web UI). The DIP switch idea is
dropped. The web UI should warn when two boards claim the same ID, and
especially two claiming ID 0.

## Hardware changes

### Remove

- LVDS: U41 (DS90LV048), U42 (DS90LV047), RJ1, RJ2, R20 to R23 (termination),
  D8 and D9 (SLVU2.8-4 protection), and their decoupling.
  `communications.kicad_sch` goes away.
- Note for the record: in the current netlist U41 pin 12 (GND) and both
  transceivers' EN* pins are unconnected. That is moot once LVDS is removed.

### Add: Wi-Fi radio

- Raspberry Pi RM2 module (Infineon CYW43439, the radio on the Pico 2 W), at a
  board edge with the antenna at or over the edge.
- Antenna keep-out: no copper under the antenna on any of the four layers,
  including both inner ground planes. Follow the RM2 datasheet exactly.
- Plastic enclosure only. A metal box kills Wi-Fi.
- Bulk capacitance at the module's supply pin for transmit current spikes, per
  the RM2 datasheet.
- Pins: the radio's SPI link needs 4 GPIOs: power enable (WL_REG_ON), one
  shared data line (data out, data in and host wake, as on the Pico 2 W), chip
  select and clock. Use **GPIO 32 to 35**, freed by the LVDS removal. The
  Pico 2 W uses GPIO 23, 24, 25 and 29, which are LED string outputs here.
- Checked in Pico SDK 2.2.0 (`pico_cyw43_driver/cyw43_bus_pio_spi.c` and
  `boards/pico2_w.h`): the pins are set per board with
  `CYW43_DEFAULT_PIN_WL_REG_ON`, `_DATA_OUT`, `_DATA_IN`, `_HOST_WAKE`,
  `_CLOCK` and `_CS` in our board header. The one rule (a `static_assert`) is
  that clock and data sit in one PIO GPIO window: all below 32, or all 16 or
  above. GPIO 32 to 35 meets it. Power enable and chip select are plain GPIOs
  with no restriction.
- The driver claims its own PIO with `pio_claim_free_sm_and_add_program_for_gpio_range`,
  which searches PIO2 first and can set a free PIO's GPIO base to 16. So the
  radio takes PIO2 at base 16, and the LED output keeps its PIO at base 0
  (GPIO 0 to 31). The LED driver uses the same claim function, which skips a
  PIO whose base doesn't cover its pins. Bring Wi-Fi up after the LED driver
  or before, either works, but nothing else may hold PIO2.
- Follow the RM2 datasheet's reference circuit for the shared data line.
- Before ordering boards, bench-test the pin choice: an RM2 module wired to
  GPIO 32 to 35 on an RP2350B board (for example `~/git/pcb_try`), running the
  Pico SDK Wi-Fi scan example with our board header.

### Remove: zone link fuses

- F5, F6 and F7 (XF-507P holders linking Z1 to Z2, Z2 to Z3 and Z3 to Z4).
  They let one supply feed every zone, but they are easy to misuse: fitted
  with two supplies connected, they tie the supplies in parallel, and in
  single-supply mode the whole board's current passes through F1 and the
  chain, so it is capped at one fuse's rating.
- Each input keeps its own fuse (F1 to F4) and zone. To run several zones from
  one supply, the user wires the supply to each input they use. The README
  should recommend a separate wire from the supply to each input (a star), not
  a jumper from terminal to terminal, so no single terminal carries every
  zone's current.
- The diodes D4 to D7 stay: the logic runs from whichever zone has power.

### Add: blown-fuse LEDs

- One LED with a series resistor across each input fuse (F1 to F4), anode on
  the input side. With the fuse intact both ends sit at the same voltage and
  the LED is dark. When the fuse blows, the input side stays at 12V and the
  zone side is pulled toward ground through its strings, so the LED lights.
  With no supply on that input it stays dark, as it should.
- About 4.7k for roughly 2 mA at 12V. In the blown state that 2 mA flows into
  the zone's strings, which is harmless. No GPIO needed.
- Place each LED next to its fuse, so a blown zone is visible at a glance.

### Change: 3.3V regulator

- The ME6217C33 is rated 300 mA. The RP2350, SD card and PSRAM already use a
  good share of that, and Wi-Fi transmit adds spikes of a few hundred mA. Expect
  brownouts and resets while transmitting if it stays.
- Use a 3.3V buck converter from the 5V rail, around 1 A. If it must stay
  linear, use a 1 A LDO in a package that can shed heat: 5V to 3.3V at 0.5 A is
  about 0.85 W, too much for SOT-23-5. The Pico 2 W uses a switching regulator
  rated around 800 mA for this reason.

### Power-up: wait, don't sense

- The TPS63070 buck-boost regulates 5V from about 2V of input, so the RP2350
  boots while the pixel 12V rail is still rising. Strings that get data during
  that ramp come up dark or half lit until the data restarts (October 2026,
  Halloween matrix).
- No 12V sense or power-good hardware. The firmware holds the LED outputs idle
  for a fixed time after boot (2 s today, `PIXEL_POWER_SETTLE_MS` in
  `src/core1_task.c`), which is long enough for the supply to settle.

### Remove: front-panel header

- J39 (display I2C, encoder and back button). The web UI, the IR remote, the
  on-board button and status LED, and USB serial cover what it did. Wi-Fi setup
  uses the access-point page on a phone. This frees GPIO 40, 41, 43, 45, 46
  and 47.

### Add: PSRAM

- 8 MB QSPI PSRAM (for example APS6404L) on the flash's QSPI lines, with its
  chip select on **GPIO 47**, one of the few pins that can carry the RP2350's
  second QSPI chip select (XIP_CS1). Check the pin options in the RP2350
  datasheet before layout. The Pimoroni Pico Plus 2 uses the same arrangement.
- Uses: buffering an upload while a show plays, decompressing xLights'
  zstd-compressed FSEQs (whose blocks don't fit in internal RAM), and caching
  short shows.

### Keep

- 32 outputs: SN74LVC8T245 level shifters, 47 ohm series resistors, PTC per
  port, four 12V zones, each with its own input and blade fuse.
- microSD on GPIO 36 to 39. Consider raising the SPI clock from 12.5 MHz (a
  firmware change) before anything else, for faster reads and uploads.
- IR receiver on GPIO 42, with SW2 and CN1.
- USB-C, BOOTSEL (SW1) and reset, for flashing and recovery.

### Add: on-board button and status LED

- **Status LED:** a plain common-anode or common-cathode RGB LED on **GPIO 40,
  41 and 43** with a resistor each, driven by PWM. No level shifting, unlike an
  addressable LED. Setup mode: blinking blue. Connected: green. Playing: slow
  pulse. SD card or config error: red.
- **Button:** on **GPIO 44**. Press: play or next show. Hold at power-up: clear
  Wi-Fi settings and start the setup access point.
- **Spare:** GPIO 45 and 46. Bring them to test pads or a small header for
  later use. GPIO 45 is ADC-capable (ADC5).

## GPIO map

| GPIO | Now | Next revision |
|---|---|---|
| 0 to 31 | LED strings | LED strings |
| 32 to 35 | LVDS HS_DATA0 to 3 | RM2 Wi-Fi |
| 36 to 39 | microSD | microSD |
| 40, 41, 43 | Panel: back, push, encoder B | Status LED (R, G, B) |
| 42 | IR in | IR in |
| 44 | LVDS DATA_IN_EN | Button |
| 45, 46 | Panel encoder A, display I2C data | Spare |
| 47 | Display I2C clock | PSRAM chip select (XIP_CS1) |

## Firmware plan

The C firmware's split stays: core 1 drives pixels, core 0 runs everything else
(now including Wi-Fi via `pico_cyw43_arch` and lwIP).

1. **Wi-Fi setup:** saved network details. With none saved, or if connecting
   fails for about 30 s, start an access point (`PixelBlit-XXXX`) with a setup
   page. mDNS name `pixelblit-<name>.local`.
2. **Upload and playlist:** HTTP upload of FSEQ, config and playlist, with
   temporary file and rename.
3. **Network time and schedule** (board 0, or any standalone board).
4. **MultiSync:** follow first, then lead (board 0).
5. **DDP input** for live preview.
6. **FPP discovery and FPP Connect uploads.**
7. **Over-the-air firmware updates**, with USB as the fallback.
8. **PSRAM:** upload buffering, then zstd FSEQ support if wanted.

Related firmware work that does not depend on this board:

- Per-port brightness limiter: estimate each string's current from the frame data
  and scale that string down before it reaches the PTC's 4 A hold current.
- Faster SD reads: higher SPI clock and multi-sector reads.
- Compressed FSEQ (zlib, small blocks) once shows get large.

## Capacity notes

- Data rate (800 kHz WS2811, 30 us per pixel): about 820 pixels per port at
  40 fps, 1,000 per port at about 30 fps, so roughly 26,000 to 32,000 pixels per
  board. Hard ceiling.
- RAM: double-buffered bit-planes cost 192 bytes per pixel row across all
  32 strings. About 1,000 rows fit easily in 520 KB.
- Power from the board: 4 A per port (PTC hold), about 20 A per zone, 80 A
  total. At 1,000 pixels per port that is 0.03 W per pixel on average, about
  5 to 10% of full white, so it needs the brightness limiter and careful show
  design.
- Voltage drop at about 2 inches per pixel: a 1,000-pixel string at 2.5 A drops
  2.7 to 6.7V from one end (18 to 22 AWG). One injection point mid-string,
  still fed from the board, keeps it under about 2V.

## Open questions

- Confirm XIP_CS1 is available on GPIO 47 of the RP2350B.
- Inner copper weight: the stackup says 1 oz, but JLC's 4-layer default is
  0.5 oz on inner layers. Confirm what is ordered.
- README.md still lists a DIP switch, LVDS and the display. Update it when this revision
  lands.
