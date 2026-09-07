# ZMK board definition — Skree Pawpaw

nRF52840 (MS88SF21) keyboard controller.

**Status: bring-up / hardware validation.** Matrix, shift registers, SK6812
underglow and the LED-power FET are all configured. Pointing devices are not.

## Building

This repo is a **ZMK user config** so GitHub Actions builds it directly — push
it and the Actions tab produces `pawpaw-zmk.uf2`. Drag that onto the UF2 drive.

The board lives in `boards/arm/pawpaw/`, found via `zephyr/module.yml`
(`board_root: .`). Pinned to **ZMK v0.3.0** in `config/west.yml`.

This layout deliberately mirrors the known-working
[ZMK-XIAOv4-SR](https://github.com/WainingForests/ZMK-XIAOv4-SR) config rather
than the `config/boards/` variant, because that one is proven to build.

To build locally instead: `west build -b pawpaw`.

### Later: split into a distributable module

The structure is already module-shaped, so shipping to customers is mostly
just moving this to its own repo and having users add it to their `west.yml`
alongside a keyboard shield built against `-b pawpaw`.

## Bring-up keymap

Sized for a 6x6 test sheet (R1–R6 × C1–C6) plus a 6-key thumb row on R7.

**Layer 0** — every wired position sends a distinct key, so a key-tester
website tells you exactly which position fired:

```
       C1  C2  C3  C4  C5  C6  | C7  C8 (unwired)
  R1    a   b   c   d   e   f  | F1  F2
  R2    g   h   i   j   k   l  | F3  F4
  R3    m   n   o   p   q   r  | F5  F6
  R4    s   t   u   v   w   x  | F7  F8
  R5    y   z   1   2   3   4  | F9  F10
  R6    5   6   7   8   9   0  | F11 F12
  R7   MO1 F13 F14 F15 F16 F17 | F18 F19
```

R7C1 is a layer hold so it types nothing — prove it works by holding it and
confirming the layer-1 keys respond.

**Layer 1** (hold R7C1) — lighting and system:

```
       C1       C2        C3        C4       C5       C6
  R1  RGB_TOG  RGB_EFF   RGB_EFR   RGB_HUI  RGB_HUD  RGB_ON
  R2  RGB_BRI  RGB_BRD   RGB_SAI   RGB_SAD  RGB_SPI  RGB_SPD
  R3  EP_ON    EP_OFF    EP_TOG     -        -        -
  R4  BOOTLDR  RESET      -         -        -        -
  R5  BT_CLR   BT_SEL0   BT_SEL1   BT_SEL2  BT_SEL3  BT_NXT
```

**Combo** — R7C5 + R7C6 together toggles external power (the SK6812 FET on
shift register index 9). A combo so you cannot trip it while walking the
matrix.

Underglow starts **on** at 50% brightness. Because ZMK ties underglow to
ext-power by default, `RGB_TOG` already drives the FET; the explicit `EP_*`
keys and the combo exist to test the FET independently.

### Set `chain-length` before you build

`pawpaw.dts` has `chain-length = <42>` (36 + 6). **Set it to your actual LED
count.** Too few and the tail of the chain stays dark; too many and effects
spread across positions that do not exist.

## Hardware notes that matter for firmware

**`CONFIG_NFCT_PINS_AS_GPIOS=y` is mandatory.** P0.09 (matrix row R2) and
P0.10 (display CS) are NFC1/NFC2 by default on the nRF52840 and will not
work as GPIO without it. This is set in `pawpaw_defconfig`.

**Shift register chain.** Two 74HC595. U2 is first (DS from MOSI), U7 second
(DS from U2's Q7S). Both share SCK and the SR_CS latch. ZMK's `&shifter`
index equals the chain stage:

| Index | Register | Q | Net |
|-------|----------|---|-----|
| 0–7   | U2 | Q0–Q7 | C8, C7, C1, C2, C3, C4, C5, C6 |
| 8     | U7 | Q0 | unconnected |
| **9** | U7 | Q1 | **`3v3en`** — SK6812 power FET |
| 10–15 | U7 | Q2–Q7 | A1–A6 (spare) |

Columns are **not** in numeric order — C8 and C7 come before C1.

**`OE#` is tied to GND and `MR#` to 3V3 on both registers**, so the outputs
are permanently enabled and cannot be tri-stated. This matters for test rigs:
a second Pawpaw cannot safely be connected column-to-column, because that is
595-against-595 push-pull contention. Use a tester with direct MCU GPIOs.

**SPI is shared** between the shift registers (CS = SR_CS, P1.13) and the
display connector (CS = SC-CS, P0.10). Display traffic shifts junk through
the 595s but never pulses their latch, and the next 595 write shifts 16 fresh
bits in, so it self-corrects.

**High-voltage mode.** VDDH is fed from VBUS/battery and VDD comes from the
nRF52840's internal REG0. `UICR.REGOUT0` must be 3.3V — the Adafruit
bootloader sets this on first boot.

## Not done yet

- **Pointing devices.** Connector is P0.08 / P1.09 / P0.12 (high speed),
  P0.15 (CS), P0.13 (IRQ). Drivers generally live in external modules that
  track ZMK `main`, so expect to bump the pinned revision.
- **Encoder.** ADC1 (P0.02) / ADC2 (P0.29) can host an EC11 as a quadrature
  pair — the quickest way to prove both pins without a joystick in hand.

## Known risk

ZMK's ext-power initialises early, and driving `&shifter 9` requires an SPI
transaction. See [zmkfirmware/zmk#2945](https://github.com/zmkfirmware/zmk/issues/2945)
for `-EAGAIN` errors when setting 595 outputs. If ext-power fails to come up,
that is the first thing to look at — not the hardware.
