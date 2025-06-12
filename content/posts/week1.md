+++
title = "Week 1: Memory Allocation"
date = "2025-06-01T11:27:05-04:00"
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["Bhyve", "vmm"]
keywords = ["Bhyve", "VMM"]
description = "A deep dive into the memory subsystems of Bhyve and VMM"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
toc = true
+++

The main goal for week 1 is the mapping of memory segments from `QEMU` to vmm. This blog post will dive into how memory segments and EPT tables work in vmm (specifically Intel VMX). The last part will detail how these functions will be tailored to map QEMU regions to vmm.

## Bhyve
Lets look at how some of these segments are mapped in Bhyve itself. Primarily it will be worthwhile to look at how RAM and the bootrom are mapped.

RAM:
```C
...
	error = vm_setup_memory(ctx, memsize, VM_MMAP_ALL);
...
```
> bhyverun.c

Bootrom:
```C
...
	romptr = vm_create_devmem(ctx, VM_BOOTROM, "bootrom", BOOTROM_SIZE);
...
```
> `init_bootrom` in bootrom.c

```C
vm_mmap_memseg(ctx, gpa, VM_BOOTROM, segoff, len, prot);
```
> `bootrom_alloc` in bootrom.c

## `vmmapi`

Memory is handled through these functions in `vmmapi.h`
```C
int	vm_get_memseg(struct vmctx *ctx, int ident, size_t *lenp, char *name,
	    size_t namesiz);

/*
 * Iterate over the guest address space. This function finds an address range
 * that starts at an address >= *gpa.
 *
 * Returns 0 if the next address range was found and non-zero otherwise.
 */
int	vm_mmap_getnext(struct vmctx *ctx, vm_paddr_t *gpa, int *segid,
	    vm_ooffset_t *segoff, size_t *len, int *prot, int *flags);

int	vm_get_guestmem_from_ctx(struct vmctx *ctx, char **guest_baseaddr,
				 size_t *lowmem_size, size_t *highmem_size);

/*
 * Create a device memory segment identified by 'segid'.
 *
 * Returns a pointer to the memory segment on success and MAP_FAILED otherwise.
 */
void	*vm_create_devmem(struct vmctx *ctx, int segid, const char *name,
	    size_t len);

/*
 * Map the memory segment identified by 'segid' into the guest address space
 * at [gpa,gpa+len) with protection 'prot'.
 */
int	vm_mmap_memseg(struct vmctx *ctx, vm_paddr_t gpa, int segid,
	    vm_ooffset_t segoff, size_t len, int prot);

int	vm_munmap_memseg(struct vmctx *ctx, vm_paddr_t gpa, size_t len);

int	vm_setup_memory(struct vmctx *ctx, size_t len, enum vm_mmap_style s);
void	*vm_map_gpa(struct vmctx *ctx, vm_paddr_t gaddr, size_t len);
/* inverse operation to vm_map_gpa - extract guest address from host pointer */
vm_paddr_t vm_rev_map_gpa(struct vmctx *ctx, void *addr);
int	vm_gla2gpa_nofault(struct vcpu *vcpu,
		   struct vm_guest_paging *paging, uint64_t gla, int prot,
		   uint64_t *gpa, int *fault);
uint32_t vm_get_lowmem_limit(struct vmctx *ctx);
void	vm_set_memflags(struct vmctx *ctx, int flags);
int	vm_get_memflags(struct vmctx *ctx);
const char *vm_get_name(struct vmctx *ctx);
size_t	vm_get_lowmem_size(struct vmctx *ctx);
vm_paddr_t vm_get_highmem_base(struct vmctx *ctx);
size_t	vm_get_highmem_size(struct vmctx *ctx);
```
### `vm_setup_memory`
On amd64 machines, the memory layout looks like this:
| Address Range                     | Purpose                              |
|----------------------------------|--------------------------------------|
| `0x00000000–0xBFFFFFFF` (0–3 GB) | Regular guest RAM ("lowmem")         |
| `0xC0000000–0xFFFFFFFF` (3–4 GB) | **MMIO hole**, not usable for RAM    |
| `0x100000000+` (4 GB+)           | High memory region ("highmem")       |

Devices like PCI, APIC, video memory, and firmware ROMs live in this space.

As such `vm_setup_memory` first creates memory segments in the vmctx for low memory:
```C
...
	if (memsize > VM_LOWMEM_LIMIT) {
		ctx->memsegs[VM_MEMSEG_LOW].size = VM_LOWMEM_LIMIT;
		ctx->memsegs[VM_MEMSEG_HIGH].size = memsize - VM_LOWMEM_LIMIT;
		objsize = VM_HIGHMEM_BASE + ctx->memsegs[VM_MEMSEG_HIGH].size;
	} else {
		ctx->memsegs[VM_MEMSEG_LOW].size = memsize;
		ctx->memsegs[VM_MEMSEG_HIGH].size = 0;
		objsize = memsize;
	}

	error = vm_alloc_memseg(ctx, VM_SYSMEM, objsize, NULL);
...
```
If the total guest memory > 3Gb, the first 3G goes into the lowmem section. The remaining memory goes into the highmem section. Objsize is the total amount of memory needed. `vm_alloc_memseg` allocates the memory in the kernel, but nothing is yet mapped into the GPA.

```C
gpa = 4GB;
len = highmem_size;
setup_memory_segment(ctx, gpa, len, baseaddr);
```

```C
gpa = 0;
len = lowmem_size;
setup_memory_segment(ctx, gpa, len, baseaddr);
```

### `setup_memory_segment`

This function performs a dual mapping:

a) Map into guest physical address space
```C
vm_mmap_memseg(ctx, gpa, VM_SYSMEM, gpa, len, PROT_ALL)
```

b) Map same segment into host virtual space:
```C
mmap(baseaddr + gpa, len, PROT_RW, MAP_SHARED|MAP_FIXED, ctx->fd, gpa)
```

The vmm module has a `cdevsw` mmap function pointer that handles this mapping.

## Bootrom allocation
```C
...
	romptr = vm_create_devmem(ctx, VM_BOOTROM, "bootrom", BOOTROM_SIZE);
...
```

```C
...
	if (vm_mmap_memseg(ctx, gpa, VM_BOOTROM, segoff, len, prot) != 0) {
		int serrno = errno;
		warn("%s: vm_mmap_mapseg", __func__);
		return (serrno);
	}
...
```

The `vm_create_devmem` creates a named memory segment for a virtual machine and maps it into the host process's virtual address space.

```C
...
	fd = -1;
	ptr = MAP_FAILED;
	if (name == NULL || strlen(name) == 0) {
		errno = EINVAL;
		goto done;
	}

	error = vm_alloc_memseg(ctx, segid, len, name);
	if (error)
		goto done;

	strlcpy(pathname, "/dev/vmm.io/", sizeof(pathname));
	strlcat(pathname, ctx->name, sizeof(pathname));
	strlcat(pathname, ".", sizeof(pathname));
	strlcat(pathname, name, sizeof(pathname));

	fd = open(pathname, O_RDWR);
	if (fd < 0)
		goto done;
...
```
 The memory is named and has an actual `/dev/vmm.io/` descriptor associated with it. Then, the memory is mapped into the host space:

```C
...
	len2 = VM_MMAP_GUARD_SIZE + len + VM_MMAP_GUARD_SIZE;
	base = mmap(NULL, len2, PROT_NONE, MAP_GUARD | MAP_ALIGNED_SUPER, -1,
	    0);
	if (base == MAP_FAILED)
		goto done;

	flags = MAP_SHARED | MAP_FIXED;
	if ((ctx->memflags & VM_MEM_F_INCORE) == 0)
		flags |= MAP_NOCORE;

	/* mmap the devmem region in the host address space */
	ptr = mmap(base + VM_MMAP_GUARD_SIZE, len, PROT_RW, flags, fd, 0);
...
```
The mmap's map the kernel allocated memory into userspace. We first mmap an anonymous segment with some overflow regions (unmapped in GPA). Then, we map the kernel memory into the host address space.

## QEMU + `vmmapi` details
The physical memory layout on an x86_64 system looks like this:
0x00000000 - 0x000BFFFF: 768 KB of Low RAM
0x000C0000 - 0x000DFFFF: 128 KB of Device ROM (e.g., VGA BIOS) 
0x000E0000 - 0x000FFFFF: 128 KB of BIOS Extension ROM
0x00100000 - 0x7FFFFFFF: 2 GB - 1 MB of Main RAM
0x80000000 - 0xFFFBFFFF: (Implicit) MMIO Region / Gap (Guest OS doesn't use this for general RAM)
0xFFFC0000 - 0xFFFFFFFF: 256 KB of Main System BIOS ROM
0x100000000 - 0x200000000: 4 GB of High RAM

1) Qemu first allocates an N Gb block (where N is the size specifified by the `-m` flag. Then, the low memory and high memory segments are mapped into memory. (2Gb Low, N - 2Gb High)

2) Bios is allocated (256 Kb). PCIROM is allocated (128Kb).

3) Bios is mapped at 0xfffc0000 (full BIOS region). Entry point at 0xFFFF0 jumps to this full bios region mapped just below the highmem segment starts.

4) Initial 2Gb section is unmapped from the GPA.

5) 768Kb RAM segment is mapped at 0x00. 128kb ROM segment is mapped at 0xc0000 (Option ROM).

6) 128kb out of the 256kb bios is mapped into at 0xe0000. Therefore the first 1Mb of the physical address space is occupied for mostly legacy devices.

7) Finally the rest of the Lowmem Ram is mapped (again). It is mapped at 1Mb, and the size is 2Gb - 1Mb (that was already mapped).