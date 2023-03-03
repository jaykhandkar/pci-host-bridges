---
layout: post
title:  "PCI Bridges, Host Bridges, Memory Controllers, Interrupts and more..."
date:   2022-10-14 21:52:50 +0530
categories: PCI PCIe x86 
---

When you have more than one PCI host bridge/PCI Express root complex in a system, such as a modern high end multi socket board (each processor package
usually has an integrated PCIe root complex these days), an interesting set of problems comes into play:

 - Each host bridge must respond to a unique range of bus numbers for configuration space accesses. It must know the bus number on the other side of it
 and the highest bus number below it (the subordinate bus number). In single host bridge systems, the first is zero by default and the latter doesn't
 matter.
 - Each host bridge must respond to a unique range of memory and I/O addresses on the host bus to pass on to the PCI(e) fabric behind them. For modern
 Intel systems with a single root complex, the PCI address range is usually subtractively decoded (whatever addresses are 'left over' unclaimed by other
 targets are forwarded to PCI). This, obviously, cannot work when you have more than one host bridge because you now have to decide exactly which addresses
 are allocated to which host bridge.

 The first restriction applies only when both host bridges implement their PCI Configuration Address/Data or the base address of their enhanced
 configuration mechanism at the same address. But I haven't come accross a system yet that actually implements them at different addresses.
 Initialising the system properly while keeping these restrictions in mind is the job of the firmware (at least on x86). I was curious about how exactly
 this is done and wanted to see it in action with an actual system. I didn't get too far with modern PCIe systems. You might find a register here and
 there that is relevant in the datasheets that Intel provides for its high end Xeon processors that are used in multi socket systems, but they will leave
 you with more questions than before. I also found a paper discussing address decoding on Xeon CPUs, but it focused more on DRAM and not PCIe. The thing
 is, address decoding on modern server level systems is very complex. In old PCI based Pentium systems, all processors interfaced with the memory
 controller on the chipset through a common front side bus. Today, you have memory controllers integrated into the processor package, requiring other
 solutions based on more complex topologies such as rings and meshes, such as Intel's QPI (Quick Path Interconnect) and UPI (Ultra Path Interconnect).
 To make things even more complex, each processor package also has a PCIe root complex and a separate logical PCI bus for configuration purposes. The
 result is that decoding information from whatever little documentation Intel provides for these systems is next to impossible.

 By contrast, it is quite simple to understand old multi socket PCI based Pentium systems. I found a relatively thorough description of the exact aspects
 discussed above in the datasheet for Intel's old (as old as 1996) 450 KX/GX "PCIset" chipset. This chipset supports upto four Pentium Pro processors, two
 PCI host bridges (the GX variant) and two memory controllers:

 ![pciset](https://user-images.githubusercontent.com/23404671/194386857-78baec3d-e721-4a73-a5e7-56ac910ed96b.png)

How, then, are the various aspects mentioned above managed in this chipset? First, let's talk about how PCI configuration space accesses are handled. 

## Configuration Space

To allow software to configure bus numbers, memory address ranges, IO address ranges and other aspects, the two bridges expose themselves as PCI devices 
residing on PCI bus 0. In fact, not just the bridges, but also the memory controller is mapped as a PCI device on bus 0. These 'pseudo - PCI' devices use device numbers greater than 15,
whereas numbers below 15 are free and are mapped to a unique AD[31:16] line (more on mapping of device numbers to AD lines, IDSEL# etc. later) so that
the hierarchy below a PCI bridge can begin with bus zero. For example, on reset, the first PCI host bridge (PB from hereon), also referred to as the
compatibility PB since it has a PCI to ISA bridge below it, is assigned a device number of 25, whereas the other PB, referred to as the Auxiliary PB is
assigned a device number of 26. Here is a description of how the two bridges behave after reset and after proper configuation -

 - Firstly, both host bridges implement a register to change the bus number on their secondary side and the subordinate bus number for the buses below
 them:


![450gx-pbusnum](https://user-images.githubusercontent.com/23404671/195123003-8a39f54d-ef2a-46b9-b427-ad6d9f054a72.png)
![450gx-subbusnum](https://user-images.githubusercontent.com/23404671/195123020-54e60c86-7b04-40c4-9358-db8e20388b0b.png)

 - At reset, both of these values are zero. This might not make sense at first because you might think both bridges will respond to configuration
 accesses to bus number zero. However, only the compatibility PB acts as the response agent for the transaction. If the device number is greater than 15
 and matches its own device number, it responds to the transaction itself. The auxiliary PB simply snarfs the data being written to the configuration
 address register, and if the bus number is zero with the device number matching its own, it will respond to the transaction instead. If the bus number
 is zero but the device number less than 16, it is the compatibility PB that generates a configuration cycle on the PCI bus behind it. 
 In this way, there is a unique responder to each type of configuration access on reset.
 - Once the firmware has discovered all the devices and buses behind the compatibility PB, it will set the subordinate bus number for the compatibility
 PB and the primary bus number for the auxiliary PB, which will cause the auxiliary PB to start responding to configuration accesses targeted to
 devices with device number below 15, thus enabling firmware to discover all the devices and buses behind the auxiliary PB.
