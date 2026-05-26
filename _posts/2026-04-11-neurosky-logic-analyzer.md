---
title: Decoding the communication interface from a Neurosky Mindflex module
description: Used a logic analyzer and PulseView to reverse engineer the Neurosky ThinkGear UART protocol. Prototyped a packet parser in Python to decode raw EEG band data and validate checksum integrity.
date: 2026-04-11
categories:
  - Blog
  - Electronics
tags:
  - uart
  - logic-analyzer
  - protocol-decoding
  - neurosky-thinkgear
  - reverse-engineering
  - hardware-hacking
  - embedded-systems
math: true
mermaid: false
---

As part of a EEG prototype I am working on, we are using the neurosky modules that come inside Mindflex toys, or similar, in order to read brainwaves. This module uses two electrodes, one as a reference, and gives us the magnitude of the following band frequencies[^5]

| Band       | Range   |
| ---------- | ------- |
| Delta      | 1-3Hz   |
| Theta      | 4-7Hz   |
| Low alpha  | 8-9Hz   |
| High alpha | 10-12Hz |
| Low beta   | 13-17Hz |
| High beta  | 18-30Hz |
| Low gamma  | 31-40Hz |
| High gamma | 41-50Hz |

The module also outputs two estimators, one called attention, which is a measure of high frequency activity in the brain, and meditation, which is a measure of low frequency activity on the brain.

The module consists of a ThinkGear module plugged into an rf board, although I couldn't find any more information on whether this board served another purpose.

After some searching, I came to the conclusion that the device uses the [ThinkGear Communication Protocol](https://developer.neurosky.com/docs/doku.php?id=thinkgear_communications_protocol) (if you can't open the link, try using the wayback machine, hopefully it doesn't get taken down). Although there is an [Arduino library](https://github.com/kitschpatrol/brain) already, I think there is much to gather from inspecting the protocol oneself, and it is a good excuse to get a logic analyzer.

> This post focuses only on decoding the UART protocol of a Neurosky ThinkGear module. Due to project constraints, I’ll focus only on the signal acquisition and decoding process.
{: .prompt-info}

## Setup

### Power pins

First, we need to power it. I cut in half two of those pre-made dupont female cables, stripped them, and soldered them to the power terminals on the circuit. The documentation states if works on 2.6-3.6v, and, at least the ThinkGear module pulls about 15mA @ 3.3v. It should run fine on an MCUs 3.3v supply.

![Power cables soldered to the board](assets/images/neurosky-logic-analyzer/neurosky_power_cables.jpg){: w="400" h="400" }
_Power cables soldered to the board._

### Communication pins

The documentation states it communicates through UART. There are markings on the module which denote the transmission (Tx) and reception (Rx) pins with "T" and "R" respectively. I soldered dupont wire with female terminals as in the previous step.

![Comms cables soldered to the board](assets/images/neurosky-logic-analyzer/neurosky_comm_cables.jpg){: w="400" h="400" }
_Communication ports cables soldered to the board. on the top left of each pin the silkscreen annotation is barely visible._

Sadly, the markings on the board are not that visible, as there is some burning from previous use, but the yellow wire corresponds to "T" and the orange wire corresponds to "R".

Powering it through the 3v pin on an arduino nano, we got it to turn on.

Then we connect both data pins to signal pins on the logic analyzer, and since we are using the ground from the arduino, we also connect the ground from the logic analyzer to a GND pin on the arduino.

![Connection diagram](assets/images/neurosky-logic-analyzer/neurosky-diagram.png){: w="400" h="400" }
_Diagram of how i connected all of the modules together._

## Reading the data

I am using a cheap chinese logic analyzer, with [PulseView](https://sigrok.org/wiki/PulseView), to see the waveforms on the logic analyzer.

The documentation states that depending on the position of two resistors on the board and/or additional configuration it may use a baudrate of 1200, 9600, or 57.6k.

I set it to 200kHz, which is plenty if we assume the baudrate to be 9600, and I took 1M samples, to see many packages being sent.

![Packages being visualized in pulseview](assets/images/neurosky-logic-analyzer/nsky-packages.png)
_We can see that we captured three packages._

In the image of the data taken we can see the three packages that were sent on the Tx line, and nothing happening on the Rx line, which is why we are able to use it with one wire.

Zooming in on one frame, and measuring the width of the smallest data bit I found, we can indeed see that the bit width is about 9.5kHz, so it is safe to assume it is operating at 9600 baud.

![Baudrate visualized in pulse view](assets/images/neurosky-logic-analyzer/nsky-baudrate.png)
_The duration and corresponding frequency can be seen at the top of the highlighted block._

This baudrate changes if the operating mode is changed, as stated in the documentation.
### UART Protocol

#### Start and end bits

UART stands for **Universal Asynchronous Receiver/Transmitter**, since it is asynchronous, we need a way for the receiver to know a transmission has started[^2].

The most common way this is implemented is to pull the line low for the duration of a bit, which PulseView already marks as "S" in yellow.

Both devices must know then, the number of data bits in the transmission, which in this case is 8. At the end, the transmitter pulls the line high for the duration of a bit and a half or two bits to signal the end of the transmission.

Again, PulseView already marks these bits; the data bits are automatically decoded to decimal, and the stop bit is marked with a "T" in orange. 

![Start and end bits being visualized in pulseview](assets/images/neurosky-logic-analyzer/nsky-bytes.png)
_We see the start bits highlighted in yellow, and the stop bits highlighted with orange._

#### Parity bits

Although not the case, sometimes an extra bit is included before the stop bit for naive error checking[^1].

For example, we could make this bit 0 if the number of ones in our data is even, and 1 if it is odd, which would allow us to discover if an odd number of bits changed. It is not very powerful, but useful for early error detection.

### Signal properties

So far, these are the properties of the signal.

| Property     | value         |
| ------------ | ------------- |
| Protocol     | UART          |
| Baud rate    | 9600          |
| Data bits    | 8             |
| Bit order    | lsb first     |
| Parity       | none          |
| Package size | 170 bytes max |
| Package rate | ~1Hz          |

> When using an ADC, such as the ADS1115, it takes certain amount of analog samples, processes them, and outputs a result on a deterministic time according to a clock speed. As we are reading analog voltages from the electrodes, one could presume this device also uses an ADC of such sorts, but I measured the times between packages, from the end of a package to the start of a new package, and it is not the same between packages. In use, the serial peripheral of an MCU handles serial with interrupts, and should be enough to get the maximum polling rate possible.
{: .prompt-info}
## Decoding the data

Once we are sure we are reading the bits correctly, we need to know what each byte stands for[^3].

The documentation states each package corresponds of a header, the payload, and a checksum.

### Header bytes

As in many other applications, headers provide useful metadata. In this case, we use it to know we are getting a package from the beginning, and to know we are reading the data correctly. It could also be used, for example, to identify the device sending the data.

From the documentation, we know the first three bytes are the header. Byte 1 and byte 2 mark the beginning of the package, and byte 3 marks the amount of data bytes we have in the package. In this case, byte 1 is 170, byte 2 is 170 as well, and byte 3 is 32.

![Header being visualized in pulseview](assets/images/neurosky-logic-analyzer/nsky-header.png)
_The first three purple blocks delimit the header bytes._

In our driver, with the first two bytes, we would check for a value of 170 followed by another value of 170 to know we are reading the package correctly. So for example, if we turn on the receiver in the middle of a transmission, it would immediately know to discard that package.

Then, we would use byte 3 to know we need to read and store the next 32 bytes of data.

But then, we have already said the package contains 36 bytes, of which the first three are the header, then 32 data bytes, so what is the last byte for?

### The checksum

The last byte is the value of the checksum. A checksum is a data block derived from another data block used to detect transmission errors, and verify data integrity[^4].

In this case, the checksum is obtained by adding all of the data bytes, ignoring overflow, and taking the one's complement (which just means inverting all the zeroes and ones).

This way, the receiver can add up all of the data bytes and the checksum, and if the complement is non-zero, know that the package was corrupted in the transmission, and discard it.

> There are error correcting codes, such as the [Hamming code](https://en.wikipedia.org/wiki/Hamming_code), which not only allow for error detection, but enable the receiver to know where the error is, and thus correct it.
{: .prompt-info}

### Data bytes

So we already have the data bytes, and have already verified no data was lost or transmitted. How do we decode it?

The package is structured as several blocks of `[code][data]`, where each code tells us how the data in that block should be read.

#### Single-byte codes

These codes state that the data is only a single byte.

| Code | Meaning        |
| ---- | -------------- |
| 0x02 | signal quality |
| 0x03 | heart rate     |
| 0x04 | attention      |
| 0x05 | meditation     |
| 0x06 | 8-bit raw wave |
| 0x07 | raw marker     |

#### Multi-byte codes

These codes state the size of the data to be read

| Code | Size | Meaning                     |
| ---- | ---- | --------------------------- |
| 0x80 | 2    | 16-bit Raw wave             |
| 0x81 | 32   | float EEG frequency bands   |
| 0x83 | 24   | integer EEG frequency bands |
| 0x86 | 2    | time between two R-peaks    |

#### Reserved codes

| Code | Meaning                                     |
| ---- | ------------------------------------------- |
| 0x55 | Used to represent an extended code (EXCODE) |
| 0xAA | Header sync bytes                           |

In our case, we are only using:

- **Signal quality:** Number between 0 and 255, where any non-zero value indicates low signal integrity, and results in attention and meditation not being calculated. A value of 200 is special, in that it means the electrode is not not contacting the person's skin. It is output around every second.
- **Attention**: Ranges from 0 to 100, and is a measure of high frequency activity of the brain. It is output around every second.
- **Meditation**: Same properties as attention, but is a measure of low frequency activity.
- **Integer EEG frequency bands**: Eight big endian 24-bit unsigned integers that correspond to the frequency bands described at the beginning of the post. They are also output around every second.

In our case, by default, our packages are structured like

```
[Header][Signal quality][Integer EEG frequency bands][Attention][Meditation][Checksum]
```

## Driver Simulation

Knowing exactly how the data is organized in the signal, we can now make a driver. Before implementing one in C++ for the arduino, I am going to make a quick sketch in python to simulate the logic.

The MCU normally deals with reading the serial port, and gathering the bytes, so for now I will focus on what to do with the package itself.

PulseView has several export options, including exporting the annotations from the UART decoder, which i will do for a single, and gives us a file like this

```
251335-251356 UART: TX data: Start bit
251356-251523 UART: TX data: 170
251523-251544 UART: TX data: Stop bit
251542-251563 UART: TX data: Start bit
251563-251730 UART: TX data: 170
251730-251751 UART: TX data: Stop bit
251749-251770 UART: TX data: Start bit
251770-251937 UART: TX data: 32
251937-251958 UART: TX data: Stop bit
251956-251977 UART: TX data: Start bit
```

First we need to parse the file. Only the numbered data bytes are of our interest, and with a regular expression such as `\ \d{1,3}` we can select them, and store them in an array

```python
import re

# Prepare the array in which the bytes will be
bytes = []

with open("export", "r") as file:
	for line in file:
		# Select only the data byte
		data = re.findall(r"\ \d{1,3}", line)
		# Select only non empty selections
		if data:
			# Append the data, not the array
			bytes.append(int(*data))
```

We will use `cycle` from itertools to simulate a never-ending stream of the package, whose next element can be accessed with `next`.

```python
from itertools import cycle

# Simulate the data stream
stream = cycle(bytes)
```

Then we read the package

```python
import time

while True:

	# Added a short delay for readability
	time.sleep(0.2)
	
	# Prepare the array in which we will store the package
	package = []
	
	# Prepare the variable in which we will store the checksum
	checksum = 0
	
	# Check for the first header bit
	if next(stream) != 170:
		continue
	
	# Check for the second header bit, to know we are synced
	if next(stream) != 170:
		continue
	
	print("Package synced!")
	
	# Save the package length
	data_length = next(stream)
	
	# verify package length is within bounds
	if data_length > 170:
		continue
	
	print("Valid package length!")
	
	# Store the data of the package
	for i in range(data_length):
		package.append(next(stream))
	
	# Store the checksum
	checksum = next(stream)
	
	# Add all read bytes to the checksum
	for byte in package:
		checksum+=byte
	
	# Invert the checksum, and check if it is zero
	if ( ~(checksum % 2**8) & (2**8 - 1))!=0:
		continue
	
	print("Package verified!")
```

Since python endows us with arbitrary length integers, we calculate the $\text{mod}(2^{8})$ to get the number we would have gotten ignoring the overflow.

Moreso, thanks to this arbitrary length, the inversion of a negative number won't give its two's complement, e.g `bin(-3)=-0b00000011` instead of `bin(-3)=0b11111101`, which is why use a bitwise AND operator, which forces the result to use 8 bits, and then the inversion treats it as an 8 bit unsigned integer.

Now that we always have a valid package stored in `package`, we must parse it using the codes described in the previous section

```python
# Definitions of the codes in use
signal_quality_code = 0x02
eeg_frequencies_code = 0x83
attention_code = 0x04
meditation_code = 0x05

# Prepare the variables in which the information will be stored
signal_quality = 0
eeg_frequencies_size = 0
eeg_frequencies = []
attention = 0
meditation = 0

# Prepare the index
i = 0

# While the package is not fully parsed through
while i < data_length:

	# For single-byte codes, read the next byte and advance
	# the index accordingly
	if package[i] == signal_quality_code:
		signal_quality = package[i+1]
		i+=2
	elif package[i] == attention_code:
		attention = package[i+1]
		i+=2
	elif package[i] == meditation_code:
		meditation = package[i+1]
		i+=2
	
	# For multi-byte codes, we read the size of the data,
	# advance the index, and in this case, read the next three bytes
	# bit shift each package into place,
	# and XOR the 24-bit big endian number together
	elif package[i] == eeg_frequencies_code:
		eeg_frequencies_size = package[i+1]
		i+=2
		for j in range(0,eeg_frequencies_size,3):
			eeg_frequencies.append(
				(package[i+j] << 16) | (package[i+j+1] << 8) | (package[i+j+2])
			)
	i+=24

# Visualize the output
print(f"signal quality: {signal_quality},attention: {attention}, meditation: {meditation}")
for i, magnitude in enumerate(eeg_frequencies):
	print(f"{i}: {magnitude:024b}")
```

and we get as an output

```
signal quality: 200,attention: 0, meditation: 0
0: 000000000000010001001101
1: 000000000000010011000010
2: 000000000000001101000100
3: 000000000000000111011100
4: 000000000000011000101100
5: 000000000000011000000111
6: 000000000000001001001011
7: 000000000000001000101101
```

Since we do not have any electrode, it corresponds that the signal quality is 200, and we don't have any attention or meditation values.

Then we have our 24-bit numbers. Lets check they are stitched together correctly.

![big integer being visualized in pulseview](assets/images/neurosky-logic-analyzer/nsky-bigint.png)
_The last three purple blocks delimit the first of the eight 24-bit integers._

Here we can see the code (0x83 is 131 in decimal), the number of bytes the data for that code has (since it is multi-byte) and the data bytes of the first number.

Since it is transmitted lsb-first, the bytes are read like `00000000`, `00000100`, `01001101`. It being a big-endian number means the msb is transmitted first, and the lsb is transmitted last. Therefore, we know it must be stitched like

```
00000000 00000100 01001101
```

Comparing it with our result from the code

```
0: 000000000000010001001101
```

We can see it is the same, so it was read correctly.

Having the logical flow of the driver down, the next step is to port it to arduino/esp32, where we can now focus on platform specifics, optimization, and error handling, instead of logic.

In future posts i will cover changing modes, to get the 16-bit wave data, and integration with embedded systems.

## References

[^1]: https://en.wikipedia.org/wiki/Parity_bit#

[^2]: D. S. Dawoud and P. Dawoud, **Serial communication protocols and standards: RS232/485, UART/USART, SPI, USB, INSTEON, Wi-Fi and WiMAX**. Aalborg: River Publishers, 2020. doi: [10.1201/9781003339496](https://doi.org/10.1201/9781003339496).

[^3]: https://www.analog.com/en/resources/analog-dialogue/articles/uart-a-hardware-communication-protocol.html

[^4]: https://en.wikipedia.org/wiki/Checksum

[^5]: https://support.neurosky.com/kb/science/eeg-band-frequencies
