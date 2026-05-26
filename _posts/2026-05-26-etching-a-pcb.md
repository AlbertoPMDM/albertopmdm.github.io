---
title: Etching a PCB
description: Using transfer paper, and ferric chloride, we test how much resolution we can get in a single-sided homemade PCB.
date: 2026-04-09
categories: [Blog, Electronics]
tags:
  - pcb
  - fabrication
  - kicad
  - electronics
  - smd
  - ferric-chloride
  - etching
math: true
mermaid: false
image:
 path: assets/images/etching-a-pcb/closeup.jpeg
---

There are many incentives to going from a protoboard or something like a perfboard to a PCB. Wiring is easier, done in the computer, and we get access to more modern components, most of which are surface-mount (SMD). The easiest way is, of course, to have them made by a Chinese manufacturer, and shipped. But at least in my country, shipping is rather expensive, and recently, it has come to my attention that the necessary components to etch one at home are cheap and accessible.

If one has their PCB manufactured overseas, it can almost be assured that it will be of good quality, can have many layers, and traces can be as thin as 6 mils, so now we ask the question: How good of a single sided PCB one could make at home?

## The method

I will be using a test pattern with common footprints, printed with a laser printer onto PCB transfer paper (which has a texture reminiscent of magazine paper, and some people do use magazine paper), adhering the pattern to a copper plate with a clothes iron, and finally submerging it in a solution of ferric chloride, until all of the copper which is not covered in toner dissolves.

This process is called etching, and works because the toner that laser printers use is powdered plastic, and it does not adhere to the transfer paper as much as to the copper, causing the pattern to leave the paper. This part then becomes the etch resist, called that because it makes copper under it dissolve at a much slower rate, due to the decrease in surface area exposed to the etchant. Still, given enough time, even the copper under the toner will dissolve.

## Making a test pattern

In order to benchmark our etching capabilities, using the KiCAD footprint editor, I drew traces with widths of 5, 10, 15, 20, 25, and 30 mils, and pads to make it easier to test for continuity with a multimeter once etched. I won't worry so much about the spacing, as the footprints of the smallest components should give me a good idea of what can be or cannot be routed.

Additionally, I included some common IC packages, such as SOT, TQFP, TSSOP, DIP, SOIC, QFN, and an ESP module footprint. I also included some common passive component sizes such as 1206, 0805, 0603, and 0402.

>One could make a test pattern in order to test tight tolerances, and leave out the bigger components, but I also want it to be a way to practice soldering the components, that is why a lot of space is dedicated to bigger footprints.
{: .prompt-tip}


I made sure it all fit in a 50x50 mm square, because copper plates this size are readily available in electronic stores in my area.

![footprint-editor](assets/images/etching-a-pcb/footprint-editor.png){: w="400" h="400" }
_Chosen footprints in KiCAD._

After that, I like to open another instance of the PCB editor, and import the design in order to tile it, and get several copies in a single PCB transfer paper page. For printing, I used the following settings

![printing-settings](assets/images/etching-a-pcb/printing-settings.png){: w="400" h="400" }
_Settings menu when selecting the plotting option in KiCAD._

- I only included the F.Cu layer, but also plotted the Edge.Cuts layer as a reference to know where to cut.
- The image probably will have to be mirrored, unless your design takes into account the way the orientation changes when it is transferred from the paper to the copper plate.
- I used small drill marks to center the bit when drilling the holes to final size.
- Choosing 1:1 scaling is essential, so the dimensions in the drawing are printed correctly.
- I also selected black and white only output, to avoid gray-scale colors, and ensure the maximum amount of toner possible is deposited in the paper.

After printing, making sure the printer is set up to print in a 1:1 scale, we get our pattern. After cutting it up, I select the best looking print.

![pattern-on-paper](assets/images/etching-a-pcb/pattern-on-paper.jpeg){: w="400" h="400" }
_A square of PCB transfer paper with the pattern on it. As you can see, there aren't any major imperfections such as smudges, or parts missing._

## Preparing the copper plate

One must prepare the surface of the copper plate for proper toner transfer. Some people do it with fine grit sandpaper (nearing 2000) or a scouring pad. 

I have gotten very good results using fine steel wool, and going in circular motions all over the pad until it is smooth and reflective

![untreated-plate](assets/images/etching-a-pcb/untreated-plate.jpeg){: w="400" h="400" }
_Copper plate before preparing the surface, take note of some scratches and the overall oxidation._

![treated-plate](assets/images/etching-a-pcb/treated-plate.jpeg){: w="400" h="400" }
_The same copper plate after going over it with steel wool. Note none of the scratches are there, and it has obtained a mirror-like finish._

## Transferring the pattern

With a smooth surface, we can transfer the toner. The process I got to after some experimenting is the following:

- ironing for a few seconds half of the pattern until it sticks,
- then leaving the iron on for 1 minute, 
- applying light pressure and moving the iron around for 2 minutes, paying attention to places with thin traces, 
- repeating the aforementioned steps, 
- and letting it cool down to room temperature.

After it cools down, you can soak it in water, so the paper dissolves, or at least softens up, and unsticks from the toner easily. I am using the cheapest toner transfer paper I could find, and if there aren't many traces, it just comes off when it cools down. Still, I usually leave it 15 minutes in water, and it peels off nicely.

![paper-peel](assets/images/etching-a-pcb/paper-peel.jpeg){: w="400" h="400" }
_Transfer paper unsticking from the pattern easily after being soaked in water for 15 minutes._

Then i like to clean up with a pointy tool, to remove any toner particles that may have been left behind, and the Edge.cuts layer lines, because it is much easier to remove the toner than to remove copper later. You may also bridge any gaps that may have formed with a permanent marker.

![transfered-pattern](assets/images/etching-a-pcb/transfered-pattern.jpeg){: w="400" h="400" }
_Copper plate with the test pattern. You can see scratches from where extra toner was removed_

## Etching the pattern

Now comes the etching. A ferric chloride solution is prepared as per the indications on the container (for me it was two parts ferric chloride and one part water), then the copper plate is submerged on it until everything other than the material below the traces is dissolved. As many others, I've found it works best when tilting the container side to side to keep the solution moving, and only takes about 15 minutes.

Once it is done, I like to dry it with some paper towels before rinsing the plate in another container with clean water, dedicated for that purpose. NO FERRIC CHLORIDE SHOULD GO IN THE DRAIN. You should also use eye protection and latex/nitrile gloves.

>I will not be commenting on how to dispose of the solution properly, but it is important you read the datasheet, and disposal instructions of any chemical you work with.
{: .prompt-warning}

![etched-pattern](assets/images/etching-a-pcb/etched-pattern.jpeg){: w="400" h="400" }
_The board after being rinsed in clean water. As you can see, the traces still remain, but all of the copper on the surface is gone._

![finished-pattern](assets/images/etching-a-pcb/finished-pattern.jpeg){: w="400" h="400" }
_The board after being cleaned with acetone. The substrate absorbed quite a bit of ink, so it's darker than before, but all of the traces are clearly visible._


## Inspecting the PCB

After wiping the board with some acetone to remove the toner (the boards I'm using somehow soak up the toner, so they become really dark), we can see that the copper underneath the pattern is almost untouched. We now perform a visual inspection of the board with a magnifying glass to find possible defects, and test for continuity on our traces with a multimeter.

As an observation, although the implication is not serious, a lot of sharp corners have been etched away. This means the number of sharp corners should be kept to a minimum, to ensure a constant trace width.

![defect-1](assets/images/etching-a-pcb/defect-1.jpeg){: w="400" h="400" }
_The defect on the 10 mils test trace is located on the bottom leftmost part of the trace, approximately the width of a hair. There are other scratches and surface defects, but not big enough to cause an open circuit._

In my case, the 10 mils trace has a slight scratch that causes an open circuit. This still is not reason to discard it, as I plan to apply solder to all of the traces to protect them from corrosion, and that should bridge such a small gap.

Bridging small defects like this should not be a problem unless you have really specific impedance requirements, but that's a reason to make more than one board at a time, buy better equipment, change the fabrication method, or outsource the boards.

For having made only a few boards, and although in this case the 10 mils trace had a defect, I consider being able to produce electrically viable 5 mils traces and even QFN-16 packages an excellent result. Good enough for prototyping, and doing away with messy cables on more complex projects, as well as being able to use more modern SMD components, which were our goals from the start.

![closeup](assets/images/etching-a-pcb/closeup.jpeg){: w="400" h="400" }
_Another close up shot of the finished board, where the resolution and in general, quality of the QFN package and 5 mils trace can be better seen._