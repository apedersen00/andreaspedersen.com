---
title: "TT6581 — SID-Inspired Audio Synthesizer"
date: 2025-12-01
description: "A MOS6581-inspired digital audio synthesizer implemented in SystemVerilog for Tiny Tapeout."
summary: "Open-source audio synthesizer ASIC inspired by the legendary Commodore 64 SID chip. Three voices, ADSR envelopes, state-variable filter, and delta-sigma DAC — all in 2×2 Tiny Tapeout tiles."
tags: ["fpga", "asic", "audio", "synthesizer", "tiny-tapeout", "systemverilog"]
categories: ["Hardware"]
weight: 1
showTableOfContents: true
---

{{< github repo="apedersen00/tt6581" >}}

## Overview

Inspired by the legendary **MOS6581 Sound Interface Device (SID)** chip used in the Commodore 64, the TT6581 is an original digital interpretation supporting nearly the entire original feature set, implemented in 2×2 tiles for [Tiny Tapeout](https://tinytapeout.com/).

### Features

- Full control through a **Serial-Peripheral Interface** (SPI)
- **Three** independently synthesised voices
- Four waveform types: triangle, sawtooth, pulse, and noise
- **ADSR** envelope shaping per voice
- **Chamberlin State-Variable Filter** (SVF) — LP, HP, BP, and BR modes
- Second-order **delta-sigma DAC** with 10 MHz PDM output (OSR = 200)

## Architecture

The diagram below shows the datapath in the TT6581. A tick generator triggers the generation of a single audio sample at 50 kHz.

![TT6581 Datapath](tt6581_datapath.png)

1. **Voice Generation** — One 10-bit voice at a time is generated. Internal phase registers maintain each voice's state while inactive. Supported waveforms are triangle, sawtooth, pulse, and noise.

2. **Envelope** — An ADSR envelope generator produces an 8-bit amplitude value per voice, applied by multiplication.

3. **Wave Accumulation** — The three voices are accumulated (mixed) by addition. Depending on each voice's filter-enable bit, the signal is routed through the SVF or bypasses it.

4. **State-Variable Filter** — A Chamberlin SVF processes the filter accumulator. It supports low-pass, high-pass, band-pass, and band-reject modes with tuneable cutoff frequency and resonance (Q).

5. **Global Volume** — The SVF output is summed with the bypass accumulator and a global 8-bit volume is applied.

6. **Delta-Sigma PDM** — An error-feedback delta-sigma modulator converts the final mix to 1-bit PDM at 10 MHz.

To fit the strict 2×2 tile area requirement, the entire synthesis pipeline is time-multiplexed — most modules are state machines, and a single **24×16 shared multiplier** handles all arithmetic.

## Pin Mapping

| Pin        | Direction | Function                     |
| ---------- | --------- | ---------------------------- |
| `uio[0]`   | Input     | SPI Chip Select (active low) |
| `uio[1]`   | Input     | SPI MOSI                     |
| `uio[2]`   | Output    | SPI MISO                     |
| `uio[3]`   | Input     | SPI SCLK                     |
| `uio[4:7]` | —         | Unused                       |
| `uo[0]`    | Output    | PDM audio output             |
| `uo[1:7]`  | —         | Unused                       |
| `ui[0:7]`  | —         | Unused                       |

The PDM output should be passed through a **4th-order Bessel low-pass filter** for best analogue reconstruction.

## Deep Dives

Detailed write-ups on each subsystem:

- [Register Map & SPI Protocol](registers/)
- [State-Variable Filter](filter/)
- [Delta-Sigma DAC](delta-sigma/)
- [Simulation & Verification](simulation/)
