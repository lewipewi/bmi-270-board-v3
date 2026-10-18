# BMI270 IMU Node — Board v3

Wearable 6-axis motion node for an IMU gesture controller. Each node samples a
Bosch BMI270 at 200 Hz and streams it over ESP-NOW to a USB receiver on a PC,
where a classifier turns boxing motions into gamepad inputs (Tekken 7 / Steve Fox).
Three identical nodes are worn: left wrist, right wrist, chest.

v3 is a full redesign of v2. v2 was built, debugged through five faults, and
finally found to be **unflashable** (see [What went wrong on v2](#what-went-wrong-on-v2)).
v3's priorities, in order: **(1) flashable, (2) debuggable, (3) charging,
(4) everything else.**

**Status:** layout complete, DRC clean (documented exclusions only), Gerbers
generated and checked, ordered from JLCPCB. Not yet assembled.

---

## At a glance

| | |
|---|---|
| MCU | ESP32-C3-MINI-1-N4 (RISC-V, 4 MB flash, WiFi/BLE, antenna overhangs board edge) |
| IMU | Bosch BMI270, I2C @ 0x68 (CHIP_ID 0x24) |
| Power in | USB-C (5 V) or 1-cell LiPo via JST-PH 2-pin |
| Charging | TP4056, ~255 mA, CHRG status to a GPIO |
| Power path | Schottky + P-FET load sharing (runs from USB when plugged in) |
| Regulator | AP2112K-3.3 LDO, enabled by slide switch |
| Programming | **Native USB** (USB-Serial-JTAG) — no buttons needed; UART header fallback |
| Board | 40.0 × 44.0 × 1.6 mm, **4 layers**, 2 mm corner radii, 2 × M2 holes |
| Assembly | Hand soldered (iron + hot air): 0603 hand-solder passives minimum |

---

## Block diagram / power tree

```
USB-C 5V (VBUS) ── USBLC6 ESD on D+/D−, 5.1k on CC1 and CC2
 ├─► TP4056 charger (R_PROG 4.7k ≈ 255 mA) ──► VBAT ◄── JST-PH battery
 │                                              │
 │                                              └─► Q1 AO3401 P-FET (ON only when USB absent)
 │                                                      │
 └─► D2 MBR0520 Schottky (5 V → ~4.7 V) ───────────────┴──► VSYS (22 µF)
                                                              │
                                        AP2112K-3.3 LDO  (EN ← slide switch, 1M pull-down)
                                                              │
                                                            3V3 ──► ESP32-C3, BMI270,
                                                                    pull-ups, power LED
VBAT ── 1M/1M divider + 100 nF ──► IO0 (battery voltage)
TP4056 CHRG ── 10k pull-up ──► IO1 (charge status)
```

- **USB present:** Q1's gate = VBUS (5 V) > source (VSYS ≈ 4.7 V) → Q1 off; board
  runs from USB through D2; the battery sees only the charger, so charge
  termination works correctly.
- **USB absent:** R5 (100k) pulls Q1's gate to 0 V → Q1 fully on (~50 mΩ); battery
  feeds VSYS. Q1's body diode catches VSYS during the hand-over.
- **Slide switch** only drives the LDO's enable pin — it never carries load current.
  Charging works with the switch OFF; running or flashing needs it ON.
- LDO drops out around a **3.4–3.5 V** battery — firmware should warn/sleep there.

---

## Pin map

| GPIO | Function | Notes |
|---|---|---|
| IO0 | VBAT_SENSE | ADC1_CH0, reads VBAT ÷ 2 |
| IO1 | CHRG | Active low = charging; high = done **or** no USB |
| IO2 | — | Strapping pin, 10k pull-up only |
| IO3 | LED2 | 330 Ω, active high |
| IO4 | BMI270 INT1 | |
| IO5 | I2C SDA | 4.7k pull-up |
| IO6 | I2C SCL | 4.7k pull-up |
| IO7 | BMI270 INT2 | |
| IO8 | — | Strapping pin, **10k pull-up (the v2 fix)** |
| IO9 | BOOT button | Strapping pin, 10k pull-up; usable as a user button after boot |
| IO10 | LED1 | 330 Ω, active high |
| IO18 / IO19 | USB D− / D+ | Native USB |
| IO20 / IO21 | UART0 RX / TX | 1×4 debug header |
| EN | RESET button | 10k + 1 µF |

Power LED (LED3) is hard-wired to 3V3 through 1k.

**Boot straps:** power-up → IO2 = 1, IO8 = 1, IO9 = 1 → boots from flash.
BOOT held → IO9 = 0, IO8 = 1 → download mode. Nothing else can pull a strapping
pin low.

---

## Changes from v2

### Fixes for v2's failures

| v2 | v3 | Why |
|---|---|---|
| **IO8 unconnected** — floated low, so BOOT entered USB_BOOT instead of download mode; board could not be flashed | **10k pull-up on IO8** (and IO2) | Espressif requires IO8 high for the serial bootloader |
| USB D+/D− (IO18/IO19) wired to **two LEDs** | **USB-C D+/D− → IO19/IO18** (native USB); LEDs moved to IO10/IO3 | Flash + console + JTAG over USB with no buttons; deletes the whole CH343P path where most v2 faults lived |
| CH343P USB-serial bridge | **Removed** | Native USB replaces it; 1×4 UART header kept as fallback |
| Rail problems hard to diagnose (D2 fault hid behind a working CH343P) | **12 test points** + power LED | Every measurement that was hard on v2 is now a probe touch |
| CC cold joint made the board "dead" one way up, unprobeable | **CC1 and CC2 test points**, separate 5.1k each | Verify both CC lines directly |
| UART pads under the module, unprobeable | UART on a through-hole header + test points | |
| One pin of each tactile switch pair used | **Both pins of each pair connected** | A single cold joint is no longer fatal |
| EN RC: 10k + 100 nF | **10k + 1 µF** | Espressif C3 hardware guideline |

### New features

| Feature | Details |
|---|---|
| Battery charging | TP4056, 4.7k PROG (~255 mA = 0.5C for ~500 mAh), TEMP tied to GND, CHRG → IO1 |
| Load sharing | D2 Schottky from VBUS + AO3401 P-FET from battery (drain = VBAT, source = VSYS, gate = VBUS with 100k pull-down) |
| Slide switch moved | v2: selected VBAT **or** VBUS as LDO input (in the power path). v3: on the LDO **enable** pin only, with 1M pull-down for mid-slide |
| Battery sense | 1M/1M on **VBAT** (1.9 µA) — v2 used 100k/100k (19 µA) on the switched rail |
| BMI270 INT2 | Connected (IO7) — separates FIFO watermark from data-ready |
| BMI270 unused pins | ASDx/ASCx → VDDIO, OCSB/OSDO no-connect, per Bosch datasheet Table 22. (v2 left all four floating; an earlier plan to ground all four would have violated Table 22.) |
| Power LED | On 3V3 — instant "is the rail up?" |
| Battery protection | Relies on the pack's own protection board (DW01A/FS8205 not on the PCB) |

### Pin map, v2 → v3

| Signal | v2 | v3 |
|---|---|---|
| SDA / SCL | IO5 / IO6 | IO5 / IO6 |
| INT1 | IO4 | IO4 |
| INT2 | not connected | IO7 |
| VBAT sense | IO0 (unconfirmed) | IO0 |
| LEDs | IO19 / IO18 (USB pins!) | IO10 / IO3 |
| USB D+ / D− | LEDs | IO19 / IO18 |
| CHRG | — | IO1 |
| IO8 | **floating** | 10k pull-up |
| IO2 | free | 10k pull-up |
| UART0 | → CH343P | → 1×4 header |

### Layout changes

| | v2 | v3 |
|---|---|---|
| Layers | 2 | **4**: L1 signals + GND pour / L2 solid GND / L3 +3V3 plane with VBUS island / L4 signals + GND pour |
| Stitching | — | 121 GND vias, ground pours tied down across the board |
| USB pair | — | 0.3 mm / 0.21 mm gap, ≈ 90 Ω differential, 27.8 / 27.4 mm, top layer only over solid GND, no vias |
| Module grounding | — | Via in each of the 9 centre GND pads |
| TP4056 | — | 6 thermal vias in the exposed pad |
| Footprints | — | Hand-solder variants; 10 µF in 0805, 22 µF in 1206 for DC-bias derating |
| Mechanical | — | IMU at board centre, away from charger/LDO/holes/edges; antenna overhang, plastic-only keep-out |

---

## Bill of materials (key parts)

| Ref | Part | Notes |
|---|---|---|
| U4 | ESP32-C3-MINI-1-N4 | |
| U5 | BMI270 | LGA-14 — hardest part to hand solder |
| U2 | TP4056 (ESOP-8) | |
| U1 | AP2112K-3.3 (Tech Public) | Clone — check dropout |
| Q1 | AO3401 P-FET | Verify P-channel with a meter before fitting |
| D2 | MBR0520LT1G | |
| D1 | USBLC6-2SC6 | |
| USB1 | USB-C 16-pin (TYPE-C 16PIN 2MD(073)) | Locating pegs set to NPTH |
| J1 | JST-PH 2-pin, vertical (B2B-PH-K-S) | Check pin 1 (+) against the battery lead |
| J2 | 1×4 2.54 mm right-angle header | 3V3 / GND / TX / RX (3V3 is a reference, not an input) |
| SW1, SW2 | TS-1187A tactile | RESET, BOOT |
| SW3 | SS-12D00-G3 slide switch | Power (LDO enable). No 3D model |

Resistors 0603: R1/R2 5.1k (CC), R3 10k (CHRG pull-up), R4 4.7k (PROG), R5 100k
(Q1 gate), R6/R7 1M (VBAT divider), R8 10k (EN), R9/R10/R11 10k (IO2/IO9/IO8),
R12/R13 330 (LEDs), R14 1M (LDO EN pull-down), R15 1k (power LED), R16/R17 4.7k (I2C).
Capacitors: C1 10 µF (VBUS), C2 10 µF (VBAT), C3 22 µF (VSYS), C4 10 µF (LDO out),
C5 1 µF (LDO in), C6 100 nF (VBAT sense), C7 10 µF + C8 100 nF (module),
C9 1 µF (EN), C10/C11 100 nF (BMI270 VDD/VDDIO). Use ≥ 10 V rating on 10/22 µF.

---

## Fabrication

- JLCPCB, 4 layers, 1.6 mm, stackup **JLC04161H-7628**, HASL (with lead), tented vias.
- Gerbers: KiCad 9, Protel extensions, 4.6 mm, X2 off, subtract mask from silk.
- Drill: Excellon, mm, decimal, PTH/NPTH separate, oval holes in route mode
  (the four USB-C shell tabs are slots).
- Documented DRC exclusions: USB-C peg-hole clearance (footprint geometry, NPTH),
  module pads 11 and 51 single thermal spoke (module GND tied internally).

---

## Bring-up checklist

Populate and test in stages so faults show up one at a time.

1. **Bare board:** no short from 3V3, VBUS, VSYS, VBAT to GND.
2. **USB-C, ESD, CC resistors, D2:** VBUS = 5 V **with the cable both ways up**;
   VSYS ≈ 4.7 V.
3. **LDO, its caps, slide switch, power LED:** switch ON → 3V3 = 3.3 V, LED lit.
4. **Module + boot parts:** appears as a COM port with no buttons pressed; flash blink.
   PlatformIO needs `-DARDUINO_USB_MODE=1 -DARDUINO_USB_CDC_ON_BOOT=1` or `Serial`
   goes to the UART header.
5. **BMI270 + pull-ups:** I2C scan finds 0x68, CHIP_ID reads 0x24. If not, probe
   SDA/SCL test points.
6. **Battery + charger:** check battery polarity with a meter first; CHRG low while
   charging; board keeps running when USB is unplugged.

If native USB stops working: hold BOOT, tap RESET, release BOOT, upload.
Last resort: UART header with a spare ESP32 dev board as the adapter.

---

## Known limitations

- **CHRG can't distinguish "charge done" from "no USB"** — only one status pin;
  combine with the VBAT reading.
- **No reverse-polarity protection on the battery.** Check every new cell's
  polarity before plugging in.
- VBUS sees ~33 µF at plug-in (USB 2.0 limit is 10 µF) — common on dev boards,
  fine in practice.
- USB 500 mA budget is slightly exceeded with charging + WiFi TX peaks — fine on a
  USB-C laptop port.
- The BMI270 is rotated 180° on the board; define the body frame in firmware.
- STEP export: the ESP32 module and USB-C 3D models are offset from their real
  (footprint) positions by ~2.7 mm and ~1.2 mm. Use footprint dimensions for
  enclosure design.

---

## What went wrong on v2

Kept here because it explains most of v3's decisions.

1. **Pre-power checks passed** (VBAT–GND = 196 kΩ confirmed the sense divider).
2. **USB-C CC cold joint** — board dead one way up; fixed by flipping the cable.
   Lesson: a C-to-C source gives 0 V until CC negotiation succeeds, and the CC pins
   can't be probed under the connector.
3. **D2 not conducting** (5.31 V in, 0.63 V out) — reworked. Masked by the CH343P
   enumerating fine from upstream of D2.
4. **No UART output** — cold joints on the module castellations; resoldered.
   Boot log then showed a correct blank-chip loop (`invalid header: 0xffffffff`).
5. **Could not enter download mode** — holding BOOT gave `boot:0x0 (USB_BOOT)`
   because IO8 floated low, and USB went to LEDs. v2 abandoned.
