---
title: "DIY: Boombox"
date: 2020-03-29T23:29:40+03:00
draft: false
tags: [diy, projects, lang-english, audio]
---

I lost my [JBL Flip bluetooth speaker](https://www.jbl.com/flip/), and I always wanted a bit more "boom", so I decided to build a real boombox.

Few things I want to have:
- Has to be cheap
- Has to sound atleast OK
- Battery operated, with decent runtime
- Bluetooth, AUX

Optional, "nice to have":
- Lightweight (I don't really care about weight, but the boombox should be atleast "portable")
- CD, Radio, USB
- Charger for phone

So, let's get started!

## Case

I had a older [Pelican 1450](https://www.pelican.com/us/en/product/cases/protector/1450) clone lying around, so I tought that hey, thats a perfect case for a boombox!
The case itself is lightweight, made from glassfiber reinforced nylon, has a carrying handle, is airtight (important for that BASS) and looks pretty cool (black).

{{< img src="pelican-hard-watertight-lifetime-case.jpg" alt="Photo courtesy of Pelican USA" >}}

## Speakers/crossover

After I ditched the Kicx speakers out of my [SQ competition car](https://nenimein.fi/blog/post/emma-3/), I had no use for them.
So, the [Kicx LL 6.5 V2](https://www.autoviihde.com/2012625.html) 6.5" speakers were used to get some beats installed into the case!

First, I drilled two 165mm holes for the speakers, and then installed the speakers with M4 stainless bolts, through the plastic case.

{{< img src="kicx_installed.jpg" alt="Kicx Speakers installed" >}}

Now with the mids done, let's install the tweeters, to get those nice "highs".
I bought two older Hertz DT16 tweeters, and drilled two 32mm holes for them to the upper corners of the case. If I want, I can someday upgrade to a 3-way system, by just drilling 3" hole and swapping out the crossovers.

I fastened the tweeters with 2-component epoxy, so they are "weatherproof" and don't leak air trough

{{< img src="hertz_installed.jpg" alt="Hertz tweeters installed" >}}

Now add the crossovers to the mix, to get better sound, and protect the tweeters from low frequencies.
I chose Focus Audio 2-Way crossovers I found lying around. If you want better sound, match the crossovers and speakers from same manufacturer and product line.

{{< img src="crossover.jpg" alt="Pioneer installed" >}}



## AMP/Headunit

For the amplifier, I got a older [Pioneer DEH-P5100UB](https://www.crutchfield.com/S-84psDNb22OS/p_130P5100UB/Pioneer-DEH-P5100UB.html) headunit. Great things about using headunits designed for cars are:
- 12V, so no need for any voltage conversions
- CD player that doesn't skip when moving
- AUX, Bluetooth (new ones atleast, mine has bluetooth added via CD changer input) and USB for inputs
- 4x50W, enough power... for now :)
- Small

First, I made a standard "1-DIN" sized hole to the case, and attached the metal support bracket to the plastic by turning the metal tabs over. I also installed a 12V switch to the ACC line. The switch has small a light, to show that the power is on.

{{< img src="pioneer_hole.jpg" alt="Pioneer installed" >}}

{{< img src="pioneer_bracket.jpg" alt="Pioneer installed" >}}

## Wiring/battery

Wiring is simple, just connect the wires according to headunit wiring diagram.
I'm using pioneer wiring, converted to a standard ISO plug.

{{< img src="pioneer_pinout.jpg" alt="Pioneer pinout" >}}
{{< img src="Car-Stereo-ISO-Radio-Wiring-Harness-Headunit-Connector.jpg" alt="Pioneer installed" >}}

To not drain the battery, and switch off the headunit, add a switch between battery +12V and ACC wire, like in cars. Connect the red (constant +12V) and black (negative) directly to the battery.

TODO;
- Wire a USB charger
- Wire battery charger

## Completed unit

Pics from inside, with one installed 12V 5Ah battery

{{< img src="final_inside.jpg" alt="Final inside" >}}

Pics outside

{{< img src="outside_1.jpg" alt="Outside 1" >}}
{{< img src="outside_2.jpg" alt="Outside 2" >}}
{{< img src="outside_3.jpg" alt="Outside 3" >}}
{{< img src="outside_4.jpg" alt="Outside 4" >}}

Video from testing
{{< youtube SBAjLhTW--4 >}}