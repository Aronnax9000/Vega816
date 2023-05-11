# Vega BB816 
A Multiprocessed Implementation of the Kohlbecker 65816 Breakout Board.

Adrien Kohlbecker has developed a breakout circuit board for the WDC 65C816 microprocessor. The breakout board demultiplexes exposes all of the 65C816's control, address, and data lines. All signals are provided on a convenient row of breakout pins, plus two connections provided for VDA and VPA outputs.

The 65816 features hardware support for these external features:
* Protected memory operations using ABORTB.
* Independent external caching of instructions (VPA) and data (VDA).
* Interrupt prioritization, via vector rewriting when VPB is active. (Seven apparently reserved contiguous vectors for IRQ $FFEE-$FFFA).
* Cycle steal DMA (VPA and VDA, and RDY).

The present design aims to implement all of these features (protected memory operation, caching, interrupt prioritization, and DMA operations) based on the Kohlbecker BB816 breakout board. 

Two processors are arranged in a cycle steal configuration.

Through-hole RAM has a worst case access time of . It is a little lower than the clock speed that the Commodore 64's VIC-II chip used as its master frequency. (NTSC: 14.31818, PAL:  17.734475 MHz)


Time-slice multiprocessing may also be built to respect "dynamic logic", access to the bus on the rising edge by one device, and on the falling edge by the other.

Supported clock speeds:
```

NTSC VIC-II: 14.31818  MHz x2 = 28.63636 MHz x2 = 57.27272 MHz.
             70 ns = 14.285714 MHz
PAL VIC-II:  17.734475 MHz x2 = 35.46895 MHz x2 = 70.9379 MHz.
             55 ns = 18.18181 MHz.
```
The VIC-II chip does clock division by 8 in order to provide PHI2.
```
NTSC: 14.31818  MHz /8 = 1.7897725 MHz
PAL:  17.734475 MHz /8 = 2.216809375 MHz
```

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
* Bus Mastering: Four Kohlbecker 65816 breakout style slots with auxiliary connections for VDA/VPA. Each slot is capable of bus mastering. (e.g. VIC-II or DMA controller). Slots 0 and 2 are on CLK0 and slots 1 and 3 are on CLK1. The normal usage pattern is for device 2 to perform DMA operations during VDA+VPA low from device 0, and similarly, device 3 during VDA+VPA low on device 1.

* Four expansion slots supporting two devices apiece (e.g. two VIAs) with independently routable/prioritized IRQs. Expansion ports are memory mapped to $0280 to $02FF, 16 bytes per device, 32 bytes per card.

* Each of the eight slots has a 8 bit latch for settings, delivered by the VIA in charge of the bus. To set the latch, the VIA first asserts the 8 bits of data on PB. Then the device number is asserted on PA5 and PA4, triggering the latch on the appropriate device, and the settings from PB are latched in to the device's settings latch.
```
Bus Master Slot Settings
7 6 5 4 3 2 1 0
T               Timeslice (0 = CLK0, 1 = CLK1)
  D             DMA only (monitor VDA/VPA of master, only go if low).
    P           Puppy bit: anti-RDY the master on this timeslice while running.
      K         taKe control of the clock while RDY.
              R RDY (Enable).

Expansion Slot Settings

7 6 5 4 3 2 1 0
I I I            Interrupt priority 0-7 (7 = RESET!)
      M M        Bus master slot to receive IRQ 0-4
              R  RDY (Enable).
      
```
