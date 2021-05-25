These are files related to CAT, the ATmega328p that serves as CERBERUS 2080™'s I/O processor and system master. 'boards.txt' should be used to change the registery of a standard ATmega328p so to prepare it for use with CERBERUS (see Manual in the root directory). The other file is CERBERUS's BIOS (Basic Input/Output System) code, which runs on CAT. Note that the BIOS code is now moved to a subdirectory to support Arduino IDE compilation in place.

# 0xFE BIOS

The 0xFE BIOS is an update of the CAT BIOS to add new features focussed on developer convenience and games software. It includes:

* Serial keyboard input via FTDI
* Data upload over serial (Assemble directly to CERBERUS)
* Switchable 50Hz interrupts for time-sensitive applications
* Switchable keyboard buffer for CPU access to scan codes (key up/down)

# Serial Input

Connecting an FTDI <-> USB adaptor to the Adaptor Port allows a host computer to send keyboard input over serial. Configure the host for 9600 baud, 8N1 and use
a serial console to send ASCII values to CERBERUS. The PS2 keyboard works as normal, but input from either the keyboard or serial link will be interpreted by 
the CAT BIOS, and passed to the running CPU via the mailbox as usual.

Note that lines are terminated with a Carriage Return ('\r'), not a Newline ('\n').

# Data Upload over Serial

The CAT data upload command has been extended to support and report checksums over serial. This allows host applications to reliably upload data to CERBERUS
without the need to restart or swap memory cards. This is supported by the [Cerberus Emulator](https://feertech.com/legion/cerberus.html) so that code can
be assembled and uploaded directly from a compatible browser (Chrome or Edge).

## Command format:

`0xAAAA  BB BB BB...  [#CCCC]` - Load bytes `BB..` from address `0xAAAA` and optionally compare with checksum `CCCC`. (All values are hexadecimal)

If a checksum is not given, the data is uploaded as per the original CERBERUS BIOS. If a checksum is given and does not match the calculated value,
CAT will beep and display the upload address above the status line. In addition, the start address and calculated checksum are sent over the serial
link (if connected) so that a host application can verify that data has been correctly received.

The checksum is calculated as follows (pseudocode):

```
  uint8  checkA = 1
  uint8  checkB = 0

  for( value in bytes_input ) {
     checkA += value;
     checkB += checkA;
  }

  uint16 checksum = (checkA << 8) | checkB;
```

So as not to overwhelm the serial link, host applications should send at most eight bytes and then verify the received checksum before sending additional data.

# Interactive Mode

In some situations, it is useful to have a time reference and more direct access to keyboard events so that software running on the CPU can respond accurately.
The **0xFE** BIOS supports 'interactive mode', which enables a 50Hz interrupt and a 16 byte keyboard buffer that the CPU can access to respond to key down 
and key up events.

## Toggling Interactive Mode

When running, the CPU should write the value `0xFE` to the CERBERUS Mailbox address, 0x200. Interactive mode will be enabled *on the next key press*. Note that 
the keyboard buffer should be initialised if required before starting Interactive Mode.

To leave Interactive Mode, the CPU should write any other value to the Mailbox address. Interactive mode will end *on the next key press*.

Note that the Escape key will return control to the CAT BIOS in all situations.

## 50Hz Interrupt

In Interactive Mode, a 50Hz non-maskable interrupt is sent to the CPU. Note that the interrupt is not synchronised to the display.

## Keyboard Buffer

PS2 Scan codes are written to a 16 byte circular buffer at address `0xFE00`. New bytes are written by CAT to the `head`, and read by the CPU
from the `tail`. The `head` location is stored at address `0xFE10`, and the `tail` location at address `0xFE11`. 

Before entering Interactive Mode, these two values should be set to `0`.

The CPU should regularly check the `head` value. If it is not the same as `tail` then the byte at `0xFE00 + tail` is valid. `tail` should then be
incremented by one (modulo 16) and the check repeated. Note that if the buffer is full, CAT will discard additional key scan codes until space
becomes available.

# Additional Changes

A number of additional small changes have been made to the BIOS to support the new features. These include:

* Code changes to reduce memory consumption where possible
* Empty command lines are ignored
* BIOS version is reported on boot

# Building the BIOS

The BIOS can be built in exactly the same way as the normal CERBERUS BIOS. Note however, that this version requires the most up to date release of Paul Stoffregen's
PS2 Keyboard library. This is **NOT** the version that is installed by default by the Arduino Library Manager. The latest version (confusingly still
reported as v2.4) is available at the GitHub repository here: https://github.com/PaulStoffregen/PS2Keyboard

Alternatively, the pre-built BIOS is available here as a `.hex` file and can be uploaded to CAT with an Arduino hex loader.
