Feature Comparison: Windows Hypervisor vs. RISC-V Linux Hypervisor
After reading through the entire MyHypervisorDriver and MyHypervisorApp codebases, here are the features present in the Windows/x86 hypervisor that are absent from the RISC-V hypervisor and could realistically be ported as architectural equivalents.

1. EPT Page Hooking (Hidden Hooks)
Windows implementation: Ept.c, HiddenHooks.c, SyscallHook.c

The Windows hypervisor can change EPT page permissions at runtime to intercept specific memory accesses. It uses execute-only pages to redirect code execution through a "fake page" while keeping data reads going to the original page. This enables transparent inline hooking of kernel functions (e.g., NtCreateFile).

RISC-V equivalent: Dynamically modify guest page table entries (hgatp-based) at runtime to revoke/grant R/W/X permissions on specific guest physical pages. When the guest faults, the trap handler can emulate or redirect the access. The RISC-V H-extension's guest-page faults (scause 20/21/23) already trap into the hypervisor — you just need infrastructure to selectively trigger them.

Difficulty: Medium

2. Dynamic Page Splitting
Windows implementation: EptSplitLargePage() in Ept.c, VMM_EPT_DYNAMIC_SPLIT struct

The Windows hypervisor uses 2MB large pages by default for performance, then dynamically splits them into 4KB pages only when fine-grained hooking is needed on a specific page.

RISC-V equivalent: The current RISC-V hypervisor always maps at 4KB granularity. You could add superpage (2MB/1GB megapage) support in guest_page_table.rs and split on-demand when a hook targets a specific 4KB region within a superpage.

Difficulty: Medium

3. Structured VMCALL/Hypercall Interface
Windows implementation: Vmcall.c, Vmcall.h

A dispatch table of numbered hypercalls (VMCALL_TEST, VMCALL_CHANGE_PAGE_ATTRIB, VMCALL_UNHOOK_ALL_PAGES, etc.) allowing the host or a privileged guest to request hypervisor services dynamically.

RISC-V equivalent: Currently only SBI calls are handled. You could define custom SBI extension IDs (vendor-specific range 0x09000000–0x09FFFFFF) for hypervisor-specific operations like page permission changes, debug commands, or performance queries.

Difficulty: Low

4. Interrupt/Event Injection
Windows implementation: Events.c, Events.h — EventInjectBreakpoint(), EventInjectGeneralProtection(), EventInjectUndefinedOpcode()

The hypervisor can inject arbitrary exceptions/interrupts into the guest via the VMCS VM-entry interrupt info field.

RISC-V equivalent: The H-extension provides the hvip (Hypervisor Virtual Interrupt Pending) CSR to inject virtual interrupts into VS-mode. You could implement timer interrupt injection (currently set_timer is a stub), software interrupts, and external interrupt injection through hvip and vsip/vsie.

Difficulty: Medium

5. MSR Bitmap → CSR Access Trapping
Windows implementation: HvSetMsrBitmap() in HypervisorRoutines.c, MSR bitmap VMCS field

Selectively traps reads/writes to specific MSRs without trapping all of them.

RISC-V equivalent: The hedeleg/hideleg registers already delegate some exceptions. The virtual instruction trap (scause=22) fires when the guest accesses certain CSRs. You could extend the trap handler to intercept and virtualize specific CSR accesses (e.g., stimecmp, satp, sstatus) for monitoring or policy enforcement.

Difficulty: Low-Medium

6. Multi-Core (Multi-Hart) Support
Windows implementation: GuestState array indexed by processor ID, BroadcastToProcessors(), per-core VMCS/VMXON regions

Each logical core has its own virtualization state and can be independently virtualized.

RISC-V equivalent: Currently single-hart only. You'd need a per-hart VCpu array, IPI (inter-processor interrupt) handling, and per-hart trap vectors. The SBI HSM (Hart State Management) extension calls from the guest would need handling.

Difficulty: High

7. Structured Logging with Ring Buffers
Windows implementation: Logging.c, Logging.h — per-core ring buffers, log levels (Info/Warning/Error), immediate vs. batched messages, kernel→usermode delivery via IRP

RISC-V equivalent: Currently just println! via SBI putchar. You could add:

Log levels with filtering
Per-hart ring buffers in shared memory
Batched output (currently the guest console buffer does this minimally)
Timestamps from rdtime
Difficulty: Low

8. Monitor Trap Flag (Single-Step Execution)
Windows implementation: HvSetMonitorTrapFlag(), EptHandleMonitorTrapFlag() — single-steps the guest one instruction, used to restore hook pages after execution

RISC-V equivalent: Use the RISC-V debug trigger module (tdata1/tdata2 CSRs with mcontrol type) to set instruction-count triggers. Alternatively, temporarily set the guest page as non-executable after handling a hook, then re-enable after one instruction using a second fault.

Difficulty: Medium-High

9. Timer Virtualization
Windows implementation: VMX preemption timer (GUEST_PREEMPTION_TIMER VMCS field)

RISC-V equivalent: The set_timer SBI call is currently a no-op stub. You could properly virtualize the timer by:

Programming the real stimecmp to fire at the guest's requested time
Injecting a timer interrupt into the guest via hvip when it fires
This would let Linux's scheduler actually work instead of hanging
Difficulty: Medium

10. Graceful Hypervisor Teardown (VMXOFF)
Windows implementation: VmxVmxoff(), VMX_VMXOFF_STATE, HvTerminateVmx() — cleanly exits VMX mode, restores host state, allows unloading the driver

RISC-V equivalent: Currently vcpu.run() is -> ! (never returns). You could add a mechanism for the guest to trigger shutdown (e.g., SBI SRST extension or a custom hypercall), gracefully restore the hypervisor stack, deallocate resources, and return to OpenSBI/M-mode.

Difficulty: Low-Medium

11. User-Mode Control Application (IOCTL Interface)
Windows implementation: MyHypervisorApp.cpp, Driver.c — user-space app communicates with hypervisor via IOCTL, can start/stop VMX, read logs, trigger hooks

RISC-V equivalent: Could add a virtio-console or shared-memory region where a privileged process (or a second hart running a control program) can send commands to the hypervisor — e.g., "hook address X", "dump guest state", "inject interrupt".

Difficulty: Medium-High

12. CPUID Virtualization → SBI/Device-Tree Spoofing
Windows implementation: HvHandleCpuid() — intercepts CPUID and can return custom values (e.g., hide hypervisor presence or advertise features)

RISC-V equivalent: The SBI probe_extension and get_mvendorid/marchid/mimpid calls already return values. You could virtualize these more sophisticatedly — e.g., present different ISA strings, hide extensions, or emulate extensions not present in hardware.

Difficulty: Low

13. Pool/Slab Allocator for VMX-Root Operations
Windows implementation: PoolManager.c — pre-allocates buffers since you can't call the kernel allocator from VMX root mode (high IRQL)

RISC-V equivalent: The current bump allocator never frees. For longer-running hypervisors, you'd want a proper slab or free-list allocator that can reclaim memory from released page tables, removed hooks, etc.

Difficulty: Low-Medium

Summary: Prioritized Upgrade Roadmap
Priority	Feature	Effort	Impact
1	Timer virtualization	Medium	Linux boots further, scheduler works
2	Interrupt injection (hvip)	Medium	Enables real device interrupts to guest
3	Custom hypercall interface	Low	Foundation for all other features
4	Structured logging	Low	Better debugging experience
5	Graceful shutdown	Low-Medium	Clean exit without hard reset
6	Dynamic page permission hooks	Medium	Security monitoring, introspection
7	CSR access trapping	Low-Medium	Policy enforcement, transparency
8	Multi-hart support	High	Run SMP Linux
9	Pool allocator upgrade	Low-Medium	Long-running stability
10	Superpage + dynamic splitting	Medium	Performance + fine-grained hooks
Items 1–5 would be the most impactful "quick wins" that bring the RISC-V hypervisor closer to the Windows one's capability level while staying within the project's educational spirit.