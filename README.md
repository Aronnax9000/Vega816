# Vega SMP
A Multiprocessor Implementation of the Kohlbecker 65816 Breakout Board

Features:
* Twin WDC 65C816 processors share alternating clock cycles: no bus contention.
* VIC-II style DMA takes place on the primary clock cycle, stealing cycles from primary processor.
* On reset, RDY line of secondary processor is held low by a VIA under control of the primary.
* On reset, primary copies EEPROM into shadow RAM in the upper 32K of its Bank 0, then enters Native mode. The E line goes low, which banks out the ROM and speeds up the clock.
* Entering native mode, primary loads instructions and initial state into the data bank mapped as bank 0 to the secondary processor, and brings RDY high on CPU1 via the dedicated VIA.
* Memory is mapped so that each processor sees a 64K offset (one data bank) into shared memory with respect to its predecessor. Each CPU therefore has its own Bank 0.
* Dedicated VIA 65C22 for bus, CPU control, and SPI operations.
* Dedicated VIA 65C22 for front panel operations.
* Dedicated Dual ACIA W65C51N for RS-232 console and paper tape operation.
* Support for up to 8 additional VIAs with IRQs fully software steerable across four timesliced CPUs.
* Software IRQ prioritization for up to seven independent IRQ handlers, per processor.
* IRQ Priority routing. When VPB is low, the LSB of the base address supplied by the processor is summed with twice the priority value before delivery to the bus for vector lookup. Lower number means higher priority.

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

# Physical Design
* Bus Mastering: Four Kohlbecker 65816 breakout style slots with auxiliary connections for VDA/VPA. Each slot is capable of bus mastering. (e.g. VIC-II or DMA controller).

* Four expansion slots supporting two devices apiece (e.g. two VIAs) with independently routable/prioritized IRQs. Expansion ports are memory mapped to $0280 to $02FF, 16 bytes per device, 32 bytes per card.

* Each of the eight slots has a 8 bit latch for settings, delivered by the VIA in charge of the bus. To set the latch, the VIA first asserts the 8 bits of data on PB. Then the device number is asserted on PA5 and PA4, triggering the latch on the appropriate device, and the settings from PB are latched in to the device's settings latch.
```
Bus Master Slot Settings
7 6 5 4 3 2 1 0
T               Timeslice (0 = CLK0, 1 = CLK1)
  D             DMA only (monitor VDA/VPA of master, only go if low).
    P           Puppy bit: can anti-RDY the master on this timeslice.
              R RDY (Enable).

Expansion Slot Settings

7 6 5 4 3 2 1 0
I I I            Interrupt priority 0-7 (7 = RESET!)
      M M        Bus master slot to receive IRQ 0-4
              R  RDY (Enable).
      
```
