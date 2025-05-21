+++
title = "Bhyve vs Qemu: Part 1"
date = "2025-05-15T14:22:08-04:00"
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["Bhyve", "Qemu"]
keywords = ["", ""]
description = "Insights into the way that Bhyve and QEMU handle devices"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
draft = true
+++

This will be a dive into how Bhyve and Qemu both handle devices. We will be looking at the AHCI SATA controller as an example on both platforms.

## Bhyve implementation
The AHCI controller is a PCI device. It is first initialized in the `init_pci` function (`/pci_emul.c`).

### PCI Initialization
First, the base addresses for IO ports, the 32-bit MMIO region, and the 64-bit MMIO region are set up on aligned boundaries

```C
pci_emul_iobase = PCI_EMUL_IOBASE;
pci_emul_membase32 = PCI_EMUL_MEMBASE32;

pci_emul_membase64 = vm_get_highmem_base(ctx) + vm_get_highmem_size(ctx);
pci_emul_membase64 = roundup2(...);
```

Then, Bhyve iterates over PCI buses, slots, and functions. For each bus, a `businfo` struct is allocated which tracks base addresses.

### Device registration
Each PCI device in Bhyve is described by a `pci_devemu`. This structure contains function pointers for device initialization (`pe_init`), config space access, BAR read and write, and more.

Example from pci_ahci.c:
```C
static const struct pci_devemu pci_de_ahci = {
	.pe_emu =	"ahci",
	.pe_init =	pci_ahci_init,
	.pe_legacy_config = pci_ahci_legacy_config,
	.pe_barwrite =	pci_ahci_write,
	.pe_barread =	pci_ahci_read,
#ifdef BHYVE_SNAPSHOT
	.pe_snapshot =	pci_ahci_snapshot,
	.pe_pause =	pci_ahci_pause,
	.pe_resume =	pci_ahci_resume,
#endif
};
PCI_EMUL_SET(pci_de_ahci);

static const struct pci_devemu pci_de_ahci_hd = {
	.pe_emu =	"ahci-hd",
	.pe_legacy_config = pci_ahci_hd_legacy_config,
	.pe_alias =	"ahci",
};
PCI_EMUL_SET(pci_de_ahci_hd);
```

The pe_init attribute is a function pointer for device initialization, and the devemu struct also features config space access, BAR read and write, and more.

### Device initialization
Bhyve parses the `-s` options, specifying PCI slot, device type, and configuration.


```C
init_pci
...
				emul = get_config_value_node(nvl, "device");
				pde = pci_emul_finddev(emul);
				fi->fi_pde = pde;
				error = pci_emul_init(ctx, pde, bus, slot,
				    func, fi);
...
```

Each device has a 256-byte (4KB for PCIe) config space stored in `pi->pi_cfgdata`. This cfgdata is set in the initialization function of each `pci` device.

Example from AHCI init function:
```C
...
	sc->cap = AHCI_CAP_64BIT | AHCI_CAP_SNCQ | AHCI_CAP_SSNTF |
	    AHCI_CAP_SMPS | AHCI_CAP_SSS | AHCI_CAP_SALP |
	    AHCI_CAP_SAL | AHCI_CAP_SCLO | (0x3 << AHCI_CAP_ISS_SHIFT)|
	    AHCI_CAP_PMD | AHCI_CAP_SSC | AHCI_CAP_PSC |
	    (slots << AHCI_CAP_NCS_SHIFT) | AHCI_CAP_SXS | (sc->ports - 1);

	sc->vs = 0x10300;
	sc->cap2 = AHCI_CAP2_APST;
	ahci_reset(sc);

	pci_set_cfgdata16(pi, PCIR_DEVICE, 0x2821);
	pci_set_cfgdata16(pi, PCIR_VENDOR, 0x8086);
	pci_set_cfgdata8(pi, PCIR_CLASS, PCIC_STORAGE);
	pci_set_cfgdata8(pi, PCIR_SUBCLASS, PCIS_STORAGE_SATA);
	pci_set_cfgdata8(pi, PCIR_PROGIF, PCIP_STORAGE_SATA_AHCI_1_0);
	p = MIN(sc->ports, 16);
	p = flsl(p) - ((p & (p - 1)) ? 0 : 1);
	pci_emul_add_msicap(pi, 1 << p);
	pci_emul_alloc_bar(pi, 5, PCIBAR_MEM32,
	    AHCI_OFFSET + sc->ports * AHCI_STEP);

	pci_lintr_request(pi);
...
```

### BAR Mapping
Device memory is mapped to physical memory through BARs (Base Address Register). Offsets to this BAR are how device registers and memory are accessed. The `pe_init` function calls `pci_emul_alloc_bar` to allocate a BAR of a given size and type.

```C
pci_emul_alloc_bar(pi, 0, PCIBAR_MEM32, size);
```

### Memory Mapping

Bhyve tracks the mapping, and ensures that guest accesses to the BAR are routed to the emulation handlers.

```C
...
	lowmem = vm_get_lowmem_size(ctx);
	bzero(&mr, sizeof(struct mem_range));
	mr.name = "PCI hole";
	mr.flags = MEM_F_RW | MEM_F_IMMUTABLE;
	mr.base = lowmem;
	mr.size = (4ULL * 1024 * 1024 * 1024) - lowmem;
	mr.handler = pci_emul_fallback_handler;
	error = register_mem_fallback(&mr);
	assert(error == 0);

	/* PCI extended config space */
	bzero(&mr, sizeof(struct mem_range));
	mr.name = "PCI ECFG";
	mr.flags = MEM_F_RW | MEM_F_IMMUTABLE;
	mr.base = PCI_EMUL_ECFG_BASE;
	mr.size = PCI_EMUL_ECFG_SIZE;
	mr.handler = pci_emul_ecfg_handler;
	error = register_mem(&mr);
	assert(error == 0);
...
```

The `register_mem` function adds the mem_range to a `rb_tree` of mmio regions, which is then accessed when there is an EPT violation and the mmio regions need to be emulated.

```C
static int
vmexit_inst_emul(struct vmctx *ctx __unused, struct vcpu *vcpu,
    struct vm_run *vmrun)
{
	struct vm_exit *vme;
	struct vie *vie;
	int err, i, cs_d;
	enum vm_cpu_mode mode;

	vme = vmrun->vm_exit;

	vie = &vme->u.inst_emul.vie;
	if (!vie->decoded) {
		/*
		 * Attempt to decode in userspace as a fallback.  This allows
		 * updating instruction decode in bhyve without rebooting the
		 * kernel (rapid prototyping), albeit with much slower
		 * emulation.
		 */
		vie_restart(vie);
		mode = vme->u.inst_emul.paging.cpu_mode;
		cs_d = vme->u.inst_emul.cs_d;
		if (vmm_decode_instruction(mode, cs_d, vie) != 0)
			goto fail;
		if (vm_set_register(vcpu, VM_REG_GUEST_RIP,
		    vme->rip + vie->num_processed) != 0)
			goto fail;
	}

	err = emulate_mem(vcpu, vme->u.inst_emul.gpa, vie,
	    &vme->u.inst_emul.paging);
```

## QEMU Devices 