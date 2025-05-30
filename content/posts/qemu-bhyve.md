+++
title = "Adding vmm(4) as an accelerator to QEMU"
date = "2025-05-23T11:31:25-04:00"
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["QEMU", "Bhyve"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

This post will be continually updated and serve as a reference for how to use `vmm(4)` as an accelerator for QEMU.

# Machine Initialization
Machine initialization is a key part of setting up the accelerator. In AccelClass, we must provide a machine intialization function:
```C
typedef struct AccelClass {
    /*< private >*/
    ObjectClass parent_class;
    /*< public >*/

    const char *name;
    int (*init_machine)(MachineState *ms);
    bool (*cpu_common_realize)(CPUState *cpu, Error **errp);
    void (*cpu_common_unrealize)(CPUState *cpu);
} AccelClass;
```

## System Memory Allocation
QEMU's machine initialization code determines how much RAM to allocate based on user configuration. It then uses `qemu_ram_alloc` to allocate a chunk of host memory that will serve as the guest's RAM.

Allocated memory is represented by a RAMBlock structure, and contains information about the HVA. The accelerator registers a RAMBlockNotifier (special memory listener for RAM Blocks) which runs a callback function that registers the memory.

Memory is allocated from the HostMemoryBackend of RAM. The memory allocator looks like this:

```C
static bool
ram_backend_memory_alloc(HostMemoryBackend *backend, Error **errp)
{
    g_autofree char *name = NULL;
    uint32_t ram_flags;

    if (!backend->size) {
        error_setg(errp, "can't create backend with size 0");
        return false;
    }

    name = host_memory_backend_get_name(backend);
    ram_flags = backend->share ? RAM_SHARED : RAM_PRIVATE;
    ram_flags |= backend->reserve ? 0 : RAM_NORESERVE;
    ram_flags |= backend->guest_memfd ? RAM_GUEST_MEMFD : 0;
    return memory_region_init_ram_flags_nomigrate(&backend->mr, OBJECT(backend),
                                                  name, backend->size,
                                                  ram_flags, errp);
}
```

At the heart of `memory_region_init_ram_flags_nomigrate` is `*qemu_ram_alloc_internal`. Which then registers RAM through `RAM_block_add`:

```C
static void ram_block_add(RAMBlock *new_block, Error **errp)
{
    const bool noreserve = qemu_ram_is_noreserve(new_block);
    const bool shared = qemu_ram_is_shared(new_block);
    RAMBlock *block;
    RAMBlock *last_block = NULL;
    bool free_on_error = false;
    ram_addr_t ram_size;
    Error *err = NULL;

    qemu_mutex_lock_ramlist();
    new_block->offset = find_ram_offset(new_block->max_length);

    if (!new_block->host) {
        if (xen_enabled()) {
            xen_ram_alloc(new_block->offset, new_block->max_length,
                          new_block->mr, &err);
            if (err) {
                error_propagate(errp, err);
                qemu_mutex_unlock_ramlist();
                return;
            }
        } else {
            new_block->host = qemu_anon_ram_alloc(new_block->max_length,
                                                  &new_block->mr->align,
                                                  shared, noreserve);
            if (!new_block->host) {
                error_setg_errno(errp, errno,
                                 "cannot set up guest memory '%s'",
                                 memory_region_name(new_block->mr));
                qemu_mutex_unlock_ramlist();
                return;
            }
            memory_try_enable_merging(new_block->host, new_block->max_length);
            free_on_error = true;
        }
    }
    ...
    
    ram_block_notify_add(new_block->host, new_block->used_length,
                            new_block->max_length);
}
```

When the `ram_block` is added, the notifier function triggers in the accelerator code. This is how that looks for NVMM:
```C
static void
nvmm_ram_block_added(RAMBlockNotifier *n, void *host, size_t size,
                     size_t max_size)
{
    struct nvmm_machine *mach = get_nvmm_mach();
    uintptr_t hva = (uintptr_t)host;
    int ret;

    ret = nvmm_hva_map(mach, hva, max_size);

    if (ret == -1) {
        error_report("NVMM: Failed to map HVA, HostVA:%p "
            "Size:%p bytes, error=%d",
            (void *)hva, (void *)size, errno);
    }
}

static struct RAMBlockNotifier nvmm_ram_notifier = {
    .ram_block_added = nvmm_ram_block_added
};
```

Because `vmm` requires you to request memory (it can't simply mmap userspace allocated memory), I plan to delay allocation from the qemu ram block add function to the notifier.
*TODO*: Ask QEMU developer mailing list the best way to accomplish this.

# State Synchronization
Another important piece of an accelerator is synchronization between the vCPUs and QEMU's CPUState.

The fundamental accelerator class `AccelOpsClass` requires several operations that will synchronize the states after system operations.

```C
...
    void (*create_vcpu_thread)(CPUState *cpu); /* MANDATORY NON-NULL */
    void (*kick_vcpu_thread)(CPUState *cpu);
    bool (*cpu_thread_is_idle)(CPUState *cpu);

    void (*synchronize_post_reset)(CPUState *cpu);
    void (*synchronize_post_init)(CPUState *cpu);
    void (*synchronize_state)(CPUState *cpu);
    void (*synchronize_pre_loadvm)(CPUState *cpu);
    void (*synchronize_pre_resume)(bool step_pending);
...
```

For example, this is how NVMM implements them:
```C
static void
do_nvmm_cpu_synchronize_state(CPUState *cpu, run_on_cpu_data arg)
{
    nvmm_get_registers(cpu);
    cpu->accel->dirty = true;
}

static void
do_nvmm_cpu_synchronize_post_reset(CPUState *cpu, run_on_cpu_data arg)
{
    nvmm_set_registers(cpu);
    cpu->accel->dirty = false;
}

static void
do_nvmm_cpu_synchronize_post_init(CPUState *cpu, run_on_cpu_data arg)
{
    nvmm_set_registers(cpu);
    cpu->accel->dirty = false;
}

static void
do_nvmm_cpu_synchronize_pre_loadvm(CPUState *cpu, run_on_cpu_data arg)
{
    cpu->accel->dirty = true;
}
```

## Register Synchronization 

vmm features similar functions that we can use to synchronize state between `CPUState*` and `vcpu*`.
```C
int	vm_set_register(struct vcpu *vcpu, int reg, uint64_t val);
int	vm_get_register(struct vcpu *vcpu, int reg, uint64_t *retval);
int	vm_set_register_set(struct vcpu *vcpu, unsigned int count,
    const int *regnums, uint64_t *regvals);
int	vm_get_register_set(struct vcpu *vcpu, unsigned int count,
    const int *regnums, uint64_t *regvals);
```

## Interrupt Delivery 
`vmm` features several peripherals in kernel for performance reasons (IOAPIC, LAPIC, i8259 PIC, HPET, RTC, PIT). For an initial implementation of an accelerator, these devices will be turned off, essentially allowing QEMU to emulate the entire. 

I plan on adding a separate flag to the `libvmmapi`/`vm_openf()` function that will specify that the virtual machine is being operated under QEMU. This QEMU mode will disable the following devices:

```C
...
	vm->vioapic = vioapic_init(vm);
	vm->vhpet = vhpet_init(vm);
	vm->vatpic = vatpic_init(vm);
	vm->vatpit = vatpit_init(vm);
	vm->vpmtmr = vpmtmr_init(vm);
	if (create)
		vm->vrtc = vrtc_init(vm);
...
```

And in the vcpu:

```C
	vcpu->vlapic = vmmops_vlapic_init(vcpu->cookie);
```

I will then create a generic event injection mechanism in `libvmmapi` in QEMU mode that plugs directly into QEMU's event mechanisms. In the vmm backend, these events will be directly injected into the VMCS or VMCB.

### Kernel LAPIC
The downside of this approach is that the performance will be somewhat limited. MSI/MSI-X will be slightly slower as they will operate in user mode. This means that every interrupt operation requires a VMExit into the hypervisor, followed by a trap into QEMU, and then re-entry into the VM. This process is altogether quite expensive.

Keeping the LAPIC in the kernel means that there is no user-kernel transition when a guest sends an IPI or receives an interrupt. The interrupts can be directly injected into the vCPU. This `kernel_irqchip` functionality is part of my plan to be implemented for the second part of this project. I plan to create a new QOM (Qemu Object Model) device (vmm-apic) that will be used on the backend of QEMUs LAPIC.

## VMExits
### MMIO
One special `vmexit` case is MMIO. The process to handle these is somewhat straightforward.

1. Check the VM Exit reason
Verify the exit was called for an EPT Violation.

2. Get the vCPU state
Load all relevant CPU state to accurately emulate

3. Fetch the instruction

4. Decode the instruction
Helps understand the type, operands, size, and direction of the instruction. 

5. Prepare data and call callback function in QEMU.
Because Callback function will use the decoded information to somehow alter the QEMU address space. This is what an MMIO callback looks like:
```C
...
    cpu_physical_memory_rw(mem->gpa, mem->data, mem->size, mem->write);

    /* Needed, otherwise infinite loop. */
    current_cpu->accel->dirty = false;
...
```
 Because MMIO is not directly mapped to the GPA, the callback function modifies the QEMU address space.

6. Next, the guest state needs to be updated to reflect the emulation of the instructions

### PMIO
Port mapped IO has a very similar process.

1. Check VM Exit reason
Ensure that the exit reason was for IO

2. Fetch state

3. Call the I/O callback
Similarly for PMIO, the function modifies the io address space. Here is an example from NVMM:
```C
static void
nvmm_io_callback(struct nvmm_io *io)
{
    MemTxAttrs attrs = { 0 };
    int ret;

    ret = address_space_rw(&address_space_io, io->port, attrs, io->data,
        io->size, !io->in);
    if (ret != MEMTX_OK) {
        error_report("NVMM: I/O Transaction Failed "
            "[%s, port=%u, size=%zu]", (io->in ? "in" : "out"),
            io->port, io->size);
    }

    /* Needed, otherwise infinite loop. */
    current_cpu->accel->dirty = false;
}
```

4. Update instruction pointer

### MSR
An MSR is a special-purpose register for controlling or reporting processor-specific features. They are accessed via the `RDMSR` and `WRMSR` instructions. Each RDMSR and WRMSR needs to have a specified handler.