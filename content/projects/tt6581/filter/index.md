---
title: "TT6581 — State-Variable Filter"
date: 2025-12-01
description: "Chamberlin SVF implementation in the TT6581 audio synthesizer."
summary: "How the Chamberlin state-variable filter works in fixed-point hardware, including stability limits and coefficient calculation."
tags: ["fpga", "dsp", "filter", "audio"]
categories: ["Hardware"]
weight: 3
showTableOfContents: true
---

## Overview

The TT6581 uses a **Chamberlin State-Variable Filter (SVF)** — a second-order IIR filter topology originally described in *Musical Applications of Microprocessors* by Hal Chamberlin. It simultaneously produces low-pass, high-pass, band-pass, and band-reject outputs from a single structure.

## Filter Topology

The SVF maintains two state registers, `reg_low` (low-pass) and `reg_band` (band-pass), and computes all four outputs each sample period:

```
hp = input - reg_low - Q_coeff * reg_band
reg_band += F_coeff * hp
reg_low  += F_coeff * reg_band
bp = reg_band
lp = reg_low
br = hp + lp
```

Where:
- `F_coeff` controls the cutoff frequency (Q1.15 fixed-point)
- `Q_coeff` controls the damping / resonance (Q4.12 fixed-point)

## Fixed-Point Implementation

All internal arithmetic uses **14-bit signed** signals. The multiplications (coefficient × state) are performed using the shared 24×16 shift-add multiplier, with results truncated back to 14 bits.

The filter is fully time-multiplexed and executes as a state machine:

1. Compute `hp` = input − low − (Q × band)
2. Compute `new_band` = band + (F × hp)
3. Compute `new_low` = low + (F × new_band)
4. Select output based on mode register

Each multiply takes multiple clock cycles through the shared multiplier, but the entire filter computation completes well within the 50 kHz sample period.

## Coefficient Calculation

### Cutoff Frequency

The cutoff coefficient is calculated as:

$$
F = \left\lfloor 2 \cdot \sin\!\left(\frac{\pi \cdot f_c}{F_s}\right) \cdot 32768 \right\rfloor
$$

This produces a Q1.15 value written across `F_LO` and `F_HI`.

**Example:** For a 1 kHz cutoff at \(F_s = 50\) kHz:

$$
F = \lfloor 2 \cdot \sin(\pi \cdot 1000 / 50000) \cdot 32768 \rfloor = 0x1013
$$

### Resonance (Q Factor)

The damping coefficient is:

$$
Q_{\text{coeff}} = \left\lfloor \frac{1}{Q} \cdot 4096 \right\rfloor
$$

This produces a Q4.12 value written across `Q_LO` and `Q_HI`.

**Example:** For Butterworth (\(Q = \tfrac{1}{\sqrt{2}}\)):

$$
Q_{\text{coeff}} = \lfloor \sqrt{2} \cdot 4096 \rfloor = 0x16A1
$$

## Stability Limits

The Chamberlin SVF topology becomes unstable when the cutoff frequency approaches \(F_s / 6\). For the TT6581 running at 50 kHz:

$$
f_{c,\max} \approx \frac{50000}{6} \approx 8333 \text{ Hz}
$$

Setting the cutoff coefficient above this value will cause the filter to oscillate and produce distorted output. This is a fundamental limitation of the Chamberlin topology and mirrors the behaviour of the original MOS6581.

## Filter Modes

The `EN_MODE` register's lower 3 bits select the filter output:

| MODE | Output      | Description   |
| ---- | ----------- | ------------- |
| 001  | `reg_low`   | Low-pass      |
| 010  | `reg_band`  | Band-pass     |
| 100  | `hp`        | High-pass     |
| 101  | `hp + lp`   | Band-reject   |

## Bode Response

The SVF frequency response can be verified using the `tt6581_bode` Verilator testbench, which sweeps a sine wave through the full audio range and records the delta-sigma PDM output. A Python script then reconstructs the analogue signal and plots the magnitude response.
