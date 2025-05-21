+++
title = "How a QEMU Accelerator works"
date = "2025-05-18T13:14:09-04:00"
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["QEMU", "NVMM"]
keywords = ["Qemu", "nvmm"]
description = "Inside the NVMM accelerator for QEMU"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

Qemu has a modular accelerator framework that is made of two components:
- `AccelClass`: The base accelerator capabilities
- `AccelOpsClass`: CPU specific operations for the accelerator

## Type Registration
These two types are initiated in `target/i386/nvmm/nvmm-all.c` and `target/i386/nvmm/nvmm-accel-ops.c`

```C
static void
nvmm_accel_class_init(ObjectClass *oc, const void *data)
{
    AccelClass *ac = ACCEL_CLASS(oc);
    ac->name = "NVMM";
    ac->init_machine = nvmm_accel_init;
    ac->allowed = &nvmm_allowed;
}

static const TypeInfo nvmm_accel_type = {
    .name = ACCEL_CLASS_NAME("nvmm"),
    .parent = TYPE_ACCEL,
    .class_init = nvmm_accel_class_init,
};

static void
nvmm_type_init(void)
{
    type_register_static(&nvmm_accel_type);
}

type_init(nvmm_type_init);
```
> Initializes nvmm machine and provides other functions to assist with code execution.

and

```C
static void nvmm_accel_ops_class_init(ObjectClass *oc, const void *data)
{
    AccelOpsClass *ops = ACCEL_OPS_CLASS(oc);

    ops->create_vcpu_thread = nvmm_start_vcpu_thread;
    ops->kick_vcpu_thread = nvmm_kick_vcpu_thread;

    ops->synchronize_post_reset = nvmm_cpu_synchronize_post_reset;
    ops->synchronize_post_init = nvmm_cpu_synchronize_post_init;
    ops->synchronize_state = nvmm_cpu_synchronize_state;
    ops->synchronize_pre_loadvm = nvmm_cpu_synchronize_pre_loadvm;
}

static const TypeInfo nvmm_accel_ops_type = {
    .name = ACCEL_OPS_NAME("nvmm"),

    .parent = TYPE_ACCEL_OPS,
    .class_init = nvmm_accel_ops_class_init,
    .abstract = true,
};

static void nvmm_accel_ops_register_types(void)
{
    type_register_static(&nvmm_accel_ops_type);
}
type_init(nvmm_accel_ops_register_types);
```
> Executes the main thread for the vCPU and qemu `CPUState` and nvmm vCPU synchronization.

## CPU Class integration
AccelCPUState. This is the core of the accelerator functionality. It features an `nvmm_vcpu`, a task priority register, and other accelerator state. Each Qemu CPU is tied to an `AccelCPUState`.
```C
struct AccelCPUState {
    struct nvmm_vcpu vcpu;
    uint8_t tpr;
    bool stop;
    bool dirty;

    /* Window-exiting for INTs/NMIs. */
    bool int_window_exit;
    bool nmi_window_exit;

    /* The guest is in an interrupt shadow (POP SS, etc). */
    bool int_shadow;
};

struct qemu_machine {
    struct nvmm_capability cap;
    struct nvmm_machine mach;
};
```

### vCPU initialization
```C
int
nvmm_init_vcpu(CPUState *cpu)
{
    struct nvmm_machine *mach = get_nvmm_mach();
    struct nvmm_vcpu_conf_cpuid cpuid;
    struct nvmm_vcpu_conf_tpr tpr;
    Error *local_error = NULL;
    AccelCPUState *qcpu;
    int ret, err;

    nvmm_init_cpu_signals();

    /* Initialize AccelCPUState with glib */
    qcpu = g_new0(AccelCPUState, 1);

    /* Create vcpu (nvmm.h) */
    ret = nvmm_vcpu_create(mach, cpu->cpu_index, &qcpu->vcpu);
    if (ret == -1) {
        err = errno;
        error_report("NVMM: Failed to create a virtual processor,"
            " error=%d", err);
        g_free(qcpu);
        return -err;
    }

    /*
        Set cpuid information.
        CPUID gives software a standardized way to query processor features.

        Leaf is just the function number passed in the EAX register before executing CPUID. Which category of information to return.

        Mask is the bitmask that extracts the feature flags from the results.
    */
    memset(&cpuid, 0, sizeof(cpuid));
    cpuid.mask = 1;
    cpuid.leaf = 0x00000001;
    cpuid.u.mask.set.edx = CPUID_MCE | CPUID_MCA | CPUID_MTRR;
    ret = nvmm_vcpu_configure(mach, &qcpu->vcpu, NVMM_VCPU_CONF_CPUID,
        &cpuid);
        
    if (ret == -1) {
        err = errno;
        error_report("NVMM: Failed to configure a virtual processor,"
            " error=%d", err);
        g_free(qcpu);
        return -err;
    }

    /* Configures the callbacks (io callback, mem callback) */
    ret = nvmm_vcpu_configure(mach, &qcpu->vcpu, NVMM_VCPU_CONF_CALLBACKS,
        &nvmm_callbacks);
    if (ret == -1) {
        err = errno;
        error_report("NVMM: Failed to configure a virtual processor,"
            " error=%d", err);
        g_free(qcpu);
        return -err;
    }

    /* Sets TPR */
    if (qemu_mach.cap.arch.vcpu_conf_support & NVMM_CAP_ARCH_VCPU_CONF_TPR) {
        memset(&tpr, 0, sizeof(tpr));
        tpr.exit_changed = 1;
        ret = nvmm_vcpu_configure(mach, &qcpu->vcpu, NVMM_VCPU_CONF_TPR, &tpr);
        if (ret == -1) {
            err = errno;
            error_report("NVMM: Failed to configure a virtual processor,"
                " error=%d", err);
            g_free(qcpu);
            return -err;
        }
    }

    /* Sets the CPUState accelClass to the AccelCPUState */
    qcpu->dirty = true;
    cpu->accel = qcpu;

    return 0;
}
```

### Register Synchronization
NVMM provides functions to synchronize registers between Qemu CPUState and NVMM vCPU:
```C
// QEMU CPU → NVMM
static void nvmm_set_registers(CPUState *cpu) {
    // Transfer GPRs, RIP/RFLAGS, segments, control registers, etc.
    state->gprs[NVMM_X64_GPR_RAX] = env->regs[R_EAX];
    state->gprs[NVMM_X64_GPR_RCX] = env->regs[R_ECX];
    state->gprs[NVMM_X64_GPR_RDX] = env->regs[R_EDX];
    state->gprs[NVMM_X64_GPR_RBX] = env->regs[R_EBX];
    state->gprs[NVMM_X64_GPR_RSP] = env->regs[R_ESP];
    state->gprs[NVMM_X64_GPR_RBP] = env->regs[R_EBP];
    state->gprs[NVMM_X64_GPR_RSI] = env->regs[R_ESI];
    state->gprs[NVMM_X64_GPR_RDI] = env->regs[R_EDI];
    ...
}

// NVMM → QEMU CPU
static void nvmm_get_registers(CPUState *cpu) {
    // Transfer GPRs, RIP/RFLAGS, segments, control registers, etc.
    /* GPRs. */
    env->regs[R_EAX] = state->gprs[NVMM_X64_GPR_RAX];
    env->regs[R_ECX] = state->gprs[NVMM_X64_GPR_RCX];
    env->regs[R_EDX] = state->gprs[NVMM_X64_GPR_RDX];
    env->regs[R_EBX] = state->gprs[NVMM_X64_GPR_RBX];
    env->regs[R_ESP] = state->gprs[NVMM_X64_GPR_RSP];
    env->regs[R_EBP] = state->gprs[NVMM_X64_GPR_RBP];
    env->regs[R_ESI] = state->gprs[NVMM_X64_GPR_RSI];
    env->regs[R_EDI] = state->gprs[NVMM_X64_GPR_RDI];
    ...
}
```

These functions are called when CPU State needs synchronization, after CPU reset, initialization, and before loading VM state.

### vCPU execution thread
This is the actual thread function for each qemu CPU. `accel-ops-class.c`
```C
static void *qemu_nvmm_cpu_thread_fn(void *arg)
{
    CPUState *cpu = arg;
    int r;

    assert(nvmm_enabled());

    rcu_register_thread();

    /* Acquire QEMU Lock */
    bql_lock();
    qemu_thread_get_self(cpu->thread);
    cpu->thread_id = qemu_get_thread_id();
    /* Set current CPU */
    current_cpu = cpu;

    /* Create vCPU */
    r = nvmm_init_vcpu(cpu);
    if (r < 0) {
        fprintf(stderr, "nvmm_init_vcpu failed: %s\n", strerror(-r));
        exit(1);
    }

    /* signal CPU creation */
    cpu_thread_signal_created(cpu);
    qemu_guest_random_seed_thread_part2(cpu->random_seed);

    do {
        if (cpu_can_run(cpu)) {
            r = nvmm_vcpu_exec(cpu);
            if (r == EXCP_DEBUG) {
                cpu_handle_guest_debug(cpu);
            }
        }
        while (cpu_thread_is_idle(cpu)) {
            /* Condition variable with BQL */
            qemu_cond_wait_bql(cpu->halt_cond);
        }
        qemu_wait_io_event_common(cpu);
    } while (!cpu->unplug || cpu_can_run(cpu));

    /* Destroy vCPU after CPU is unplugged */
    nvmm_destroy_vcpu(cpu);
    cpu_thread_signal_destroyed(cpu);
    bql_unlock();
    rcu_unregister_thread();
    return NULL;
}

static void nvmm_start_vcpu_thread(CPUState *cpu)
{
    char thread_name[VCPU_THREAD_NAME_SIZE];

    snprintf(thread_name, VCPU_THREAD_NAME_SIZE, "CPU %d/NVMM",
             cpu->cpu_index);
    /* Start vCPU thread */
    qemu_thread_create(cpu->thread, thread_name, qemu_nvmm_cpu_thread_fn,
                       cpu, QEMU_THREAD_JOINABLE);
}
```

## vCPU execution
The execution loop takes place in the `nvmm_vcpu_loop` function:
```C
static int
nvmm_vcpu_loop(CPUState *cpu) {
    struct nvmm_machine *mach = get_nvmm_mach();
    AccelCPUState *qcpu = cpu->accel;
    struct nvmm_vcpu *vcpu = &qcpu->vcpu;
    X86CPU *x86_cpu = X86_CPU(cpu);
    CPUX86State *env = &x86_cpu->env;
    struct nvmm_vcpu_exit *exit = vcpu->exit;
    cpu_exec_start(cpu);

    /*
     * Inner VCPU loop.
     */
    do {
        if (cpu->accel->dirty) {
            nvmm_set_registers(cpu);
            cpu->accel->dirty = false;
        }

        if (qcpu->stop) {
            cpu->exception_index = EXCP_INTERRUPT;
            qcpu->stop = false;
            ret = 1;
            break;
        }

        nvmm_vcpu_pre_run(cpu);

        if (qatomic_read(&cpu->exit_request)) {
#if NVMM_USER_VERSION >= 2
            nvmm_vcpu_stop(vcpu);
#else
            qemu_cpu_kick_self();
#endif
        }

        /* Read exit_request before the kernel reads the immediate exit flag */
        smp_rmb();
        ret = nvmm_vcpu_run(mach, vcpu);
        if (ret == -1) {
            error_report("NVMM: Failed to exec a virtual processor,"
                " error=%d", errno);
            break;
        }

        nvmm_vcpu_post_run(cpu, exit);
        ...
        switch (exit->reason) {
        case NVMM_VCPU_EXIT_NONE:
            break;
#if NVMM_USER_VERSION >= 2
        case NVMM_VCPU_EXIT_STOPPED:
            /*
             * The kernel cleared the immediate exit flag; cpu->exit_request
             * must be cleared after
             */
            smp_wmb();
            qcpu->stop = true;
            break;
#endif
        case NVMM_VCPU_EXIT_MEMORY:
            ret = nvmm_handle_mem(mach, vcpu);
            break;
        case NVMM_VCPU_EXIT_IO:
            ret = nvmm_handle_io(mach, vcpu);
            break;
        case NVMM_VCPU_EXIT_INT_READY:
        case NVMM_VCPU_EXIT_NMI_READY:
        case NVMM_VCPU_EXIT_TPR_CHANGED:
            break;
        case NVMM_VCPU_EXIT_HALTED:
            ret = nvmm_handle_halted(mach, cpu, exit);
            break;
        case NVMM_VCPU_EXIT_SHUTDOWN:
            qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
            cpu->exception_index = EXCP_INTERRUPT;
            ret = 1;
            break;
        case NVMM_VCPU_EXIT_RDMSR:
            ret = nvmm_handle_rdmsr(mach, cpu, exit);
            break;
        case NVMM_VCPU_EXIT_WRMSR:
            ret = nvmm_handle_wrmsr(mach, cpu, exit);
            break;
        case NVMM_VCPU_EXIT_MONITOR:
        case NVMM_VCPU_EXIT_MWAIT:
            ret = nvmm_inject_ud(mach, vcpu);
            break;
        default:
            error_report("NVMM: Unexpected VM exit code 0x%lx [hw=0x%lx]",
                exit->reason, exit->u.inv.hwcode);

}
```

The loop calls a prerun function, executes the VM in VMX mode, and returns an exitcode at `vmexit`. Then, we provide match the different exitcodes to various handlers.

### Pre-run and Post Run
The pre-run is called before the vCPU is run. It injects events generated by the I/O thread, and synchronizes the guest TPR.

```C
...
    tpr = cpu_get_apic_tpr(x86_cpu->apic_state);
    if (tpr != qcpu->tpr) {
        qcpu->tpr = tpr;
        sync_tpr = true;
    }

    /*
     * Force the VCPU out of its inner loop to process any INIT requests
     * or commit pending TPR access.
     */
    if (cpu->interrupt_request & (CPU_INTERRUPT_INIT | CPU_INTERRUPT_TPR)) {
        cpu->exit_request = 1;
    }

    if (!has_event && (cpu->interrupt_request & CPU_INTERRUPT_NMI)) {
        if (nvmm_can_take_nmi(cpu)) {
            cpu->interrupt_request &= ~CPU_INTERRUPT_NMI;
            event->type = NVMM_VCPU_EVENT_INTR;
            event->vector = 2;
            has_event = true;
        }
    }

    if (!has_event && (cpu->interrupt_request & CPU_INTERRUPT_HARD)) {
        if (nvmm_can_take_int(cpu)) {
            cpu->interrupt_request &= ~CPU_INTERRUPT_HARD;
            event->type = NVMM_VCPU_EVENT_INTR;
            event->vector = cpu_get_pic_interrupt(env);
            has_event = true;
        }
    }

    /* Don't want SMIs. */
    if (cpu->interrupt_request & CPU_INTERRUPT_SMI) {
        cpu->interrupt_request &= ~CPU_INTERRUPT_SMI;

    /* Sync TPR */

    /* Inject event into vCPU. Event is injected at the next interrupt window */
    if (has_event) {
        ret = nvmm_vcpu_inject(mach, vcpu);
        if (ret == -1) {
            error_report("NVMM: Failed to inject event,"
                " error=%d", errno);
        }
    }
```

The post run is called after vCPU runs (and exits). We synchronize the host view of the TPR and RFLAGS.

```C
static void
nvmm_vcpu_post_run(CPUState *cpu, struct nvmm_vcpu_exit *exit)
{
    AccelCPUState *qcpu = cpu->accel;
    X86CPU *x86_cpu = X86_CPU(cpu);
    CPUX86State *env = &x86_cpu->env;
    uint64_t tpr;

    env->eflags = exit->exitstate.rflags;
    qcpu->int_shadow = exit->exitstate.int_shadow;
    qcpu->int_window_exit = exit->exitstate.int_window_exiting;
    qcpu->nmi_window_exit = exit->exitstate.nmi_window_exiting;

    /* Synchronize TPR */
    tpr = exit->exitstate.cr8;
    if (qcpu->tpr != tpr) {
        qcpu->tpr = tpr;
        bql_lock();
        cpu_set_apic_tpr(x86_cpu->apic_state, qcpu->tpr);
        bql_unlock();
    }
}
```

## Machine Initialization
Qemu initializes the machine as follows:
```C
int accel_init_machine(AccelState *accel, MachineState *ms)
{
    ms->accelerator = accel;
    *(acc->allowed) = true;
    ret = acc->init_machine(ms); // Calls nvmm_accel_init()
    // ...
}
```

The `init_machine` function is one of the defining attributes of the `AccelState`:

```C

static int
nvmm_accel_init(MachineState *ms)
{
    int ret, err;

    ret = nvmm_init();

    ret = nvmm_capability(&qemu_mach.cap);

    ret = nvmm_machine_create(&qemu_mach.mach);

    memory_listener_register(&nvmm_memory_listener, &address_space_memory);
    ram_block_notifier_add(&nvmm_ram_notifier);

    return 0;
}
```

### Map Memory
```C
static void
nvmm_update_mapping(hwaddr start_pa, ram_addr_t size, uintptr_t hva,
    bool add, bool rom, const char *name)
{
    struct nvmm_machine *mach = get_nvmm_mach();
    int ret, prot;

    if (add) {
        prot = PROT_READ | PROT_EXEC;
        if (!rom) {
            prot |= PROT_WRITE;
        }
        ret = nvmm_gpa_map(mach, hva, start_pa, size, prot);
    } else {
        ret = nvmm_gpa_unmap(mach, hva, start_pa, size);
    }

    if (ret == -1) {
        error_report("NVMM: Failed to %s GPA range '%s' PA:%p, "
            "Size:%p bytes, HostVA:%p, error=%d",
            (add ? "map" : "unmap"), name, (void *)(uintptr_t)start_pa,
            (void *)size, (void *)hva, errno);
    }
}

static void
nvmm_process_section(MemoryRegionSection *section, int add)
{
    MemoryRegion *mr = section->mr;
    hwaddr start_pa = section->offset_within_address_space;
    ram_addr_t size = int128_get64(section->size);
    unsigned int delta;
    uintptr_t hva;

    if (!memory_region_is_ram(mr)) {
        return;
    }

    /* Adjust start_pa and size so that they are page-aligned. */
    delta = qemu_real_host_page_size() - (start_pa & ~qemu_real_host_page_mask());
    delta &= ~qemu_real_host_page_mask();
    if (delta > size) {
        return;
    }
    start_pa += delta;
    size -= delta;
    size &= qemu_real_host_page_mask();
    if (!size || (start_pa & ~qemu_real_host_page_mask())) {
        return;
    }

    hva = (uintptr_t)memory_region_get_ram_ptr(mr) +
        section->offset_within_region + delta;

    nvmm_update_mapping(start_pa, size, hva, add,
        memory_region_is_rom(mr), mr->name);
}

static void
nvmm_region_add(MemoryListener *listener, MemoryRegionSection *section)
{
    memory_region_ref(section->mr);
    nvmm_process_section(section, 1);
}

static MemoryListener nvmm_memory_listener = {
    .name = "nvmm",
    .begin = nvmm_transaction_begin,
    .commit = nvmm_transaction_commit,
    .region_add = nvmm_region_add,
    .region_del = nvmm_region_del,
    .log_sync = nvmm_log_sync,
    .priority = MEMORY_LISTENER_PRIORITY_ACCEL,
};
```

This object specifies how to add and delete memory regions to the guest physical space. `nvmm`, unlike `vmm` on FreeBSD, has memory allocated in userspace, and then mapped into the HVA of the virtual machine. I will dive into how this works in a later blog.

System memory is allocated as such:
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