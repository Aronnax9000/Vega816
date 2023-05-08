# Vega SMP
A Multiprocessor Implementation of the Kohlbecker 65816 Breakout Board

Features:
* WDC 65C816 processors share alternating clock cycles: no bus contention.
* Up to three auxiliary processors.
* On reset, RDY line of secondary processor is held low by a VIA under control of the primary.
* On reset, primary copies EEPROM into shadow RAM in the upper 32K of its Bank 0, then enters Native mode. The E line goes low, which banks out the ROM and speeds up the clock.
* Entering native mode, primary loads instructions and initial state into the data bank mapped as bank 0 to the secondary processor, and brings RDY high on CPU1 via the dedicated VIA.
* Memory is mapped so that each processor sees a 64K offset (one data bank) into shared memory with respect to its predecessor. Each CPU therefore has its own Bank 0.
* Dedicated VIA 65C22 for bus, CPU control, and SPI operations.
* Dedicated VIA 65C22 for front panel operations.
* Dedicated ACIA W65C51N for RS-232 console operation.
* Software IRQ steering for up to seven add on peripherals (shared VIAs, etc.).
* IRQ Priority routing. When VPB is low, the LSB of the base address supplied by the processor is rewritten according to the priority of the interrupt received.

Here is the interrupt vector table for the 65816, augmented to show the
extra vectors
```
$00FFE8 ABORT 
$00FFEA NMI 
$00FFEE IRQ0 
$00FFF0 IRQ1 
$00FFF2 IRQ2 
$00FFF4 IRQ3 
$00FFF6 IRQ4 
$00FFF8 IRQ5 
$00FFFA IRQ6 
$00FFFC RESET 
```
