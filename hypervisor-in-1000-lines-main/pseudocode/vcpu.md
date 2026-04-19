# VCpu - Pseudocode

Represents a Virtual CPU - holds the full guest register state and hypervisor
configuration needed to enter/exit guest mode via the RISC-V H-extension.

## Data Structures

```c
// Virtual CPU state - all fields are saved/restored on VM exit/entry
struct VCpu {
    // Hypervisor internal state
    uint64_t host_sp;    // Host stack pointer (for trap handler)
    uint64_t hstatus;    // Hypervisor status register
    uint64_t hgatp;      // Guest address translation (page table pointer)
    uint64_t hedeleg;    // Hypervisor exception delegation register
    uint64_t sstatus;    // Supervisor status register
    uint64_t sepc;       // Supervisor exception program counter (guest PC)

    // Guest general-purpose registers (x1-x31)
    uint64_t ra;   // x1  - return address
    uint64_t sp;   // x2  - stack pointer
    uint64_t gp;   // x3  - global pointer
    uint64_t tp;   // x4  - thread pointer
    uint64_t t0;   // x5  - temporary
    uint64_t t1;   // x6  - temporary
    uint64_t t2;   // x7  - temporary
    uint64_t s0;   // x8  - saved register / frame pointer
    uint64_t s1;   // x9  - saved register
    uint64_t a0;   // x10 - argument / return value
    uint64_t a1;   // x11 - argument / return value
    uint64_t a2;   // x12 - argument
    uint64_t a3;   // x13 - argument
    uint64_t a4;   // x14 - argument
    uint64_t a5;   // x15 - argument
    uint64_t a6;   // x16 - argument (SBI function ID)
    uint64_t a7;   // x17 - argument (SBI extension ID)
    uint64_t s2;   // x18 - saved register
    uint64_t s3;   // x19 - saved register
    uint64_t s4;   // x20 - saved register
    uint64_t s5;   // x21 - saved register
    uint64_t s6;   // x22 - saved register
    uint64_t s7;   // x23 - saved register
    uint64_t s8;   // x24 - saved register
    uint64_t s9;   // x25 - saved register
    uint64_t s10;  // x26 - saved register
    uint64_t s11;  // x27 - saved register
    uint64_t t3;   // x28 - temporary
    uint64_t t4;   // x29 - temporary
    uint64_t t5;   // x30 - temporary
    uint64_t t6;   // x31 - temporary
};
```

## Functions

### vcpu_new

```c
// Create and initialize a new VCpu for running a guest
VCpu vcpu_new(GuestPageTable *table, uint64_t guest_entry) {
    VCpu vcpu = {0};  // Zero-initialize all registers

    // Configure hstatus:
    //   VSXL = 2 (64-bit XLEN for VS-mode)
    //   SPV  = 1 (return to virtualized mode on sret)
    vcpu.hstatus = 0;
    vcpu.hstatus |= (2ULL << 32);  // VSXL = 64-bit
    vcpu.hstatus |= (1ULL << 7);   // SPV = 1 (guest mode)

    // Configure hedeleg: delegate certain exceptions to the guest
    // so the guest OS can handle them without hypervisor intervention
    vcpu.hedeleg = 0;
    vcpu.hedeleg |= (1 << 0);   // Instruction address misaligned
    vcpu.hedeleg |= (1 << 1);   // Instruction access fault
    vcpu.hedeleg |= (1 << 2);   // Illegal instruction
    vcpu.hedeleg |= (1 << 3);   // Breakpoint
    vcpu.hedeleg |= (1 << 4);   // Load address misaligned
    vcpu.hedeleg |= (1 << 5);   // Load access fault
    vcpu.hedeleg |= (1 << 6);   // Store/AMO address misaligned
    vcpu.hedeleg |= (1 << 7);   // Store/AMO access fault
    vcpu.hedeleg |= (1 << 8);   // Environment call from U-mode
    vcpu.hedeleg |= (1 << 12);  // Instruction page fault
    vcpu.hedeleg |= (1 << 13);  // Load page fault
    vcpu.hedeleg |= (1 << 15);  // Store/AMO page fault

    // Configure sstatus:
    //   SPP = 1 (previous privilege = S-mode / VS-mode)
    vcpu.sstatus = (1ULL << 8);

    // Set the guest page table
    vcpu.hgatp = guest_page_table_hgatp(table);

    // Set initial guest program counter
    vcpu.sepc = guest_entry;

    // Allocate a dedicated stack for the hypervisor trap handler (512KB)
    size_t stack_size = 512 * 1024;
    void *stack_base = alloc_pages(stack_size);
    vcpu.host_sp = (uint64_t)stack_base + stack_size;  // Stack grows down

    return vcpu;
}
```

### vcpu_run

```c
// Enter guest mode - restores all guest state and executes sret.
// This function never returns; on VM exit, trap_handler is invoked,
// which eventually calls vcpu_run again to re-enter the guest.
__attribute__((noreturn))
void vcpu_run(VCpu *self) {
    // 1. Write hypervisor CSRs
    csr_write(hstatus, self->hstatus);
    csr_write(sstatus, self->sstatus);
    csr_write(sscratch, (uint64_t)self);  // Trap handler will read this
    csr_write(hgatp, self->hgatp);        // Guest page table
    csr_write(hedeleg, self->hedeleg);    // Exception delegation
    csr_write(hcounteren, 0b11);          // Allow guest to read cycle & time
    csr_write(sepc, self->sepc);          // Guest program counter

    // 2. Restore all guest general-purpose registers from VCpu
    //    (using a0 as base pointer temporarily)
    asm("mv a0, self");
    asm("ld ra,  offsetof(VCpu, ra)(a0)");
    asm("ld sp,  offsetof(VCpu, sp)(a0)");
    asm("ld gp,  offsetof(VCpu, gp)(a0)");
    asm("ld tp,  offsetof(VCpu, tp)(a0)");
    asm("ld t0,  offsetof(VCpu, t0)(a0)");
    asm("ld t1,  offsetof(VCpu, t1)(a0)");
    asm("ld t2,  offsetof(VCpu, t2)(a0)");
    asm("ld s0,  offsetof(VCpu, s0)(a0)");
    asm("ld s1,  offsetof(VCpu, s1)(a0)");
    asm("ld a1,  offsetof(VCpu, a1)(a0)");
    // ... (a2-a7, s2-s11, t3-t6 similarly)
    asm("ld a0,  offsetof(VCpu, a0)(a0)");  // Restore a0 last

    // 3. Return to guest mode
    //    sret uses hstatus.SPV to determine we're entering VS-mode
    //    and jumps to the address in sepc
    asm("sret");

    __builtin_unreachable();
}
```

### vcpu_read_register / vcpu_write_register (helpers)

```c
// Read a guest register by index (0=x0/zero, 1=ra, ..., 31=t6)
uint64_t vcpu_read_register(VCpu *vcpu, uint64_t reg) {
    switch (reg) {
        case 0:  return 0;          // x0 is hardwired to 0
        case 1:  return vcpu->ra;
        case 2:  return vcpu->sp;
        case 3:  return vcpu->gp;
        case 4:  return vcpu->tp;
        case 5:  return vcpu->t0;
        case 6:  return vcpu->t1;
        case 7:  return vcpu->t2;
        case 8:  return vcpu->s0;
        case 9:  return vcpu->s1;
        case 10: return vcpu->a0;
        case 11: return vcpu->a1;
        case 12: return vcpu->a2;
        case 13: return vcpu->a3;
        case 14: return vcpu->a4;
        case 15: return vcpu->a5;
        case 16: return vcpu->a6;
        case 17: return vcpu->a7;
        case 18: return vcpu->s2;
        case 19: return vcpu->s3;
        case 20: return vcpu->s4;
        case 21: return vcpu->s5;
        case 22: return vcpu->s6;
        case 23: return vcpu->s7;
        case 24: return vcpu->s8;
        case 25: return vcpu->s9;
        case 26: return vcpu->s10;
        case 27: return vcpu->s11;
        case 28: return vcpu->t3;
        case 29: return vcpu->t4;
        case 30: return vcpu->t5;
        case 31: return vcpu->t6;
        default: __builtin_unreachable();
    }
}

// Write a value to a guest register by index
void vcpu_write_register(VCpu *vcpu, uint64_t reg, uint64_t value) {
    switch (reg) {
        case 0:  break;               // x0 writes are ignored
        case 1:  vcpu->ra  = value; break;
        case 2:  vcpu->sp  = value; break;
        case 3:  vcpu->gp  = value; break;
        case 4:  vcpu->tp  = value; break;
        case 5:  vcpu->t0  = value; break;
        case 6:  vcpu->t1  = value; break;
        case 7:  vcpu->t2  = value; break;
        case 8:  vcpu->s0  = value; break;
        case 9:  vcpu->s1  = value; break;
        case 10: vcpu->a0  = value; break;
        case 11: vcpu->a1  = value; break;
        case 12: vcpu->a2  = value; break;
        case 13: vcpu->a3  = value; break;
        case 14: vcpu->a4  = value; break;
        case 15: vcpu->a5  = value; break;
        case 16: vcpu->a6  = value; break;
        case 17: vcpu->a7  = value; break;
        case 18: vcpu->s2  = value; break;
        case 19: vcpu->s3  = value; break;
        case 20: vcpu->s4  = value; break;
        case 21: vcpu->s5  = value; break;
        case 22: vcpu->s6  = value; break;
        case 23: vcpu->s7  = value; break;
        case 24: vcpu->s8  = value; break;
        case 25: vcpu->s9  = value; break;
        case 26: vcpu->s10 = value; break;
        case 27: vcpu->s11 = value; break;
        case 28: vcpu->t3  = value; break;
        case 29: vcpu->t4  = value; break;
        case 30: vcpu->t5  = value; break;
        case 31: vcpu->t6  = value; break;
        default: __builtin_unreachable();
    }
}
```
