---
title: "HPE Proliant Microserver Gen7 fan mod"
date: 2023-05-23T21:09:55+03:00
draft: false
tags: [HPE, projects, diy, cooling, lang-english]
---

Modding my HPE Proliant Microserver Gen7 (N54L) to be less loud.
Also listing the fans which do **not** work and fans which work.

Mostly following the now "defunct" guide from [SilentPCReview](http://www.silentpcreview.com/article1193-page7.html). Working guide can be found at [Internet Archive](https://web.archive.org/web/20180618153444/http://www.silentpcreview.com/article1193-page7.html)

## Problem

Stock fans, especially the PSU fan is very noisy. It doesnt output much desibels, but it certainly outputs them at very annoying frequencies.

Here are REW graphs, measured with MiniDSP UMIK-1 measuring microphone approx one meter away from the device, with fans pointing away from the microphone. The server is sitting under my desk, in a corner. Red graph is NAS turned off and green is NAS running with stock fans.

{{< img src="nas_before.jpg" alt="REW Graph of NAS before fan mods" >}}

As your can see, the fan ouputs about 5dB across 150Hz-4kHz, which is right in the middle of humans most sensitive hearing range.

Some raw data from averages, what is interesting here is the A- and C-weighted measures, which differ alot.

NAS not running:
- 65536-point 1/48 octave RTA using Hann window and 32 averages
- Input RMS 59.61 dB
- 48.7 dB C, 33.7 dB A
- 49.0 dB 22 - 22k UNW
- 18.3 dB >22k

NAS running:
- 65536-point 1/48 octave RTA using Hann window and 32 averages
- Input RMS 52.39 dB
- 48.0 dB C, 38.3 dB A
- 48.3 dB 22 - 22k UNW
- 19.0 dB >22k

### Fans

I purchased following Noctua fans:
- [Noctua NF-A12x25 PWM](https://noctua.at/en/nf-a12x25-pwm), a 120x25mm 12V PWM fan (Did not work, too slow start RPM...)
- [Noctua NF-A4x20 5V](https://noctua.at/en/products/fan/nf-a4x20-5v), a 40x20mm 5V fan (Did not work, too noisy...)
- [Noctua NF-A4x20 FLX](https://noctua.at/en/products/fan/nf-a4x20-flx), a 40x20mm 12V fan with LNA&ULNA adapters (Works wonderfully!)

As the new Noctua 120mm fan was too slow to run, I returned the fan back to original. The motherboard requires atleast 500RPM signal from the fan to boot. This is unfortunately not changeable in the BIOS. 

Here are the graphs comparing all of the options. Red is NAS not running, green is NAS running with stock fans, purple is NAS running with 5V Noctua PSU fan and orange is NAS running with 12V Noctua PSU fan.

{{< img src="nas_after.jpg" alt="REW Graph of NAS after fan mods" >}}

As you can see, the 5V fan is wayy too loud, most likely as the PSU is outputting 5V and the fan is designed to operate at 5000RPM with that voltage.

Swapping the fan to Noctua 12V fan, we see huge improvement over 5V fan, but also to the noisy stock 12V fan. Differences in the hearing range are almost 5dB, and the fan has more "pleasant" sound.

### Installation guide

# PROCEED AT YOUR OWN RISK
Capacitors inside PSU's will hold enough power to instantly kill you for long periods after the power has been removed. Do not touch any capacitors/exposed wiring/PCB traces!

Remove the PSU from the case by undoing the thumbscrew at the back, removing the top case cover. Then remove the motherboard to get easy access to the 12V motherboard connector and unplug it. Unplug all the MOLEX connectors at the top and fish the DVD drive SATA power connector from the hole to the front. Now unscrew three bolts behind the server near the power plug and smaller fan.

{{< img src="IMG_7212.jpeg" alt="Case opened" >}}

Now you will have the PSU removed from the case. Open the four screws holding the top plate on and the two fan screws.

{{< img src="IMG_7210.jpeg" alt="PSU removed" >}}

Cut the fan cable at the middle and slide the fan out from the case.

{{< img src="IMG_7213.jpeg" alt="FAN removed" >}}

Connect the two cables, red to red and black to black with connectors included in the Noctua kit. Slide the fan back in to the slot.

{{< img src="noctua_connector.png" alt="FAN installed" >}}
{{< img src="IMG_7214.jpeg" alt="FAN installed" >}}

Tuck the connectors and cables neatly inside the PSU and screw on the new fan with two of the included screws.

{{< img src="IMG_7216.jpeg" alt="cables installed" >}}

Close up the PSU case, install the PSU, route the cables, attach top panel and enjoy your nearly silent HPE Proliant Microserver Gen7!