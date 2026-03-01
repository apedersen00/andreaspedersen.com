---
title: "TT6581: SID-Inspired Audio Synthesizer"
date: 2026-03-01
description: "A MOS6581-inspired digital audio synthesizer implemented in SystemVerilog for Tiny Tapeout."
summary: "Open-source audio synthesizer ASIC inspired by the legendary Commodore 64 SID chip. Three voices, ADSR envelopes, state-variable filter, and delta-sigma DAC — all in 2×2 Tiny Tapeout tiles."
tags: ["asic", "audio", "tiny-tapeout", "systemverilog"]
categories: ["Hardware"]
weight: 1
showTableOfContents: true
---

{{< github repo="apedersen00/tt6581" >}}

## Introduction

> [!NOTE] This post serves as a deep dive in the design of the TT6581. For a brief overview of the project, visit the projects [Github page](https://github.com/apedersen00/tt6581/).

Inspired by the legendary **MOS6581 Sound Interface Device (SID)** chip used in retro computers such as the Commodore 64, the _Tiny Tapeout 6581_ (TT6581) is an original digital interpretation supporting nearly the entire original MOS6581 feature set, implemented in 2x2 tiles for [Tiny Tapeout](https://tinytapeout.com/).

To demonstrate the chip's capabilities, here is Rob Hubbard's _Monty on the Run_. The track was played using a Verilator RTL testbench, and the final audio was produced by capturing the PDM output and processing it through a 4th-order Bessel filter in Python.

<audio controls src="/audio/monty_on_the_run.mp3"></audio>

## Architecture

The diagram below shows the datapath in the TT6581. A tick generator triggers the generation of a single audio sample at 50 kHz.

![TT6581 Datapath](svg/tt6581.svg "TT6581 Datapath")

1. **Voice Generation** — One 10-bit voice at a time is generated. Internal phase registers maintain each voice's state while inactive. Supported waveforms are triangle, sawtooth, pulse, and noise.

2. **Envelope** — An ADSR envelope generator produces an 8-bit amplitude value per voice, applied by multiplication.

3. **Wave Accumulation** — The three voices are accumulated (mixed) by addition. Depending on each voice's filter-enable bit, the signal is routed through the SVF or bypasses it.

4. **State-Variable Filter** — A Chamberlin SVF processes the filter accumulator. It supports low-pass, high-pass, band-pass, and band-reject modes with tuneable cutoff frequency and resonance (Q).

5. **Global Volume** — The SVF output is summed with the bypass accumulator and a global 8-bit volume is applied.

6. **Delta-Sigma PDM** — An error-feedback delta-sigma modulator converts the final mix to 1-bit PDM at 10 MHz.

To fit the strict 2×2 tile area requirement, the entire synthesis pipeline is time-multiplexed — most modules are state machines, and a single **24×16 shared multiplier** handles all arithmetic.

## Deep Dives

Detailed write-ups on each subsystem:

- [Register Map & SPI Protocol](registers/)
- [State-Variable Filter](filter/)
- [Delta-Sigma DAC](delta-sigma/)
- [Simulation & Verification](simulation/)
