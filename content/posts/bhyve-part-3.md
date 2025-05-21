+++
title = "Bhyve Part 3"
date = "2025-05-15T14:22:03-04:00"
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["Bhyve", "FreeBSD"]
keywords = ["", ""]
description = "Machine Initialization Routines in Bhyve"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
toc=true
+++

At this point, we're ready to initialize the machine by starting all the vCPUs:
```C
	for (int vcpuid = 0; vcpuid < guest_ncpus; vcpuid++)
		bhyve_start_vcpu(vcpu_info[vcpuid].vcpu, vcpuid == BSP);
```

## Start vCPUs

```C
void
bhyve_start_vcpu(struct vcpu *vcpu, bool bsp)
{
	int error;

	if (bsp) {
		if (bootrom_boot()) {
			error = vm_set_capability(vcpu,
			    VM_CAP_UNRESTRICTED_GUEST, 1);
			if (error != 0) {
				err(4, "ROM boot failed: unrestricted guest "
				    "capability not available");
			}
			error = vcpu_reset(vcpu);
			assert(error == 0);
		}
	} else {
		bhyve_init_vcpu(vcpu);

		/*
		 * Enable the 'unrestricted guest' mode for APs.
		 *
		 * APs startup in power-on 16-bit mode.
		 */
		error = vm_set_capability(vcpu, VM_CAP_UNRESTRICTED_GUEST, 1);
		assert(error == 0);
	}

	fbsdrun_addcpu(vcpu_id(vcpu));
}
```

The bootstrap processor is started first. If the guest is being booted with a bootrom, the UNRESTRICTED_GUEST capability is set, and the vcpu is reset. Non bootsstrap processors are just now initialized.

```C
void
fbsdrun_addcpu(int vcpuid)
{
	struct vcpu_info *vi;
	pthread_t thr;
	int error;

	vi = &vcpu_info[vcpuid];

	error = vm_activate_cpu(vi->vcpu);
	if (error != 0)
		err(EX_OSERR, "could not activate CPU %d", vi->vcpuid);

	CPU_SET_ATOMIC(vcpuid, &cpumask);

	error = vm_suspend_cpu(vi->vcpu);
	assert(error == 0);

	error = pthread_create(&thr, NULL, fbsdrun_start_thread, vi);
	assert(error == 0);
}
```

The thread function is what calls the `libvmmapi` `vm_loop` in each thread.

```C
static void
vm_loop(struct vmctx *ctx, struct vcpu *vcpu)
{
	struct vm_exit vme;
	struct vm_run vmrun;
	int error, rc;
	enum vm_exitcode exitcode;
	cpuset_t active_cpus, dmask;

	error = vm_active_cpus(ctx, &active_cpus);
	assert(CPU_ISSET(vcpu_id(vcpu), &active_cpus));

	vmrun.vm_exit = &vme;
	vmrun.cpuset = &dmask;
	vmrun.cpusetsize = sizeof(dmask);

	while (1) {
		error = vm_run(vcpu, &vmrun);
		if (error != 0)
			break;

		exitcode = vme.exitcode;
		if (exitcode >= VM_EXITCODE_MAX ||
		    vmexit_handlers[exitcode] == NULL) {
			warnx("vm_loop: unexpected exitcode 0x%x", exitcode);
			exit(4);
		}

		rc = (*vmexit_handlers[exitcode])(ctx, vcpu, &vmrun);

		switch (rc) {
		case VMEXIT_CONTINUE:
			break;
		case VMEXIT_ABORT:
			abort();
		default:
			exit(4);
		}
	}
	EPRINTLN("vm_run error %d, errno %d", error, errno);
}
```

This is a pretty basic virtualization workflow. The vCPU is set to run until a vmexit occurs. The vmexit is then sent to the handler, which repeats the process.

## PCI device walkthrough
It's useful to see how the lifecycle of a device in Bhyve as well