
# DMA Controller

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
![DMA Controller](schematics/Vega816-4%20Channel%20DMA%20Ctrl.svg)


### DMA Channels and Address Interleaving

When no request granularity is set for a CPU on the DMA Control Shim, all DMA requests take place against DMA 0.

If only the primary granularity is set, DMA requests will take place against DMA 0 and DMA 2.
If only the secondary granularity is set, DMA requests will take place against DMA 0 and DMA 1.

If both granularities are set, DMA requests will take place against DMA 0-3. The maximum granularity for four channel DMA is 4 MB, with primary set at 8 MB and secondary set at 4MB. (0x1100 0000). In that case,

```
Requests for the range 
(0x00?? ????) go to DMA channel 0
(0x01?? ????) go to DMA channel 1
(0x10?? ????) go to DMA channel 2
(0x11?? ????) go to DMA channel 3
```

The DMA controller maps requests so that each CPU sees a different DMA channel as its channel 0, with subsequent channels in order. The next CPU's channel 0 is the present CPU's channel 1, and channel 0 for the next CPU after that is channel 2. The order wraps to form a closed ring.

```
           Requested           Physical
CPU   A    B      C      D
      0    1      2      3      DMA 0
      1    2      3      0      DMA 1
      2    3      0      1      DMA 2
      3    0      1      2      DMA 3
```

From the chart it can be seen that the channel that CPU A sees as DMA 0 is seen by CPU B as DMA channel 3, etc.


DMA request granularity may or may not match the largest installed address size.

Since the address bits used to select the DMA channels are suppressed in the address presented to the destination channel, if a CPU's request granularity is set to a smaller value than the installed memory size on any given DMA channel, then a range of that memory will not be accessible to that CPU.



CPU A has primary DMA on 1M, and secondary DMA set to 512K. 
CPU A issues requests against Banks 0 to 7.

What DMA channels are addressed?

```
0x000 Bank 0x000 0 on DMA 0
0x001 Bank 0x000 0 on DMA 1
0x010 Bank 0x000 0 on DMA 2
0x011 Bank 0x000 0 on DMA 3
0x100 Bank 0x100 4 on DMA 0
0x101 Bank 0x100 4 on DMA 1
0x110 Bank 0x100 4 on DMA 2
0x111 Bank 0x100 4 on DMA 3
```

### How the DMA Controller routes requests

Each request enters the DMA controller as a two bit code, representing the two address lines selected by jumper settings on the DMA Control Shim. These two bits are augmented with a third bit, a CPU Priority bit supplied for that CPU. The CPU Priority bit is active low, and pulled high. 

The three bit request is decoded into a one in eight choice, each representing a request for one of four DMA channels, with and without priority. The eight decoded request lines are then routed, together with the decoded request lines from the other CPUs, into priority encoders, one for each of the four DMA channels. The eight request lines entering each DMA channel's priority encoder are wired so that the four priority requests occur with highest priority, followed by the four no-priority request lines. Among the two sets of priority requests, Each DMA channel gives greatest priority to a particular CPU, followed by the others in ring order.

```
DMA Channel   Order
     0        ADCB                  
     1        BADC            
     2        CBAD
     3        DCBA            
```

When more than one CPU requests the same DMA channel, the losing CPU(s) must be informed, since they will not have access to the DMA channel for the duration of the current clock cycle. 

The W65C816 provides two mechanisms for this, the READY signal, and the ABORTB interrupt signal. The READY signal pauses the processor, if pulled low. Releasing the low condition, allowing READY to return high, will cause the CPU to resume processing on the rising edge of the next clock cycle.

Adrien Kohlbecker's BB816 CPU breakout provides the input to the READY signal as RDY_IN. This signal is that which is used by the DMA Controller to pause the CPU.

If the losing CPU is losing access to its Bank Zero channel (due to the winning CPU having priority bit active), the losing CPU is paused, by pulling RDY_IN low. 

Any other losing CPU is informed by pulling the losing CPU's ABORTB low, triggering the losing CPU's ABORT Interrupt handler. In most cases, the handler will simply return control to the interrupted process so that another attempt at access to the DMA channel can be made on the next clock cycle.


### DMA Request timing


CPU to DMA routing must be complete, with time to spare for the devices on the DMA channel to decode incoming address, before the routing is latched in on the falling edge of the clock. Now latched, the CPU to DMA channel routing persists through the remainder of the clock cycle, until unlatched at the rising edge of the clock at the beginning of the next cycle, so that arbitration may begin again.

