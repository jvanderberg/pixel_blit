# v5 design audit (October 2026)

Four independent reviews of the v5 board at commit `1ffcc0a`: power planes
and heat, signal integrity and emissions, power protection, and ESD. Their
findings are merged and ranked here. Each item gives the evidence measured on
the board and the proposed fix. Strike items through and note the commit as
they are done.

## Fix before ordering

### 1. ~~The 32 LED data pins have no protection~~ (done)
**Done:** D19–D50, MDD SMF5.0A (LCSC C193402, SOD-123FL), cathode on each
port's data line on the terminal side of the 47 Ω, anode to a GND via beside
it (In1/In2). Each sits within about 5–9 mm of its terminal where the
housings and the data bundles leave room. J9 and J30 tap their traces
about 8 mm from pin 2. The data nets are now named `Net-(Dn-C)`. A few
+/− polarity marks were nudged clear. The 47 Ω resistors are unchanged.

- **Where:** U5–U8 B outputs, through R19–R50 (47 Ω, 0402), to pin 2 of J5–J36.
- **Evidence:**
  - The 47 Ω sits within 1.5–6.7 mm of the level shifter.
  - 12–152 mm of bare F.Cu runs from the resistor to the terminal; J20 is the longest.
  - No pin-2 net has a clamp to GND.
- **Risk:**
  - **ESD:** a discharge from a hand or a long string goes straight into the SN74LVC8T245.
  - **Miswire:** zone power (up to 24 V) on a data pin, or a string whose ground comes off first, pushes about 0.4 A through the output clamp into +5V. D1 and D3 block it from going anywhere else, so +5V rises past 6 V, the absolute maximum of U4 and the shifters, and the excess can reach +3V3.
- **Fix:**
  - A 5 V unidirectional TVS from pin 2 to pin 3 (GND) at each terminal: SMF5.0A in SOD-123FL (LCSC C320547, C2759874 or C193402). There is room just above each terminal body, about 2 mm from the GND pad.
  - Optionally a 100 Ω resistor in 0603: it softens the edges and fuses on a miswire.

### 2. ~~Wi-Fi SPI routes are 150–175 mm long~~ (done)
**Done:** rerouted by hand, mostly on In2 around the east side of the
3.3 V island: WL_CLK 83 mm (including R75), WL_DATA 85, WL_CS 70, WL_ON 83.
The lanes cross the SD bus only at short angles. The lower ends were moved
0.55 mm east so WL_CLK no longer sits over SD_MOSI (B.Cu, x 88.4). R75 (33 Ω,
0402, C25105) is in series on WL_CLK about 4 mm north of U1 pin 44, before
the In2 via. Still open: CLK runs about 35 mm within 0.45 mm (3× width) of
WL_DATA, at a 0.4 mm pitch with a 0.25 mm gap.

- **Where:** WL_CLK (173 mm), WL_DATA (165), WL_CS (148), WL_ON (147), all on F.Cu, from U1 to U9.
- **Evidence:**
  - These are the old LVDS routes: north to y 52.8, east to x 101.9, past U6, then south along x 73.1. The module is about 50 mm from the MCU.
  - WL_CLK runs beside WL_DATA for 85 mm with an edge gap of 0.35 mm or less, down to 0.18 mm along x 72.8–73.1.
  - WL_CLK also runs beside /LED_B for 43 mm at 0.15 mm.
- **Risk:** ringing or double-clocking of the CYW43439 at full SPI speed; bench tests with short wires won't show it.
- **Fix:**
  - Reroute directly, about 60 mm.
  - Add a 22–33 Ω series resistor on WL_CLK at U1, and space CLK from DATA by at least 3× the trace width.

### 3. ~~The IR input isn't safe for 5 V~~ (done)
**Done:** SW2's IR pins are now the net IR_RAW. R76 (1 kΩ, 0402, C11702) sits in
series on the line beside SW2, and D51 (B5819W, already on the BOM) clamps
IR_IN to +3V3 right after it. +3V3 comes from the J38 branch through a via at
(179, 95) and a 1.8 mm In2 stub. A 10k/20k divider was rejected: the receiver's
internal pull-up (about 30 kΩ to 5 V) would pull the high level down to about
1.7 V. With the clamp, the idle level is about 3.45 V at about 45 µA, and a 5 V
short while plugging in pushes about 1.4 mA into +3V3. The 100 pF at U1 was
not added.

- **Where:** CN1 → SW2 → IR_IN → U1 pin 53 (GPIO42), 172 mm on F.Cu.
- **Evidence:**
  - GPIO 40–47 are ADC pins and are not 5 V tolerant (check the RP2350 datasheet's pin table).
  - The receiver runs on +5V through R2 (100 Ω), and its output idles near 5 V.
  - Plugging in the 3.5 mm jack briefly shorts 5 V onto the signal.
  - There is no ESD path, and IR_IN runs beside /LED_R (62 mm) and /LED_B (48 mm).
- **Fix:**
  - At CN1: a series resistor and a clamp to +3V3, as a divider (10k/20k) or a BAT54S.
  - Optionally 100 pF at U1.

### 4. ~~The 3.3 V feed to the RM2 and MCU is one long 1 mm In2 track~~ (done)
**Done:** the In2 trunk is now 4 mm wide (2.5 mm for the last 8 mm beside
USB1). The GND stitching vias along it were moved out to keep 0.25 mm
clearance, and three on the north side with no room were removed. The RM2
feed from the trunk is three vias and a 1 mm F.Cu track. At 0.5 oz inner
copper: about 58 mΩ from L3 to U1 and about 55 mΩ to U9 pin 14, so the RM2
sees about 3.26 V at a 750 mA peak. 1 oz inner copper is no longer needed.

- **Where:** from U4/L3 at (188, 98), through JP1 and 3 vias at (187–190, 105.7), then In2 to the via at (60.5, 99.4) and U9 pin 14.
- **Evidence:**
  - About 190 mm of 1 mm track with no parallel path.
  - 197 mΩ to U9 and 201 mΩ to U1 with 0.5 oz inner copper; 114 and 104 mΩ with 1 oz.
  - At the 750 mA peak the drop is 85–150 mV, so the RM2 sees 3.18–3.24 V, below the 3.2 V RF threshold.
- **Fix:** any of these:
  - widen the track to 2.5–3 mm;
  - add a parallel F.Cu or B.Cu run;
  - move U4 near the MCU and RM2;
  - at minimum, order 1 oz inner copper.

### 5. ~~The flash chip select runs 109 mm to the Boot button~~ (done)
**Done:** SW1 moved to the left edge below USB1 at (57, 84), rotated 270, and R1 placed
2 mm from U2 pin 1 at (63.4, 85.6). QSPI_SS now ends at R1; only the 6 mm
button side (R1-Pad2) runs to SW1. The 109 mm B.Cu branch, including the
run beside VZ3 along y 56.17, is gone.

- **Where:** QSPI_SS on B.Cu, from U2 at (71.8, 80.1) along y 56.17 to R1 at (145.05, 65.96) and SW1.
- **Evidence:**
  - The line is 15 mm from the MCU to the flash, then a 109 mm dead-end stub to the button.
  - For 64.9 mm it runs between the VZ3 pour and the GND pour on B.Cu, 0.25 mm from VZ3.
  - R1 (1 kΩ) is at the far end, so the stub is unbuffered.
- **Risk:**
  - A step on every chip-select edge.
  - Noise from LED load changes and hot-plugs coupling into XIP flash and PSRAM reads.
  - ESD from the button reaching the line before R1.
- **Fix:** put R1 next to U2 or U1, so that only the button side of R1 runs to SW1. Better, move SW1 near the flash. Keep the line away from VZ3.

### 6. ~~The zone TVSs can't act as a crowbar~~ (done)
**Done:** each zone input now has an ideal diode after its fuse: an LM74700QDBVTQ1
(U11-U14) driving an Infineon BSC014N06NS (Q1-Q4, C113391; 60 V, 1.45 mOhm, about 0.6 W at 20 A),
with 100 nF for VCAP, input and output. A reversed supply is blocked instead of
relying on the TVS to blow the fuse. The fused node is the new net Zn_F;
the SMBJ26A TVS (D10-D13) and the SS34 OR diode stay on VZn, after the FET,
and the blown-fuse LEDs now read across the fuse only (Jn_IN to Zn_F).

Layout:
- Each FET sits just west of its fuse's output pins, source toward the fuse.
- A Zn_F pour on F.Cu and B.Cu links the fuse to the FET, and the Jn_IN pour
  was cut back from the output pins.
- An F.Cu VZn pour on the drain is stitched to the B.Cu VZn band by 7-12 vias.
  Zone 4 uses seven 0.8 mm vias, because the VZ4/VZ3 band boundary is close.
- Each enable pin reaches its anode through a short In2 jumper.
- D4's VIN_SYS track, which ran through the zone-1 FET site, now crosses on In2
  along x 225. Zone 4's VIN_SYS run moved from y 42.7 to y 39.85.
- Zone 4's FET sits 2 mm south of the fuse-pin midpoint, so its controller
  clears the J21 terminal body.
- The zone-border silk was regenerated.

Not covered: an over-voltage supply (above about 29 V) still reaches the TVS.

- **Where:** D10–D13 (SMBJ26A) and fuses F1–F4. Each TVS side has two 0.6 mm stubs and two 0.3 mm vias.
- **Evidence:**
  - With reverse polarity or a supply that's too high, the plan relies on the TVS shorting and blowing the 20 A fuse.
  - Most LED supplies current-limit or hiccup below 20 A, so the fuse doesn't blow.
  - The TVS then dissipates 15–30 W, against a rating of about 5 W.
  - The vias, at about 5–7 A each, burn open first, and the zone is left unprotected.
  - While reversed, the ports sit at −1 to −3 V and the data lines are pulled below GND.
- **Fix:**
  - **Real protection:** an ideal-diode controller with a 40 V N-FET per zone (LM74700-Q1 or LM74610), which also blocks backfeed.
  - **At minimum:** set the TVS pads in solid VZ and GND copper with 6 or more vias per side, and state in the README that the TVS is for spikes only.

## Should fix

### 7. J38's pin order invites a short
- **Where:** J38 pins 3V3 / 5V / VIN / GND.
- **Evidence:** one slipped jumper puts up to 28 V on +5V or +3V3, killing the MCU, flash, PSRAM and RM2. A VIN-to-GND short is limited only by the 20 A fuses, so the 3 A SS34 OR diodes burn first.
- **Fix:**
  - Reorder to VIN, GND, 5V, 3V3.
  - Add a PTC of about 1.5–2 A / 30 V after the OR node (the converter draws about 1.3 A at 4.5 V), or a small PTC on J38's VIN pin.

### 8. A 5 V zone barely starts the logic
- **Where:** the VIN_SYS track from D4 at (225, 146) to the U3 input at (184, 63): 0.5 mm wide, 131 mm long, 129 mΩ.
- **Evidence:**
  - 5.0 V on zone 1 alone, minus 0.45 V in the SS34 and 0.17 V in the track, leaves about 4.38 V against UVLO turn-on of 3.98–4.18 V.
  - Lead drop under pixel load can stop startup.
- **Fix:** widen the track to ≥1.5 mm or pour it, and/or lower UVLO (R11 27k gives about 3.55 V).

### 9. Reset and button lines are long, with the resistor at the far end
- **RUN:** 44 mm to R18 (1 kΩ) at the button end, with no filter. It runs beside SWDIO and STRING30 on B.Cu at 0.10 mm.
- **/BTN_USER:** 63 mm on B.Cu, with no series resistor.
- **Fix:**
  - Move R18 next to U1 and add 100 nF on RUN.
  - Add a series resistor on BTN_USER near U1.

### 10. Regulator decoupling
- **U4 (TPS62A02A):**
  - GND pin 1 reaches GND through a 0.15 × 0.4 mm stub and one via.
  - The input caps are 2.7–4.1 mm from VIN.
  - Fix: widen the stub, add a second via, and move C27 within 1 mm.
- **U3 (TPS552872):**
  - No HF output cap; C46–C49 sit 2–8 mm away.
  - Fix: add 100 nF 0402 at VOUT/PGND. Optionally trim the SW pours (about 35 mm² each) to the inductor pads.

## Minor and documentation

- **Fuse rating:**
  - 20 A per zone runs the ATO fuse at 100% of its rating, about 2 W per fuse.
  - Label the continuous rating at about 15–16 A per zone, and check the XF-507P holder's rating.
- **Supply limits to state:**
  - Maximum 26.4 V input (SMBJ26A breakdown is 28.9 V minimum; some "24 V" supplies can be trimmed up to 28.8 V).
  - Supplies that current-limit at 20 A or less (the PTCs' Imax is 40 A); no batteries.
- **Heat:**
  - D4–D7 (SS34) have only a 0.5 mm cathode track, an estimated 50–70 °C rise at 5 V input.
  - D3 (B5819W) is near its package limit at 1 A.
  - Fix: add small pours, or an SMA part for D3.
- **No zone voltage sensing:**
  - Data is sent into unpowered zones and phantom-powers their first pixel.
  - Fix: a divider from each VZn to an ADC pin, so firmware can blank dead zones and flag a mismatched voltage.
- **Cheap insurance:**
  - 27 Ω on USB D+/D− at U1 (as on the Pico 2);
  - 22–33 Ω on SD SCK/MOSI;
  - a capacitor on USB VBUS;
  - 1 µF / 50 V on each zone rail at the TVS;
  - a 5 V TVS on the IR jack's 5 V pin;
  - 100 nF on RUN;
  - extra GND stitching vias at the board centre (x 140–160, y 82–115) and along the input strip (x > 215).
- **Reference plane change:** /PSRAM_CS and part of QSPI_SS cross the In2 +3V3 island edge at y ≈ 83.7. Small effect.
- **Open vias in pads:** each PTC and SS34 pad has untented vias. Order filled and capped vias if the price is right.
- **Voltage drop at 5 V:** from the input to the farthest port at 20 A the drop is about 0.25–0.35 V. Note it in the README.

## Checked and fine

- **Zone pours (B.Cu VZ1–VZ4):** no real necks; 0.8–1.0 W and an estimated 15–25 °C rise near the fuse end at 20 A.
- **Input pours:** 8–12 mV drop at 20 A.
- **Pad connections:** every power pad has a solid connection, no thermal-relief spokes.
- **PTC vias:** five vias per PTC.
- **Port output tracks:** 1.5 mm.
- **Ground return at 80 A:** In1 and In2 carry it with 37 mV at the worst port.
- **U3 switching loops:** tight, over solid In1; the PGND strip has well-stitched vias.
- **U4 switch node:** about 1.4 mm to L3.
- **Flash and PSRAM bus:** daisy-chained with no stubs, on F.Cu over In1; SCLK is 9 mm to U2 and 21 mm to U10.
- **USB:** about 87 Ω differential, well matched. The USBLC6 (D2) sits between the connector and the MCU, and the CC resistors are fitted.
- **RM2 antenna keepout:** on all layers, with the module flush to the board edge.
- **In1 GND:** solid under every F.Cu signal, and every signal via has a GND via within 0.6–5 mm.
- **Mounting holes:** H1–H4 are plated, grounded and stitched.
- **Fuse LEDs:** the polarity is right and R68–R71 dissipate about 48 mW in a 100 mW part.
- **Fuse, TVS and LED order:** correct.
- **Voltage ratings:** VIN_SYS caps are 50 V; U3's EN divider stays under its limit at 42 V.
- **Mixed zone voltages:** the SS34 OR diodes see at most 19 V reverse between zones.
- **Rail ORing:** D1 and D3 OR USB and the regulator cleanly.
- **Power-up:** the level shifters keep their outputs off until 3.3 V is up.
