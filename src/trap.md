# Trap Handler - Pseudocode

Handles VM exits (traps) from guest mode. When the guest executes an
operation that requires hypervisor intervention (SBI calls, MMIO access),
the CPU traps into the hypervisor and this code processes the event.

## Global State

```c
// Buffer for guest console output (line-buffered printing)
static spinlock_t console_lock;
static uint8_t console_buffer[4096];
static size_t console_len = 0;
```

## Functions

### trap_handler (assembly entry point)

```c
// Naked function placed at .text.stvec - the CPU jumps here on any trap.
// The stvec CSR points to this function.
__attribute__((naked, section(".text.stvec")))
__attribute__((noreturn))
void trap_handler(void) {
    // Swap a0 with sscratch (sscratch holds pointer to VCpu struct)
    asm("csrrw a0, sscratch, a0");

    // a0 now points to VCpu. Save all guest general-purpose registers.
    // (ra, sp, gp, tp, t0-t6, s0-s11, a1-a7 saved at known offsets)
    asm("sd ra,  offsetof(VCpu, ra)(a0)");
    asm("sd sp,  offsetof(VCpu, sp)(a0)");
    asm("sd gp,  offsetof(VCpu, gp)(a0)");
    asm("sd tp,  offsetof(VCpu, tp)(a0)");
    asm("sd t0,  offsetof(VCpu, t0)(a0)");
    asm("sd t1,  offsetof(VCpu, t1)(a0)");
    asm("sd t2,  offsetof(VCpu, t2)(a0)");
    asm("sd s0,  offsetof(VCpu, s0)(a0)");
    asm("sd s1,  offsetof(VCpu, s1)(a0)");
    asm("sd a1,  offsetof(VCpu, a1)(a0)");
    // ... (a2-a7, s2-s11, t3-t6 similarly)

    // Recover original a0 from sscratch and save it
    asm("csrr t0, sscratch");
    asm("sd t0, offsetof(VCpu, a0)(a0)");

    // Switch to the hypervisor's dedicated stack
    asm("ld sp, offsetof(VCpu, host_sp)(a0)");

    // Call the C trap handler with vcpu pointer as argument
    asm("call handle_trap");
}
```

### handle_trap

```c
// Main trap dispatch - called after guest state is saved
__attribute__((noreturn))
void handle_trap(VCpu *vcpu) {
    uint64_t scause = csr_read(scause);  // Trap cause
    uint64_t sepc   = csr_read(sepc);    // Trapped instruction address
    uint64_t stval  = csr_read(stval);   // Trap value (faulting address, etc.)

    switch (scause) {
        case 10:  // Environment call from VS-mode (SBI call from guest)
            handle_sbi_call(vcpu);
            vcpu->sepc = sepc + 4;  // Skip past the ecall instruction
            break;

        case 21:  // Load guest-page fault (MMIO read)
        case 23:  // Store/AMO guest-page fault (MMIO write)
        {
            uint64_t htinst = csr_read(htinst);  // Trapped instruction hint
            uint64_t htval  = csr_read(htval);   // Guest physical address >> 2
            assert(htinst != 0);

            // Reconstruct the full guest physical address
            uint64_t guest_addr = (htval << 2) | (stval & 0x3);
            bool is_write = (scause == 23);

            // Decode instruction width from htinst
            uint64_t opcode = htinst & 0x7F;
            uint64_t funct3 = (htinst >> 12) & 0x7;
            int width;
            switch (opcode) {
                case 0x03:  // Load instructions
                    switch (funct3) {
                        case 0: case 4: width = 1; break;  // lb, lbu
                        case 1: case 5: width = 2; break;  // lh, lhu
                        case 2: case 6: width = 4; break;  // lw, lwu
                        case 3:         width = 8; break;  // ld
                    }
                    break;
                case 0x23:  // Store instructions
                    switch (funct3) {
                        case 0: width = 1; break;  // sb
                        case 1: width = 2; break;  // sh
                        case 2: width = 4; break;  // sw
                        case 3: width = 8; break;  // sd
                    }
                    break;
                default: width = 4; break;
            }

            if (is_write) {
                uint64_t rs2 = (htinst >> 20) & 0x1F;  // Source register
                handle_mmio_write(vcpu, guest_addr, rs2, width);
            } else {
                uint64_t rd = (htinst >> 7) & 0x1F;    // Destination register
                handle_mmio_read(vcpu, guest_addr, rd, width);
            }

            // Advance PC past the faulting instruction
            bool is_compressed = (htinst & (1 << 1)) == 0;
            int inst_len = is_compressed ? 2 : 4;
            vcpu->sepc = sepc + inst_len;
            break;
        }

        default:
            panic("trap handler: %s at 0x%lx (stval=0x%lx)",
                  scause_to_string(scause), sepc, stval);
    }

    // Resume guest execution
    vcpu_run(vcpu);
}
```

### handle_sbi_call

```c
// Handle an SBI (Supervisor Binary Interface) call from the guest
void handle_sbi_call(VCpu *vcpu) {
    uint64_t eid = vcpu->a7;  // Extension ID
    uint64_t fid = vcpu->a6;  // Function ID
    int64_t error = 0;
    int64_t value = 0;

    if (eid == 0x00 && fid == 0x0) {
        // Legacy Set Timer - not implemented, just acknowledge
        printf("[sbi] WARN: set_timer is not implemented, ignoring\n");
        error = 0; value = 0;
    }
    else if (eid == 0x10 && fid == 0x0) {
        // Get SBI specification version
        error = 0; value = 0;
    }
    else if (eid == 0x10 && fid == 0x3) {
        // Probe SBI extension - report not available
        error = -1;
    }
    else if (eid == 0x10 && (fid == 0x4 || fid == 0x5 || fid == 0x6)) {
        // Get machine vendor/arch/implementation ID
        error = 0; value = 0;
    }
    else if (eid == 0x1 && fid == 0x0) {
        // Console Putchar - buffer output, flush on newline
        uint8_t ch = (uint8_t)vcpu->a0;
        lock(&console_lock);
        if (ch == '\n') {
            console_buffer[console_len] = '\0';
            printf("[guest] %s\n", console_buffer);
            console_len = 0;
        } else {
            console_buffer[console_len++] = ch;
        }
        unlock(&console_lock);
        error = 0; value = 0;
    }
    else if (eid == 0x2 && fid == 0x0) {
        // Console Getchar - not supported
        error = -1;
    }
    else {
        panic("unknown SBI call: eid=0x%lx, fid=0x%lx", eid, fid);
    }

    // Return result per SBI convention: a0=error, a1=value
    if (error == 0) {
        vcpu->a0 = 0;
        vcpu->a1 = (uint64_t)value;
    } else {
        vcpu->a0 = (uint64_t)error;
    }
}
```

### handle_mmio_write

```c
// Handle a guest store to an unmapped (MMIO) address
void handle_mmio_write(VCpu *vcpu, uint64_t guest_addr, uint64_t reg, uint64_t width) {
    // Read the value from the source register
    uint64_t value = vcpu_read_register(vcpu, reg);

    if (guest_addr >= PLIC_ADDR && guest_addr < PLIC_END) {
        // PLIC writes are silently ignored (stub)
        printf("[MMIO]: ignore write to PLIC at 0x%lx\n", guest_addr);
    } else {
        panic("[MMIO]: invalid write at 0x%lx (value=0x%lx, width=%lu)",
              guest_addr, value, width);
    }
}
```

### handle_mmio_read

```c
// Handle a guest load from an unmapped (MMIO) address
void handle_mmio_read(VCpu *vcpu, uint64_t guest_addr, uint64_t reg, uint64_t width) {
    uint64_t value;

    if (guest_addr >= PLIC_ADDR && guest_addr < PLIC_END) {
        // PLIC reads return 0 (stub)
        printf("[MMIO]: ignore read from PLIC at 0x%lx\n", guest_addr);
        value = 0;
    } else {
        panic("[MMIO]: invalid read at 0x%lx (width=%lu)", guest_addr, width);
    }

    // Write the value into the destination register
    vcpu_write_register(vcpu, reg, value);
}
```
