# Boot sequence

## Hardware boot sequence
From rp2040-datasheet.pdf, section 2.8.1:

1. Both cores enter bootrom
2. Which core(CPUID) am I?
    1. If CPUID == 1 -> Sleep until given entry point through FIFO mailbox
3. PoR rescue flag set?
    1. If set: Clear flag and halt
4. Watchdog boot-to-SRAM set?
    1. If set: Set SP and jump to entry point
5. 100 us delay (pullup on flash CSn)
6. Read flash CSn (bootsel button)
    1. If Low(USB device), jump to X
7. Configure SSI and connect to pads
8. Load 256 bytes from flash (to SRAM5)
9. Checksum pass?
    1. If no, has more than 0.5s passed since boot?
        1. If Yes, goto 10.
        2. If No, Increment CPOL, CPHA, delay 100us and goto 8
    2. If yes, execute code at SRAM5 (diverges)
10. Start crystal oscillator
11. Crystal present?
    1. No: halt
12. Start PPLs, Sys, USB clocked at 49 MHz
13. Enter USB device mode bootcode

## Flash boot second stage
> The flash second stage must configure the SSI and the external flash for the best possible execute-in-place
performance. This includes interface width, SCK frequency, SPI instruction prefix and an XIP continuation code for
address-data only modes. Generally some operation can be performed on the external flash so that it does not require
an instruction prefix on each access, and will simply respond to addresses with data.
Until the SSI is correctly configured for the attached flash device, it is not possible to access flash via the XIP address
window. Additionally, the Synopsys SSI can not be reconfigured at all without first disabling it. Therefore the second
stage must be copied from flash to SRAM by the bootrom, and executed in SRAM.
Alternatively, the second stage can simply shadow an image from external flash into SRAM, and not configure executein-place.
This is the only job of the second stage. All other chip setup (e.g. PLLs, Voltage Regulator) can be performed by
platform initialisation code executed over the XIP interface, once the second stage has run.

There is code at [pico-sdk/src/rp2\_common/boot\_stage2](https://github.com/raspberrypi/pico-sdk/tree/6a7db34ff63345a7badec79ebea3aaef1712f374/src/rp2_common/boot_stage2) for different devices and one generic one.

TODO: How to know what flash device I have?

## Reset vector
The second stage flash boot program will return at the reset vector in the vector lut at (XIP\_BASE + 0x100). Why 0x100??
