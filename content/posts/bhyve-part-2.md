+++
title = "Bhyve Part 2"
date = "2025-05-15T11:58:06-04:00"
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["Bhyve", "FreeBSD"]
keywords = ["Bhyve", "FreeBSD"]
description = "Part 2 of Architectural walkthrough of FreeBSD Bhyve"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
toc = true
+++

## Initializing the Platform
Next, Bhyve initializes the rest of the hardware with the bootstrap processor:
```C
	if (bhyve_init_platform(ctx, bsp) != 0)
		exit(4);
```

The init platform function initiates several pieces of hardware:
```C
int
bhyve_init_platform(struct vmctx *ctx, struct vcpu *bsp __unused)
{
	int error;

	error = init_msr();
	if (error != 0)
		return (error);
	init_inout();
	kernemu_dev_init();
	atkbdc_init(ctx);
	pci_irq_init(ctx);
	ioapic_init(ctx);
	rtc_init(ctx);
	sci_init(ctx);
	error = e820_init(ctx);
	if (error != 0)
		return (error);
	error = bootrom_loadrom(ctx);
	if (error != 0)
		return (error);

	return (0);
}
```

In this post, we'll dive into each of them.

### In/out Handlers
The `init_inout` function (amd64/inout.c) sets handlers for port mapped I/O. The `in` and `out` instructions access this space, leaving the memory bus for RAM.

The CPU places a 16-bit port address on dedicated lines. Each device watches for port-address ranges (ex: PIT timer: ports 0x40-0x43). When the CPU pulses the I/O-read or I/O-write strobe, the selected device will place data on the bus (`in`), or latch data from the data bus (`out`)

In Bhyve, the handlers are set as such (amd64/inout.c):
```C
void
init_inout(void)
{
	struct inout_port **iopp, *iop;

	/*
	 * Set up the default handler for all ports
	 */
	register_default_iohandler(0, MAX_IOPORTS);

	/*
	 * Overwrite with specified handlers
	 */
	SET_FOREACH(iopp, inout_port_set) {
		iop = *iopp;
		assert(iop->port < MAX_IOPORTS);
		inout_handlers[iop->port].name = iop->name;
		inout_handlers[iop->port].flags = iop->flags;
		inout_handlers[iop->port].handler = iop->handler;
		inout_handlers[iop->port].arg = NULL;
	}
}
```

The `inout_port_set` is defined via the `INOUT_PORT` macro:
```C
#define	INOUT_PORT(name, port, flags, handler)				\
	static struct inout_port __CONCAT(__inout_port, __LINE__) = {	\
		#name,							\
		(port),							\
		1,							\
		(flags),						\
		(handler),						\
		0							\
	};								\
	DATA_SET(inout_port_set, __CONCAT(__inout_port, __LINE__))
```

### Kernel Emulated Device Initialization
The `vmm` kernel hypervisor that underlies Bhyve defines several devices to improve the performance of the virtual machine.

#### APIC
The IOAPIC and LAPIC are critical components of the PC architecture. They allow for the management of interrupts in a multi-core environment.

The LAPIC resides in each CPU core (or vCPU) to handle local interrupts and receive external interrupts routed to it.It processes interrupts delivered to its CPU via the Interrupt Vector Table.

The IOAPIC is a system-wide controller that manages external interrupts from devices (disk, network) and routes them to a CPU's LAPIC.

Example:
Kernel issues a read to an AHCI disk. Virtual AHCI controller completes the I/O and generates an interrupt (IRQ). The interrupts is sent to the vIOAPIC.

vIOAPIC receives the IRQ and checks its redirection table entry (RTE). The RTE specifies teh interrupt vector, the target vCPU, and the delivery mode. The vIOAPIC routes the interrupt to vCPU 0's LAPIC.

The vCPU receives the interrupt vector 0x40. The LAPIC checks its state, and if the `EFLAGS.IF` is set, the interrupt is queued into the Interrupt Pending Register (IPR). If the vCPU is running, LAPIC injects the interrupt immediately. If not, it's delivered when the vCPU resumes.
#### HPET
The high precision event timer (HPET) is a hardware timer that provides high resolution programmable timing for operating systems. It generates one-shot interrupts at nanosecond resolution, replacing older timers like the PIT or RTC.

It can operate at a high resolution and has a MMIO interface. Interrupts are generally routed to the IOAPIC and LAPIC.
#### Bhyve initialization
Kernel devices are initialized in `amd64/kernemu_dev.c`:

```C
void
kernemu_dev_init(void)
{
	int rc;

	rc = register_mem(&lapic_mmio);
	if (rc != 0)
		errc(4, rc, "register_mem: LAPIC (0x%08x)",
		    (unsigned)lapic_mmio.base);
	rc = register_mem(&ioapic_mmio);
	if (rc != 0)
		errc(4, rc, "register_mem: IOAPIC (0x%08x)",
		    (unsigned)ioapic_mmio.base);
	rc = register_mem(&hpet_mmio);
	if (rc != 0)
		errc(4, rc, "register_mem: HPET (0x%08x)",
		    (unsigned)hpet_mmio.base);
}
```

A memory range is defined like this:
```C
static int
apic_handler(struct vcpu *vcpu, int dir, uint64_t addr, int size,
    uint64_t *val, void *arg1 __unused, long arg2 __unused)
{
	if (vm_readwrite_kernemu_device(vcpu, addr, (dir == MEM_F_WRITE),
	    size, val) != 0)
		return (errno);
	return (0);
}

static struct mem_range lapic_mmio = {
	.name = "kern-lapic-mmio",
	.base = DEFAULT_APIC_BASE,
	.size = PAGE_SIZE,
	.flags = MEM_F_RW | MEM_F_IMMUTABLE,
	.handler = apic_handler,

};
```

Bhyve uses red-black trees to organize memory regions, using per vCPU caching for performance. The `register_mem` function sets the memory region base and end into the red black tree with `mmio_rb_add`.

### `atkbdc` initialization
The AT Keyboard controller sets up emulation of the PS/2 keyboard and mouse controller.

#### ATKBDC
The ATKBDC is a legacy hardware component that interfaces the CPU with PS/2 keyboards and mice. It remains emulated for backwards compatibility. It translates key presses and mouse movements into interrupts.

#### Bhyve implementation
```C
void
atkbdc_init(struct vmctx *ctx)
{
...
	bzero(&iop, sizeof(struct inout_port));
	iop.name = "atkdbc";
	iop.port = KBD_STS_CTL_PORT;
	iop.size = 1;
	iop.flags = IOPORT_F_INOUT;
	iop.handler = atkbdc_sts_ctl_handler;
	iop.arg = sc;

	error = register_inout(&iop);
	assert(error == 0);

	bzero(&iop, sizeof(struct inout_port));
	iop.name = "atkdbc";
	iop.port = KBD_DATA_PORT;
	iop.size = 1;
	iop.flags = IOPORT_F_INOUT;
	iop.handler = atkbdc_data_handler;
	iop.arg = sc;

	error = register_inout(&iop);
	assert(error == 0);
...
}
```

An IO device is registered similarly to before:
```C
...
	for (i = iop->port; i < iop->port + iop->size; i++) {
		inout_handlers[i].name = iop->name;
		inout_handlers[i].flags = iop->flags;
		inout_handlers[i].handler = iop->handler;
		inout_handlers[i].arg = iop->arg;
	}
...
```

### PCI device initialization
The PCI device IRQs are initiated as follows:
```C
void
pci_irq_init(struct vmctx *ctx __unused)
{
	int i;

	for (i = 0; i < NPIRQS; i++) {
		pirqs[i].reg = PIRQ_DIS;
		pirqs[i].use_count = 0;
		pirqs[i].active_count = 0;
		pthread_mutex_init(&pirqs[i].lock, NULL);
	}
	for (i = 0; i < NIRQ_COUNTS; i++) {
		if (IRQ_PERMITTED(i))
			irq_counts[i] = 0;
		else
			irq_counts[i] = IRQ_DISABLED;
	}
}
```

This clears the IRQs and sets the permission for each.

#### Asserting an IRQ
```C
void
pci_irq_assert(struct pci_devinst *pi)
{
	struct pirq *pirq;
	int pin;

	pin = pi->pi_lintr.irq.pirq_pin;
	if (pin > 0) {
		assert(pin <= NPIRQS);
		pirq = &pirqs[pin - 1];
		pthread_mutex_lock(&pirq->lock);
		pirq->active_count++;
		if (pirq->active_count == 1 && pirq_valid_irq(pirq->reg)) {
			vm_isa_assert_irq(pi->pi_vmctx, pirq->reg & PIRQ_IRQ,
			    pi->pi_lintr.irq.ioapic_irq);
			pthread_mutex_unlock(&pirq->lock);
			return;
		}
		pthread_mutex_unlock(&pirq->lock);
	}
	vm_ioapic_assert_irq(pi->pi_vmctx, pi->pi_lintr.irq.ioapic_irq);
}
```

If there are more than one devices currently asserting, use the IOAPIC (ISA only supports one device per INT#.

Deasserting subtracts one from the active_count.

### Initializing the IOAPIC

This function queries the IOAPIC pincount, adjusts the count to reserve the first 16 pins for legacy ISA devices, and stores the remaining pins for PCI device interrupts.
```C
static int pci_pins;

void
ioapic_init(struct vmctx *ctx)
{

	if (vm_ioapic_pincount(ctx, &pci_pins) < 0) {
		pci_pins = 0;
		return;
	}

	/* Ignore the first 16 pins. */
	if (pci_pins <= 16) {
		pci_pins = 0;
		return;
	}
	pci_pins -= 16;
}
```

### RTC Initialization
The `rtc` timer is another device that is implemented at the kernel level. It maintains the wall clock even when the system is powered off with a small battery. It can trigger wake up events (e.g. from sleep) at a predefined time (e.g. scheduled backups).

UEFI requires the systems RTC to report guest memory size in the CMOS NVRAM. Additionally, we set the time based on the hosts `rtc_time()`.
```C
void
rtc_init(struct vmctx *ctx)
{
	size_t himem;
	size_t lomem;
	int err;

	lomem = (vm_get_lowmem_size(ctx) - m_16MB) / m_64KB;
	err = vm_rtc_write(ctx, RTC_LMEM_LSB, lomem);
	assert(err == 0);
	err = vm_rtc_write(ctx, RTC_LMEM_MSB, lomem >> 8);
	assert(err == 0);

	himem = vm_get_highmem_size(ctx) / m_64KB;
	err = vm_rtc_write(ctx, RTC_HMEM_LSB, himem);
	assert(err == 0);
	err = vm_rtc_write(ctx, RTC_HMEM_SB, himem >> 8);
	assert(err == 0);
	err = vm_rtc_write(ctx, RTC_HMEM_MSB, himem >> 16);
	assert(err == 0);

	err = vm_rtc_settime(ctx, rtc_time());
	assert(err == 0);
}
```

### SCI
The ACPI SCI is a specialized hardware interrupt that allows the OS to receive notifications about power changes, thermal events, device notifications, etc.

```C
void
sci_init(struct vmctx *ctx)
{

	/*
	 * Mark ACPI's SCI as level trigger and bump its use count
	 * in the PIRQ router.
	 */
	pci_irq_use(SCI_INT);
	vm_isa_set_irq_trigger(ctx, SCI_INT, LEVEL_TRIGGER);
}
```


## Mapping bootrom
```C
...
	/* Map the bootrom into the guest address space */
	if (bootrom_alloc(ctx, rom_size, PROT_READ | PROT_EXEC,
	    BOOTROM_ALLOC_TOP, &ptr, NULL) != 0) {
		goto done;
	}

	/* Read 'romfile' into the guest address space */
	for (i = 0; i < rom_size / PAGE_SIZE; i++) {
		rlen = read(fd, ptr + i * PAGE_SIZE, PAGE_SIZE);
		if (rlen != PAGE_SIZE) {
			EPRINTLN("Incomplete read of page %d of bootrom "
			    "file %s: %ld bytes", i, romfile, rlen);
			goto done;
		}
	}

	if (varfd >= 0) {
		var.mmap = mmap(NULL, var_size, PROT_READ | PROT_WRITE,
		    MAP_SHARED, varfd, 0);
		if (var.mmap == MAP_FAILED)
			goto done;
		var.size = var_size;
		var.gpa = (gpa_alloctop - var_size) + 1;
		gpa_alloctop = var.gpa - 1;
		rv = register_mem(&(struct mem_range){
		    .name = "bootrom variable",
		    .flags = MEM_F_RW,
		    .handler = bootrom_var_mem_handler,
		    .base = var.gpa,
		    .size = var.size,
		});
		if (rv != 0)
			goto done;
	}
...
```

Part 3 covers the initialization of the machine