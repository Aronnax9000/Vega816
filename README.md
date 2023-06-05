# Vega816 - Dual 65C816 Computer with Vector Pull rewrite

My goal is to demonstrate symmetrical multiprocessing and dual channel DMA with the W65C816 processor, beginning with Adrien Kohlbecker's fine BB816 CPU breakout board. 

The system also includes a programmable interrupt controller (PIC), which can be configured at runtime to direct IRQ from any device to one or the other connected CPU.

The interrupt handling system features hardware Vector Pull rewrite support for up to seven levels of IRQ prioritization, programmable per device (seven + RESB). The PIC architecture offers support for I/O devices with 8, 16, 32, and 64 byte address space requirements (e.g. VIA, ACIA, SID, VIC). Multiple PICs may be installed to support any number of devices whose combined address space will fit within that DMA channel's Bank 0. 

Memory modules are provided to provide RAM and ROM services. ROM can be separately banked in for read or write, or completely banked out for higher clock speeds.

An Expansion Bus board may be set, via jumper, to any of three pages in Bank Zero for the board's DMA channel: $0200, $0400, or $0600. The board splits its page of I/O address space across 4 expansion slots, each of which spans 64 bytes (1/4th) of the breakout's assigned page of address space, providing further address decoding into chip select signals for up to four devices per slot on 16 byte boundaries. The address decoder provides a signal to inhibit the selection of RAM within the assigned address range of the expansion bus. The Programmable Interrupt Controller (PIC) uses the page of RAM immediately above the assigned page of I/O address space in order to store each device's IRQ priority and target CPU, so the address decoder will assert Chip Select not just for the address space of any 16 byte device, but also for the corresponding range in the page above it. For example, the Chip Select line for a device mapped at $0240-$024F is also active for $0340-$034F.

The Programmable Interrupt Controller performs further subdivision of the 16 byte device granularity of the Expansion Bus into 8 byte device address spaces, such as used by the W65C51 ACIA.

Since the complete system supports two DMA channels, each with three two-page ranges reserved for I/O, supporting up to one PIC per each of four 64 byte expansion slot and each PIC supports up to eight devices per double-page, the complete system can support up to 2x3x4x8 = 

Finally, a reference System Controller based on the W65C22 VIA is presented, allowing software control over system hardware, such as clock speed, CPU and DMA control. 

## Limitations of the Design

The chief limitation of the present design is that it supports a maximum of two CPUs running against a maximum of two DMA channels. It is easy to imagine future directions. Additional CPU buffers, could be added to each processor in order to implement caching. In those cases when the CPU is attempting to reach address space within a busy DMA channel, a DMA controller could direct the CPU to a DMA channel representing cache. The fact that VPA and VDA are given treatment through the CPU Buffer board design, despite being offered only as test pins on the BB816 breakout board, is intended to facilitate exploration of that possibility in future designs, as maintaining separate caches for program code and program data will improve cache efficiency.

If the design were expanded to offer two separate bits of the bank address to the DMA controller instead of one, the address space could be sliced into up to four DMA channels, so that four CPUs could share the same address space. Such an architecture would lend itself to "Connection Machine" type multiprocessor topologies, with each CPU sharing address space with three of its neighbors.

Such an expansion would also necessitate a rework of the Programmable Interrupt Controller, as a device on any given DMA channel might be expected to direct IRQs to any one of four different CPUs.

The complicated logic presented here introduces timing delays which I have not yet modeled. These delays must certainly introduce limitations to the maximum clock speed of a 65816 based computer.

## CPU Buffer Board

The CPU Buffer board provides two straight-wired connectors for the BB816 CPU breakout board, called "the CPU end", and buffers communications between the two of them and dual output connectors with the same pinout, called "the DMA end".

The extra straight wired connectors are included for ease of stacking the boards, both on the CPU end, and on the DMA end, in a multiplexed configuration, to implement shared memory multiprocessing. 

The circuit uses a 74AHCT245 octal bus transceiver for bi-directional communication for the 8 data lines, and seven 74HC541 octal buffers for the remaining signals provided by the BB816 CPU breakout board, including VPA and VDA. One of the buffers is allocated to CPU inputs (NMIB, IRQB, ABORTB, RDY_IN, etc), buffering signals from the DMA end. The remaining 74HC541s buffer lines from the CPU end onto the DMA end.

The bus transceiver and buffers' output enable lines are tied together and provided as an active low input called DMAB, so that a single signal suffices to turn communication on and off. A 1K pulldown resistor ensures that the board is active unless DMAB is raised high by external hardware.

![CPU Buffer Board](schematics/CPU%20Buffer.svg)

## DMA Control Shim and Vector Pull Rewrite Shim

If we assign each DMA channel its own address space, we can use address decoding in order to select which DMA channel is to be accessed by a particular CPU. In order to do that, we need to intercept the address lines from the CPU, pass decoding information to the DMA controller, and potentially rewrite the address being asserted by the CPU, before granting the CPU buffer access to the correct DMA channel.

The W65C816's VPB (Vector Pull) line also allows rewriting of the address asserted by the CPU when it is pulling an interrupt vector. It is convenient to provide hardware to intercept and rewrite the vector pull address as part of the same adapter.

Together, the DMA Control Shim and the Vector Pull Rewrite Shim provide facilities for DMA control and Vector Pull Rewrite to be performed by external hardware. 
![CPU Buffer Board](schematics/Vega816-Vector%20Pull%20+%20Dual%20Channel%20DMA%20Control%20Shim.svg)
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

The connector for the external Vector Pull Rewrite controller provides the E signal from the CPU, so that the Vector Pull Rewrite controller can know which scheme of vector pull (W65C02, W65C816) is being used.

### CAVEAT: Offset of 7 triggers RESB interrupt handler

Please note that, when the 65816 is running in native mode, if an IRQ priority of 7 (0x111) is asserted on Y0-Y2, the vector offset will result in the RESET vector being pulled. Since no actual hardware reset has occurred, if the reset handler relies on a hardware reset having taken place, unpredictable state may occur. It is generally advised to avoid the use of IRQ priority 7 when the CPU is in native mode.

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

Note that the last nybble of each of the standard interrupt vectors is unique, which aids in designing a hook for any one of them (the same is true for 6502 Emulation mode, see below). A test of these four bits suffices to uniquely identify the vector being pulled. In particular, both bytes of the two-byte IRQ vector ($FFEE and $FFEF) are the only two bytes in this table with $E or $F (0x1110, 0x1111) as the last nybble of address, i.e., A1-A3 high.

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

### Effect of Vector Pull Rewrite in Emulation (65C02) Mode
The 6502's IRQ vector, like its 65816 counterpart, has 0x111x in the last nybble, so it is detected by the Vector Pull Rewrite Shim. 
```
$FFFA NMI
$FFFC RESB
$FFFE BRK/IRQ 
```
Unlike the 65816's IRQ vector, the 6502 IRQ vector occurs at the very top of its page, so additions of 1-7 to the bits may force A4 to overflow to 0 (no carry). In particular, $FFFE + $02 = $FFE0, etc. This means the augmented
IRQ vector map for the 6502 looks like this:

```
$FFE0 BRK/IRQ 1
$FFE2 BRK/IRQ 2
$FFE4 BRK/IRQ 3
$FFE6 BRK/IRQ 4
$FFE8 BRK/IRQ 5
$FFEA BRK/IRQ 6
$FFEC BRK/IRQ 7
...
$FFFA NMI
$FFFC RESB
$FFFE BRK/IRQ 0
```
## Dual DMA Channel to Dual CPU IRQ Dispatcher with 0-7 Priority

Devices mapped within either DMA channel may issue IRQ against either CPU. The W65C816 supports Vector Pull Rewrite through its VPB line. The dispatcher uses two 74LS348 8-to-3 Priority Encoders to receive IRQs from up to two DMA channels, and routes the IRQ signal to one of two CPUs. If multiple IRQs are asserted against the same processor, the IRQ with the lowest priority wins. 

The priority is encoded as a three bit number and placed on outputs Y0-Y2, where it may be used to rewrite the address of the vector pull. For example, the Vector Pull Rewrite Shim (described above) will add the offset on Y0-Y2, multiplied by two, to the low byte of the vector pull. 

IRQs received from DMA channel 0 and intended for CPU A or CPU B will be routed to those CPUs. For DMA channel 1, the situation is reversed: IRQs which DMA channel 1 intended for CPU A will be routed to CPU B, and vice versa. 
![CPU Buffer Board](schematics/Vega816-Dual%20CPU%20IRQ%20Dispatcher.svg)
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
Since the DMA controller will dispatch requests from up to two CPUs, CPU A and CPU B, the signals are distinguished within the controller with these designations:

Inputs
```
DMA_REQB_A DMA_REQB_A
RDY_IN_A  RDY_IN_B
ABORTB_A  ABORTB_B
VA_A      VA_B
```
Since the DMA channels are cross-mapped in address range between the two CPUs, so that each CPU's DMA 0 corresponds to the lower part of its address space with respect to DMA 1, the outputs are:
```
DMA_0_A  = DMA_1_B
DMA_1_A  = DMA_0_B
```
This cross mapping, which takes place in the connector wiring of the DMA Controller board, means that two identically configured units may be connected as CPU A or CPU B, provided that their DMA buffer modules are connected with the sense of DMA 0 and DMA 1 reversed.

The "home" channel for a CPU is the DMA channel containing that CPU's Bank Zero.

Here are the possible scenarios for DMA arbitration:

1. Both CPUs are asserting valid address, each against their own home DMA channels. CPU A is using DMA 0, and CPU B is using DMA 1.
2. Both CPUs are asserting valid address, against the opposite CPU's home DMA channel. CPU A is asserting against DMA 1, and CPU B is asserting against DMA 0.
3. Both CPUs are asserting valid address against the same DMA channel. CPU A is asserting against DMA 0, and so is CPU B.
4. Both CPUs are asserting valid address against the same DMA channel. CPU A is asserting against DMA 1, and so is CPU B.


The DMA controller monitors the VA (Valid Address) from one or two CPUs, as well as an additional input, DMA_REQB. If one CPU asserts DMA_REQB, the controller checks the VA line from the opposite CPU. If it is not active, the controller asserts DMA
![CPU Buffer Board](schematics/Vega816-Dual%20Channel%20DMA%20Controller.svg)


## Quad 64-byte device I/O Bus and IRQ Multiplexer
The I/O bus board decodes a selected page (256 bytes) of address space for device I/O, together with the page above it, which can be used by a programmable interrupt controller to store metadata such as IRQ priority, and which CPU is to receive IRQs from a given device. 

Jumpers are used to locate the I/O address space on any even-numbered page boundary between $0000-$7E00, although, in order to avoid the default page zero and stack areas at $0000 and $01000, it is advisable to avoid these two pages, with $0200-$7E00 being the most acceptable range. This allows for up to 63 choices of I/O page, per DMA channel. A system with two DMA channels could therefore support 2 x 63 x 256 = 32256 bytes of I/O register area, enough for 2,016 VIAs or 4,032 ACIA RS-232 ports, for use in controlling factories or large scale bulletin board system (BBS) installations over dial-up modem.

Four I/O expansion ports are provided, each spanning 64 bytes of the assigned I/O page. Each port is provided with a chip select signal based on a further decoding of the address, down to a 16 byte resolution, which is the register width of the W65C22 VIA, and double the width of the W65C51 ACIA.

Boards which use any of the 64-byte expansion ports may perform additional address decoding in order to accommodate more than four 16-byte devices. For example, one port can support not just 4 VIAs, but 3 VIAs and 2 ACIAs, if one of the 16-byte address ranges is further decoded into two eight byte ranges.

64 bytes was chosen as the granularity for the expansion ports because it is the next whole power of two larger than the widest device used by the Commodore 64, the VIC-II, which has a register width of 47 bytes. (The SID's register width is 29 bytes, giving examples within the C64 of devices requiring (rounding to next whole number power of two) 8, 16, 32, and 64 byte register address spaces

Devices which require more than 16 bytes of address space within a 64 byte range can perform a logical OR on the 16-byte chip select lines. Devices which require more than 64 bytes of register address space may use a similar strategy, bridging across multiple slots. 

Devices which wish to subdivide a 16 byte address space (such as the 8 register ACIA) can further decode address line A5.

The 16 byte range chip select signals are raised high for both the I/O page within that 16 byte range, as well as for that range in the page immediately above it. E.g., if a device is mapped to $0210-$021F, then the chip select for that device is also active for the range $0310-$031F. This allows access to latches used to store programmable interrupt configuration, and potentially other metadata used in controlling the device.

The IRQ Multiplexer takes the 16 IRQ output lines from each expansion port (8 priorities destined for one of up to two CPUs), and multiplexes them for dispatch by the IRQ Dispatch board.
Vega816-Quad IO Bus + IRQ Multiplexer
![Quad 64-byte Expansion Bus and IRQ Multiplexer](schematics/Vega816-Quad%20IO%20Bus%20+%20IRQ%20Multiplexer.svg)
## Programmable Interrupt Controller (PIC)

A 64-byte expansion slot may include a programmable interrupt controller. The PIC subdivides the 64 byte address range into eight 8-byte device I/O spaces, two for each of the 16-byte address ranges decoded by the expansion bus controller and supplied to the PIC as chip select lines. 

Both IRQ priority (0-7) and CPU destination (CPU A or CPU B) can be programmed for each of up to eight devices, by writing to latches which shadow the device I/O space in the next higher page of address space. Four octal latches, at address offsets +$100, +$104, +$108 and +$10C offsets from the corresponding 16 byte I/O address ranges are provided. Each octal latch stores programmable interrupt information for up to two 8 byte devices: the low nybble for an eight byte device at offset +$00 from the base I/O address, and the high nybble for an eight byte device at offset $08 from the base I/O address. The low bit of the nybble determines the target CPU. The high three bits specify the priority from 0-7.
![PIC (Programmable Interrupt Controller](schematics/Vega816-8%20Device%20Programmable%20Interrupt%20Controller.svg)
