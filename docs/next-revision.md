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
  `communications.kicad_sch` goes away. **Done**: the sheet is now
  `wireless.kicad_sch` (the RM2), and the parts, their copper and the
  DATA IN/OUT silkscreen are off the board.
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
- **Done in the KiCad files** (`wireless.kicad_sch`, U9 RMC20452T, LCSC
  C49451547). The pin assignment follows the order of the four routed
  LVDS traces, so the existing copper from the RP2350 is reused:

  | GPIO | Was | Now | RM2 pad |
  |---|---|---|---|
  | 32 | HS_DATA3 | WL_ON (`_WL_REG_ON`) | 12 (Wi-Fi on), 13 (Bluetooth on) |
  | 33 | HS_DATA2 | WL_CS (`_CS`) | 9 |
  | 34 | HS_DATA1 | WL_DATA (`_DATA_OUT`, `_DATA_IN`, `_HOST_WAKE`) | 5 direct, 6 via R60 470R, 10 via R61 10k |
  | 35 | HS_DATA0 | WL_CLK (`_CLOCK`) | 3 |

  GPIO 44 (was DATA_IN_EN) is unconnected until the button lands on it.
  VIN and VDDIO are on +3V3, with C42 (10 uF) and C43/C44 (100 nF).
- Placement: antenna end flush with the left edge, module centred at y = 110
  so the space under the flash (U2) stays free for the PSRAM. The footprint
  carries the datasheet's 35.5 x 18.5 mm keep-out as a rule area on every
  copper layer. The In2 +3V3 trunk that ran up the left edge (x = 54.15)
  went through that area, so it now runs at x = 60.5, under the module body.
  The KiCad library footprint from JLC had the side pads flush with the
  module edge; they are moved out 0.5 mm to match datasheet Figure 6.
- Before ordering boards, bench-test the pin choice: an RM2 module wired to
  GPIO 32 to 35 on an RP2350B board (for example `~/git/pcb_try`), running the
  Pico SDK Wi-Fi scan example with our board header.

### Remove: zone link fuses

- **Done**: F5, F6 and F7 are off the Power sheet and the board. They had no
  traces, only the B.Cu zone pours, so the zones simply refill.
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

### Change: 5 to 24V inputs, mixed per zone

Today's real input range is 5 to 14V, not the "6-12VDC" on the silkscreen
(that dates from an LDO plan). The TPS63070 takes 2 to 16V, so 5V inputs
already work: about 4.6V after the OR diode, boosted to 5V. The 16V limit comes
from the TPS63070, the 1812L400/16 PTCs and the 16V capacitors on the zone
rails. This revision raises it to 24V nominal.

- **5V regulator:** replace the TPS63070 with a buck-boost rated 36V or more
  input, so the data rail is a regulated 5V from every input between 5 and
  24V. No buck in dropout: marginal data levels are exactly the kind of fault
  that shows up as some strings working some of the time.
  - Keep today's arrangement: the regulator is set to about 5.3V (TPS63070
    with R12 560k / R13 100k, 0.8V reference), and the B5819W (D3) that ORs it
    with USB drops that to about 4.9 to 5.0V on `+5V`. The new part needs the
    same higher setpoint, or replace D3 with an ideal-diode controller and set
    5.0V. Either way, stay under the SN74LVC8T245's 5.5V VCCB limit.
  - Candidates: the LTC3115-1 (2.7 to 40V in, integrated switches, about 2 A
    in buck mode) or a TI part in the TPS552xx family. Controller types such as
    the LM5176 work too, but need external FETs and more area. Confirm
    input range, output current at 5V input and package before choosing.
  - Load is small: MCU, SD, PSRAM, level shifters and the 3.3V buck for Wi-Fi.
    About 1 A at 5V is plenty.
- **Port PTCs:** 30V-rated parts. They are there for shorts and wiring
  accidents, not to enforce a per-port power budget (the brightness limiter
  does that), so a lower hold current is fine. Prefer an 1812 part with about
  3 A hold (about 70 W per port at 24V, more than today's 48 W at 12V) and keep
  the footprints. The 2920 size is only needed to keep 4 A. Rule: the
  maximum fault current (Imax) at rated voltage must be at least 2x the zone
  fuse rating (so at least 50 A behind a 25 A fuse). In a dead short the
  supply's own current limit (typically 105 to 150% of its rating) and its
  output capacitors set the current, and the PTC trips well before the blade
  fuse blows.
  Pick from the Littelfuse or Bourns tables.
- **Capacitors** on the zone rails and `+12V_SYS`: 35V or 50V.
- **TVS diode** on each zone input, clamping just above 24V, for hot-plugging
  supplies on long leads.
- **Unchanged:** SS34 OR diodes (40V), blade fuses (32V), terminals, copper,
  level shifters and data path. 24V pixels still take 5V logic data.
- **Blown-fuse LED resistor:** brightness follows the zone voltage. 10k gives
  about 2 mA at 24V, 1 mA at 12V and 0.3 mA at 5V. Use a high-efficiency LED
  so it is still visible on a 5V zone.

**Mixed voltages:** each zone feeds only its own 8 ports, and the OR diodes
power the logic from whichever zone is highest. So one board can run, for
example, 5V on zone 1, 12V on zones 2 and 3, and 24V on zone 4. The lower
zones' diodes are reverse-biased: 19V at most, within the SS34's 40V rating.
All zones share ground. The zone link fuses must be gone for this (see
below), since fitted they would tie two supplies together.

**Labelling**, because a 5V or 12V string on a 24V zone is destroyed at once:

- Silkscreen "5-24VDC" at the inputs.
- A write-in box for each zone's voltage.
- Optionally, a different terminal colour per zone.

**Zone silkscreen.** On the current board the zones are not runs of
consecutive ports. Each zone is one row of the staggered terminal rows, so
neighbouring terminals alternate between zones (checked in
`pixel_blit.kicad_pcb`):

| Zone | Input (fuse) | Row | Strings today (pin order) | Strings, next revision |
|---|---|---|---|---|
| Z1 | J1 (F1), bottom right | Bottom edge, outer row (J6 to J20, even) | 1, 3, 5 ... 15 | 0 to 7 |
| Z2 | J2 (F2) | Bottom edge, inner row (J5 to J19, odd) | 0, 2, 4 ... 14 | 8 to 15 |
| Z3 | J3 (F3) | Top edge, inner row (J22 to J36, even) | 17, 19, 21 ... 31 | 16 to 23 |
| Z4 | J4 (F4), top right | Top edge, outer row (J21 to J35, odd) | 16, 18, 20 ... 30 | 24 to 31 |

Wiring left to right along an edge therefore alternates between two zones,
which may be at different voltages. The silkscreen has to make the zone of
every terminal obvious without a manual:

- **Row bars:** a bold silkscreen line running the length of each terminal
  row, on the side away from the other row, labelled at both ends ("ZONE 1").
  The inner and outer rows use visibly different styles, for example a solid
  bar on one and a hatched or dashed bar on the other, so the two rows on an
  edge can't be confused.
- **Zone on every terminal:** each string label gets its zone, for example
  "S1 · Z1" in place of "STRING3" (new numbering, see below). It is the label
  the user is looking at while they wire.
- **Input to row:** each input terminal is labelled "ZONE n IN 5-24VDC" with
  the write-in voltage box next to it, and its bar style matches its row
  (bar style is the code that ties an input to its outputs). Where space
  allows, run the bar from the input along the board edge to its row.
- **Zone key:** a small table in silkscreen near the inputs: zone, bar style,
  strings, input. Also printed in the README.
- **Blown-fuse LEDs** sit next to their input fuse, labelled with the zone
  number.
- Keep text at least 1 mm high and lines 0.15 mm or more for JLC silkscreen,
  and keep it clear of terminal pads so it survives assembly.

**Renumber the strings by zone, in firmware.** The terminals and their
traces stay exactly where they are. Only the silkscreen labels and a
firmware table change, so each zone is a contiguous block of 8 strings, as
in the last column above. Zones are then contiguous in config.csv and
xLights too, which makes it harder to put a string on the wrong voltage when
setting up a model.

No board of the current revision is in anyone else's hands, so there is no
backward compatibility to keep: the zone numbering simply becomes the
numbering, in firmware, config.csv and the docs.

- Within a row, the new numbers run in the same direction as today's labels:
  left to right on the bottom edge, right to left on the top edge.
- The mapping from string number `n` to the output bit (the pin order, today's
  string number) is: zone `z = n / 8`, position `i = n % 8`,
  bit `= 16 * (z / 2) + 2 * i + (z % 2 == 0 ? 1 : 0)`. In full:
  `{1,3,5,7,9,11,13,15, 0,2,4,6,8,10,12,14, 17,19,...,31, 16,18,...,30}`.
- In the firmware (`pixel_blit_firmware`), the string number becomes an
  output bit in exactly two places: `mask = 1u << string` in `pb_set_pixel`
  and `pb_get_pixel` (`lib/pb_led_driver/pb_led_driver.c`). Both become
  `1u << string_to_bit[string]`, a fixed 32-byte table. No cost per frame
  beyond one table lookup per pixel write.
- The PIO pin range must cover physical bits, not string numbers.
  `fseq_player.c` sets `num_strings` to the highest configured string + 1,
  and `pb_led_driver_hw.c` claims and drives only `gpio_base` to
  `gpio_base + num_strings - 1`. With the table, strings 0 to 3 are bits 1,
  3, 5 and 7, so a 4-string config would leave them dark. The hardware side
  always drives all 32 pins.
- Everything above the driver (config.csv rows, FSEQ channel order, rasters,
  the per-port brightness limiter, the web UI) uses string numbers and needs
  no change. Code that reads bit-planes back sees pin order, so the
  brightness limiter should work from frame data before encoding.
- Bench tools that drive GPIOs directly (`string_test.c`, `toggle_test.c`)
  go through the same table, so every tool speaks one numbering.
- The driver unit tests assume string = bit. Update them for the table and
  add a test of the mapping itself.
- Rename the schematic nets `S0_OUT` to `S31_OUT` (and the old "STRINGn"
  texts) to the new numbers, then update the PCB from the schematic. That
  changes net names only, not copper, and keeps the schematic, silkscreen and
  firmware in agreement for anyone probing with a scope.

**What 24V gets you:** the same copper and 4 A PTCs carry twice the power
(about 96 W per port, about 1.9 kW per board). Voltage drop matters about 4x
less (half the current, twice the headroom), which suits long, dense strings.
For the same light output, everything carrying current runs cooler: half the
current means a quarter of the I²R heat in string wire, terminals, PTCs, fuses
and board copper, so a bright show no longer means heavier wire.

### Add: blown-fuse LEDs

- One LED with a series resistor across each input fuse (F1 to F4), anode on
  the input side. With the fuse intact both ends sit at the same voltage and
  the LED is dark. When the fuse blows, the input side stays at 12V and the
  zone side is pulled toward ground through its strings, so the LED lights.
  With no supply on that input it stays dark, as it should.
- About 4.7k for roughly 2 mA at 12V (10k if 24V inputs are supported, see
  above). In the blown state that 2 mA flows into
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
- **Done**: J39, its six signal traces and its +3V3 feed from J38 are off the
  board. GPIO 40, 41, 43, 45 and 46 are no-connects in the schematic until the
  status LED, button and spares are placed.

### Add: PSRAM

- 8 MB QSPI PSRAM (for example APS6404L) on the flash's QSPI lines, with its
  chip select on **GPIO 47**, one of the few pins that can carry the RP2350's
  second QSPI chip select (XIP_CS1). Check the pin options in the RP2350
  datasheet before layout. The Pimoroni Pico Plus 2 uses the same arrangement.
- **Done**: U10 APS6404L-3SQR-SN (LCSC C5333729, the 3.3 V part; the cheaper
  -SQN- is 1.8 V), R62 10k pull-up on CE#, C45 100 nF. GPIO 47 is XIP_CS1
  (funcsel 9, `XIP_SS_N_1` in the SDK's `io_bank0.h`; Pimoroni's Pico Plus 2
  board files use the same pin). The pull-up matters: RP2350 GPIOs reset
  with the pull-down on, which would select the PSRAM while the bootrom
  reads the flash on the shared lines.
- Layout: U10 sits directly under the flash (U2) in the same orientation,
  so each column of pins carries one signal on both chips (top rows SD3,
  SCLK, SD0; bottom rows SD1, SD2). SD3, SCLK and SD0 come down from U2's
  top pins through the gaps in its bottom row; SD1 and SD2 go down from
  U2's bottom pins through the gaps in U10's top row; between the chips
  every trace makes the same half-pitch jog, so nothing crosses. SD1 and
  SD2 from U1 continue down the right-hand corridor into U10's bottom pads.
  The flash CS reaches U2 pin 1 on B.Cu, and PSRAM_CS runs on B.Cu from the
  old DISP_SCL via down x = 72.3 to U10 pin 1. The schematic names the shared
  nets /QSPI_SD0..3 and /QSPI_SCLK.
- Uses: buffering an upload while a show plays, decompressing xLights'
  zstd-compressed FSEQs (whose blocks don't fit in internal RAM), and caching
  short shows.
- Read-ahead buffer for playback: the SD task fills PSRAM ahead of the player,
  so a slow card read never stalls output. This is what makes high frame
  rates (around 120 fps, 8 ms per frame) dependable: today a frame read takes
  6 to 7 ms for the Halloween show, which barely fits. Sizes for 8 MB: the
  Halloween show (4,248 bytes per frame) at 40 fps is about 49 s of frames,
  so it can hold a whole short show; a full board (32,000 pixels, 96 KB per
  frame) is about 85 frames, about 2 s at 40 fps, which is plenty as a buffer.
  Note that wire speed still caps the frame rate: 120 fps allows about 270
  pixels per port.

### Keep

- 32 outputs: SN74LVC8T245 level shifters, 47 ohm series resistors, PTC per
  port, four zones (5 to 24V, see above), each with its own input and blade fuse.
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
9. **String renumbering by zone:** the `string_to_bit` table in the LED
   driver (see "Zone silkscreen" above). Small, and needed before the first
   board of this revision lights up, since the silkscreen will use the new
   numbers.

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

- ~~Confirm XIP_CS1 is available on GPIO 47 of the RP2350B.~~ Confirmed in
  Pico SDK `io_bank0.h`: GPIO47 function 9 is `XIP_SS_N_1`.
- Inner copper weight: the stackup says 1 oz, but JLC's 4-layer default is
  0.5 oz on inner layers. Confirm what is ordered.
- README.md still lists a DIP switch, LVDS and the display. Update it when this revision
  lands.
