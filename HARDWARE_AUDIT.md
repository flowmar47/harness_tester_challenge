# Harness Tester Challenge — Hardware Audit

Independent bug list for `commaai/harness_tester_challenge`.

| | |
|---|---|
| Repository revision audited | `069f7243d9ca87b31333d873bb33b08cfb54042a` |
| Schematic title block | "Hardware Hiring Challenge", comma.ai, rev A, 2024-05-23 |
| Files audited | `firmware/firmware.ino`, `firmware/CY8C9560.cpp`, `firmware/CY8C9560.h`, `kicad_files/hardware_challenge.kicad_sch`, `kicad_files/hardware_challenge.kicad_pcb` |
| Findings | 24 (13 firmware, 8 schematic, 3 PCB) |

Every finding below names an exact location, a deterministic failure mechanism, the
evidence used to reproduce it, and a primary source where the claim depends on a part
specification. Observations that did not survive an adversarial pass are listed at the
end under **Not counted**, with the reason each was rejected.

## How this was reproduced

KiCad 10.0.5 (official macOS build, run read-only from a mounted DMG) was used to
regenerate a netlist and to run ERC/DRC with zone refill and schematic-parity enabled:

```
kicad-cli sch export netlist kicad_files/hardware_challenge.kicad_sch -o netlist.net
kicad-cli sch erc            kicad_files/hardware_challenge.kicad_sch \
                             --format json --severity-all -o erc.json
kicad-cli pcb drc            kicad_files/hardware_challenge.kicad_pcb --format json \
                             --schematic-parity --all-track-errors --refill-zones \
                             --severity-all -o drc.json
```

(`--save-board` is deliberately omitted: zones are refilled in memory only and no
challenge file is modified.)

Netlist, zone-fill polygons, track/via geometry and footprint land patterns were then
queried directly out of the KiCad s-expression files so that every geometric claim is a
measurement, not a visual reading. Firmware claims are read off the source with the
relevant device register semantics taken from the CY8C9560A datasheet.

Results of that run, for context:

- **ERC: 107 items — 4 errors, 103 warnings.** The four errors are three
  `power_pin_not_driven` on power-flag symbols (`#PWR01`, `#PWR04`, `#PWR012`) and one
  `pin_not_driven` on the antenna symbol's input pin. None is a functional fault. Note
  what ERC does *not* catch: the TX↔TX short in HW-14 passes clean because the Teensy
  symbol declares its pins `bidirectional`.
- **DRC: 405 violations, 1 unconnected item, 2 schematic-parity items.** Only the
  unconnected item is used as evidence below (HW-22). The violation total is dominated by
  199 `track_width` and 83 `silk_edge_clearance` project-rule items and is *not* treated
  as a finding source.

---

# Firmware

## HW-01 — `cy.begin()` is never called, so the I/O expander is never initialised

**Location:** `firmware/firmware.ino:97-116` (`setup()`), `firmware/CY8C9560.cpp:3-15`

`CY8C9560::begin()` is the only code that calls `WIRE.begin()`, `WIRE.setClock()`, sets
`CY_RST`/`CY_IRQ_N` pin modes, issues the XRES pulse, and verifies the device ID. Nothing
in the sketch ever calls it — the only uses of the object are `cy.set_output()`,
`cy.set_pd_inputs()` and `cy.read_inputs()` inside `loop()`.

**Failure:** `Wire2` is never started, so every `beginTransmission`/`requestFrom` in the
driver operates on an uninitialised peripheral. No harness pin is ever driven and
`read_inputs()` returns garbage. The tester cannot test anything.

**Evidence:** `grep -n 'cy\.' firmware/firmware.ino` returns only lines 145, 146 and 148.
`begin()` appears nowhere in the sketch.

## HW-02 — Output GPIOs are never configured as outputs

**Location:** `firmware/firmware.ino:104-107`, and `set_status()` at `:67-71`

`setup()` calls `pinMode()` for exactly two pins, both inputs:

```c
pinMode(PIN_BTN_TEST, INPUT);
pinMode(PIN_UBX_TIMEPULSE, INPUT);
digitalWrite(PIN_UBX_SAFEBOOT, LOW);
digitalWrite(PIN_UBX_RST_N, HIGH);
```

`PIN_LED_R` (5), `PIN_LED_G` (6), `PIN_LED_B` (7), `PIN_UBX_SAFEBOOT` (3) and
`PIN_UBX_RST_N` (4) are never put into `OUTPUT` mode, yet all five are only ever used
with `digitalWrite()`.

**Failure:** on an i.MX RT1062 the pad stays configured as an input, so the GPIO data
register write has no effect on the pin. The RGB LED — the board's only user-facing
status output (README: "start button and RGB LED are the user interface") — never
lights, in any state, ever.

## HW-03 — Firmware drives `SAFEBOOT_N` low, which stops the GNSS module from booting

**Location:** `firmware/firmware.ino:106`, net `UBX-SAFEBOOT` → U3 pin 1

`digitalWrite(PIN_UBX_SAFEBOOT, LOW)` expresses the intent to hold the NEO-M8N's
`SAFEBOOT_N` pin low. That is precisely the state that must be avoided.

**Failure:** *"If the SAFEBOOT_N pin is 'low' at start up, the u-blox M8 module starts in
Safe Boot Mode and doesn't begin GNSS operation."* No NMEA output is produced, so
`time_fixed` never becomes true and `loop()` returns at line 134 forever. The NEO-M8
datasheet pin table further specifies SAFEBOOT_N should be left **OPEN**.

**Primary source:** NEO-M8 Hardware Integration Manual UBX-13003557, §1.5 I/O pins,
"SAFEBOOT_N"; NEO-M8 Data Sheet UBX-15031086, pin 1.

*Independence note:* this is a distinct defect from HW-02. HW-02 is a missing
`pinMode()`; HW-03 is the wrong intended level. Fixing either one alone leaves the design
broken — fixing only HW-02 actively asserts safe-boot mode.

## HW-04 — The NMEA parser only accepts `$GPRMC`, but the module emits `$GNRMC` by default

**Location:** `firmware/firmware.ino:75-76`

```c
if (strncmp(buf, "$GPRMC", 6) == 0) {
  if (sscanf(buf, "$GPRMC,%10[^,],...
```

**Failure:** the NEO-M8N's default configuration is *"concurrent reception of GPS and
GLONASS"*, and u-blox assigns the talker ID by GNSS configuration: **"Any combination of
GNSS → GN"**. Out of the box the receiver therefore emits `$GNRMC`, which fails the
`strncmp`. `time_fixed` never becomes true, and the tester never leaves the "waiting for
GPS time lock" state.

**Primary sources:** NEO-M8 Data Sheet UBX-15031086, Table 1 ("default: concurrent
reception of GPS and GLONASS"); u-blox 8 / M8 Receiver Description UBX-13003221 §31.1.2
"NMEA Talker IDs".

## HW-05 — 64-byte NMEA buffer overflows on every sentence; `process_nmea` also writes one past the end

**Location:** `firmware/firmware.ino:118-131` and `:74`

```c
char nmea_buf[64];
int  nmea_idx = 0;
...
nmea_buf[nmea_idx++] = UBX_SERIAL.read();          // no bound on nmea_idx
...
void process_nmea(char *buf, int len) { buf[len] = 0; ... }   // writes buf[len]
```

`nmea_idx` is never compared against `sizeof(nmea_buf)`, and `process_nmea()` writes its
terminator at `buf[len]`, i.e. at index `nmea_idx`, which is one past the last byte
stored.

**Failure:** this is not a corner case — it happens on every sentence. The RMC example
in the u-blox protocol spec,
`$GPRMC,083559.00,A,4717.11437,N,00833.91522,E,0.004,77.52,091202,,,A,V*57`, is 73
characters, 75 including CR LF. Writes therefore run to `nmea_buf[74]` on a 64-byte
buffer — 11 bytes past the end — corrupting whatever the linker placed after it in `.bss`.
Memory corruption on the very first GNSS sentence received, and it repeats at 1 Hz.

**Primary source:** u-blox 8 / M8 Receiver Description UBX-13003221 §31.2 RMC example
sentence.

## HW-06 — Test button polarity is inverted

**Location:** `firmware/firmware.ino:138`, `pinMode` at `:104`; schematic SW1, R4

```c
if (digitalRead(PIN_BTN_TEST) == LOW) return;   // "not pressed → return"
```

The hardware says the opposite. From the netlist: SW1 pins 1 and 3 → `GND`, pins 2 and 4
→ `BTN_TEST`; R4 (10 kΩ) connects `+3.3V` → `BTN_TEST`. `BTN_TEST` therefore idles HIGH
and goes LOW while the button is held.

**Failure:** the logic is exactly inverted. The tester runs a full harness test on every
`loop()` iteration whenever the button is *released*, and stops as soon as it is
*pressed*. Combined with HW-13 this means the board free-runs tests and writes SD log
lines continuously from power-up, and the start button acts as a stop button.

## HW-07 — `passed` is OR-accumulated: one matching pin passes the whole harness

**Location:** `firmware/firmware.ino:142`, `:155-157`

```c
bool passed = false;
for (int i = 0; i < NUM_HARNESS_PINS; i++) {
  ...
  if (values == EXPECTED_CONNECTIONS[i]) {
    passed = true;              // never cleared on mismatch
  }
}
```

**Failure:** the result is a logical OR across pins where it must be an AND. A harness
with 39 wrong pins and one correct pin reports "Harness passed!", lights GOOD and writes
`Passed` to `results.txt`. For a device whose entire purpose is achieving a near-zero
harness escape rate, this is the worst possible failure mode: it passes bad parts
silently.

## HW-08 — `1 << i` is a 32-bit shift, so harness pins 31-39 are never driven or reported

**Location:** `firmware/firmware.ino:144` and `:152`

```c
uint64_t output_mask = 1 << i;                        // i runs 0..39
...
DBG_SERIAL.printf("%d", (values & (1 << j)) ? 1 : 0); // j runs 0..39
```

`1` is an `int`. The shift is performed in 32-bit arithmetic and only then widened to
`uint64_t`. For `i == 31` the result is `INT_MIN`, which sign-extends on conversion to
`0xFFFFFFFF80000000` — bits 31 through 63 all set. For `i >= 32` the shift is undefined
behaviour; the observable result depends on codegen (a Cortex-M7 register shift of ≥32
yields 0, while a constant-folded shift may yield anything), but in no case is it the
intended bit.

**Failure:** the upper quarter of a 40-pin tester is unusable. Pin 31 asserts a mask with
33 bits set instead of one; pins 32-39 are never correctly stimulated, and the
`(values & (1 << j))` loop at line 152 mis-prints the same columns in the serial
connection dump. `1ULL << i` is required in both places.

## HW-09 — Harness pins 20-39 are addressed at the wrong bit positions (CY8C9560A GPort 2 is only 4 bits)

**Location:** `firmware/CY8C9560.cpp:57-59` (`read_inputs`), used by
`firmware/firmware.ino:144-155`

`read_inputs()` does `read_registers(REG_INPUT_BASE, 8)`, packing Input Port registers
0x00-0x07 into a 64-bit word as `bit = port*8 + bit_in_port`. The firmware then assumes
harness pin *n* is at bit *n*. The schematic wiring makes that assumption false:

| Harness pins | CY8C9560A port/bits | Packed word bits | Firmware assumes |
|---|---|---|---|
| CBL_0…7 | GPort0 bits 0-7 | 0-7 | 0-7 ✔ |
| CBL_8…15 | GPort1 bits 0-7 | 8-15 | 8-15 ✔ |
| CBL_16…19 | GPort2 bits 0-3 | 16-19 | 16-19 ✔ |
| CBL_20…27 | GPort3 bits 0-7 | **24-31** | 20-27 ✘ |
| CBL_28…35 | GPort4 bits 0-7 | **32-39** | 28-35 ✘ |
| CBL_36…39 | GPort5 bits 0-3 | **40-43** | 36-39 ✘ |

GPort 2 contributes only four usable I/O on this device — the datasheet's GPIO
availability table lists GPort 2 as "0-4 bit" for the CY8C9560A, which is how the part
reaches 60 I/O — but its **input register still occupies a full byte** of the packed
word. Every harness pin from CBL_20 upwards is therefore offset by exactly 4 bit
positions.

**Failure:** two independent consequences. (a) Every comparison against
`EXPECTED_CONNECTIONS[i]` for `i >= 20` compares the wrong bits. (b) Harness pins 36-39
live at packed bits 40-43, outside the 40-bit space the firmware ever drives or reads, so
those four pins can never be tested at all.

**Evidence:** pin-to-net map extracted from the generated netlist (U4 pins 95/97/99/3/4/5/6/7
→ CBL_0..7; 8/9/10/11/16/17/18/19 → CBL_20..27; 53/52/22/23 → CBL_36..39, etc.).

**Primary source:** CY8C9560A datasheet (Infineon 38-12036 Rev. \*I), Table 1 "GPIO
Availability", and the Input Port Registers (00h-07h) map.

## HW-10 — The stimulus pin is turned back into an input before it is read

**Location:** `firmware/firmware.ino:145-148`

```c
cy.set_output(output_mask, output_mask);   // drive pin i
cy.set_pd_inputs(~output_mask);            // <-- sets Pin Direction = 0xFF on ALL ports
uint64_t values = cy.read_inputs() & ~output_mask;
```

`set_pd_inputs()` writes `0xFF` to the Pin Direction register (1Ch) for every port, and
the datasheet is explicit: *"If a bit in this register is set (written with '1'), the
corresponding port pin is enabled as an input."* That includes the pin `set_output()` just
configured as the stimulus.

**Failure:** by the time `read_inputs()` runs, no pin is driving. Every harness pin is a
pulled-down input, so the read returns all zeros and no connection can ever be detected.
The two calls must be issued in the opposite order (and see HW-11).

**Primary source:** CY8C9560A datasheet, "Port Select Register (18h) / Port Direction
Register (1Ch)".

## HW-11 — `set_output()` reconfigures an entire 8-bit port as outputs, not just the masked pin

**Location:** `firmware/CY8C9560.cpp:77-84`

```c
void CY8C9560::set_output(uint64_t pins, uint64_t values) {
  write_registers(REG_OUTPUT_BASE, values, 8);
  for (int i = 0; i < 8; i++) {
    write_register(REG_PORT_SELECT, i);
    write_register(REG_PIN_DIRECTION, 0x00);                              // whole port → output
    write_register(REG_PIN_DRIVE_MODE_BASE + DRIVE_MODE_STRONG, pins >> (i * 8));
  }
}
```

The direction byte is the literal `0x00` instead of `~(pins >> (i*8))`. The `pins` mask is
only applied to the drive-mode register.

**Failure:** all 60 expander I/O are switched to outputs simultaneously and driven with
the contents of the Output registers. During a harness test this means the seven pins
sharing a port with the stimulus pin — and every pin of every other port — are actively
driven low while the harness under test may be shorting them to a pin driven high. That
is a hard contention path through the DUT on every one of the 40 iterations. It is also
why HW-10 cannot be fixed by simply swapping the two call sites: this API cannot express
"one output, the rest inputs" at all.

**Primary source:** CY8C9560A datasheet, Port Direction Register (1Ch) semantics
(0 = output) and Drive Mode Registers (1Dh-23h), "Each '1' written to this register
changes the corresponding line drive mode."

## HW-12 — A failing result is erased by the next loop iteration before anyone can see it

**Location:** `firmware/firmware.ino:134-135` vs `:163`

```c
if (!time_fixed) return;
set_status(GOOD);        // unconditional, top of every loop() iteration
...
set_status(passed ? GOOD : FAILED);   // end of a test run
```

**Failure:** `set_status(FAILED)` is overwritten by the unconditional `set_status(GOOD)`
at the top of the very next iteration, microseconds later. The red channel is therefore
never observably on. An operator watching the board sees green after a failed harness.
Even with HW-02 fixed, the tester cannot indicate a failure.

## HW-13 — No edge detection or debounce: the test and the SD log repeat continuously

**Location:** `firmware/firmware.ino:138-163`

The button is level-tested inside `loop()` with no press/release edge tracking and no
debounce, and `log_result()` is called at the end of every pass.

**Failure:** the complete 40-pin test re-runs, and a new line is appended to
`results.txt`, on every single iteration of `loop()` for as long as the input stays in the
"run" state. Since HW-06 inverts the sense, that is *all the time the button is not
pressed*: the SD card fills with thousands of duplicated result lines per second and the
per-harness statistics the log exists to provide (README: "microSD card to store test
results along with the current time (for statistical purposes)") are meaningless.

---

# Schematic

## HW-14 — The GNSS UART is wired TX-to-TX and RX-to-RX

**Location:** nets `UBX-TXD`, `UBX-RXD`

| Net | Nodes | Pin types |
|---|---|---|
| `UBX-TXD` | U2 pin 3 (Teensy pin 1, `TX1`) + U3 pin 20 (`TXD/SPI_MISO`) | two **outputs** |
| `UBX-RXD` | U2 pin 2 (Teensy pin 0, `RX1`) + U3 pin 21 (`RXD/SPI_MOSI`) | two **inputs** |

**Failure:** there is no crossover. `UBX-RXD` has no driver at all, and `UBX-TXD` ties two
push-pull outputs together. The MCU can never receive a single NMEA character, so the
firmware is stuck at line 134 permanently; the shorted output pair additionally sources
contention current whenever the two drivers disagree. Note that KiCad's ERC does not catch
this, because the Teensy symbol declares its pins `bidirectional` rather than
input/output — which is exactly why this needs to be found by reading the netlist.

## HW-15 — I²C SDA has a 4.7 kΩ pull-**down** instead of a pull-up

**Location:** R3, nets `CY_SDA` / `CY_SCL`

| Resistor | Value | Pin 1 | Pin 2 |
|---|---|---|---|
| R2 | 4k7 | `+3.3V` | `CY_SCL` |
| R3 | 4k7 | **`GND`** | `CY_SDA` |

**Failure:** I²C is an open-drain bus; both lines require a pull-up to the bus supply. SDA
is instead pulled to ground, so the recessive state is a logic low. A START condition can
never be generated, and any device attempting to release SDA is fighting a 4.7 kΩ path to
GND against its own pull-up (there is none). All communication with the CY8C9560A fails,
which alone makes the tester non-functional.

## HW-16 — The RGB LED has no current-limiting resistors

**Location:** D3 (ASMB-KTF0-0A306), nets `LED_R`/`LED_G`/`LED_B`

D3 pin 1 (common anode) → `+3.3V`; pins 2/3/4 (B/G/R cathodes) → Teensy pins 7/6/5
directly. The schematic contains only four resistors in total — R1 (1 k, PMOS gate), R2
and R3 (4k7, I²C), R4 (10 k, button pull-up) — and none of them is in series with an LED
channel.

**Failure:** each channel is a bare LED across roughly 3.3 V minus the sink pin's
saturation voltage, current-limited only by the die and the I/O pad impedance. Broadcom
rates the ASMB-KTF0-0A306 at a **DC forward current absolute maximum of 20 mA** per
channel, and PJRC's recommended maximum output current per Teensy 4.1 pin is **4 mA**.
Both the LED and the MCU pad are driven past their ratings the first time the LED is
turned on (red is worst: lowest V<sub>f</sub>, highest resulting current).

**Primary sources:** Broadcom ASMB-KTF0-0A306 datasheet, Absolute Maximum Ratings
("DC Forward Current 20 mA"); PJRC Teensy 4.1 product page ("The recommended maximum
output current is 4mA").

## HW-17 — The MAX2679 LNA is supplied at ~3.2 V, far above its supply range

**Location:** U5 pin A1 → net `Net-(U3-VCC_RF)` → U3 pin 9 (`VCC_RF`)

The LNA's Vcc is taken from the NEO-M8N's `VCC_RF` output, which the module specifies as
**V<sub>CC</sub> − 0.1 V**. With the module on the 3.3 V rail, `VCC_RF` ≈ **3.2 V**.

**Failure:** the MAX2679/MAX2679B is a sub-2 V part — Analog Devices specifies operation
from a **+1.08 V to +1.98 V single supply**. Applying 3.2 V is roughly 1.6× the maximum
operating supply and will destroy or permanently degrade the LNA, taking the GNSS receive
path with it.

**Primary sources:** NEO-M8 Data Sheet UBX-15031086, Table 10 Operating conditions
("VCC_RF voltage … VCC−0.1 V"); Analog Devices
[MAX2679/MAX2679B](https://www.analog.com/en/products/max2679.html) supply specification.
*(Disclosure: the MAX2679 PDF could not be retrieved from the audit network — Akamai
returned "Access Denied" — so the supply range is cited from Analog Devices' published
device description rather than from a locally extracted copy of the datasheet table.)*

## HW-18 — The LNA input is unmatched and DC-coupled; the series network sits on the output instead

**Location:** AE1 → U5 pin B1; U5 pin A2 → L1 → C5 → U3 pin 11

The RF chain as drawn is:

```
AE1 (patch) ──── U5.B1 (RFIN)      no DC block, no matching network
U5.A2 (RFOUT) ── L1 12 nH ── C5 100 nF ── U3.RF_IN
```

**Failure:** two separate errors in one network. (a) The amplifier input is taken straight
off the antenna with no series DC block and no matching component, so the input is
neither impedance-matched nor DC-isolated. (b) Everything that should have been at the
input has been placed on the output, and it is not a matching network in any case — both
L1 and C5 are in **series** with no shunt element. A 12 nH series inductor presents
+j119 Ω at 1.575 GHz in a 50 Ω path, i.e. a gross mismatch and several dB of insertion
loss inserted directly after the LNA, where it degrades the noise figure of everything
downstream. Compounding this, 100 nF is far past self-resonance at L1-band; u-blox's own
recommended RF DC-block for these modules is a **47 pF** capacitor.

**Primary source:** NEO-M8 Hardware Integration Manual UBX-13003557, Table 4 recommended
components ("Murata GRM1555C1E470JZ01, C, 47 pF, DC-block").

## HW-19 — A unidirectional TVS on the unprotected side of the PMOS defeats reverse-polarity protection

**Location:** D1 (SMAJ16A), net `Net-(D1-A1)` (barrel-jack centre pin + Q1 drain) to `GND`

D1 sits directly across the raw jack input, ahead of the reverse-polarity PMOS Q1. The
part fitted is an **SMAJ16A**, which is the *unidirectional* member of the SMAJ family
(the bidirectional part is SMAJ16**C**A). The schematic symbol used models it as
bidirectional: both of D1's pins are named `A1` and `A2` — two anodes — so the drawing
carries no polarity at all, while the `Diode_SMD:D_SMA` land pattern it is assigned to is
polarised.

**Failure:** in the reverse-battery case the design is supposed to survive by having Q1
block. It cannot: the unidirectional TVS forward-conducts at well under a volt in that
direction, placing a diode straight across the reversed supply, in parallel with and
ahead of the protection MOSFET. The reverse-polarity protection the README calls for is
therefore bypassed on the first reversed connection, and the TVS itself is the sacrificial
element. (In the opposite symbol orientation — which the polarity-free symbol makes
equally likely — the same diode is a dead short across the supply in *normal* operation.)

**Primary source:** Littelfuse
[SMAJ series datasheet](https://www.littelfuse.com/assetdocs/tvs-diodes-smaj-datasheet?assetguid=13c2a823-03b8-4d1f-9ddc-9b44670aed9d);
SMAJ16A is listed as unidirectional, V<sub>RWM</sub> 16 V, V<sub>BR</sub> 17.8-19.7 V.
*(Disclosure: littelfuse.com was unreachable from the audit network; the SMAJ16A
parameters were taken from distributor datasheet listings — Farnell 2691564, TME
SMAJ16A-LF — and from the KiCad symbol's own description string, "400W unidirectional
Transient Voltage Suppressor, 16.0Vr".)*

## HW-20 — A 100 V Schottky cannot clamp the PMOS gate; V<sub>GS</sub> exceeds its ±20 V rating

**Location:** Q1 (SiSS27DN) gate, D2 (PMEG10020ELR), R1 (1 kΩ)

The gate network is R1 pulling the gate hard to GND, with D2 in the gate-clamp position
(cathode on the source / `+12V` node, anode on the gate). The gate is therefore always at
Vgs = −V<sub>IN</sub>.

**Failure:** the component in the clamp position is a **100 V Schottky rectifier**, not a
Zener. Reverse-biased at −12 V it simply blocks, and there is no voltage at which it
begins to conduct before the MOSFET is destroyed — it can only ever limit *positive*
V<sub>GS</sub>, which never occurs here. The SiSS27DN's gate-source absolute maximum is
**±20 V**. During any transient large enough to put D1 into conduction the input node is
held at up to the SMAJ16A's **26 V maximum clamping voltage**, so V<sub>GS</sub> reaches
−26 V — 30 % past the gate-oxide rating — on exactly the events the protection circuit
exists to survive.

**Primary sources:** Vishay SiSS27DN datasheet, Absolute Maximum Ratings
("Gate-Source Voltage V<sub>GS</sub> ± 20 V"); Nexperia PMEG10020ELR datasheet (100 V,
2 A Schottky barrier rectifier); Littelfuse SMAJ16A V<sub>C</sub> = 26 V at
I<sub>PP</sub> = 15.4 A (see HW-19 disclosure).

## HW-21 — Teensy VIN is externally powered while USB is the only debug interface, with VUSB/VIN unseparated

**Location:** U2 pin 48 (`VIN`) ← `+5V` from U1; U2 pin 49 (`VUSB`) marked no-connect;
`firmware/firmware.ino:14` `#define DBG_SERIAL Serial`

The board feeds the Teensy 4.1 from the L7805 through `VIN`, and the firmware's entire
diagnostic output — the per-pin connection matrix dump that makes the tester usable — goes
to USB `Serial`. USB will therefore be connected while the board is externally powered,
every time the tester is used for diagnosis and every time it is programmed.

**Failure:** on an unmodified Teensy 4.1 `VUSB` and `VIN` are joined on the board. PJRC's
instruction is explicit: *"a pair of pads on the bottom side may be cut apart, to separate
VUSB from VIN, allowing power to be safely applied while USB is in use."* Nothing in the
schematic, the BOM or the board documents that modification, so as designed the host's
USB 5 V rail is tied directly to the L7805 output whenever the debug interface is used.

**Primary source:** PJRC Teensy 4.1 product page, "Power" section.

---

# PCB layout

## HW-22 — The RGB LED's common anode sits on an isolated +3.3 V copper island

**Location:** D3 pin 1 at (159, 33.5); F.Cu stub (158.1, 32.9) → via (157.226, 32.9184)

KiCad's own DRC reports the net as not fully connected after a zone refill:

```
Missing connection between items
  Track [+3.3V] on F.Cu, length 0.8556 mm  @ (158.1, 32.9)      <- D3 anode stub
  Track [+3.3V] on F.Cu, length 0.6845 mm  @ (167.320, 35.775)  <- R4 stub
```

That is the only unconnected item on the board, and it is reported identically whether the
committed zone fills are used or the zones are refilled from scratch
(`--refill-zones`). Testing the filled zone geometry directly
confirms the mechanism: the `+3.3V` pour on In1.Cu refills into **14 separate islands**,
and the D3 anode via lands on a 32-vertex fragment while every other +3.3 V consumer —
R4's via, the Teensy 3V3 pad at (156.34, 62.62), C3, U3, U4/C4 — lands on the single large
island (12 778 vertices). The five +3.3 V vias are all F.Cu↔B.Cu, and both of those layers
carry the GND pour, so the In1.Cu fragment is the only thing that could have made the
connection.

**Failure:** the common anode of the RGB LED has no supply. Independently of HW-02 and
HW-16, the board's only status indicator is dead on arrival.

**Reproduce:** `kicad-cli pcb drc --format json` → `unconnected_items`; or point-in-polygon
test of the In1.Cu `+3.3V` `filled_polygon` sets against (157.226, 32.9184) versus
(167.3352, 36.4744).

## HW-23 — An all-layer copper keepout removes the ground plane under the patch antenna and under its feed

**Location:** keepout zone (119.5, 64.5) → (144.5, 89.5); AE1 at (132, 77)

The board carries a rule-area zone with `copperpour not_allowed` on **F.Cu, In1.Cu,
In2.Cu and B.Cu simultaneously** — every copper layer — occupying a 25 × 25 mm square that
matches the ANT-GNSSCP-TH25L1 patch outline exactly. Point-in-polygon tests against the
refilled `GND` zone confirm that none of its F.Cu, In2.Cu or B.Cu fills covers the antenna
centre (132, 77), the feed pad (134.5, 77), or a mid-feed point (140, 72), while a
reference point outside the keepout (150, 60) is covered on all three.

**Failure:** a passive ceramic patch antenna radiates against its ground plane — the plane
*is* the other half of the antenna. Removing it on every layer under the element leaves
the patch grossly detuned off L1 with no defined pattern or gain. The same keepout also
strips the reference plane out from under the feed. The RF path from the antenna pin is an
arc from (134.5, 77) to (142.869, 68.631) — an 11.8 mm chord, entirely inside the keepout —
then a straight 3.69 mm run to (146.558, 68.631) whose first 1.6 mm is still inside it,
then a 0.127 mm-wide stub into U5 pad B1 at (147.998, 68.644). About 13 of the roughly
17 mm of feed therefore crosses bare laminate with no ground on any layer beneath it: no
50 Ω controlled impedance and no continuous return path between antenna and LNA. The trace
additionally steps abruptly from 0.8128 mm to 0.127 mm at the LNA pad.

*Scope note:* this is deliberately reported as **one** defect, not two. The dead plane and
the unreferenced feed are two consequences of the same keepout polygon and are fixed by the
same change.

## HW-24 — The CY8C9560A land pattern is the wrong package

**Location:** U4 footprint `Package_QFP:TQFP-100_12x12mm_P0.4mm`

Measured directly from the board file, U4's land pattern is: 100 pads, 25 per side,
**0.400 mm pitch**, pad-centre span 13.400 mm, F.Fab body outline exactly **12.0 × 12.0 mm**.
The footprint's own description string confirms it: *"100-Lead Plastic Thin Quad Flatpack
(PT) - 12x12x1 mm Body … [TQFP]"*.

The part specified is `CY8C9560A-24AXIT`, whose datasheet package outline is
**"100-pin TQFP (14 × 14 × 1.0 mm)"** (Figure 12, Infineon/Cypress package spec 51-85048).

**Failure:** a 14 × 14 mm body cannot be placed on a 12 × 12 mm land pattern — the lead
frame overhangs the pad ring by ~1 mm per side and the lead pitch does not match the pad
pitch, so the leads do not register with the pads at all. The board cannot be assembled
with the specified device.

**Primary source:** CY8C9560A datasheet (Infineon 38-12036 Rev. \*I), Figure 12,
"100-pin TQFP (14 × 14 × 1.0 mm) Package Outline".

---

# Not counted

Candidates that were investigated and deliberately **rejected**, with the reason. These
are listed so the reasoning can be checked, not as additional submissions.

**Investigated and found to be non-bugs**

- *RMC navigation-status field `V` is ignored.* The `%*c` does skip the A/V flag, but the
  claim does not survive: in a no-fix `$G_RMC` the latitude/longitude/speed fields are
  empty, the first `%*f` fails to convert, `sscanf` returns 1 not 2, and the sentence is
  rejected anyway. Not an independently reachable failure.
- *Date/time freeze after the first fix.* `sscanf` is evaluated before the `&& !time_fixed`
  short-circuit, so `utc_time` and `date` are in fact rewritten on every accepted sentence.
  The claim is simply false.
- *Connection-matrix defect.* All 40 rows are present and 40 bits wide; parsed as C
  compiles them (MSB-first) the matrix is perfectly symmetric with zero diagonal
  self-connections, which is consistent with `values & ~output_mask` masking the driven
  pin. Verified programmatically. No defect.
- *CY8C9560A reset pulse polarity.* XRES is *"Active high external reset with internal pull
  down"*, so `begin()`'s HIGH→delay→LOW sequence is correct and an undriven pin is not held
  in reset.
- *CY8C9560A I²C address `0x20` and device-ID check `== 0x06`.* The datasheet gives the
  default multi-port address format as `010000A0X` with A0 (pin 30) grounded → 0x20, and
  the Device ID/Status register default as `60h` for the CY8C9560A → high nibble 6. Both
  correct.
- *`Wire2` selection.* Teensy 4.1 SCL2/SDA2 are pins 24/25; the netlist connects U2 pin 16
  (`24_A10_TX6_SCL2`) → `CY_SCL` and pin 17 (`25_A11_RX6_SDA2`) → `CY_SDA`. Correct.
- *UART baud rate.* NEO-M8 default is 9600 8N1, matching `UBX_SERIAL.begin(9600)`.
- *`VDD_USB` tied to GND / `V_BCKP` tied to `+3.3V` / `D_SEL` and `EXTINT` left open.*
  All match u-blox guidance for a UART-only design with no USB and no backup battery.
- *SW1 schematic-parity warning ("no pad found for pins 3 and 4").* The `SW_Push_Dual`
  symbol has two parallel contacts (1↔2 and 3↔4) wired identically between `GND` and
  `BTN_TEST`; the PTS645 land pattern has two pads. The switch still closes `BTN_TEST` to
  `GND`. Cosmetic only.
- *PMOS source/drain orientation.* Drain on the jack side, source on the load side puts the
  body diode in the conducting direction for normal current and lets R1 pull V<sub>GS</sub>
  to −V<sub>IN</sub>. Correct.
- *NEO-M8N `RESET_N` never driven (HW-02's missing `pinMode`).* Harmless in isolation: the
  module has an internal 11 kΩ pull-up on RESET_N, so an undriven pin is de-asserted. Folded
  into HW-02 rather than counted separately.
- *Built-in SD-card API usage (`BUILTIN_SDCARD`).* Correct for Teensy 4.1.
- *C6/U5 courtyard overlap (DRC).* Courtyards overlap but the component bodies clear each
  other by ~0.56 mm. Assembly-rule warning, not a fault.

**Real but not show-stopping, so excluded from the count**

- The U4 symbol labels pin 62 `RESET_N` when the device pin is the active-**high** `XRES`.
  A misleading label; the firmware nonetheless drives it with the correct polarity.
- Total 3.3 V decoupling is two 100 nF parts (C3, C4) for the Teensy, NEO-M8N and a 100-pin
  I/O expander, with no bulk capacitor on the rail. Poor practice; no deterministic failure
  could be established without assembly-specific numbers.
- The 3.3 V rail is generated by the Teensy 4.1's on-board regulator (PJRC: 250 mA
  recommended maximum for external use). The aggregate steady-state load is under that
  budget except when HW-16's unlimited LED current is added, so it is a consequence of
  HW-16 rather than an independent defect.
- The refilled board reports 32 isolated GND fill islands and 22 isolated +3.3 V fill
  islands. Only the one in HW-22 leaves a pin without a path to its net; the rest are
  cosmetic pour fragments.

**DRC noise explicitly not used as evidence**

199 `track_width` and 37 `clearance` violations arise from a 0.2 mm project rule against
0.127 mm routing and ~0.0889 mm actual spacing (mostly the CY_RST_N / CY_INT pair on
B.Cu); four of the clearance items are simply the MAX2679's own WLP-4 0.4 mm-pitch pad
spacing (0.150 mm) measured against the same rule. 83 `silk_edge_clearance`, 25
`lib_footprint_issues`, 2 `copper_edge_clearance` and the text-size items are project-rule
or library bookkeeping. None is a functional fault against a stated manufacturer or
fabricator limit, so none is claimed.

---

# Sources

| Component / topic | Document |
|---|---|
| CY8C9560A I/O expander | [Infineon 38-12036 Rev. \*I datasheet](https://www.infineon.com/dgdl/Infineon-CY8C9520A_CY8C9540A_CY8C9560A_20-_40-_and_60-Bit_I_O_Expander_with_EEPROM-DataSheet-v10_00-EN.pdf?fileId=8ac78c8c7d0d8da4017d0ebd16ae2f29) |
| NEO-M8 module | [Data sheet UBX-15031086](https://content.u-blox.com/sites/default/files/NEO-M8-FW3_DataSheet_UBX-15031086.pdf) |
| NEO-M8 integration | [Hardware Integration Manual UBX-13003557](https://content.u-blox.com/sites/default/files/NEO-M8_HardwareIntegrationManual_%28UBX-13003557%29.pdf) |
| NMEA / talker IDs | [u-blox 8 / M8 Receiver Description UBX-13003221](https://content.u-blox.com/sites/default/files/products/documents/u-blox8-M8_ReceiverDescrProtSpec_UBX-13003221.pdf) |
| Teensy 4.1 | [PJRC Teensy 4.1](https://www.pjrc.com/store/teensy41.html) · [UART](https://www.pjrc.com/teensy/td_uart.html) · [Wire](https://www.pjrc.com/teensy/td_libs_Wire.html) |
| MAX2679 LNA | [Analog Devices MAX2679](https://www.analog.com/en/products/max2679.html) |
| GNSS patch antenna | [TE ANT-GNSSCP-TH25L1](https://www.te.com/en/product-ANT-GNSSCP-TH25L1.html) |
| RGB LED | [Broadcom ASMB-KTF0-0A306](https://docs.broadcom.com/doc/ASMB-KTF0-0A306-DS100) |
| TVS diode | [Littelfuse SMAJ series](https://www.littelfuse.com/assetdocs/tvs-diodes-smaj-datasheet?assetguid=13c2a823-03b8-4d1f-9ddc-9b44670aed9d) |
| Schottky | [Nexperia PMEG10020ELR](https://assets.nexperia.com/documents/data-sheet/PMEG10020ELR.pdf) |
| P-channel MOSFET | [Vishay SiSS27DN](https://www.vishay.com/docs/62847/siss27dn.pdf) |

Two documents could not be retrieved from the audit network and are cited from the
manufacturer's published device description and from distributor datasheet listings
instead; each is disclosed inline at the finding that relies on it (HW-17, HW-19/HW-20).
