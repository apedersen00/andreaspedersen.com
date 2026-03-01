---
title: "TT6581 — Delta-Sigma DAC"
date: 2025-12-01
description: "Second-order error-feedback delta-sigma modulator in the TT6581."
summary: "How the 1-bit PDM output is generated from the digital audio stream using a second-order error-feedback delta-sigma modulator."
tags: ["fpga", "dsp", "delta-sigma", "audio", "dac"]
categories: ["Hardware"]
weight: 4
showTableOfContents: true
---

## Overview

The TT6581 converts its internal digital audio signal to analogue using a **second-order error-feedback delta-sigma modulator**. The modulator outputs a 1-bit **pulse-density modulated** (PDM) signal at 10 MHz — an oversampling ratio (OSR) of 200 relative to the 50 kHz audio sample rate.

## Architecture

The error-feedback topology is ideal for resource-constrained implementations. It consists of:

1. An input scaling stage
2. Two cascaded integrators (error accumulators)
3. A 1-bit quantiser (sign-bit extraction)
4. Error feedback from the quantiser back to the integrators

```
                  ┌─────────────┐
  audio_in ──×16──┤             ├──── PDM out (1-bit @ 10 MHz)
                  │  2nd-order  │
                  │  error-     │
                  │  feedback   │
                  │  Σ-Δ        │
                  └─────────────┘
```

### Input Scaling

The audio signal arriving at the delta-sigma modulator typically uses only a fraction of the available dynamic range due to cascaded ÷256 divisions in the signal chain (envelope multiply and volume multiply). To maximise SNR, the input is **scaled by ×16** before entering the modulator:

```systemverilog
audio <= {audio_i[13], audio_i, 4'b0};  // sign-extend + shift left 4
```

This maps the 14-bit input into a 19-bit working range (±32,768 reference).

### Quantiser

The quantiser is simply the **sign bit** of the first integrator's output:

- Positive → PDM = `1` (maps to +32,768 reference)
- Negative → PDM = `0` (maps to −32,768 reference)

The quantisation error is fed back through two integrator stages, shaping the noise spectrum so that most quantisation noise is pushed above the audio band.

## Noise Shaping

The second-order topology provides **40 dB/decade** noise shaping. Combined with OSR = 200, this yields a theoretical SNR improvement of approximately:

$$
\text{SNR} \approx 6.02 \cdot 1 + 1.76 + (2 \cdot 2 + 1) \cdot 10 \cdot \log_{10}(\text{OSR})
$$

In practice, the effective resolution is around **12–13 bits** within the audio band.

## Reconstruction

The PDM output must be filtered externally to recover the analogue audio signal. A **4th-order Bessel low-pass filter** with a cutoff around 20–25 kHz provides optimal results:

- Flat passband (no ringing, important for audio)
- Linear phase response
- Sufficient attenuation of the shaped quantisation noise above \(F_s / 2\)

In simulation, this reconstruction is performed digitally using `scipy.signal.bessel` and `sosfilt`.

## Amplitude Headroom

The signal chain introduces cascaded ÷256 divisions:

| Stage            | Effect                    | Nominal range after        |
| ---------------- | ------------------------- | -------------------------- |
| Voice output     | 10-bit (±512)             | ±512                       |
| × Envelope (÷256)| 8-bit envelope            | ±2 per voice               |
| Accumulate ×3    | Sum of 3 voices           | ±6                         |
| SVF (14-bit)     | Passthrough or filtered   | ±6 (Q boost can increase)  |
| × Volume (÷256)  | 8-bit global volume       | ±6 (at max volume)         |
| ×16 scaling      | Delta-sigma input stage   | ±96                        |

With a reference of ±32,768, the modulator operates well within its linear range for typical signals. High-Q filter resonance with multiple voices at full amplitude can theoretically clip at around ±2,048 input (pre-scaling), though this is rare in practice.
