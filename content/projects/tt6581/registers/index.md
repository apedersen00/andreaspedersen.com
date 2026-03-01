---
title: "TT6581 — Registers & SPI Protocol"
date: 2025-12-01
description: "Register map and SPI interface for the TT6581 audio synthesizer."
summary: "Full register map, SPI framing, and worked examples for programming the TT6581."
tags: ["fpga", "spi", "registers", "tiny-tapeout"]
categories: ["Hardware"]
weight: 2
showTableOfContents: true
---

## SPI Protocol

The SPI interface uses **CPOL=0, CPHA=0** (data sampled on the rising edge of SCLK). Each transaction is a 16-bit frame while CS is held low:

| Bit   | 15    | 14:8             | 7:0           |
| ----- | ----- | ---------------- | ------------- |
| Field | R/W   | Address \[6:0\]  | Data \[7:0\]  |

- **Bit 15** = `1` for write, `0` for read
- **Bits 14:8** = 7-bit register address
- **Bits 7:0** = write data (ignored on reads)

Data is transmitted MSB first.

## Register Map

The TT6581 has a 7-bit address space with 26 8-bit registers. Three identical voice groups are followed by filter and volume registers.

### Voice Registers

Each voice occupies 7 consecutive registers. Voice 0 starts at `0x00`, Voice 1 at `0x07`, and Voice 2 at `0x0E`.

| Offset | Name    | Bits | Description                        |
| ------ | ------- | ---- | ---------------------------------- |
| 0x00   | FREQ_LO | 7:0  | Frequency control word — low byte  |
| 0x01   | FREQ_HI | 7:0  | Frequency control word — high byte |
| 0x02   | PW_LO   | 7:0  | Pulse width — low byte             |
| 0x03   | PW_HI   | 3:0  | Pulse width — high byte            |
| 0x04   | CONTROL | 7:0  | Waveform select and voice control  |
| 0x05   | AD      | 7:0  | Attack (7:4) / Decay (3:0)         |
| 0x06   | SR      | 7:0  | Sustain (7:4) / Release (3:0)      |

### CONTROL Register Bit Fields

| Bit | Name     | Description                    |
| --- | -------- | ------------------------------ |
| 7   | NOISE    | Select noise waveform          |
| 6   | PULSE    | Select pulse (square) waveform |
| 5   | SAW      | Select sawtooth waveform       |
| 4   | TRI      | Select triangle waveform       |
| 3   | —        | Reserved                       |
| 2   | RING_MOD | Enable ring modulation         |
| 1   | SYNC     | Enable oscillator sync         |
| 0   | GATE     | Gate (1 = attack, 0 = release) |

### Filter and Volume Registers

| Address | Name    | Bits | Description                            |
| ------- | ------- | ---- | -------------------------------------- |
| 0x15    | F_LO    | 7:0  | Filter cutoff coefficient — low byte   |
| 0x16    | F_HI    | 7:0  | Filter cutoff coefficient — high byte  |
| 0x17    | Q_LO    | 7:0  | Filter damping coefficient — low byte  |
| 0x18    | Q_HI    | 7:0  | Filter damping coefficient — high byte |
| 0x19    | EN_MODE | 5:0  | Filter enable and mode select          |
| 0x1A    | VOLUME  | 7:0  | Global volume (0x00–0xFF)              |

### EN_MODE Bit Fields

| Bit | Name    | Description                                          |
| --- | ------- | ---------------------------------------------------- |
| 5   | FILT_V2 | Route Voice 2 through filter                         |
| 4   | FILT_V1 | Route Voice 1 through filter                         |
| 3   | FILT_V0 | Route Voice 0 through filter                         |
| 2:0 | MODE    | Filter mode: `001`=LP, `010`=BP, `100`=HP, `101`=BR  |

## Formulas

### Frequency Control Word (FCW)

$$
\text{FCW} = \frac{f_{\text{desired}} \cdot 2^{19}}{F_s}
$$

where \(F_s = 50\) kHz. The 16-bit FCW is split across `FREQ_LO` (bits 7:0) and `FREQ_HI` (bits 15:8).

Maximum representable voice frequency:

$$
f_{\max} = \frac{65535 \cdot F_s}{2^{19}} \approx 6250 \text{ Hz}
$$

### Filter Cutoff Coefficient (Q1.15 signed)

$$
\text{FCC} = \left\lfloor 2 \cdot \sin\!\left(\frac{\pi \cdot f_c}{F_s}\right) \cdot 32768 \right\rfloor
$$

where \(f_c\) is the desired cutoff frequency in Hz. The maximum stable cutoff is approximately **8 kHz** (\(F_s / 6\)).

### Filter Damping Coefficient (Q4.12 signed)

$$
\text{FDC} = \left\lfloor \frac{1}{Q} \cdot 4096 \right\rfloor
$$

where \(Q\) is the desired resonance.

## Quick Start Example

Playing a 440 Hz sawtooth on Voice 0 with instant attack and full sustain:

```text
SPI Write: addr=0x1A, data=0xFF    # Volume = max
SPI Write: addr=0x00, data=0x05    # FREQ_LO = 0x05  (FCW = 0x1205)
SPI Write: addr=0x01, data=0x12    # FREQ_HI = 0x12
SPI Write: addr=0x05, data=0x00    # AD = 0x00 (attack=0, decay=0)
SPI Write: addr=0x06, data=0xF0    # SR = 0xF0 (sustain=15, release=0)
SPI Write: addr=0x04, data=0x21    # CONTROL = sawtooth + gate on
```

To release the note:

```text
SPI Write: addr=0x04, data=0x20    # CONTROL = sawtooth + gate off
```
