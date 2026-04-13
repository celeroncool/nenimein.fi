---
title: "Tesla Model X speaker measurements"
date: 2022-02-04T19:43:58+02:00
draft: false
tags: [car, car-audio, projects]
---

Here are some measurements of a Tesla Model X, with "basic" audio package (two speakers per door).

I have also included the [full REW measurement data here](./rew_data.mdat), so you do not have to trust pictures only.

## Measuring setup

### Equipment

- RTA MIC: Measurements were done with MiniDSP UMIK-1, microphone pointing up, and 90deg calibration file loaded to REW.
- Speaker line measuring: Mackie ProFX6v3, with speaker line going in to high-level input.
- Source: USB stick, Audison BitOne test CD track "03 Pink noise 10min" in WAV format (lossless), played with volume 5 on Tesla's head unit/entertainment system.

### Method

Method is mostly based on [this great guide](https://www.diymobileaudio.com/threads/first-timers-guide-to-measuring-your-system.163234/) from diymobileaudio author Hanatsu.

Each measurement is taken by moving the microphone around near the driver's ear, while sitting in drivers seat. We take 30 samples using REW RTA with the following options:
- Mode: RTA 1/12 octave
- Smoothing: no smoothing
- FFT length: 64k
- Averages: Forever
- Window: Hann
- Max overlap: 50%

This gives us the best possibility to get a "real" average. Unfortunately we cannot input REW sweeps to the H/U as it does not have AUX in.

### Results

Left front speakers (Psychoacoustic smoothing applied):
{{< img src="left_average.jpg" alt="Left speaker averages." >}}

Right front speakers (Psychoacoustic smoothing applied):
{{< img src="right_average.jpg" alt="Right speaker averages." >}}

### Direct speaker line measurements

We attached the Mackie ProFX6v3 straight to speaker wires, so we can figure out what frequencies we get in to the OEM speakers. We are installing Audison Prima AP8.9 Bit to this car, and need to figure out where to get full range signal into the DSP, more on this later...

Here are the front midrange and tweeter measurements combined, as the front has a "door unit" which probably has a crossover built in:
{{< img src="front_line.jpg" alt="Front speaker line measurements." >}}

And here is the rear speaker measured, rear only has a capacitor at tweeter input, so rear has full-range signal:
{{< img src="rear_line.jpg" alt="Front speaker line measurements." >}}