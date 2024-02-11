## Vector Pull Rewrite 

The W65C816 datasheet gives this table of interrupt vectors which lie between $00FFE4 and $00FFFC, when the processor is in Native mode:

```
Native Mode (65C816) Interrupt Vector Table (datasheet)


$FFE4 COP      
$FFE6 BRK      
$FFE8 ABORT    
$FFEA NMI      
$FFEE IRQ      
$FFFC RESET    
```
Note that in native mode, between the addresses for the IRQ and RESET vectors, there are six vectors' (twelve bytes) worth of unused addresses, $FFF0-$FFFB. 
This strongly suggests the possibility of using the vector pull line to up to add additional six different IRQ vectors, by rewriting the address of the vector to be pulled. The feature described here alters the addresses fetched by the CPU when it fetches an IRQ vector. The result is the following expanded table of interrupts.

```
W65C816 Native Interrupt Vector Table (after rewrite)

$FFE4 COP      
$FFE6 BRK      
$FFE8 ABORT    
$FFEA NMI      
$FFEE IRQ 0 
$FFF0 IRQ 1 
$FFF2 IRQ 2 
$FFF4 IRQ 3 
$FFF6 IRQ 4 
$FFF8 IRQ 5 
$FFFA IRQ 6 
$FFFC RESET

```

## Theory of operation
The circuit has 7 input lines, active low, designated IRQ0B through IRQ6B. While one or more of them is asserted low, IRQ on the processor is pulled low. A priorty encoder encodes the number of the lowest-numbered active input. Twice this number is offered to a 4-bit adder added to the low order byte of the vector pull.
When
The circuit detects vector pulls to the IRQ vector, $FFEE, and rewrites the low order byte of the fetch.

When E is low (native mode), and VPB goes low (active), the CPU is in native mode and asserting the address of an interrupt vector on the address lines. The current memory access fetches the low byte of the vector, and the next memory access fetches the high byte. Since the 

and the high three bits of the low order nybble of the address are examined. 



A three bit number between 0-7 is added to the least significant bits (but one) of the fetch address, A1-A3, with carry into A4.

The VPB line is brought active low by the 65C816 during the fetch of any interrupt vector, not just IRQ. However, the standard IRQ vector's address has a unique bit pattern which makes it easy to detect, so that an offset when fetching an IRQ vector, and not for any other kind of interrupt.

When VPB is active (low) and A1-A3 are all high, indicating that an interrupt vector is being fetched by the processor, the shim requests a vector pull offset from the IRQ Dispatcher by asserting VP_REQ (positive logic). Pulldown resistors keep the offset equal to zero when Y0-Y2 are not connected (no IRQ Dispatcher) or in a high impedance state. 

A 74HC283 adder is used to add an offset to A1-A4 equal to the three bit number provided as input to the shim from the IRQ Dispatcher, on pins Y0-Y2. This has the effect of altering the fetch of the IRQ vector at $FFEE to a fetch of a vector at one of the following eight addresses: 


Y0-Y2 are pulled low by 10K resistors, for cases in which no Vector Pull controller is connected. Bypass jumpers are also provided for A1-A4, in case the 74HC283 adder and supporting logic are not installed.



#### Details of Vector Pull address rewriting

The datasheet also specifies that when the 65C816 is addressing an interrupt vector, it pulls its VPB line low. This is to allow the system to override the fetched interrupt vector with any other. 

VPB is pulled low for any interrupt vector fetch, not just IRQ, but the present design purposefully limits Vector Pull rewriting to only those cases when IRQ is the vector being fetched.

Note that the last nybble of each of the standard interrupt vectors is unique, which aids in designing a hook for any one of them (the same is true for 6502 Emulation mode, see below). A test of these four bits suffices to uniquely identify the vector being pulled. In particular, both bytes of the two-byte IRQ vector ($FFEE and $FFEF) are the only two bytes in this table with $E or $F (0x1110, 0x1111) as the last nybble of address, i.e., A1-A3 high.

Note also the gap of 12 bytes between the IRQ vector and the RESET vector. Since each vector occupies two bytes, this gap provides sufficient space to specify six (6) additional vectors. 

```
Native Mode Interrupt Vector Table (with IRQ0-7)

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

While an IRQ vector is being pulled, if an offset [0..7]*2 is added to the address asserted on the bus, then the vector fetched will be as shown above, going forward from IRQ0.

## IRQ Priority Dispatch

The Vector Pull Rewrite portion of the CPU shim exposes eight inputs, active low, as IRQ0B-IRQ7B. Devices connected to any DMA channel may direct an IRQ to the CPU, by asserting one of these inputs. 

An 8-to-3 priority encoder is used to prioritize eight different levels of IRQ, from 0-7 (lowest value wins).

The priority is encoded as a three bit offset, and this value is added to A1-A3, effectively adding double the offset to the address to be pulled. (Double, since each vector occupies two consecutive bytes)

## Native Mode: IRQ7 triggers RESET interrupt handler 

Note that, when the 65816 is running in native mode, if an IRQ priority of 7 (0x111) is asserted on Y0-Y2, the vector offset will result in the RESET vector being pulled. Since no actual hardware reset has occurred, if the reset handler relies on a hardware reset having taken place, unpredictable state may occur. It is generally advised to avoid the use of IRQ priority 7 when the CPU is in native mode. Alternatively, IRQ priority 7 may be rerouted to NMI.

## IRQ Priority 7 to NMI Option

A jumper on the CPU Shim allows IRQ7 to trigger an NMI instead of participating in Vector Pull Rewrite.
This is intended to support devices such as the RESTORE key on the C64, which expects to be able to
assert NMIB. Another motivation is to provide a useful alternative to the default effect of asserting
IRQ7 within the rewrite scheme, which is to force the pull of the RESET vector.

## Effect of Vector Pull Rewrite in Emulation (65C02) Mode
The 6502's IRQ vector, like its 65816 counterpart, has 0x111x in the last nybble, so it is detected by the Vector Pull Rewrite Shim. 

```
Emulation Mode Interrupt Vector Table  (datasheet)

$FFFA NMI
$FFFC RESB
$FFFE BRK/IRQ 
```

Unlike the 65816's IRQ vector, the 6502 IRQ vector occurs at the very top of its page, so additions of 1-7 to the bits may force A4 to overflow to 0 (no carry). In particular, $FFFE + $02 = $FFE0, etc. This means the augmented
IRQ vector map for the 6502 looks like this:


```
Emulation Mode Interrupt Vector Table (with IRQ0-7)

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



