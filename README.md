# Vega65816
A Multiprocessor Implementation of the Kohlbecker 65816 Breakout Board

Features:
* IRQ Priority routing. When VPB is low, the base address 
Here is the interrupt vector table for the 65816, augmented to show the
extra vectors
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
