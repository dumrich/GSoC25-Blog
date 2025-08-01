+++
title = "Week 2: Project Setup Details and vLAPIC"
date = "2025-06-12T14:48:12-04:00"
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["QEMU", "vmm"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
toc = true
+++

## QEMU configuration script
```C
$ cd qemu/build
$ ../configure \
  --target-list=x86_64-softmmu \
  --enable-debug \
  --extra-cflags="-I$HOME/freebsd-src/lib/libvmmapi" \
  --extra-ldflags="-L$HOME/vmmtest_root/usr/lib" \
  --disable-werror
$ gmake -j4
```

This ensures that qemu will fetch the custom `vmmapi` located in $HOME/vmmtest_root/usr/lib.

## `libvmmapi`

```C
$ cd $HOME/freebsd-src/lib/libvmmapi/
$ make DEBUG_FLAGS="-g -O1 -fPIC" WITHOUT_MAN=yes all
$ doas cp libvmmapi.a $HOME/vmmtest_root/usr/lib
```

## `vmm`
Build new module:
```C
$ cd freebsd-src/sys/modules/vmm
$ make KERNCONF=THINKBSD \
     KMODDIR=/boot/kernel \
     MACHINE=amd64 \
     MACHINE_ARCH=amd64 \
     SYSDIR=/home/chabi/freebsd-src/sys \
     SRCCONF=/dev/null \
     __MAKE_CONF=/dev/null all
```

Load new module:
```C
# kldunload vmm
# cd freebsd-src/sys/modules/vmm
# kldload ./vmm.ko
```

## Virtual Lapic
The `vmm` kernel module includes virtual peripherals in each VM that aim to potentially improve the performance of a VM.

Many of these peripherals are reimplemented by QEMU, so we must create a bridge between those modules in `vmm` and `QEMU`. Luckily, QEMU provides a template to accomplishing this for one of these peripherals, via its `APICCommonState`.

### What is a LAPIC
The virtual LAPIC will be the first in-kernel peripheral to have a bridge to QEMU, and will bring possibly the greatest performance improvement.

The LAPIC is a per-core interrupt controller that is used to deliver external, internal, and IP (inter-processor) interrupts. It is mapped to the physical adress 0xFEE00000 (or MSR IA32_APIC_BASE), and accessed via memory-mapped registers by software.

It has registers (4 bytes wide) accessed via MMIO.

### Interrupt Delivery
External Interrupts:
- Come through IOAPIC or Legacy PIC
- Routed through LAPIC through the APIC bus or system bus
- LAPIC receives the interrupt, and checks its priority via the TPR
- If unmasked, the interrupt is delivered via the CPU interrupt gate

Timer Interrupts:
- LAPIC timer fires on configured interval (simple clock divider + counter mechanism)
- Triggers a local interrupt

Inter-processor interrupts (IPIs): Sent from processor to processor. Used in SMP, SIPI (AP initialization), scheduling, TLB shootdowns.
- One CPU sends an IPI using LAPIC ICR register
- Target LAPIC receieves and interrupts its local core


### Interrupt Delivery details
Once the CPU finishes running the interrupt handler (specified in the IDT), it writes to the EOI register in the LAPIC, which then clears the interrupt, and allows new ones. Each register in the LAPIC is offset some bytes from the base.

If an interrupt is masked, the LAPIC will not deliver it to the CPU - even if its pending.

### Task Priority Register (TPR)
The TPR sets the minimum priority vector that can be accepted by the LAPIC. TPR stores a priority threshold. If an incoming interrupt's priority is < TPR -> it is then masked.

### AP Initialization
1) On Reset
APs are initially halted, except for the BSP.

2) 16-bit real mode BIOS phase
Sets up LAPICs for enumerated CPUs

3) Bootloader/Kernel Phase
The OS or bootloader (from the Bootstrap processor), will send an INIT IPI to each AP. After 10ms, it will send an SIPI that starts executing trampoline code.

Trampoline code is the 16-bit real mode code that switches from real->protected->long mode.

### VMX challenges and solutions
In a VM:
1) The guest believes it's writing/reading MMIO registers at 0xFEE00000.
2) The hypervisor must intercept, emulate, and redirect each access.
3) Doing this with VM exits on a LAPIC access becomes very expensive.

Virtual-APIC Page
1) Guest LAPIC register state is stored in a memory page (4kb)
2) Hypervisor and hardware both share access to this page
3) No need to trap MMIO - guest reads/writes the memory page
