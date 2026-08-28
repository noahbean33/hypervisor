# Bao Hypervisor — Implemented Feature Reference

A detailed inventory of what this source tree actually implements, derived by reading the code in `archive/bao/`.

- **License:** Apache-2.0 (`SPDX-License-Identifier: Apache-2.0` on every file)
- **Size:** ~37,900 lines of C / headers / assembly across 476 files
- **Type:** Bare-metal (type-1) **static-partitioning** hypervisor for embedded/mixed-criticality systems
- **Language:** Freestanding C11 (`-ffreestanding -std=c11 -fno-pic -fno-pie`), plus per-arch assembly

| Area | LOC |
| --- | --- |
| `src/arch/armv8` | 10,090 |
| `src/arch/riscv` | 6,968 |
| `src/platform` (24 boards + 11 UART drivers) | 6,835 |
| `src/core` | 5,892 |
| `src/arch/tricore` | 3,722 |
| `src/arch/rh850` | 3,134 |
| `src/lib` | 937 |
| `scripts` + `configs` | 368 |

---

## 1. Core design model

The hypervisor implements **static partitioning**, not time-sharing virtualization. The consequences of that choice show up everywhere in the code:

- **No scheduler.** There is no run queue, no time slicing, no vCPU migration. Every vCPU is statically pinned 1:1 to a physical CPU at boot (`vmm_assign_vcpu()` in `src/core/vmm.c`). `vcpu_run()` (`src/core/vm.c:423`) either enters the guest, parks the core in standby, or powers it down — there is nothing else to run.
- **Compile-time configuration.** The entire system topology (VMs, memory regions, devices, interrupts, shared memory) is a C struct (`struct config`) linked into the binary. There is no runtime VM creation, destruction, or reconfiguration.
- **Boot-time-only resource assignment.** Memory, interrupts, and devices are handed to VMs during `vm_init()` and never revoked.
- **Minimal trusted computing base.** The hypervisor emulates only the interrupt controller and the minimum of platform firmware calls; everything else is either passed through to a guest or delegated to a *backend VM* over the Remote I/O interface.

### Boot sequence (`src/core/init.c`)

```
cpu_init()        -> per-CPU structures, per-CPU stack
mem_init()        -> hypervisor address space, page pools, optional self-recoloring
platform_init()   -> platform description, cache enumeration, IOMMU
console_init()    -> map + init UART, master CPU only
interrupts_init() -> interrupt controller, IPI reservation
vmm_init()        -> vCPU assignment, VM allocation/install, vm_init(), vcpu_run()
```

Every phase is separated by `cpu_sync_barrier()` on `cpu_glb_sync`; a designated **master CPU** performs all singleton work while the others wait.

---

## 2. Architecture support matrix

| Architecture | Sub-arch / profile | Memory protection | Virtual IRQ controller | Firmware interface emulated |
| --- | --- | --- | --- | --- |
| **Armv8-A** | AArch64, AArch32 | Stage-2 MMU | vGICv2 / vGICv3 | PSCI (via SMC trapping) |
| **Armv8-R** | AArch64, AArch32 (Cortex-R52) | MPU (optionally MMU per-VM on AArch64) | vGICv3 | PSCI |
| **RISC-V** | RV64, RV32 (H-extension) | Stage-2 MMU (`hgatp`) | vPLIC / vAPLIC + vIMSIC (AIA) | SBI (Base/TIME/IPI/RFENCE/HSM) |
| **Renesas RH850** | U2A16 | MPU (hardware guest mode, SPID/GPID) | vINTC + vIPIR + virtual boot controller | — (native guest mode) |
| **Infineon TriCore** | TC4Dx | MPU + slave-side MMIO protection (PROT/APU) | vIR (virtual interrupt router) | — |

Selected at build time via `ARCH`, `ARCH_SUB`, `ARCH_PROFILE` in the platform makefile. The core is compiled against either the **MMU** back end (`src/core/mmu/`) or the **MPU** back end (`src/core/mpu/`), chosen by `arch_mem_prot` and exposed as `-DMEM_PROT_MMU` / `-DMEM_PROT_MPU`.

---

## 3. Memory management

### 3.1 Two interchangeable memory-protection back ends

Both implement the same core interface (`src/core/inc/mem.h`): `mem_alloc_page`, `mem_alloc_map`, `mem_alloc_map_dev`, `mem_unmap`, `mem_map_cpy`, `mem_translate`.

**MMU back end** (`src/core/mmu/`)

- Multi-level page tables with an architecture-supplied descriptor (`struct page_table`, `src/core/inc/page_table.h`); Armv8 LPAE tables in `src/arch/armv8/armv8-a/page_table.c`, RISC-V tables in `src/arch/riscv/page_table.c`.
- Separate hypervisor and per-VM (stage-2) address spaces, tagged with VMID/ASID.
- Address-space **sections** (`SEC_HYP_GLOBAL`, `SEC_HYP_PRIVATE`, `SEC_HYP_VM`, `SEC_VM_ANY`) that segregate global, per-CPU, and per-VM mappings.
- TLB maintenance hooks per architecture (`arch/tlb.h`, `tlb_vm_inv_all`).
- Root page tables built in assembly at boot (`root_pt.S`, `pagetables.S`).

**MPU back end** (`src/core/mpu/`)

- A software "virtual MPU" of 64 entries per address space (`VMPU_NUM_ENTRIES`), kept as an ordered list of `struct mp_region` with `MPE_S_FREE / MPE_S_INVALID / MPE_S_VALID` states.
- Region merging/splitting, lockable entries, broadcast updates to other cores.
- A thin physical-MPU port layer per architecture: `mpu_init`, `mpu_enable`, `mpu_map`, `mpu_unmap`, `mpu_update`, `mpu_perms_compatible` — implemented for Armv8-R (`armv8-r/mpu.c`), RH850 (`rh850/mpu.c`), and TriCore (`tricore/mpu.c`).
- Supports **non-unified memory** platforms (`plat_mem := non_unified`, `-DMEM_NON_UNIFIED`) where code and data live in separate physical regions with independently relocatable base addresses.

### 3.2 Physical page allocator

- Per-memory-region page pools (`struct page_pool`) with a bitmap allocator, a `last`-position hint, and a spinlock (`src/core/mmu/mem.c`).
- Supports alignment-constrained allocation (`MEM_ALIGN_REQ`) for page-table levels.

### 3.3 Cache coloring (partition-level cache isolation)

A fully implemented, non-trivial feature:

- `cache_enumerate()` / `cache_calc_colors()` (`src/core/cache.c`) derive `COLOR_NUM` and `COLOR_SIZE` from the enumerated cache hierarchy — requires the last shared level to be **unified and PIPT**, and accounts for first-level way size to avoid aliasing.
- `pp_alloc_clr()` allocates page runs matching a `colormap_t`.
- `mem_map_reclr()` **re-colors an already-loaded VM image in place**: it counts the pages that fall outside the VM's color set, allocates replacements in the right colors, copies page by page, flushes the caches, and frees the original pages.
- `mem_color_hypervisor()` colors **the hypervisor itself** by copying its own image and allocated structures into a newly colored region, jumping into it, and freeing the old one.
- Colors are assigned per VM (`.colors` in `struct vm_config`), per shared-memory object, and for the hypervisor (`config.hyp.colors`).

---

## 4. VM and vCPU model

`struct vm` / `struct vcpu` (`src/core/inc/vm.h`):

- VM holds: id, config pointer, master CPU id, sync token, vCPU array, physical CPU bitmap, address space, per-arch state, emulated-memory and emulated-register lists, an interrupt ownership bitmap, IPC objects, and Remote I/O devices.
- Single allocation per VM: `vmm_alloc_vm()` computes a page-aligned block holding `struct vm` plus the `struct vcpu` array with correct per-type alignment, then installs it into every participating core's address space (`vmm_get_vm_install_info` / `vmm_vm_install`).
- vCPU/pCPU translation helpers in both directions, plus mask translation (`vm_translate_to_pcpu_mask`, `vm_translate_to_vcpu_mask`).
- **CPU affinity**: the `cpu_affinity` bitmap gives preferred physical cores; unclaimed cores are then assigned to any VM still short of its `cpu_num`. Cores left over are powered down.

### VM image loading

Three modes, all in `src/core/vm.c`:

1. **Built-in** — `VM_IMAGE(name, path)` uses an `.incbin` in a dedicated `.vm_image_*` linker section; the image is linked into the hypervisor binary and copied into the VM's memory.
2. **Separately loaded** — `VM_IMAGE_LOADED(base, load, size)`, image placed by a bootloader.
3. **In place** — `.image.inplace`, the image is mapped where it already sits (with re-coloring if the VM has a color set), avoiding a copy.

Loading validates that the load region does not overlap the runtime region, and flushes the caches over the destination.

---

## 5. Interrupt virtualization

### 5.1 Generic layer (`src/core/interrupts.c`)

- A global bitmap of assigned interrupts plus a per-VM ownership bitmap, guarded by a reservation spinlock.
- `interrupts_reserve()` claims a physical line for the hypervisor and binds a handler; `interrupts_vm_assign()` assigns a line to a VM after an architecture-specific conflict check.
- `interrupts_handle()` decides per IRQ: **forward to the owning VM** (`vcpu_inject_hw_irq`) or **handle in the hypervisor**, and errors out on unknown IDs.
- Architecture port surface: `interrupts_arch_init/reserve/enable/check/clear/ipi_send/vm_assign/conflict/ipi_init`, with weak default implementations for platforms using a single IPI ID.

### 5.2 Arm — vGIC (`src/arch/armv8/vgic*.c`, ~1,850 lines)

- Version-agnostic core (`vgic.c`) plus **GICv2** (`vgicv2.c`) and **GICv3** (`vgicv3.c`) back ends, selected by `GIC_VERSION` at build time.
- Full MMIO emulation of the distributor register groups: `ISENABLER`, `ICENABLER`, `ISPENDR`, `ICPENDR`, `ISACTIVER`, `ICACTIVER`, `ICFGR`, `IPRIORITYR`, plus `ITARGETSR` (GICv2) and `IROUTER` (GICv3).
- **GICv3 redistributor** emulation (`GICR_*` registers, per-vCPU `TYPER` with correct affinity and Last bit), and trapping of the `ICC_SGI1R_EL1` system register for SGI generation.
- List-register management: `vgic_add_lr` / `vgic_remove_lr`, per-vCPU `curr_lrs[]` tracking, and interrupt **ownership transfer** between vCPUs (`vgic_get_ownership` / `vgic_yield_ownership`) coordinated by inter-core messages.
- Hardware-interrupt mapping (`vgic_set_hw`) so a physical IRQ deactivates correctly when the guest EOIs the virtual one.
- SGI delivery across cores via `vgic_send_sgi_msg`.

### 5.3 RISC-V — two interrupt controller families

- **PLIC / vPLIC** (`irqc/plic/`): emulates priorities, per-context enable bitmaps, pending/active state, and thresholds; two emulated MMIO windows (global and threshold/claim).
- **AIA** (`irqc/aia/`): **APLIC** (`aplic.c`) plus **IMSIC** (`imsic.c`) with their virtual counterparts. `vaplic` emulates `domaincfg`, `sourcecfg[]`, `ip`/`ie` bitmaps, `target[]`, `idelivery`, `iforce`, `ithreshold`, and `topi`/`claimi` per IDC. `vimsic` maps each vCPU's guest interrupt file to the physical IMSIC page of the core it is pinned to, using guest interrupt files (`num_guest_files`).
- IPIs via **ACLINT SSWI** (`aclint.c`) or via **SBI** `send_ipi`, chosen by the platform's `IPIC`.
- Interrupt injection uses `hvip` for virtual supervisor interrupts.

### 5.4 RH850 and TriCore

- **RH850**: `vintc.c` (virtual interrupt controller with per-VM reset), `vipir.c` (virtual inter-processor interrupt register), `vbootctrl.c` (virtual boot controller so guests can start their own secondary cores). Uses the RH850 hardware guest mode: GPID in `EIPSWH`/`FEPSWH`, SPID and `MPID6` fixed to the VM id for memory isolation.
- **TriCore**: `ir.c` / `vir.c` implement a virtual interrupt router, including GPSR service-request group assignment to VMs for guest-generated IPIs, with one group reserved for the hypervisor.

---

## 6. Inter-core communication (IPI messaging)

`src/core/cpu.c`, `src/core/inc/cpu.h`:

- A per-CPU circular queue of `struct cpu_msg {handler, event, data}`, sized by default to 253 entries so `struct cpuif` fits in one 4 KiB page (`IPI_MAX_EVENTS`, overridable).
- Handlers register themselves through the `CPU_MSG_HANDLER(fn, id)` macro, which places the function pointer into a dedicated linker section (`.ipi_cpumsg_handlers`) — a compile-time handler table.
- `cpu_sync_and_clear_msgs()` is a barrier that keeps draining messages while waiting, avoiding deadlock during initialization.
- Users of this mechanism: vGIC ownership/SGI, PSCI CPU_ON, SBI IPI/HSM, IPC notification, and Remote I/O.

---

## 7. Inter-VM communication

### 7.1 Shared memory (`src/core/shmem.c`)

- A global list of `struct shmem` objects declared in the configuration, each with size, colors, optional fixed physical placement, a lock, and a `cpu_masters` bitmap tracking which cores host a VM that maps it.
- Non-placed shared memory is allocated from the page pools at boot with the requested colors.

### 7.2 IPC objects + notification hypercall (`src/core/ipc.c`)

- A VM binds a shared-memory object into its address space through an `struct ipc` entry (base, size, `shmem_id`, and a list of notification interrupt IDs).
- `HC_IPC` hypercall: `ipc_hypercall(ipc_id, event_id)` sends an `IPC_NOTIFY` CPU message to every core hosting another VM that maps the same shared memory; the receiving core injects the configured virtual interrupt for that event into its vCPU.
- Bounds-checked against `ipc_num` and shared-memory validity, returning `HC_E_INVAL_ARGS`.

### 7.3 Remote I/O — para-virtual device offload (`src/core/remio.c`, ~640 lines)

A complete split-driver framework letting one VM (the **backend**) implement devices for another (the **frontend**), with the hypervisor brokering MMIO:

- Devices are declared per VM as `struct remio_dev` with a `bind_key` that pairs a frontend with a backend, a shared-memory region for the data path, an MMIO window, and an interrupt.
- Frontend MMIO accesses trap and land in `remio_mmio_emul_handler()`, which allocates a `struct remio_request` from an object pool and enqueues it as **pending**.
- Request lifecycle: `REMIO_STATE_FREE -> PENDING -> PROCESSING -> COMPLETE`.
- The backend drives it through the `HC_REMIO` hypercall with four operations: `REMIO_HYP_ASK` (fetch the next pending request), `REMIO_HYP_READ` / `REMIO_HYP_WRITE` (complete one), `REMIO_HYP_NOTIFY` (raise the frontend's interrupt, e.g. for buffer/config changes).
- Completion and notification are propagated with CPU messages (`REMIO_CPU_MSG_READ/WRITE/NOTIFY`) so the injecting core is the one that owns the target vCPU.
- Ready/handshake flags on both sides; `remio_assign_vm_cpus()` picks the lowest-id vCPU of each VM as the injection target.

---

## 8. Hypercall interface (`src/core/hypercall.c`)

A deliberately tiny ABI — two calls:

| ID | Name | Purpose |
| --- | --- | --- |
| 1 | `HC_IPC` | Notify peers sharing a shared-memory object |
| 2 | `HC_REMIO` | Remote I/O backend operations |

Return codes: `HC_E_SUCCESS`, `HC_E_FAILURE`, `HC_E_INVAL_ID`, `HC_E_INVAL_ARGS`. Arguments and returns are marshalled through architecture-defined registers (`HYPCALL_IN_ARG_REG` / `HYPCALL_OUT_ARG_REG` in each `arch/hypercall.h`). On Arm the entry point is an `HVC` in the vendor hypervisor SMCCC range.

---

## 9. Trap and emulation framework

- Generic `struct emul_access` describes a faulting access: address, direction, width, target register, register width, sign extension, and multi-register (64-bit `MRRC`-style) accesses.
- Two per-VM registries: **emulated memory ranges** (`vm_emul_add_mem` / `vm_emul_get_mem`) and **emulated registers** (`vm_emul_add_reg` / `vm_emul_get_reg`).
- **Arm** (`src/arch/armv8/aborts.c`): decodes `ESR_EL2` and dispatches on exception class — lower-EL data aborts (with ISS-based instruction decoding, translation/permission fault checks, and PC advance honoring the IL bit), `HVC32`/`HVC64`, `SMC32`/`SMC64`, and system-register traps (`ESR_EC_SYSRG`, `ESR_EC_RG_32`, `ESR_EC_RG_64`, including `ICC_SGI1R`). Correctly adjusts the return address for trapped SMCs.
- **RISC-V** (`src/arch/riscv/sync_exceptions.c`): guest page-fault handler with a load/store instruction decoder (`ins_ldst_decode`), compressed-instruction and pseudo-instruction handling, and a `sync_handler_table` for VS-mode ecalls.
- **TriCore** (`src/arch/tricore/traps.c` + `decode.c`): data-protection traps with instruction decoding, plus a bus-error trap handler that suppresses spurious errors when a guest touches the first register of a device region under slave-side MMIO protection.

---

## 10. Firmware / power-management interfaces

### PSCI (Arm, `src/arch/armv8/psci.c`)

Both a **client** (Bao calls EL3 to power on secondary cores and to standby/power down) and a **server** (guests' PSCI SMCs are trapped and emulated). Implemented functions:

- `PSCI_VERSION` (reports 1.1)
- `CPU_SUSPEND` (SMC32/SMC64)
- `CPU_OFF`
- `CPU_ON` (SMC32/SMC64) — target validated against the VM's vCPUs, entry point and context id delivered by CPU message, `ALREADY_ON` / `INVALID_PARAMS` handled
- `AFFINITY_INFO` (SMC32/SMC64)
- `PSCI_FEATURES`
- `MIGRATE_INFO_TYPE` (reports `TOS_NOT_PRESENT_MP`)

vCPU power state is tracked per vCPU (`psci_ctx.state = ON/OFF`) with its own lock.

### SBI (RISC-V, `src/arch/riscv/sbi.c`)

Bao is both an SBI client (to the M-mode firmware) and the SBI implementation seen by guests:

- **Base** — spec version, impl id/version, `probe_extension`, `mvendorid`/`marchid`/`mimpid`
- **TIME** — `set_timer`, using the **Sstc** extension (`vstimecmp`) when present, otherwise trapping to the firmware and juggling `hvip.VSTIP` / `sie.STIE`
- **IPI** — `send_ipi`, translated from virtual hart mask to physical harts
- **RFENCE** — `remote_fence_i`, `remote_sfence_vma`, `remote_sfence_vma_asid`
- **HSM** — `hart_start`, `hart_stop`, `hart_status`, `hart_suspend`, with a per-vCPU state machine (`STARTED / STOPPED / START_PENDING / STOP_PENDING`) and vCPU reset on start
- A **vendor `SBI_EXTID_BAO` extension** (`0x08000ba0`) for Bao-specific calls

### RISC-V H-extension configuration (`src/arch/riscv/vmm.c`)

Delegates VS-level interrupts (`hideleg`) and the guest-relevant exceptions (`hedeleg`: breakpoint, ecall-from-U, instruction/load/store page faults), then probes and enables — each with a WARL read-back sanity check — **Sstc**, **Svpbmt**, **Zicboz**, **Zicbom**, **Ssstateen** (per-bit control of `C`, `FCSR`, `CTX`/Sdtrig, `CSRIND`/Sscsrind, `IMSIC`+`AIA`, `ENVCFG`, `SEO`).

---

## 11. DMA protection (IOMMU)

Generic interface `io_init` / `io_vm_init` / `io_vm_add_device` (`src/core/inc/io.h`), with a real implementation on the MMU path and a no-op on the MPU path.

- **Arm SMMUv2** (`src/arch/armv8/armv8-a/smmuv2.c`, ~350 lines): feature/version checking, context bank allocation and programming (stage-2 translation pointed at the VM's root page table with the VM's VMID), stream-match register (SMR) allocation with mask/ID matching, stream-to-context (S2CR) programming, group support, and reuse detection for compatible stream-match entries. Per-VM and platform-wide stream-ID global masks are configurable.
- **RISC-V IOMMU** (`src/arch/riscv/iommu.c`): a 1-level extended-format Device Directory Table (64 entries), a 64-entry Fault Queue with interrupt-driven fault reporting (WSI mode), version check against `RV_IOMMU_SUPPORTED_VERSION`, and capability checks for `Sv39x4` second-stage translation and MSI flat mode.
- Devices are bound to VMs by the `deviceid_t id` (bus-master / stream id) field of `struct vm_dev_region`.

---

## 12. Platform and driver support

**24 platform ports** (`src/platform/`), each providing a `struct platform` description (CPU count, memory regions, MMIO regions, console base, architecture-specific IRQ controller/SMMU addresses) and a `platform.mk`:

| Arm (Armv8-A) | Arm (Armv8-R) | RISC-V | Other |
| --- | --- | --- | --- |
| qemu-aarch64-virt, fvp-a, fvp-a-aarch32, rpi4, zcu102, zcu104, ultra96, tx2, hikey960, imx8mn, imx8mp-verdin, imx8qm, s32g3, agilex5 | fvp-r, fvp-r-aarch32, mps3-an536, s32z270, e3650 | qemu-riscv64-virt, qemu-riscv32-virt, k3-com260 | rh850-u2a16 (RH850), tc4dx (TriCore) |

**11 UART drivers** (`src/platform/drivers/`): `pl011_uart`, `8250_uart`, `imx_uart`, `nxp_uart`, `zynq_uart`, `linflexd_uart`, `cmsdk_uart`, `sbi_uart` (RISC-V SBI console), `asclin_uart` (TriCore), `renesas_rlin3` (RH850), `e3650_uart`.

The console layer (`src/core/console.c`) maps the UART as a device page, serializes output with a spinlock, adds CR before LF, and provides a `printf`-style `console_printk()` with a 256-byte buffer.

---

## 13. Configuration format (`src/core/inc/config.h`)

A single `struct config` per build:

```c
struct config {
    struct { bool relocate; paddr_t base_addr;       // hypervisor placement (MPU platforms)
             bool data_relocate; paddr_t data_addr;  // non-unified memory platforms
             colormap_t colors; } hyp;
    size_t shmemlist_size; struct shmem* shmemlist;  // shared memory objects
    size_t vmlist_size;    struct vm_config* vmlist; // the VMs
};
```

Per VM (`struct vm_config`):

- `image` — `base_addr` (guest PA), `load_addr` (hyp PA), `size`, `separately_loaded`, `inplace`
- `entry` — guest entry point
- `cpu_affinity` — preferred physical CPU bitmap
- `colors` — cache color bitmap
- `platform` — the **virtual** platform:
  - `cpu_num`
  - `regions[]` — RAM regions, optionally at a fixed physical address (`place_phys` / `phys`) and with their own colors
  - `devs[]` — pass-through devices: physical address, guest virtual address, size, interrupt list, and IOMMU device id
  - `ipcs[]` — shared-memory bindings with notification interrupts
  - `remio_devs[]` — Remote I/O frontend/backend devices
  - `mmu` — on MPU platforms that also support an MMU, opt this VM into MMU-based isolation
  - `arch` — architecture specifics (Arm: vGIC `gicd_addr`/`gicc_addr`/redistributor, SMMU stream groups; RISC-V: PLIC/APLIC/IMSIC base addresses; TriCore: GPSR groups)

Two configurations ship in-tree: `configs/example/config.c` (a documented 2-VM Arm setup with shared memory, pass-through UARTs, and IPC interrupts) and `configs/null/config.c` (empty, used to smoke-test the build).

---

## 14. Build system (`Makefile`, `scripts/`)

- Invocation: `make PLATFORM=<plat> CONFIG=<cfg>`, producing `bin/<PLATFORM>/<CONFIG>/bao.elf`, `bao.bin`, a disassembly (`.asm`), and a readelf dump (`.txt`).
- **Both GCC and Clang/LLVM toolchains** are supported (`CROSS_COMPILE`, auto-detected, switching between `gcc/ld/objcopy` and `clang/ld.lld/llvm-objcopy` and adding `--target=`).
- Configuration repositories are pluggable via `CONFIG_REPO`, so configs can live outside the tree (including absolute paths, which are staged under `config/external/`).
- **Three code generators** compiled and run on the host during the build:
  - `scripts/config_defs_gen.c` -> `CONFIG_VM_NUM`, `CONFIG_VCPU_NUM`, `CONFIG_HYP_BASE_ADDR`, `CONFIG_HYP_DATA_ADDR`, `CONFIG_REMIO_DEV_NUM` — so static arrays are sized exactly for the configuration
  - `scripts/platform_defs_gen.c` (+ per-arch variants for RISC-V, RH850, TriCore) -> platform constants
  - `asm_defs.c` per architecture -> struct offsets for assembly
- Feature macros derived from the platform/arch makefiles: `MEM_PROT_MMU`, `MEM_PROT_MPU`, `MEM_NON_UNIFIED`, `PHYS_IRQS_ONLY`, `MMIO_SLAVE_SIDE_PROT`, `CC_IS_GCC`/`CC_IS_CLANG`, `GIC_VERSION`, `IRQC`, `IPIC`, `RV_XLEN`, with consistency checks (e.g. slave-side MMIO protection requires an MPU; AArch64 rejects non-unified memory).
- Strict warning set treated as errors: `-Wall -Werror -Wextra -Wconversion -Wsign-conversion -Wshadow -Wcast-qual -Wvla -Wswitch-default -Wmissing-prototypes -pedantic-errors`, and more.
- The version string is derived from `git describe` and linked in as a symbol.
- The linker script (`src/linker.ld`) is preprocessed per build; non-debug builds are stripped.

---

## 15. Support library (`src/lib/`)

Freestanding replacements — no libc dependency: `string.c` (`memcpy`, `memset`, `strcmp`, …), `printk.c` (`vsnprintf`-class formatter), `bitmap.c`, and headers for bit manipulation (`bit.h`), intrusive linked lists (`list.h`), a circular queue (`circular_queue.h`), and general utilities (`util.h`). Plus an object pool allocator in the core (`src/core/objpool.c`) with bitmap-based allocation and id-based lookup, used by Remote I/O.

---

## 16. Notable *absences* (by design)

These are not gaps in the port — they follow from the static-partitioning model:

- **No scheduler / no time-sharing.** More vCPUs than physical cores is not supported.
- **No dynamic VM lifecycle.** No VM creation, destruction, save/restore, or live migration at runtime; configuration is fixed at link time.
- **No in-hypervisor device models.** No virtio, block, or network back ends inside the hypervisor — that role is delegated to a backend VM through Remote I/O.
- **No nested virtualization.**
- **No GICv3 ITS / LPI (MSI) virtualization** on Arm; MSI support on RISC-V comes via IMSIC guest interrupt files instead.
- **No SMMUv3** — Arm DMA protection is SMMUv2 only.
- **No secure-world / TrustZone management** (`MIGRATE_INFO_TYPE` explicitly reports no trusted OS).
- **No device tree parsing at runtime** — the platform description is compiled in.
- **No dynamic memory allocator** beyond the page pools and object pools.

---

## 17. Source map

```
Makefile                    build system, feature macros, generators
configs/                    example + null configurations
scripts/                    host-side definition generators (config, platform, per-arch)
src/linker.ld               preprocessed linker script

src/core/                   architecture-independent hypervisor
  init.c vmm.c vm.c cpu.c   boot, VM assignment/creation, vCPU model, IPI messaging
  mem.c cache.c             page pools, cache coloring
  interrupts.c              interrupt ownership and routing
  ipc.c shmem.c remio.c     inter-VM communication and Remote I/O
  hypercall.c console.c     hypercall dispatch, UART console
  objpool.c config.c        object pools, configuration plumbing
  mmu/                      MMU-based memory protection back end
  mpu/                      MPU-based memory protection back end

src/arch/armv8/             Armv8 (aarch32/aarch64 sub-arch, armv8-a/armv8-r profiles)
  gic*.c vgic*.c            physical + virtual GICv2/GICv3
  aborts.c psci.c smc.c     trap dispatch, PSCI, SMCCC
  armv8-a/                  stage-2 MMU, page tables, SMMUv2, per-core init
  armv8-r/                  MPU, Cortex-R52 support, generic timer bring-up

src/arch/riscv/             RISC-V H-extension
  vmm.c vm.c sbi.c          hgatp/hedeleg/hideleg/henvcfg/hstateen setup, SBI
  sync_exceptions.c         guest page faults, instruction decoding
  irqc/plic/ irqc/aia/      PLIC/vPLIC, APLIC+IMSIC/vAPLIC+vIMSIC
  iommu.c aclint.c          RISC-V IOMMU, ACLINT SSWI IPIs

src/arch/rh850/             Renesas RH850 (hardware guest mode, MPU)
  vintc.c vipir.c vbootctrl.c   virtual INTC, IPIR, boot controller

src/arch/tricore/           Infineon TriCore TC4
  ir.c vir.c traps.c csa.c decode.c   interrupt router, traps, context save areas

src/platform/               24 board descriptions + 11 UART drivers
src/lib/                    freestanding string/printk/bitmap/list utilities
```
