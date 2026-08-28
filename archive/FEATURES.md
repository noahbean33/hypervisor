# Archive Feature Reference (excluding `bao/`)

This document details what is implemented in each project under `archive/`, **other than the Bao hypervisor** (which is documented separately in [`bao/FEATURES.md`](bao/FEATURES.md)).

The archive root is a Visual Studio solution — **"Hypervisor From Scratch"** — a Windows kernel-mode Intel VT-x hypervisor, accompanied by three smaller sibling projects:

| Project | What it is | Language | Size |
| --- | --- | --- | --- |
| **Hypervisor From Scratch** (`MyHypervisorDriver` + `MyHypervisorApp`) | Type-2-style Windows kernel hypervisor that virtualizes the running OS live using Intel VT-x / EPT | C (driver) + C++ (app) + MASM64 | ~7,700 LOC driver |
| **VMCS-Auditor** | Standalone tool that validates a VMCS layout against Intel/Bochs consistency checks before `VMLAUNCH` | C++ | ~4,200 LOC |
| **rust/** | Minimal RISC-V (H-extension) type-1 hypervisor that boots a real Linux kernel | Rust (`no_std`) | ~840 LOC |
| **x64-Driver-Inline-Assembly** | WDK sample showing how to call MASM64 assembly from a C x64 kernel driver | C + MASM64 | small |

License: **MIT** (root `LICENSE`).

---

# Part 1 — Hypervisor From Scratch (the main project)

An educational but fully working Windows kernel-mode hypervisor built on **Intel VT-x (VMX)**. Unlike a classic "boot a fresh guest" hypervisor, it does **live virtualization**: it takes the already-running Windows system and demotes it into VMX non-root (guest) mode on every logical processor, then sits underneath it in VMX root mode. This is the "blue-pill" architecture.

Two build outputs:
- **`MyHypervisorDriver.sys`** — the hypervisor itself (kernel driver)
- **`MyHypervisorApp.exe`** — a user-mode control/console app

## 1.1 Driver architecture (`MyHypervisorDriver/`)

Standard WDM/WDF driver skeleton in `Driver.c`:

- `DriverEntry` creates a device (`\Device\MyHypervisorDevice`) and symbolic link, wires up the IRP major functions (`CREATE`, `CLOSE`, `READ`, `WRITE`, `DEVICE_CONTROL`), enables NX pool opt-in, and allocates a per-processor `VIRTUAL_MACHINE_STATE` array sized to `KeQueryActiveProcessorCount()`.
- `IRP_MJ_CREATE` triggers `HvVmxInitialize()` (start virtualizing); `IRP_MJ_CLOSE` triggers `HvTerminateVmx()` (tear down).
- **IOCTL interface** (`DrvDispatchIoControl`):
  - `IOCTL_REGISTER_EVENT` — register an IRP-based or event-based notification channel for kernel→user log delivery
  - `IOCTL_RETURN_IRP_PENDING_PACKETS_AND_DISALLOW_IOCTL` — flush pending log IRPs and stop accepting new ones (used during shutdown)

## 1.2 VMX bring-up and teardown

**Initialization** (`HypervisorRoutines.c`, `Vmx.c`, `VmxRegions.c`):

- `HvIsVmxSupported()` — checks CPUID feature bit and `IA32_FEATURE_CONTROL` MSR lock/enable bits.
- Virtualization is driven across **all logical processors** via DPC broadcast (`KeGenericCallDpc` / `HvDpcBroadcastInitializeGuest`), so every core is put into VMX operation.
- Per-core VMX structures are allocated:
  - **VMXON region** (`VmxAllocateVmxonRegion`) with the revision ID from `IA32_VMX_BASIC`
  - **VMCS region** (`VmxAllocateVmcsRegion`)
  - **VMM stack** (`VmxAllocateVmmStack`)
  - **MSR bitmap** (`VmxAllocateMsrBitmap`)
- `AsmEnableVmxOperation` sets CR4.VMXE (bit 13); VMXON is then executed.
- `VmxVirtualizeCurrentSystem` captures the current CPU context and launches the guest from the point of the call, so the running OS continues seamlessly in non-root mode.

**Teardown** (`HvTerminateVmx` / `VMCALL_VMXOFF`): broadcast DPC executes `VMXOFF` on each core, restores state, and frees all regions.

## 1.3 VMCS configuration (`VmxSetupVmcs`)

A complete guest/host state setup, including:

- **Host state**: segment selectors (masked to RPL 0), CR0/CR3/CR4, GDTR/IDTR/TR bases, FS/GS bases, SYSENTER MSRs, `HOST_RIP = AsmVmexitHandler`, `HOST_RSP = VMM stack`. `HOST_CR3` is set to the System process directory table base (`FindSystemDirectoryTableBase`) so the VM-exit handler runs in a valid, stable address space regardless of the interrupted process.
- **Guest state**: mirrors the current live CPU (segment descriptors filled from the real GDT via `HvFillGuestSelectorData`/`HvGetSegmentDescriptor`), `GUEST_RIP = AsmVmxRestoreState`, `GUEST_RSP` = captured stack, DEBUGCTL, RFLAGS, SYSENTER MSRs.
- **Execution controls**, adjusted through `HvAdjustControls` against the capability MSRs (honoring the `IA32_VMX_BASIC` "true controls" hint):
  - Primary proc-based: **activate MSR bitmap**, **activate secondary controls**
  - Secondary: **EPT**, **VPID**, **RDTSCP**, **INVPCID**, **XSAVES/XRSTORS**, **user wait/pause**
  - VM-entry / VM-exit controls set IA-32e (64-bit) mode
- `MSR_BITMAP`, `EPT_POINTER`, and `VIRTUAL_PROCESSOR_ID` (VPID tag = 1) wired in.
- `VMCS_LINK_POINTER = ~0`.

## 1.4 VM-exit handling (`Exit.c`, `AsmVmexitHandler.asm`)

`AsmVmexitHandler` saves the full guest GPR context (`GUEST_REGS`) and calls `VmxVmexitHandler`, which dispatches on the exit reason:

| Exit reason | Handling |
| --- | --- |
| `TRIPLE_FAULT` | logged as error |
| All VMX instructions (`VMCLEAR/VMPTRLD/VMPTRST/VMREAD/VMRESUME/VMWRITE/VMXON/VMXOFF/VMLAUNCH`) | inject failure by setting guest RFLAGS.CF (report "VMX not available" to the guest) |
| `CR_ACCESS` | `HvHandleControlRegisterAccess` — emulates MOV to/from CR0/CR3/CR4 |
| `MSR_READ` | `HvHandleMsrRead` — emulates RDMSR, with valid-range and reserved-range guards |
| `MSR_WRITE` | `HvHandleMsrWrite` — emulates WRMSR |
| `CPUID` | `HvHandleCpuid` — passes through, but can hide/annotate hypervisor presence |
| `IO_INSTRUCTION` | stubbed (not implemented) |
| `EPT_VIOLATION` | `EptHandleEptViolation` — core of the hidden-hook mechanism |
| `EPT_MISCONFIG` | `EptHandleMisconfiguration` (fatal diagnostic) |
| `VMCALL` | dispatched (see below) |
| `EXCEPTION_NMI` | handles guest exceptions via the exception bitmap; re-injects `#BP` and logs breakpoints with the faulting process id + RIP |
| `MONITOR_TRAP_FLAG` | single-step restore point for EPT hooks |
| `HLT` | handled |

After handling, `HvResumeToNextInstruction` advances guest RIP by the VM-exit instruction length (unless a handler cleared `IncrementRip`), then `VMRESUME`.

## 1.5 Extended Page Tables (EPT) — `Ept.c` / `Ept.h` (~2,000 LOC combined)

The most substantial subsystem:

- **Feature detection** (`EptCheckFeatures`): validates page-walk-length-4, write-back memory type, and 2 MB page support from the EPT/VPID capability MSR.
- **MTRR-aware identity map** (`EptBuildMtrrMap`, `EptAllocateAndCreateIdentityPageTable`): builds a full identity-mapped guest-physical → host-physical EPT using 2 MB large pages (PML4→PDPT→PD), honoring the platform's MTRR memory-type ranges so device/uncacheable regions get the correct EPT memory type.
- **Dynamic large-page splitting** (`EptSplitLargePage`): converts a 2 MB PDE into 512× 4 KB PTEs on demand (pre-allocated buffers from the pool manager), so a single page can be hooked at 4 KB granularity.
- **EPT pointer / logical-processor init** (`EptLogicalProcessorInitialize`).

### Hidden hooks (EPT-based, invisible to the guest)

`EptPageHook` / `EptPerformPageHook` implement stealth hooking using split EPT permissions:

- The target page is split to 4 KB, and a **shadow "fake" copy** of the page is created (`FakePageContents`).
- Permissions are separated: the real page is left executable-but-not-readable (or read/write-trapped), and the fake page holds the patched code.
- **Execute hooks**: an inline trampoline is written using the **LDE64 length-disassembler engine** (`Libraries/LDE64x64.lib`, declared as `LDE()`), which measures how many whole instructions must be overwritten so the original bytes can be relocated into an executable trampoline (`EptHookInstructionMemory`, `EptHookWriteAbsoluteJump`). This gives a detour hook where `OrigFunction` still runs the displaced instructions.
- **Read/write hooks**: the page's EPT read/write bits are cleared so any access faults out to the hypervisor.
- On an EPT violation to a hooked page (`EptHandleHookedPage` / `EptHandlePageHookExit`), the hypervisor flips permissions, sets the **Monitor Trap Flag** to single-step one instruction, then restores the protected state — so the guest transparently sees the real page for exactly one instruction and never notices the hook.
- Unhooking: `EptPageUnHookSinglePage` / `EptPageUnHookAllPages`, coordinated across cores by DPC broadcast and EPT invalidation.

The `HiddenHooks.c` file demonstrates hooking `ExAllocatePoolWithTag`.

## 1.6 VMCALL interface (`Vmcall.c`)

Guest→hypervisor calls, authenticated by magic values in R10/R11/R12 (`HVFS` / `VMCALL` / `NOHYPERV`, set by `AsmVmxVmcall`) so the handler can distinguish its own hypercalls from Hyper-V's (unrecognized ones are forwarded via `AsmHypervVmcall`):

| VMCALL number | Action |
| --- | --- |
| `VMCALL_TEST` (0x1) | test/echo call |
| `VMCALL_VMXOFF` (0x2) | turn off VMX on the current core |
| `VMCALL_CHANGE_PAGE_ATTRIB` (0x3) | install an EPT hook (attribute mask in the upper 32 bits selects read/write/exec) |
| `VMCALL_INVEPT_ALL_CONTEXTS` (0x4) | invalidate all EPT contexts |
| `VMCALL_INVEPT_SINGLE_CONTEXT` (0x5) | invalidate one EPT context |
| `VMCALL_UNHOOK_ALL_PAGES` (0x6) | remove all page hooks |
| `VMCALL_UNHOOK_SINGLE_PAGE` (0x7) | remove one page hook |

## 1.7 TLB / cache management

- **INVEPT** (`Invept.c` + `AsmEpt.asm`): single-context and all-context EPT invalidation.
- **INVVPID** (`Vpid.c`): individual-address, single-context, all-context, and single-context-retaining-globals variants. VPID = 1 keeps EPT translations separate from the OS's own TLB entries.

## 1.8 Event injection (`Events.c`)

VM-entry interruption injection helper (`EventInjectInterruption`) plus wrappers to inject `#BP` (breakpoint, with instruction-length fix-up), `#GP` (general protection, with error code), and `#UD` (invalid opcode) into the guest.

## 1.9 Syscall hooking (`SyscallHook.c`)

A separate, higher-level hooking technique:

- `SyscallHookGetKernelBase` — locates the ntoskrnl base from the system module list.
- `SyscallHookFindSsdt` — resolves the **SSDT** (`KeServiceDescriptorTable`) and Win32k shadow table by pattern-scanning the kernel.
- `SyscallHookGetFunctionAddress` — resolves a syscall by its service (API) number, decoding the SSDT offset-encoded entries.
- `NtCreateFileHook` — an example hook of `NtCreateFile`.

## 1.10 Support infrastructure

- **Pool manager** (`PoolManager.c/.h`): a pre-allocation system that reserves non-paged pool at PASSIVE_LEVEL for use inside VMX root mode (where you cannot safely allocate). Tracks allocations by intent (`TRACKING_HOOKED_PAGES`, `EXEC_TRAMPOLINE`, `SPLIT_2MB_PAGING_TO_4KB_PAGE`) and refills on demand via `PoolManagerCheckAndPerformAllocation`.
- **Logging** (`Logging.c/.h`): a kernel→user message-tracking system with IRP-based and event-based delivery, immediate vs. buffered modes, info/warning/error levels, and optional WPP tracing — all configurable through `Shared Headers/Configuration.h`.
- **Spinlocks** (`Spinlock.c`): custom test-and-set spinlocks for VMX-root synchronization.
- **Assembly modules** (MASM64): `AsmVmxOperation.asm` (VMXON enable, VMCALL), `AsmVmexitHandler.asm` (exit stub + context save), `AsmVmxContextState.asm` (save/restore state), `AsmSegmentRegs.asm` (read segment registers/GDT/IDT), `AsmCommon.asm`, `AsmEpt.asm` (INVEPT/INVVPID).
- **MSR/segment definitions** (`Msr.h`, `Common.h`, `Vmx.h`) — VMCS field encodings, exit-reason constants, control-bit masks, segment/descriptor structures.

## 1.11 User-mode app (`MyHypervisorApp/MyHypervisorApp.cpp`)

- Verifies the CPU is `GenuineIntel` and that CPUID reports VMX support (inline-asm CPUID probe).
- Opens `\\.\MyHypervisorDevice`, which loads the hypervisor.
- Spawns a worker thread that continuously pulls kernel log buffers via `IOCTL_REGISTER_EVENT` and prints info/warning/error messages.
- On keypress, sends the shutdown IOCTL and closes the handle (turning VMX off).

---

# Part 2 — VMCS-Auditor (`VMCS-Auditor/`)

A standalone Windows console tool (`VmcsAuditor.exe`, prebuilt in `x64/Release/`) that **validates a VMCS before `VMLAUNCH`/`VMRESUME`**. It is forked from the **Bochs** emulator's VMX consistency-check implementation, so it reproduces almost the entire Intel SDM VM-entry checklist without needing the actual CPU to reject the launch (which gives only a coarse error code).

Workflow (`VmcsAuditor.cpp`): the user pastes in the capability MSRs (`IA32_VMX_PINBASED_CTLS` 0x481, `PROCBASED_CTLS` 0x482, `PROCBASED_CTLS2` 0x48B, EXIT/ENTRY CTLS, EPT/VPID cap, CR0/CR4 fixed-bit masks, EFER mask) and then the VMCS field values; the auditor reports which specific check fails.

The check engine (`Auditor.cpp`, ~2,150 LOC) implements the three SDM check phases as three functions:

- **`VMenterLoadCheckVmControls`** — validates VM-execution, VM-exit, and VM-entry control fields: reserved-bit conformance against the capability MSRs, MSR-bitmap/IO-bitmap/TPR-shadow/APIC-access/EPTP/VPID validity, CR3-target count, exception bitmap / page-fault error-code mask-match, VMCS-link-pointer, VM-entry interruption-information consistency, and MSR-load/store area addresses.
- **`VMenterLoadCheckHostState`** — validates host CR0/CR3/CR4 against fixed-bit masks, host segment selectors (RPL/TI rules), canonical-address checks on host FS/GS/GDTR/IDTR/TR/SYSENTER bases and RIP, and IA-32e-mode consistency.
- **`VMenterLoadCheckGuestState`** — the largest: guest CR0/CR3/CR4/DR7, IA32_DEBUGCTL / PAT / EFER, segment registers and their access-rights bytes (type, S, DPL, present, granularity, usable/unusable), GDTR/IDTR limits, RIP/RFLAGS canonical and reserved-bit checks, activity state and interruptibility-state consistency, pending debug exceptions, and VMCS-link-pointer.

Supporting infrastructure ported from Bochs: `parse_selector`, segment-descriptor decoders, the 32-entry `exceptions_info` table (fault/trap/abort class and error-code delivery), `IsValidPageAlignedPhyAddr`, and the capability-bit initialization (`init_vmx_extensions_bitmask`). `Auditor.h` (~1,860 LOC) carries the VMCS-cache structure, all VMX field/flag constants, and error-code enums.

**In short:** it is not a hypervisor — it is a companion debugging/validation tool for one, reproducing Intel's VM-entry consistency checks in software.

---

# Part 3 — Rust RISC-V hypervisor (`rust/`)

A compact **type-1 hypervisor for RISC-V** using the hardware **H (hypervisor) extension**, written in `no_std`/`no_main` Rust (~840 LOC). Unlike the Windows project, this one boots a **real, unmodified Linux kernel** as a guest. It targets a QEMU-`virt`-style machine.

- **Boot** (`main.rs`): a `.text.boot` entry sets up the stack, zeroes BSS, installs the trap vector (`stvec`), initializes a bump heap allocator, then loads and runs the guest.
- **Guest loading** (`linux_loader.rs`): parses the RISC-V **`Image` header** (validates the `RSCV` magic), copies the kernel into guest memory, and **generates a Flattened Device Tree at runtime** using the `vm-fdt` crate — describing a `riscv-virtio` machine with `bootargs`, memory, CPU, PLIC, and console nodes. The kernel `Image` is embedded via `include_bytes!`.
- **Two-stage guest page tables** (`guest_page_table.rs`): builds an **Sv48x4** second-stage translation table (guest-physical → host-physical), 4-level, with R/W/X/U/V PTE flags, and produces the `hgatp` value.
- **vCPU** (`vcpu.rs`): a `VCpu` struct holding all 31 GPRs plus host SP and the control CSRs. On construction it programs:
  - `hstatus` — VSXL = 64-bit, SPV (return into virtualized supervisor mode)
  - `hedeleg` — delegates the standard exception set (misaligned/access/page faults, illegal instruction, breakpoint, U-mode ecall) directly to the guest
  - `sstatus.SPP` = supervisor
  - `hgatp`, `sepc` = guest entry
  - `run()` writes the CSRs, restores all guest GPRs from the struct via hand-written inline assembly, and executes `sret` to enter the guest.
- **Trap handling** (`trap.rs`): a naked-assembly trap entry saves the full guest GPR set into the `VCpu` (swapping via `sscratch`), switches to the host stack, and calls the Rust `handle_trap`, which decodes `scause` (full table of exceptions and interrupts) and services:
  - **VS-mode ecalls → SBI emulation** (`handle_sbi_call`): implements a minimal **SBI**: Base extension (spec version, probe, vendor/arch/impl IDs), `set_timer` (stubbed), and **console putchar** (buffers guest output and prints it line-by-line as `[guest] …`). Console getchar and unknown calls return unsupported.
  - **Guest-page faults (load/store) → MMIO emulation** (`handle_mmio_read` / `handle_mmio_write`): reconstructs the faulting guest-physical address from `htval`+`stval`, decodes the trapped instruction from `htinst` (load/store opcode and width, compressed vs. 32-bit), reads/writes the correct guest register, and routes accesses in the **PLIC** address window to a (stubbed) PLIC handler. Advances `sepc` by the correct instruction length.
- **Allocator / print** (`allocator.rs`, `print.rs`): a simple linked/bump page allocator behind Rust's `GlobalAlloc`, and a `println!` macro over the SBI/console.

**Feature summary:** hardware-assisted two-stage translation, guest trap/emulation loop, SBI console + timer stubs, PLIC MMIO interception, and runtime device-tree generation — enough to bring a Linux guest to a console.

---

# Part 4 — x64 Driver Inline Assembly (`x64-Driver-Inline-Assembly/`)

Not a hypervisor — a **WDK technique sample** that exists because MSVC does **not** support `__asm` inline assembly in 64-bit code. It demonstrates the supported alternative: separate `.asm` files assembled by MASM64 and called from C via `extern` declarations.

- `Driver.c` — a KMDF driver skeleton (`DriverEntry`, `Unload`) that calls into assembly (`MainAsm()`).
- `Driver.asm` / `Source.asm` — example `PROC`s showing public-symbol export, arithmetic, register preservation (`push`/`pop`), and `int 3` breakpoints.
- Includes the `.inf` install file and VS project configured to treat `.asm` as MASM items.

It is included as a companion reference for the assembly-integration technique the main hypervisor relies on (VMXON, VMREAD/VMWRITE, INVEPT, etc. all require this pattern).

---

# Cross-project relationships

- **Hypervisor From Scratch** is the centerpiece: a live Intel VT-x hypervisor with EPT, stealth hooks, VMCALL, VPID, event injection, and SSDT syscall hooking on Windows x64.
- **VMCS-Auditor** is its debugging companion — validate the VMCS in software instead of getting an opaque `VMLAUNCH` failure.
- **x64-Driver-Inline-Assembly** documents the C↔MASM64 mechanism every VMX driver needs.
- **rust/** is an independent, contrasting design point: a different ISA (RISC-V, not x86), a different model (type-1 booting a fresh Linux guest, not live-virtualizing the host), and a memory-safe language — but the same fundamentals (second-stage page tables, trap-and-emulate, firmware-call emulation).

## Directory map

```
archive/
  Hypervisor From Scratch.sln     VS solution (driver + app)
  LICENSE                         MIT
  README.md                       project overview
  .clang-format

  MyHypervisorDriver/             the VT-x hypervisor (kernel driver)
    Driver.c                      WDM entry, IOCTL dispatch
    Vmx.c VmxRegions.c            VMXON/VMCS setup, launch/resume/off
    HypervisorRoutines.c          init, CPUID/MSR/CR handlers, DPC broadcast
    Exit.c AsmVmexitHandler.asm   VM-exit dispatch + context save
    Ept.c Ept.h                   EPT identity map, large-page split, hidden hooks
    Vmcall.c                      guest hypercall interface
    Invept.c Vpid.c               EPT/VPID TLB invalidation
    Events.c                      exception/interrupt injection
    SyscallHook.c                 SSDT resolution + NtCreateFile hook
    HiddenHooks.c                 EPT hook demo (ExAllocatePoolWithTag)
    PoolManager.c                 pre-allocated non-paged pool for VMX-root
    Logging.c                     kernel->user log transport
    Spinlock.c                    VMX-root spinlocks
    Asm*.asm                      MASM64 low-level VMX/segment routines

  MyHypervisorApp/                user-mode loader/console
  Shared Headers/                 Configuration.h, Definitions.h (IOCTLs)
  Libraries/LDE64x64.lib          length-disassembler engine (for hooks)
  Examples/                       gifs/pngs of hooks & syscall interception

  VMCS-Auditor/                   software VMCS consistency checker (Bochs-derived)
    VmcsAuditor/Auditor.cpp/.h    the three VM-entry check phases
    x64/Release/VmcsAuditor.exe   prebuilt binary

  rust/                           RISC-V H-extension type-1 hypervisor (no_std)
    main.rs                       boot, load Linux, run vCPU
    vcpu.rs                       vCPU state + run loop (hstatus/hgatp/hedeleg)
    trap.rs                       trap handler, SBI + MMIO/PLIC emulation
    guest_page_table.rs           Sv48x4 second-stage page tables
    linux_loader.rs               Image parsing + runtime device-tree build
    allocator.rs print.rs guest_memory.rs

  x64-Driver-Inline-Assembly/     WDK sample: calling MASM64 from C x64 drivers
```
