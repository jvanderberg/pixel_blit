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

Board ID is set in software (web UI or the panel menu). The DIP switch idea is
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
- Pins: the radio's SPI link needs 4 GPIOs (power enable, data, chip select,
  clock). Use **GPIO 32 to 35**, freed by the LVDS removal. The Pico 2 W uses
  GPIO 23, 24, 25 and 29, which are LED string outputs on this board, so the
  pins must be set in the board header. Confirm in the Pico SDK version in use
  that the CYW43 pins are configurable for a custom board before layout.

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

### Change: 3.3V regulator

- The ME6217C33 is rated 300 mA. The RP2350, SD card and display already use a
  good share of that, and Wi-Fi transmit adds spikes of a few hundred mA. Expect
  brownouts and resets while transmitting if it stays.
- Use a 3.3V buck converter from the 5V rail, around 1 A. If it must stay
  linear, use a 1 A LDO in a package that can shed heat: 5V to 3.3V at 0.5 A is
  about 0.85 W, too much for SOT-23-5. The Pico 2 W uses a switching regulator
  rated around 800 mA for this reason.

### Add: 12V rail sense

- The TPS63070 buck-boost regulates 5V from about 2V of input, so the RP2350
  boots while the pixel 12V rail is still rising. Strings that get data during
  that ramp come up dark or half lit until the data restarts (October 2026,
  Halloween matrix). The firmware currently holds output for 2 s after boot as a
  stopgap; it has no way to see the rail.
- Divider from +12V_SYS to **GPIO 44 (ADC4)**, freed by removing DATA_IN_EN.
  Example: 100k over 22k gives 2.16V at 12V and 2.7V at the 15V zener clamp,
  with 100 nF at the ADC pin. Disable the pad pulls on that pin.
- Firmware waits for the rail to pass a threshold (around 11V) before the first
  frame, and can report brownouts.
- Alternative or addition: a divider on the TPS63070 EN pin so 5V only starts
  once the input passes about 9V. No firmware needed, but the board would then
  ignore USB-only power for its logic, which matters for bench testing. Check
  the EN threshold in the datasheet before choosing values.

### Keep

- 32 outputs: SN74LVC8T245 level shifters, 47 ohm series resistors, PTC per
  port, four 12V zones, each with its own input and blade fuse.
- microSD on GPIO 36 to 39. Consider raising the SPI clock from 12.5 MHz (a
  firmware change) before anything else, for faster reads and uploads.
- IR receiver on GPIO 42, with SW2 and CN1.
- J39 front-panel header: display (I2C on GPIO 46 and 47) plus encoder and back
  button (GPIO 40, 41, 43, 45). The panel becomes optional: useful for showing
  the IP address, Wi-Fi status, board ID, the current show and frame, and SD or
  config errors, and for local control. Wi-Fi setup is easier from a phone.
- USB-C, BOOTSEL (SW1) and reset, for flashing and recovery.

### Add: on-board button and status LED

No GPIO is left once the radio and 12V sense are in, so:

- **Button:** wire it in parallel with the panel's push button (PSH, GPIO 41).
  Press: play or next show. Hold at power-up: clear Wi-Fi settings and start the
  setup access point.
- **Status LED:** the Pico 2 W drives its LED from a CYW43439 GPIO (WL_GPIO0).
  If the RM2 brings that pin out, use it for a status LED. Otherwise, choose
  between giving up a panel line and doing without. Check the RM2 pinout.

## GPIO map

| GPIO | Now | Next revision |
|---|---|---|
| 0 to 31 | LED strings | LED strings |
| 32 to 35 | LVDS HS_DATA0 to 3 | RM2 Wi-Fi |
| 36 to 39 | microSD | microSD |
| 40, 41, 43, 45 | Panel: back, push, encoder B, encoder A | Same (on-board button in parallel with 41) |
| 42 | IR in | IR in |
| 44 (ADC4) | LVDS DATA_IN_EN | 12V sense |
| 46, 47 | Display I2C | Display I2C |

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
5. **12V sense:** gate the first frame on the rail; replace the fixed 2 s hold.
6. **DDP input** for live preview.
7. **FPP discovery and FPP Connect uploads.**
8. **Over-the-air firmware updates**, with USB as the fallback.

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

- Does the RM2 bring out a CYW43439 GPIO for a status LED?
- Are the CYW43 pins configurable in the Pico SDK version in use?
- 12V sense alone, or also a UVLO divider on the TPS63070 EN pin?
- Inner copper weight: the stackup says 1 oz, but JLC's 4-layer default is
  0.5 oz on inner layers. Confirm what is ordered.
- README.md still lists a DIP switch and LVDS. Update it when this revision
  lands.
