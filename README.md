# Vega816 - Dual 65C816 Computer with Vector Pull rewrite
Based on Adrien Kohlbecker's BB816 project.

## CPU Buffer Board

The CPU Buffer board provides two straight-wired connectors for the BB816 CPU breakout board, called "the CPU end", and buffers communications between the two of them and dual output connectors with the same pinout, called "the DMA end".

The extra straight wired connectors are included for ease of stacking the boards, both on the CPU end, and on the DMA end, in a multiplexed configuration, to implement shared memory multiprocessing. 

The circuit uses a 74AHCT245 octal bus transceiver for bi-directional communication for the 8 data lines, and seven 74HC541 octal buffers for the remaining signals provided by the BB816 CPU breakout board, including VPA and VDA. One of the buffers is allocated to CPU inputs (NMIB, IRQB, ABORTB, RDY_IN, etc), buffering signals from the DMA end. The remaining 74HC541s buffer lines from the CPU end onto the DMA end.

The bus transceiver and buffers' output enable lines are tied together and provided as an input called DMAB, so that a single signal suffices to turn communication on and off.

## DMA Control Shim and Vector Pull Rewrite Shim

If we assign each DMA channel its own address space, we can use address decoding in order to select which DMA channel is to be accessed by a particular CPU. In order to do that, we need to intercept the address lines from the CPU, pass decoding information to the DMA controller, and potentially rewrite the address being asserted by the CPU, before granting the CPU buffer access to the correct DMA channel.

The W65C816's VPB (Vector Pull) line also allows rewriting of the address asserted by the CPU when it is pulling an interrupt vector. It is convenient to provide hardware to intercept and rewrite the vector pull address as part of the same adapter.

Together, the DMA Control Shim and the Vector Pull Rewrite Shim provide facilities for DMA control and Vector Pull Rewrite to be performed by external hardware. 

### DMA Control Shim

The DMA Control shim provides movable jumpers for each of the bits A16-A23. Any of the bits may be selected as DMA_REQ. Once the jumper has been set to select one of those lines, it is connected as the DMA_REQ output, and disconnected from output to the DMA end, and is in fact pulled low on the DMA end (10K resistor), in order to map the request into a lower part of the DMA channel's address space.

The jumper offers the user a choice of granularity of DMA channel address mapping, in multiple of 2 increments from 64K (single bank, the minimum granularity) to 4M maximum. This allows for the address space to be contiguous across all DMA channels, depending on installed memory sizes. Ordinarily, this jumper should be set to the size of the maximum address space serviced by the hardware on either DMA channel, e.g. the total installed memory on that channel, or half the total memory across the two channels.

### Vector Pull Rewrite Shim

A 74HC283 adder is used to add an offset to A1-A4 equal to the three bit number provided as input by Y0-Y2. This takes place only when VPB is active (low) and A1-A3 are all high, indicating that an interrupt vector is being fetched by the processor. This has the effect of altering the fetch of the IRQ vector at $FFEE to a fetch of a vector at one of the following eight addresses: 
```
$00FFEE IRQ 0 
$00FFF0 IRQ 1 
$00FFF2 IRQ 2 
$00FFF4 IRQ 3 
$00FFF6 IRQ 4 
$00FFF8 IRQ 5 
$00FFFA IRQ 6 
$00FFFC RESET 
```
Y0-Y2 are pulled low by 10K resistors, for cases in which no Vector Pull controller is connected. Bypass jumpers are also provided for A1-A4, in case the 74HC283 adder and supporting logic are not installed.

### Details of Vector Pull address rewriting
The W65C816 datasheet gives this table of interrupt vectors which lie between $00FFE4 and $00FFFC
```
Address Function Last Octet (binary)
$00FFE4 COP      0x1110 0100
$00FFE6 BRK      0x1110 0110
$00FFE8 ABORT    0x1110 1000
$00FFEA NMI      0x1110 1010
$00FFEE IRQ      0x1110 1110
$00FFFC RESET    0x1111 1100
```
The datasheet also specifies that when the 65C816 is addressing an interrupt vector, it pulls its VPB line low. This is to allow the system to override the fetched interrupt vector with any other. 

VPB is pulled low for any interrupt vector fetch, not just IRQ, but the present design purposefully limits Vector Pull rewriting to only those cases when IRQ is the vector being fetched.

Note that the last nybble of each of the standard vector is unique, and therefore a test of these four bits suffices to uniquely identify the vector being pulled. In particular, both bytes of the two-byte IRQ vector ($FFEE and $FFEF) are the only two bytes in this table with $E (0x1110) as the last nybble of address, i.e., A1-A3 high.

Note also the gap of 12 bytes between the IRQ vector and the RESET vector. Since each vector occupies two bytes, this gap provides sufficient space to specify six (6) additional vectors. 

```
$00FFE4 COP	
$00FFE6 BRK
$00FFE8 ABORT 
$00FFEA NMI 
$00FFEE IRQ 0 
$00FFF0 IRQ 1 
$00FFF2 IRQ 2 
$00FFF4 IRQ 3 
$00FFF6 IRQ 4 
$00FFF8 IRQ 5 
$00FFFA IRQ 6 
$00FFFC RESET 
```
If an offset 0-6 is added to A1-A3, (leaving A0 as found), then the vector fetched will be as shown above, going forward from IRQ0.

## DMA Controller

The DMA Controller is intended to control the multiplexing of one or two CPUs (called CPU A and CPU B) onto one or two communication channels (called DMA 0 and DMA 1). The two channels are cross-mapped in address space from the point of view of opposite processors, depending on a selected granularity. For example, if 64 KB (single bank) granularity is selected, when CPU A requests Bank 0, the DMA controller will connect it to DMA channel 0, whereas if CPU A requests Bank 1, the controller will connect it to DMA channel 1. The situation is reversed for CPU B. The controller will direct its requests for Bank 0 to DMA channel 1, and requests for Bank 1 to DMA channel 0.

This permits each CPU to have its Bank 0 lie within the address space of a separate physical DMA channel, so that accesses by each CPU to its own Bank 0 will not normally interfere with the opposite CPU's requests for its own Bank 0.

The effect for higher memory is that each CPU's odd numbered banks are the other CPU's even numbered banks, interleaved to the top of physical memory.

The granularity of the interleaving is specified by a jumper on the DMA Controller Shim, and provided to the DMA controller as the signal DMA_REQ. Additionally, the following CPU signals are utilized in implementing DMA:
```
RDY_IN
ABORTB
VA
```


The DMA controller monitors the VA (Valid Address) from one or two CPUs, as well as an additional input, DMA_REQB. If one CPU asserts DMA_REQB, the controller checks the VA line from the opposite CPU. If it is not active, the controller asserts DMA







## Notes
The 65816 features hardware support for these external features:
* Protected memory operations using ABORTB.
* Independent external caching of instructions (VPA) and data (VDA).
* Interrupt prioritization, via vector rewriting when VPB is active. (Seven apparently reserved contiguous vectors for IRQ $FFEE-$FFFA).
* Cycle steal DMA (VPA and VDA, and RDY).

The present design aims to implement all of these features (protected memory operation, caching, interrupt prioritization, and DMA operations) based on the Kohlbecker BB816 breakout board. 

Each bus mastering device can attach its own data bus to either DMA data bus. Address translation insures that each channel sees the two 512 K RAM segments in the opposite order in address space. In other words, Bank 0 for channel B begins at the 512K point for channel A, and vice versa. 

At startup, the primary CPU is set to DMA channel A, which talks to the lower 512K of physical RAM. The secondary CPU is set to DMA channel B, seeing the upper 512K of physical memory as beginning at its own address 0. 

