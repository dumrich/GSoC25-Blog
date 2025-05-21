+++
title = "Bhyve Part 1"
date = "2025-05-13T09:33:23-04:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
tags = ["Bhyve", "FreeBSD"]
keywords = ["Bhyve", "FreeBSD"]
description = "Quick Architectural Walkthrough of FreeBSD Bhyve"
showFullContent = false
readingTime = false
hideComments = false
toc = true
+++

## Using Bhyve and `bhyve-firmware`

Ensure `vmm.ko` is loaded and `bhyve-firmware` is installed:
```
# kldstat -m vmm
# pkg install bhyve-firmware
```

Create bridge as described [here](https://docs.freebsd.org/en/books/handbook/virtualization/#virtualization-bhyve-prep)

Create VM:
```sh
# bhyve -c 2 -m 2G -AHP -s 0:0,hostbridge -s 31,lpc \
    -s 3:0,ahci-hd,${BACKING_FILE}.img \
    -l com1,stdio \
    -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd \
    -s 2,virtio-net,tap0 \
    vm0
```

This command:
- `c 2`: Attaches 2 virtual CPUs
- `m 2G`: Allocates 2G of RAM
- `AHP`: Enables, ACPI, Hypervisor, and host CPU vendor
- `-s `: PCI device assignments (slot:function)

## Bhyve Internals (amd64)

### Main Function

1. Initialize configuration


```c
> bhyverun.c (638)
	bhyve_init_config();
	bhyve_optparse(argc, argv);
```

These functions just set the configuration options for Bhyve by parsing the command line arguments. For example, the `pci` devices are parsed as such:

```C
> bhyverun_machdep.c (183)
    case 's':
        if (strncmp(optarg, "help", strlen(optarg)) == 0) {
            pci_print_supported_devices();
            exit(0);
        } else if (pci_parse_slot(optarg) != 0)
            exit(4);
        else
            break;
```

2. Calculate Topology and Build vCPU Maps

```C
    calc_topology();
    build_vcpumaps();
```

`calc_topology` makes sure the sockets, cores, threads, and guest cpu variables are valid and set.
`build_vcpumaps` allocates a `vcpumap`. A `vcpumap` is just a double pointer to a `cpuset`

```C
    static cpuset_t **vcpumap;

    // Each vcpu gets a cpuset
	vcpumap = calloc(guest_ncpus, sizeof(*vcpumap));
	for (vcpu = 0; vcpu < guest_ncpus; vcpu++) {
		snprintf(key, sizeof(key), "vcpu.%d.cpuset", vcpu);
		value = get_config_value(key);
		if (value == NULL)
			continue;
		vcpumap[vcpu] = malloc(sizeof(cpuset_t));
		if (vcpumap[vcpu] == NULL)
			err(4, "Failed to allocate cpuset for vcpu %d", vcpu);
		parse_cpuset(vcpu, value, vcpumap[vcpu]);
	}
```

3. Creating the VM
Creating the VM is done in the `do_open` function.

```C
static struct vmctx *
do_open(const char *vmname)
{
	struct vmctx *ctx;
	int error;
	bool romboot;

	romboot = bootrom_boot(); /* Is there a bootrom? */

	/*
	 * If we don't have a boot ROM, the guest context must have been
	 * initialized by bhyveload(8) or equivalent.
	 */
	ctx = vm_openf(vmname, romboot ? VMMAPI_OPEN_REINIT : 0); // From Libvmmapi
	if (ctx == NULL) {
		if (errno != ENOENT)
			err(4, "vm_openf");
		if (!romboot)
			errx(4, "no bootrom was configured");
		ctx = vm_openf(vmname, VMMAPI_OPEN_CREATE);
		if (ctx == NULL)
			err(4, "vm_openf");
	}

	error = vm_set_topology(ctx, cpu_sockets, cpu_cores, cpu_threads, 0); // Set *calculated_topology* in vmm
	if (error)
		errx(EX_OSERR, "vm_set_topology");
	return (ctx);
}

```

The resulting `vmctx` instance includes file descriptors for the device and control interfaces, memory flags, memory segment information, and the name of the virtual machine.

4. Bootstrap CPU (and initialize others)

First, create the bootstrap processor and check the max number of cpus:

```C
	bsp = vm_vcpu_open(ctx, BSP);
	max_vcpus = num_vcpus_allowed(ctx, bsp);
```

How Bhyve checks the number of CPUs: 
```C
static int
num_vcpus_allowed(struct vmctx *ctx, struct vcpu *vcpu)
{
	uint16_t sockets, cores, threads, maxcpus;
	int tmp, error;

	/*
	 * The guest is allowed to spinup more than one processor only if the
	 * UNRESTRICTED_GUEST capability is available.
	 */
	error = vm_get_capability(vcpu, VM_CAP_UNRESTRICTED_GUEST, &tmp);
	if (error != 0)
		return (1);

	error = vm_get_topology(ctx, &sockets, &cores, &threads, &maxcpus);
	if (error == 0)
		return (maxcpus);
	else
		return (1);
}
```

Then, Bhyve initializes the bootstrap processor in machine dependent `bhyverun_machdep.c`. This involves just setting the capabilities for the vcpu. By default, the following two capabilities are set:
```C
	vm_set_capability(vcpu, VM_CAP_ENABLE_INVPCID, 1);

	err = vm_set_capability(vcpu, VM_CAP_IPI_EXIT, 1);
```

5. Initializing Virtual Hardware
Initialize per-VCPU resources:
```C
static struct vcpu_info {
	struct vmctx	*ctx;
	struct vcpu	*vcpu;
	int		vcpuid;
} *vcpu_info;


/* Allocate per-VCPU resources. */
vcpu_info = calloc(guest_ncpus, sizeof(*vcpu_info));
for (int vcpuid = 0; vcpuid < guest_ncpus; vcpuid++) {
    vcpu_info[vcpuid].ctx = ctx;
    vcpu_info[vcpuid].vcpuid = vcpuid;
    if (vcpuid == BSP)
        vcpu_info[vcpuid].vcpu = bsp;
    else
        vcpu_info[vcpuid].vcpu = vm_vcpu_open(ctx, vcpuid);
}
```


Setup memory. vm_setup_memory maps the memory to the `vmctx`:
```C
	memflags = 0;
	if (get_config_bool_default("memory.wired", false))
		memflags |= VM_MEM_F_WIRED;
	if (get_config_bool_default("memory.guest_in_core", false))
		memflags |= VM_MEM_F_INCORE;
	vm_set_memflags(ctx, memflags);
	error = vm_setup_memory(ctx, memsize, VM_MMAP_ALL);
	if (error) {
		fprintf(stderr, "Unable to setup memory (%d)\n", errno);
		exit(4);
	}
```

MMIO features are initialized via the `init_mem` function:
```C
void
init_mem(int ncpu)
{

	mmio_ncpu = ncpu;
	mmio_hint = calloc(ncpu, sizeof(*mmio_hint));
	RB_INIT(&mmio_rb_root);
	RB_INIT(&mmio_rb_fallback);
	pthread_rwlock_init(&mmio_rwlock, NULL);
}
```

6. Initializing other hardware (bootrom and qemu_fwcfg):

Bootrom is placed in the high part of the GPA to avoid conflict with guest RAM or low-memory devices
```C
void
init_bootrom(struct vmctx *ctx)
{
	vm_paddr_t highmem;

	romptr = vm_create_devmem(ctx, VM_BOOTROM, "bootrom", BOOTROM_SIZE);
	if (romptr == MAP_FAILED)
		err(4, "%s: vm_create_devmem", __func__);
	highmem = vm_get_highmem_base(ctx);
	gpa_base = highmem - BOOTROM_SIZE;
	gpa_allocbot = gpa_base;
	gpa_alloctop = highmem - 1;
}
```

QEMU `fw cfg` gives an interface for passing strings and files into the VM firmware. Operating systems can read configuration data provided by the host without needing disk or network access. Bhyve supports both this qemu fwcfg and also bhyve fwctl. We set the number of hardware CPUs in the guest for the firmware to read.

```C
	if (qemu_fwcfg_init(ctx) != 0) {
		fprintf(stderr, "qemu fwcfg initialization error\n");
		exit(4);
	}

	if (qemu_fwcfg_add_file("opt/bhyve/hw.ncpu", sizeof(guest_ncpus),
	    &guest_ncpus) != 0) {
		fprintf(stderr, "Could not add qemu fwcfg opt/bhyve/hw.ncpu\n");
		exit(4);
	}
```

Part 2 will cover initializing the platform.ww



## Sources
- Wiki: https://wiki.freebsd.org/bhyve
- Bhyvecon: https://www.youtube.com/watch?v=MTXUZLpCvfg&list=WL&index=4&t=2760s
