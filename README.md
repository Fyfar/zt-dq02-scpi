# ZT-DQ02 SCPI Reference

> Complete SCPI command reference for the **ZOYI ZT-DQ02 LCR meter** — extracted from firmware static analysis and **verified by live device testing** over USB.

This document lets you control the ZT-DQ02 programmatically from a PC, automate component sorting, and integrate the meter into test fixtures — no screen needed.

---

## Table of Contents

- [Hardware & Connection](#hardware--connection)
- [Quick Start](#quick-start)
- [Command Model](#command-model)
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

> **field [16]** is computed from NOMinal and ERRor% continuously, even when the tolerance display is off (field [12] = 0). Use it for automated pass/fail without enabling the screen display.

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

Reflected in field [11]. Used for measuring polarized electrolytic capacitors (ECAP mode applies this automatically).

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
| `COMParator ON` | ✓ | Arms comparator parameters |
| `COMParator OFF` | ✓ | Disarms |
| `COMParator` (no param) | ✓ | Toggles ON/OFF |
| `COMParator:NOMinal <value>` | ✓ | Sets reference value |
| `COMParator:ERRor <pct>` | ✓ | Sets tolerance %; range 0–40 (≥41 clamps to 40) |
| `COMParator:RANGe?` | ✓ | Accepted; returns empty |

> **Important caveats:**
>
> - `COMParator:RANGe` (write form) does not exist → `-110 Command header error`
> - All comparator query commands (`COMParator?`, `COMParator:NOMinal?`, `COMParator:RESult?`, `COMParator:RANGe?`) return empty strings. Read comparator state via `FETCh?` fields [12]–[16] only.
> - `COMParator:ERRor?` → `-110 Command header error` (query form absent in firmware)
> - `COMParator ON/OFF` arms internal parameters but does **not** change field [12] and does **not** show PASS/FAIL on the display. The front-panel menu is required to activate the on-screen tolerance display.
> - Tolerance does **not** work in AUTO mode.

### COMParator:NOMinal — shared storage

One nominal value is stored as a raw SI float, shared across all measurement modes. Setting it in any mode overwrites it for all modes. Field [14] displays with mode-appropriate units:

| MAIN mode | Display unit |
|-----------|-------------|
| `R`, `Z` | Ω |
| `C`, `ECAP`, `AUTO` | F |
| `L` | H |
| `BAT` | N/A — not used |

Set the nominal while in the target mode. Scientific notation is accepted: `COMParator:NOMinal 470e-12`

### Automated Pass/Fail Workflow

No screen or physical interaction required:

```
COMParator:NOMinal 1000   # reference: 1 kΩ
COMParator:ERRor 5        # ±5% tolerance
# insert component
FETCh?                    # field [16]: 1 = PASS, 0 = FAIL
```

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
