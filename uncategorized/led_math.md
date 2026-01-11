## LED explanation with math

A bit of explanation about your LEDs:

A 3 V 20 mA RGB Tri-Color 4-pin LED is a single package containing red, green, and blue LED chips that can be lit individually or combined to produce colors, each color typically needing about 20 mA at 3 V hence the denomination.

It comes in two wiring types: common anode, where the 3 LED chips share a positive pin and each color is lit by pulling its cathode low, and common cathode, where they share a ground pin and each color is lit by driving its anode high.

In both cases, the maximum current is per color channel (about 20 mA), so if all three are on at full brightness the LED can draw up to roughly 60 mA..

A LED by itself has no internal current limit: once its forward voltage is reached, its resistance drops sharply and it will try to draw as much current as the source can provide, quickly overheating and committing electronic suicide. To prevent this, we add a resistor in series to impose Ohm’s law (U = R × I), so the resistor drops the excess voltage and fixes the current at a safe value for the LED.

To find the resistor value R for an LED with a 3.3 V supply, you take VCC = 3.3 V, subtract the LED’s forward voltage (Vf), and then divide by the desired LED current (I). The formula is R = (3.3 − Vf) ÷ I.

An important point to know is that Each pin of your ESP can source or sink up to about 40 mA absolute maximum, but Espressif recommends keeping it at 20 mA or less (less is better).

So we should target being at or under 20mA flowing through the pin when calculating the resistance needed.

For example, for a red LED with Vf = 2.0 V and expected current I = 20 mA (so 0.02A) , R = (3.3 − 2.0) ÷ 0.02 = 65 Ω (use the next standard value, 68 Ω).

Note that red, green, and blue LEDs in an RGB LED do not have the same forward voltage, so each color usually requires its own series resistor to set the correct current.

Long story short, to find which resistor value to take, you need to read the specs of your RGB Leds and find the forward voltage for each embedded diode Red, Green and blue, decide which target current you can afford from the source and that the led will accept. If you don’t know Vf, a safe approach is to assume a typical maximum for the color: about 2 V for red and 3 V for green or blue. Then choose a resistor that limits current to well below 20 mA, for example 150–220 Ω for a 3.3 V supply.

So assuming you have a common cathode LED, your wiring would be

Pin1 — 220 Ω — [redPin — [LED] — cathode] — GND

Pin2 — 220 Ω — [greenPin — [LED] — cathode] — GND

Pin3 — 220 Ω — [bluePin — [LED] — cathode] — GND

If you set Pin1 HIGH and the others LOW, your LED will light up red. If you set Pin2 HIGH and the others LOW, it will light green, and if Pin3 HIGH with the others LOW, it will light blue. You can also combine pins, for example Pin1 and Pin2 HIGH together will produce yellow.

Something else to keep in mind: it’s important to think in terms of total current draw through the board and in practice, the total current the chip can handle across all pins is limited by the 3.3 V supply rail and package thermal constraints ➜ If you try to drive many pins at high current simultaneously, you risk brownouts or damaging the chip.

A safe rule / conservative guideline from hobbyist is to keep the sum of all pins currents under 100 mA .

You now see the challenge. If you want to light up your led fully in white, you’ll feed say 15mA per diode so that is already 3x15=45mA
If you have 2, it adds up, 90mA and at 3 you are above the conservative 100mA.

That’s why @hammy or @Railroader said you need to use external drivers (transistors, MOSFETs, etc.) for anything beyond a few LEDs.

Makes sense ?