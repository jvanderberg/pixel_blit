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
  **Done:** all 32 are now LUTE 1812L300/30GR (LCSC C48985875, imported
  with jlcimport: JLCImport symbol and footprint, pins redrawn at ±5.08 to
  match the old symbol; pads are identical to the old F1812): 3 A hold,
  5 A trip, 30 V, same 1812 body (4.73 x 3.41 mm) and footprint. Its Imax is
  40 A, and so is every 30 V polymer PTC checked at JLC, in 1812 and in 2920
  (1812L260/30, BSMD1812-200-30V, 2920L300/30GR). So the 2x rule holds with
  a **20 A zone fuse**, not 25 A; the silkscreen says "BLADE FUSES 20A".
- **Capacitors** on the zone rails and `+12V_SYS`: 35V or 50V.
- **TVS diode** on each zone input, clamping just above 24V, for hot-plugging
  supplies on long leads.
- **Unchanged:** SS34 OR diodes (40V), blade fuses (32V), terminals, copper,
  level shifters and data path. 24V pixels still take 5V logic data.
- **Blown-fuse LED resistor:** brightness follows the zone voltage. 10k gives
  about 2 mA at 24V, 1 mA at 12V and 0.3 mA at 5V. Use a high-efficiency LED
  so it is still visible on a 5V zone.

**Intake stage design (October 2026).** From each zone input to `5V_REG`.

- **Per zone, unchanged:** terminal J1 to J4, blade fuse F1 to F4, then the
  zone rail `+12V_Zn`, which should be renamed `VZn` since it is no longer
  12V. The ports hang off that rail.
- **Per zone, new:** an SMBJ26A TVS from the zone rail to GND, after the fuse
  (LCSC C726831; Littelfuse C315992). Standoff 26 V covers a 24 V supply
  turned up to 26.4 V; it breaks down at 28.9 V minimum. Behind the fuse, a
  supply that is simply too high (a 36 V brick) makes the TVS fail short and
  blow the fuse, so the ports are protected.
- **OR diodes:** keep D4 to D7 (SS34, 3 A, 40 V). Each carries at most the
  converter's input current, about 1.3 A at a 4.5 V input. The worst reverse
  voltage is 19 V (24 V on one zone, 5 V on another).
- **`+12V_SYS`**, to be renamed `VIN_SYS`, spans 4.5 to 28 V:
  - Remove D10 (BZT52C15V). It was a 15 V clamp protecting the TPS63070,
    and it would burn up at 24 V.
  - C22 and C23 (22 µF 25 V X5R 0805) and C24 (10 µF 25 V 0603) would run
    at their rating with little capacitance left. Replace them with
    2 x 10 µF 50 V X7R 1206 (C89632) at the converter and a 47 µF 50 V
    electrolytic (C72523, 6.3 x 7.7 mm). The electrolytic's ESR damps
    hot-plug ringing on long leads, which an all-ceramic input would make
    worse.
  - PWR_LED3 with R3 = 8.2k draws 2.7 mA at 24 V, and R3 dissipates 59 mW
    in an 0603 rated 100 mW. That is fine, and the LED is dim on 5 V.
  - J38 pin 3 brings this rail out. Its silkscreen says 12V; change it to
    VIN.
- **Converter: TPS552872** (TI, 36 V, 4 A fully integrated four-switch
  buck-boost, VQFN-HR-21 3 x 5 mm). LCSC stocks it as the automotive grade
  TPS552872QWRYQRQ1 (C22397842, 1.5k); the commercial TPS552872RYQR
  (C22427496) has 188. The TPS552892 (8 A, C19272254, 30k in stock) has
  the same pinout and drops in, but its 13 A peak limit would need a bigger
  inductor. Checked in the datasheet (SLVSHG1):
  - VIN and SW1 absolute maximum 42 V, above the TVS clamp.
  - **EN/UVLO absolute maximum is 20 V**, so it cannot be tied to VIN. Use a
    divider of 51k over 22k (JLC basic parts): on at 4.08 V, with
    5 µA × 51k = 0.26 V of hysteresis, and only 8.4 V on the pin at 28 V.
  - FB reference 1.2 V ±1%. R_up 100k (TI's recommendation) over
    20k + 10k in series gives **5.20 V**, which D3 brings down to about
    4.85 V on `+5V`. All three are basic parts; the exact 29.4k would be
    an extended part.
  - FSW: fSW = 10⁶ / (50·R_kΩ + 35) kHz, so 20k (basic) gives **966 kHz**.
    That is a compromise between switching loss at 28 V in and inductor
    size.
  - **MODE to VCC: forced PWM**, the same reasoning as the 3.3 V buck.
    EXTVCC left open (internal LDO; the datasheet allows high or floating,
    and it saves a crossing); VCC cap 10 µF; BOOT1 and BOOT2 0.1 µF;
    DITH/SYNC to GND (no dithering); PG and CC open, or to test pads; CDC
    open.
  - **No output current limit resistor:** ISP and ISN tie to VOUT. The part
    still has its 4 A average current limit and hiccup short-circuit
    protection, and it saves an extended part. A 25 mΩ sense resistor
    (V_SNS = 50 mV, so 2 A) can be added later if J38's 5 V pin needs it.
- **Inductor: 4.7 µH, I_sat 8 A** (Sunlord MWSA0605S-4R7MT, C408465, 7 x
  6.6 mm; the ABC GSSM05304R7M3AU, C2928302, is 5.2 x 5.7 mm). I_sat is
  above the 6.5 A peak current limit, so the inductor does not saturate in
  a fault. Ripple is 0.88 to 0.91 A p-p at 24 to 28 V in, so the peak is
  1.46 A at a 1 A load. Boost mode at 4.5 V in: I_L(DC) is 1.31 A with
  little ripple. 4.7 µH is above the inner loop's 1.2/fSW = 1.2 µH minimum.
- **Output caps:** 4 x 22 µF 16 V X5R 0805 (C98190), about 50 µF effective
  at 5.3 V. C21 (47 µF 6.3 V) runs too close to its rating at 5.3 V and
  goes.
- **Compensation** (datasheet equations 24 to 26, worst case boost at
  4.5 V in and 1 A out, so D = 0.151): f_RHPZ = 129 kHz, crossover
  f_C = 20 kHz (below f_RHPZ/5 and fSW/10), G_EA = 190 µA/V, R_SENSE
  0.055 Ω. That gives R_C = 9.5k, C_C = 14 nF, C_P = 26 pF; fitted as
  basic parts **R_C = 10k, C_C = 10 nF, C_P = 22 pF** (zero at 1.6 kHz,
  well below the 20 kHz crossover). These are calculated values; check them
  in TI's design tool or on the bench.
- **Load:** 1 A continuous at 5 V. The 3.3 V buck draws about 0.55 A at
  its 750 mA peak, the level shifters' 5 V side and LEDs about 50 mA, and
  J38's 5 V pin is budgeted at 100 mA.
- **Output OR:** keep D3 (B5819W) from `5V_REG` to `+5V`, and D1 from
  USB. An ideal-diode controller is an option later, if the extra 0.3 V
  is ever wanted back.
- **Layout (done):** U3 at (192.6, 67.4) in the old TPS63070's spot,
  following TI's layout example (SLVSHG1 figure 8-13) rotated -90°. The
  SW1 and SW2 bars exit right to the inductor on two SW pours, with the
  bootstrap caps at the corners. VIN, PGND and VOUT exit left into three
  stacked pours, keeping the old arrangement (+12V_SYS upper left, 5V_REG
  lower left next to D3). The input caps bridge VIN and PGND, the output
  caps PGND and VOUT, and the PGND strip has its own vias plus three at the
  PGND bar's right end. FSW drops to B.Cu next to the pin so the VIN neck
  stays clear, and MODE reaches VCC on a short B.Cu link. The SW copies on
  In1 under the old inductor are gone, so In1 is solid GND under the
  converter. The 47 µF electrolytic (C24) sits on the +12V_SYS feed at
  x = 179.85. The zone TVS (D10 to D13) sit beside each fuse with two vias
  to the zone plane and two to GND. The display's silkscreen outline is
  removed.

**Mixed voltages:** each zone feeds only its own 8 ports, and the OR diodes
power the logic from whichever zone is highest. So one board can run, for
example, 5V on zone 1, 12V on zones 2 and 3, and 24V on zone 4. The lower
zones' diodes are reverse-biased: 19V at most, within the SS34's 40V rating.
All zones share ground. The zone link fuses must be gone for this (see
below), since fitted they would tie two supplies together.

**Done (silkscreen):** "6-12VDC" is now "5-24VDC" and the "12V" labels
(J38 and the buck-boost input) read "VIN". Every string label is
"S‹n›·Z‹z›" in the new zone numbering, plain text at one offset per row.
Each input terminal reads "+ Z‹n› −" (zone number between the existing
polarity marks). A silkscreen border outlines each zone, following its B.Cu
pour plus its input's F.Cu pour; neighbouring zones share one line, and the
line breaks around pads, part silk and text. A zone key beside "5-24VDC"
lists zone, strings and a write-in voltage box, plus the 20 A fuse note.
The schematic nets are renamed to the new numbers (`STRINGn` = GPIO for
string n, `Sn_OUT`), so schematic, board and silkscreen agree; the firmware
`string_to_bit` table is still to do.

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

**Done.** D15-D18 (KT-0603R red) with R68-R71 (10k 0603, 100 mW: 58 mW at
24 V) from Jn_IN to the zone rail, in the strip between each fuse holder and
its input terminal, inside the input pour; the resistor drops a via to the
zone's B.Cu pour.

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

**Spec (October 2026).**

- **Input:** `+5V`, which is the 5V regulator or USB through a B5819W. It
  stays on `+5V` rather than the zone rails so the board still runs from
  USB alone. Worst case is USB at 4.75 V minus the diode, about 4.3 V.
- **Output:** 3.3 V ±3%. The tightest load is the RM2: VDDIO 3.3 V ±5%,
  and VIN below 3.2 V reduces RF performance.
- **Load budget (3.3 V).** Only the RM2 figure is from its datasheet; the
  rest are estimates to check on the bench.

  | Load | Peak | Typical |
  |---|---|---|
  | RM2, transmit (datasheet: 271 mA at MCS7, 16 dBm) | 300 mA | 50–100 mA |
  | SD card, write | 200 mA | 50 mA |
  | RP2350 (core regulator, IO, USB PHY) | 100 mA | 50 mA |
  | PSRAM + flash | 65 mA | 20 mA |
  | Level shifters (A side), IR, LEDs, pull-ups | 30 mA | 15 mA |
  | J38 3.3 V pin (external) | 50 mA | 0 |
  | **Total** | **≈ 750 mA** | **≈ 250 mA** |

  So: at least 1 A continuous. A 2 A part gives headroom and runs cool.
- **Noise:** forced PWM at a fixed frequency, not power-save mode at light
  load. Power-save mode lets the switching frequency and ripple wander with
  the load, which is unwelcome next to the radio and the ADC. The Pico 2 W
  forces PWM on its regulator for the same reason.
- **Transients:** the RM2 transmits in bursts, so the converter needs a fast
  current-mode loop and the 22 µF output capacitor at the module (C42
  10 µF + C43/C44 are already there).
- **Enable:** keep the existing `3V3_EN` (R15 100 kΩ to +5V).

**Recommended: TPS62A02ADRLR** (TI, LCSC C5820994, 14.5k in stock).
Checked in the datasheet (SLUSEG9E):

- 2.5 to 5.5 V in, 2 A, SOT-563 (pins: 1 GND, 2 SW, 3 VIN, 4 EN, 5 FB,
  6 PG).
- The `A` suffix means **forced PWM** at **2.4 MHz**. The plain TPS62A02 uses
  power-save mode.
- High-side current limit 2.7 A min / 3.4 A typical. 100% duty-cycle mode
  gives low dropout: VIN(min) ≈ 3.3 V + 0.75 A × (100 mΩ + DCR) ≈ 3.45 V,
  well under the 4.3 V worst case.
- FB 600 mV ±1%. R2 must be 100 kΩ or less: **R1 = 100 kΩ, R2 = 22 kΩ**
  gives 3.327 V (+0.8%). 100 kΩ is already in the library; 22 kΩ is JLC
  basic C25768.
- L = 1 µH, COUT = 22 µF (the datasheet's standard, recommended
  combination), CIN = 4.7 µF.
- Rated for 10–47 µF of output capacitance. C53 (22 µF) sits at the
  regulator and C28 (10 µF) stays at the 3V3 node next to it: 32 µF, with
  C16, C42 and the 100 nF parts further out on the rail.
- PG unused: leave open (the datasheet allows open or GND). The RP2350's own
  brownout handles power-up.

**Inductor:** Murata DFE201612E-1R0M=P2 (LCSC C337892, 1 µH, 4 A,
2.0 x 1.6 mm, 21k in stock). Same family as the DFE252012F in TI's
example. Sunlord MWTC201610S1R0MT (C2840989, 4.6 A) is a cheaper fallback.

**Fallback IC:** TLV62569DBVR (C141836, 416k in stock, SOT-23-5,
1.5 MHz, 2.2 µH / 10 µF). It is cheaper and more widely stocked, but it
always uses power-save mode at light load (no forced-PWM version), so it is
second choice.

**Placement (done):** U4 at (188.4, 95.6), laid out like TI's SOT563
example rotated 90°: GND | SW | VIN along the bottom, the inductor below,
a GND pour above with CIN (C52) and COUT (C53) bridging into it, and a
+3V3 output pour that reaches C28, C29, TP2 and JP1, so the In2 +3V3 trunk
is fed as before. The feedback divider sits above the IC, away from SW.

No JLC basic-library buck exists, so any choice pays the extended-part fee.

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
- USB-C, BOOTSEL (SW1) and reset, for flashing and recovery. **Moved** into
  the old display area beside the MCU (SW1 at 121,65; Reset1 at 121,77).
  QSPI_SS now drops a short B.Cu stub to R1 from its y=56 run, and RUN
  ends at R18, so the long QSPI_SS leg that split the back ground plane at
  x=145.6, its stitching vias and the RUN loop out to x=178 are gone.

### Add: on-board button and status LED

**Done.** D14, a TOGIALED TJ-S1615CY6TGLCCSRGB-A5 (C601684, common cathode)
with R72 1k (red), R73 1k (green), R74 470R (blue); SW3 "Play" to GND (use
the internal pull-up). They reuse the old display-panel routes, so the
colour/GPIO assignment follows those lines: red GPIO 43 (was TRB), green
GPIO 45 (TRA), blue GPIO 41 (PSH), button GPIO 40 (BAK, the panel's back
button). TRB and TRA are F.Cu end to end; PSH's old 30 mm B.Cu tail now runs
on F.Cu beside TRB, with a 2 mm B.Cu hop under IR_IN. BAK is restored as it
was (about 63 mm, mostly on B.Cu along y = 66.8). The LED sits where the
routes come out of their diagonal, Play at the old header row, and Boot
moved right to (141, 66) to clear TRA.

- **Status LED:** a plain common-anode or common-cathode RGB LED on **GPIO 40,
  41 and 43** with a resistor each, driven by PWM. No level shifting, unlike an
  addressable LED. Setup mode: blinking blue. Connected: green. Playing: slow
  pulse. SD card or config error: red.
- **Button:** on **GPIO 44**. Press: play or next show. Hold at power-up: clear
  Wi-Fi settings and start the setup access point.
- **Spare:** GPIO 44 and 46 (as built). GPIO 46 had the display's I2C data
  route; GPIO 44 (pad 55) would need an escape via in the pocket under IR_IN.

## GPIO map

| GPIO | Now | Next revision |
|---|---|---|
| 0 to 31 | LED strings | LED strings |
| 32 to 35 | LVDS HS_DATA0 to 3 | RM2 Wi-Fi |
| 36 to 39 | microSD | microSD |
| 40 | Panel: back | Button |
| 41, 43, 45 | Panel: push, encoder B, encoder A | Status LED blue, red, green |
| 44 | LVDS DATA_IN_EN | Spare |
| 42 | IR in | IR in |
| 46 | Display I2C data | Spare |
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
