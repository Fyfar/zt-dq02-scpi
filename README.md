# ZT-DQ02 SCPI Reference

> Complete SCPI command reference for the **ZOYI ZT-DQ02 LCR meter** — extracted from firmware static analysis and **verified by live device testing** over USB.

This document lets you control the ZT-DQ02 programmatically from a PC, automate component sorting, and integrate the meter into test fixtures — no screen needed.

---

## Table of Contents

- [Hardware & Connection](#hardware--connection)
- [Quick Start](#quick-start)
- [Command Model](#command-model)
- [Timing & Programming Notes](#timing--programming-notes)
- [FETCh? — Reading All State](#fetch--reading-all-state)
- [Measurement Commands](#measurement-commands)
  - [FREQuency](#frequency-)
  - [APERture](#aperture-)
  - [VOLTage](#voltage-)
  - [BIAS:VOLTage](#biasvoltage-)
- [FUNCtion:IMPedance Commands](#functionimpedance-commands)
  - [MAIN mode](#functionimpedancemain-)
  - [SUB parameter](#functionimpedancesub-)
  - [Circuit Model](#functionimpedancemodel-)
  - [Range](#functionimpedancerange-)
- [COMParator — Pass/Fail Testing](#comparator--passfail-testing)
  - [Automated Workflow](#automated-passfail-workflow)
  - [BAT Mode Limits](#bat-mode-tolerance)
- [IEEE 488.2 Common Commands](#ieee-4882-common-commands-)
- [SYSTem Commands](#system-commands)
- [STATus Commands](#status-commands-)
- [Unused Library Commands](#commands-present-but-unused)
- [Firmware Analysis Notes](#firmware-analysis-notes)

---

## Hardware & Connection

The ZT-DQ02 presents as a **USB CDC Virtual COM Port** — no custom driver needed on Linux or macOS. On Windows, the standard USB serial driver or a VCP driver (e.g. from STMicroelectronics) may be required.

| Setting | Value |
|---------|-------|
| Baud rate | **9600** |
| Data bits / parity / stop | 8N1 |
| Command terminator | `\n` (LF) |
| Response terminator | `\r\n` (CRLF) |

Connect the device via USB, identify the COM port (`/dev/ttyACM0` on Linux, `/dev/tty.usbmodem*` on macOS, `COMx` on Windows), and open a serial terminal or use any SCPI-capable library.

---

## Quick Start

**Verify connection:**
```
*IDN?
→ ZOYI,ZT-DQ02,<serial>,V1.12
```

**Read current measurement and settings:**
```
FETCh?
→ 987.3R,-0.12X,R,R,X,SER,SER,MED,AUTO,1000,300,0,0,R,1.0000e3R,5.0,1
```

**Automated resistor sorting (±5% around 1 kΩ):**
```
FUNCtion:IMPedance:MAIN    # cycle to R mode if not already there
COMParator:NOMinal 1000    # 1000 Ω reference
COMParator:ERRor 5         # ±5% tolerance
FETCh?                     # field [16] = 1 (PASS) or 0 (FAIL)
```

**Python example using pyserial:**
```python
import serial, time

lcr = serial.Serial('/dev/ttyACM0', 9600, timeout=1)

def cmd(s):
    lcr.write((s + '\n').encode())
    time.sleep(0.05)
    return lcr.readline().decode().strip()

print(cmd('*IDN?'))
print(cmd('FETCh?'))
```

---

## Command Model

**All instrument-specific commands are no-parameter cycle commands.** Each call advances the setting to the next fixed value. There is no way to set a value directly — only cycle through the available options.

**Individual query commands (`FREQuency?`, `APERture?`, etc.) return empty strings.** `FETCh?` is the sole mechanism to read device state.

---

## Timing & Programming Notes

All verified live:

- **`FETCh?` is fast but cached.** It returns the last completed measurement in ~1 ms; it does not wait for a fresh conversion. The actual measurement refreshes about every **0.36 s** in MED/FAST and irregularly (0.3–4 s) in SLOW. **MED and FAST refresh at the same rate** on this hardware — FAST is not faster. To capture a genuinely new reading, poll `FETCh?` until field [0] changes, or wait ≥ 0.4 s between samples.
- **Compound (semicolon) commands work.** `FREQuency;FETCh?` executes both in order and returns the FETCh? value on one line — handy to cycle a setting and read back atomically.
- **Many standard SCPI subsystems are not implemented.** `MEASure?`, `INITiate`, `TRIGger`, `CALCulate`, `DISPlay`, `SENSe`, `OUTPut`, `INPut` all return `-110 Command header error`. There is no trigger model — `FETCh?` is the only read path.
- **Numeric inputs reject unit suffixes.** Values must be bare floats; `47uF`, `0.047m`, etc. raise `-120 Numeric data error`.

---

## FETCh? — Reading All State ✓

Returns the most recent measurement plus all current settings as **17 comma-separated fields**.

**Firmware format string (offset 0x1e5cc):**
```
%s%s,%s%s,%s,%s,%s,%s,%s,%d,%0.0f,%0.0f,%d,%s,%s%s,%0.1f,%d
```

| Field | Format | Content | Example values |
|-------|--------|---------|----------------|
| [0] | `%s%s` | Primary value + unit | `1.2345e-9F`, `987.3R` |
| [1] | `%s%s` | Secondary value + unit | `-1.28X`, `0.001D` |
| [2] | `%s` | MAIN function | `AUTO` `R` `C` `L` `Z` `ECAP` `BAT` |
| [3] | `%s` | Primary parameter label | `R` `C` `L` `Z` `V` |
| [4] | `%s` | Secondary parameter label | `D` `Q` `X` `P` `R` |
| [5] | `%s` | Circuit model selection | `PAR` `SER` `AUTO` |
| [6] | `%s` | Effective circuit model | `PAR` or `SER` (AUTO resolves based on component) |
| [7] | `%s` | Measurement speed | `SLOW` `MED` `FAST` |
| [8] | `%s` | Impedance range | `AUTO` `100` `1000` `10000` `100000` (Ω) |
| [9] | `%d` | Test frequency (Hz) | `100` `120` `1000` `10000` `100000` |
| [10] | `%0.0f` | AC test voltage (mV) | `100` `300` `600` |
| [11] | `%0.0f` | DC bias voltage (mV) | `0` (off) or `500` (on) |
| [12] | `%d` | Tolerance display state | `0` = off, `1` = on (front-panel menu only) |
| [13] | `%s` | Comparator parameter type | `R` `C` `L` — matches active measurement |
| [14] | `%s%s` | Tolerance nominal value | `1.0000e3R`; set by `COMParator:NOMinal` |
| [15] | `%0.1f` | Tolerance percentage | `10.0` for ±10%; set by `COMParator:ERRor` |
| [16] | `%d` | PASS/FAIL result | `1` = PASS, `0` = FAIL |

> **field [16]** is only meaningful when the comparator is active (field [12] = 1). With the comparator off (field [12] = 0) it stays `0` regardless of NOMinal/ERRor. Enable the comparator over SCPI with the bare `COMParator` command (see below) — no front-panel menu needed.

---

## Measurement Commands

### FREQuency ✓
Cycles test frequency: `100 Hz → 120 Hz → 1 kHz → 10 kHz → 100 kHz → (repeat)`

Reflected in field [9]. 120 Hz is included for 60 Hz mains-harmonic testing.

### APERture ✓
Cycles measurement speed: `SLOW → MED → FAST → (repeat)`

Reflected in field [7]. Slower speed = more averages = more stable readings.

### VOLTage ✓
Cycles AC test signal amplitude: `100 mV → 300 mV → 600 mV → (repeat)`

Reflected in field [10].

### BIAS:VOLTage ✓
Toggles DC bias: `0 mV (off) ↔ 500 mV (on)` — fixed level, cannot be adjusted.

Reflected in field [11]. Used for measuring polarized electrolytic capacitors. ECAP mode enables it automatically, but the toggle still works in ECAP mode — verified: a 47 µF cap read ~30 µF with bias on vs. ~167 µF with bias off, so leave bias on for electrolytics.

---

## FUNCtion:IMPedance Commands

### FUNCtion:IMPedance:MAIN ✓
Cycles the primary measurement function. Reflected in field [2].

Cycle order: `AUTO → R → C → L → Z → ECAP → BAT → (repeat)`

| Mode | field [3] | field [4] default | Notes |
|------|-----------|--------------------|-------|
| `AUTO` | `C` | `D` | Auto-detect component; tolerance not available |
| `R` | `R` | `X` | Pure resistance |
| `C` | `C` | `D` | Capacitance |
| `L` | `L` | `Q` | Inductance |
| `Z` | `Z` | `P` | Complex impedance |
| `ECAP` | `C` | `D` | Electrolytic cap; applies 500 mV DC bias automatically |
| `BAT` | `V` | `R` | Battery: voltage + internal resistance |

### FUNCtion:IMPedance:SUB ✓
Cycles the secondary measurement parameter through 5 values. Starting position depends on MAIN mode. Reflected in field [4]. Has no effect in BAT mode.

| MAIN | Cycle order |
|------|-------------|
| `R` | X → P → R → D → Q → (repeat) |
| `C` / `AUTO` / `ECAP` | D → Q → X → P → R → (repeat) |
| `L` | Q → X → P → R → D → (repeat) |
| `Z` | P → R → D → Q → X → (repeat) |

Secondary codes: `D` = dissipation factor, `Q` = quality factor, `X` = reactance, `P` = phase angle, `R` = resistance/ESR.

### FUNCtion:IMPedance:Model ✓
Cycles circuit model: `PAR → SER → AUTO → (repeat)`

Reflected in field [5]; field [6] shows the resolved model when AUTO is active.

### FUNCtion:IMPedance:RANGe ✓
Cycles impedance range: `AUTO → 100 Ω → 1 kΩ → 10 kΩ → 100 kΩ → (repeat)`

Reflected in field [8].

---

## COMParator — Pass/Fail Testing

### Standard modes (R, C, L, Z, ECAP)

| Command | Verified | Notes |
|---------|----------|-------|
| `COMParator` (no param) | ✓ | **Toggles the comparator on/off** — flips field [12] between `1` and `0`. This enables PASS/FAIL over SCPI without the front-panel menu. |
| `COMParator ON` / `COMParator OFF` | ✓ | Accepted with a boolean argument; the reliable way to switch the on-screen comparator is the bare `COMParator` toggle above |
| `COMParator:NOMinal <value>` | ✓ | Sets reference value — **mode-scaled units, see below** |
| `COMParator:ERRor <pct>` | ✓ | Sets tolerance %. **SCPI clamps to 40** (any value ≥ 40 becomes 40). The front-panel menu allows up to 99.9%. |
| `COMParator:RANGe?` | ✓ | Accepted; returns empty |

> **Important caveats:**
>
> - **field [16] (PASS/FAIL) is only valid when the comparator is enabled (field [12] = 1).** With it off, field [16] reads `0` even when the part is in tolerance. Send the bare `COMParator` command to enable it.
> - **`COMParator:ERRor` clamps at 40% over SCPI.** Verified: inputs of 50, 99, 99.9, 100, 200 all store as `40.0`. The instrument itself supports up to 99.9% via the front-panel menu — this is a limitation of the SCPI command only.
> - `COMParator:RANGe` (write form) does not exist → `-110 Command header error`
> - All comparator query commands (`COMParator?`, `COMParator:NOMinal?`, `COMParator:RESult?`, `COMParator:RANGe?`) return empty strings. Read comparator state via `FETCh?` fields [12]–[16] only.
> - `COMParator:ERRor?` → `-110 Command header error` (query form absent in firmware)
> - Tolerance does **not** work in AUTO mode.

### COMParator:NOMinal — mode-scaled units, shared storage

The value you send is **not in SI base units** — it is scaled by the active mode. One raw number is stored and shared across all modes (setting it in any mode overwrites it everywhere); field [14] then displays it with that mode's scale:

| MAIN mode | Input unit | Example: set 47 µF / 1 kΩ / 831 µH |
|-----------|-----------|-------------------------------------|
| `R`, `Z` | **ohms (Ω)** — 1:1 | `COMParator:NOMinal 1000` → 1 kΩ |
| `C`, `ECAP`, `AUTO` | **nanofarads (nF)** ×10⁻⁹ | `COMParator:NOMinal 47000` → 47 µF |
| `L` | **microhenries (µH)** ×10⁻⁶ | `COMParator:NOMinal 831` → 831 µH |
| `BAT` | N/A — accepted but ignored | — |

> This is the key gotcha vs. the front-panel: the device keypad lets you type `47µF` directly, but over SCPI the C-mode unit is **nanofarads**. `COMParator:NOMinal 47e-6` does **not** mean 47 µF — it is read as 0.000047 nF ≈ 0. Use `47000`.

**Fractional values are accepted** (it is a float, not an integer): in C mode `COMParator:NOMinal 0.5` → 500 pF, `1000.5` → 1.0005 µF. **Unit suffixes are rejected:** `47uF` or `0.047m` → `-120 Numeric data error; 'u'/'m' not allowed in FLOAT`.

### Automated Pass/Fail Workflow

No physical interaction required — enable the comparator over SCPI first:

```
# --- resistor sorting, 1 kΩ ±5% (R mode) ---
COMParator:NOMinal 1000   # 1 kΩ  (R/Z mode: ohms, 1:1)
COMParator:ERRor 5        # ±5% tolerance (max 40 over SCPI)
COMParator                # toggle comparator ON — confirm field [12] = 1
FETCh?                    # field [16]: 1 = PASS, 0 = FAIL

# --- capacitor sorting, 47 µF ±20% (C / ECAP mode) ---
COMParator:NOMinal 47000  # 47 µF expressed in NANOFARADS
COMParator:ERRor 20
COMParator                # ensure field [12] = 1
FETCh?                    # field [16]
```

Verify field [12] = 1 in the FETCh? output before trusting field [16]; if it reads 0, send `COMParator` once more to toggle it on. Verified live: with the nominal matching the reading, field [16] = 1; at 2× the nominal with ±5%, field [16] = 0.

### BAT Mode Tolerance

In BAT mode, the comparator is always on and uses four independent limits instead of nominal ± tolerance:

| Limit | Meaning |
|-------|---------|
| `VOL_H` | Voltage upper limit |
| `VOL_L` | Voltage lower limit |
| `RES_H` | Internal resistance upper limit |
| `RES_L` | Internal resistance lower limit |

These limits are **not accessible via SCPI** — all probed command variants generate `-110 Command header error`. Set from front-panel menu only.

`COMParator:NOMinal` and `COMParator:ERRor` are accepted in BAT mode without error but have no effect on BAT limits.

---

## IEEE 488.2 Common Commands ✓

| Command | Description |
|---------|-------------|
| `*CLS` | Clear ESR and STB event bits |
| `*ESE <mask>` / `*ESE?` | Event Status Enable register (read/write) |
| `*ESR?` | Query and clear Event Status Register |
| `*IDN?` | Returns `ZOYI,ZT-DQ02,<HEX8>,V<ver>` |
| `*OPC` | Set OPC bit (bit 0) in ESR |
| `*OPC?` | Returns `1` immediately (synchronous device) |
| `*RST` | Resets SCPI status registers only — **does not reset measurement settings** |
| `*SRE <mask>` / `*SRE?` | Service Request Enable register (read/write) |
| `*STB?` | Query Status Byte |
| `*TST?` | Returns empty (not implemented or silent pass) |
| `*WAI` | Returns immediately (synchronous device) |

---

## SYSTem Commands

| Command | Notes |
|---------|-------|
| `SYSTem:ERRor?` / `SYSTem:ERRor:NEXT?` | Dequeue oldest error (same handler) |
| `SYSTem:ERRor:ALL?` | Query all errors without dequeuing |
| `SYSTem:ERRor:CLEAR` ✓ | Clear error queue |
| `SYSTem:ERRor:COUNt?` ✓ | Number of errors in queue |
| `SYSTem:ERRor:CODE?` / `SYSTem:ERRor:CODE:NEXT?` | Next error code only (same handler) |
| `SYSTem:ERRor:CODE:ALL?` | All error codes |
| `SYSTem:VERSion?` ✓ | Returns `1999.0` |

---

## STATus Commands ✓

Standard SCPI status register model. Both Operation and Questionable registers provide:
`EVENt?`, `CONDition?`, `ENABle <val>` / `ENABle?`. Top-level `?` is an alias for `EVENt?`.

`STATus:PRESet` clears both enable registers to 0.

> **Firmware limitation:** `ENABle` accepts only boolean values (`0`/`1` or `ON`/`OFF`), not integer bitmasks. Passing `255` → `-120 Numeric data error: Invalid BOOL value`. Writing `1` sets the register to `65535` (all bits); `0` clears it.

---

## Commands Present But Unused

These commands are artifacts of the embedded [scpi-parser](https://github.com/j123b567/scpi-parser) library and serve no instrument function:

| Command | Behavior |
|---------|----------|
| `USeRERRor` | Pushes `10,"Custom error; Custom error message..."` to error queue |
| `DATA:BLOB` | Binary block data transfer test |
| `ERROR_FALLBACK` | Error fallback handler test |
| `CHARData` | Character data type test |

---

## Firmware Analysis Notes

`ZT-DQ02_v112.bin` — 126,812 bytes, ARM Cortex-M Thumb-2, STM32 family, flash base `0x08000000`. Uses the open-source [scpi-parser](https://github.com/j123b567/scpi-parser) library by Jan Breuer.

**SCPI dispatch tables:**
- Instrument commands: file offset `0x0107D4`, stride 88 bytes
- IEEE 488.2 / SYSTem commands: file offset `0x01124C`, stride 88 bytes

Each entry: up to 3-level path strings (null-terminated) + handler pointer at offset +72 + flags at offset +76.

**Alias pairs (identical handler addresses):**
- `SYSTem:ERRor?` = `SYSTem:ERRor:NEXT?`
- `SYSTem:ERRor:CODE?` = `SYSTem:ERRor:CODE:NEXT?`
- `STATus:OPERation?` = `STATus:OPERation:EVENt?`
- `STATus:QUEStionable?` = `STATus:QUEStionable:EVENt?`

**IDN format string (offset 0x0f084):** `ZOYI,ZT-DQ02,%08X,V%s`

---

*All commands marked ✓ were confirmed on a physical ZT-DQ02 unit via USB CDC at 9600 baud 8N1.*
