# Plan: RISC-V Type-1 Hypervisor in C

Target: a bare-metal HS-mode hypervisor, written in C11 + a small amount of assembly, that boots an unmodified Linux `Image` as a VS-mode guest on QEMU `virt` with OpenSBI as M-mode firmware. The Rust tree you supplied is the reference behavior; this plan treats it as a working spec with known gaps, not as something to transliterate line by line.

---

## 1. What the reference actually does

Read the Rust as a state machine before porting anything. It is roughly 850 lines and the whole thing is one loop with no return path:

```
OpenSBI (M-mode)
   └─ jumps to 0x80200000 in S-mode  ──►  boot()  (.text.boot)
                                            │  sp = __stack_top
                                            ▼
                                          main()
                                            │  zero .bss, stvec = trap_handler
                                            │  bump allocator init over __heap..__heap_end
                                            │  build G-stage tables, copy Image + DTB into
                                            │    static arrays, map them GPA→HPA
                                            │  vcpu.a0 = hartid, vcpu.a1 = GUEST_DTB_ADDR
                                            ▼
                                       VCpu::run()  ── csrw hstatus/sstatus/hgatp/hedeleg/
                                            │          hcounteren/sepc, restore x1..x31, sret
                                            ▼
                                    ══ VS-mode: Linux ══
                                            │  ecall / guest-page fault
                                            ▼
                                     trap_handler (naked)
                                            │  csrrw a0, sscratch, a0  → a0 = &VCpu
                                            │  save x1..x31 into VCpu, sp = vcpu.host_sp
                                            ▼
                                      handle_trap(vcpu) -> !
                                            │  scause 10 → SBI, 21/23 → MMIO
                                            │  advance vcpu.sepc
                                            └──────── tail-calls VCpu::run() again
```

Two design facts matter for the port:

- **The host has no persistent stack frame.** `host_sp` is reloaded from the vCPU struct on every trap entry, so the apparently-recursive `handle_trap → run → trap_handler → handle_trap` chain never grows the stack. It also means the host cannot keep anything in local variables across a guest entry.
- **`sscratch` holds the vCPU pointer while the guest runs.** The `csrrw a0, sscratch, a0` on entry swaps the pointer into `a0` and the guest's `a0` into `sscratch`, which is then read back and stored. `run()` rewrites `sscratch` on every entry, which is what keeps this consistent.

Address map as configured:

| Region | Guest physical | Backing |
|---|---|---|
| Guest DTB | `0x7000_0000`, 64 KiB | `DTB_MEMORY` static array, mapped R |
| Guest RAM | `0x8000_0000`, 64 MiB | `GUEST_MEMORY` static array, mapped RWX |
| PLIC | `0x0c00_0000`, +4 MiB in code / +64 MiB in the DT | unmapped, trapped, stubbed |

Everything not in the G-stage table faults with scause 21/23 and lands in the MMIO decoder.

---

## 2. Target environment and toolchain

```
QEMU:      qemu-system-riscv64 -machine virt -cpu rv64,h=true -smp 1 -m 512M \
             -nographic -bios default -kernel build/hypervisor.elf
```

`-cpu rv64,h=true` is required; the H extension is not on by default on the generic `rv64` CPU. Older QEMU spells it `x-h=true`. `-bios default` loads the bundled OpenSBI, which enters your `-kernel` payload at `0x8020_0000` in S-mode with `a0 = hartid`, `a1 = host DTB`.

Give it more RAM than the reference implies. 64 MiB of guest RAM lives inside the hypervisor image's memory footprint, plus heap, plus the kernel copy. `-m 512M` removes a whole class of confusing early failures.

Toolchain: `riscv64-unknown-elf-gcc` (or `riscv64-linux-gnu-gcc` used freestanding, or clang `--target=riscv64`). The one real toolchain risk is H-extension CSR *names* in the assembler. Two mitigations, use both:

1. Try `-march=rv64imac_zicsr_zifencei_h`. If binutils rejects the `h`, drop it — nothing breaks as long as you follow (2).
2. Never write `csrr t0, hgatp` in source. Define CSRs numerically and go through macros. Numeric CSR addresses assemble on every toolchain, forever.

```c
/* include/csr.h */
#define __STR(x) #x
#define STR(x)   __STR(x)

#define CSR_SSTATUS     0x100
#define CSR_SIE         0x104
#define CSR_STVEC       0x105
#define CSR_SCOUNTEREN  0x106
#define CSR_SSCRATCH    0x140
#define CSR_SEPC        0x141
#define CSR_SCAUSE      0x142
#define CSR_STVAL       0x143
#define CSR_SIP         0x144
#define CSR_SATP        0x180

#define CSR_HSTATUS     0x600
#define CSR_HEDELEG     0x602
#define CSR_HIDELEG     0x603
#define CSR_HIE         0x604
#define CSR_HTIMEDELTA  0x605
#define CSR_HCOUNTEREN  0x606
#define CSR_HGEIE       0x607
#define CSR_HENVCFG     0x60a
#define CSR_HTVAL       0x643
#define CSR_HIP         0x644
#define CSR_HVIP        0x645
#define CSR_HTINST      0x64a
#define CSR_HGATP       0x680
#define CSR_HGEIP       0xe12

#define CSR_VSSTATUS    0x200
#define CSR_VSIE        0x204
#define CSR_VSTVEC      0x205
#define CSR_VSSCRATCH   0x240
#define CSR_VSEPC       0x241
#define CSR_VSCAUSE     0x242
#define CSR_VSTVAL      0x243
#define CSR_VSIP        0x244
#define CSR_VSTIMECMP   0x24d
#define CSR_VSATP       0x280

#define csr_read(csr) ({                                                  \
    unsigned long __v;                                                    \
    __asm__ __volatile__ ("csrr %0, " STR(csr) : "=r"(__v) : : "memory"); \
    __v; })

#define csr_write(csr, val) ({                                            \
    unsigned long __v = (unsigned long)(val);                             \
    __asm__ __volatile__ ("csrw " STR(csr) ", %0" :: "rK"(__v) : "memory"); })

#define csr_set(csr, val) ({                                              \
    unsigned long __v = (unsigned long)(val);                             \
    __asm__ __volatile__ ("csrs " STR(csr) ", %0" :: "rK"(__v) : "memory"); })

#define csr_clear(csr, val) ({                                            \
    unsigned long __v = (unsigned long)(val);                             \
    __asm__ __volatile__ ("csrc " STR(csr) ", %0" :: "rK"(__v) : "memory"); })
```

Same treatment for `hfence.gvma`, which older assemblers also refuse:

```c
/* HFENCE.GVMA x0, x0 : funct7=0b0110001, rs2=x0, rs1=x0, funct3=0, rd=x0, op=0x73 */
static inline void hfence_gvma_all(void) {
    __asm__ __volatile__ (".insn r 0x73, 0, 0x31, x0, x0, x0" ::: "memory");
}
```

---

## 3. Repository layout

```
hypervisor/
  Makefile
  boot/
    boot.S               entry from OpenSBI, stack, bss clear
    hypervisor.ld        link script, memory map, guest RAM region
  arch/riscv/
    entry.S              vcpu_enter / guest_exit  (the only performance-critical asm)
    csr.h                CSR numbers + accessor macros
    trap.c               scause dispatch, exit-reason decode
    vcpu.c               vCPU init, entry/exit wrapper, CSR programming
    gstage.c             Sv48x4 / Sv39x4 G-stage page tables
    hlv.h                hlv/hlvx wrappers for reading guest memory
  vmm/
    vm.c                 VM object: memory, devices, vcpus
    sbi.c                SBI service implementation (guest-facing)
    sbi_call.c           SBI client (host → OpenSBI)
    mmio.c               MMIO bus dispatch
    loader.c             Image header parse, guest RAM population
    fdt.c                flattened device tree writer
  dev/
    plic.c               SiFive PLIC model
    virtio_mmio.c        virtio transport
    virtio_blk.c         block device
  lib/
    printf.c string.c bump.c ring.c
  include/
    ...
  guest/
    Image                (or referenced out-of-tree)
    rootfs.cpio
  test/
    host/                native unit tests for fdt.c, gstage.c, plic.c, virtio
```

The `test/host/` directory is the single highest-leverage decision in this port. `fdt.c`, `gstage.c`, `plic.c`, and the virtqueue walker are pure logic with no privileged instructions. Write them so they take an allocator callback and a base pointer, and they compile and run under a normal x86 test runner with ASan and UBSan on. You will otherwise be debugging a page table walker through a serial port.

---

## 4. Module-by-module port map

### 4.1 `allocator.rs` → `lib/bump.c`

Drop the `spin::Mutex` for now (single hart), but leave the API shaped so a lock can be added:

```c
void  bump_init(void *start, void *end);
void *bump_alloc(size_t size, size_t align);
void *bump_alloc_pages(size_t size);   /* align 4096, zeroed */
```

Two things Rust was doing for you that C is not:

- `next_multiple_of` and `saturating_add`. Write the check as `if (size > (uintptr_t)end - addr) panic(...)` so there is no overflow to reason about.
- Alignment must be a power of two. `assert((align & (align - 1)) == 0)`.

Add `bump_stats()` returning used/free bytes and print it after setup. Silent heap exhaustion in a page table walker is miserable to diagnose.

### 4.2 `guest_page_table.rs` → `arch/riscv/gstage.c`

This is where the reference has its most consequential bug. **Sv48x4 and Sv39x4 root tables are 16 KiB, not 4 KiB.** The G-stage root is four pages, holds 2048 entries, must be 16 KiB-aligned, and the top-level index is 11 bits, not 9:

```
Sv48x4:  gpa[50:39] → root index (11 bits, 2048 entries, 16 KiB table)
         gpa[38:30] → level 2 (9 bits)
         gpa[29:21] → level 1
         gpa[20:12] → level 0
```

The reference allocates 4096 bytes and masks the top index to 9 bits. It happens to work because every GPA in use is below 2^39, so the top index is always 0 and only the first 512 entries are touched — but `hgatp.PPN` still has to be 16 KiB-aligned, and `alloc_pages` only guarantees 4 KiB. That is a coin flip on whether the mode field is honored. Fix both:

```c
#define PTE_V (1UL << 0)
#define PTE_R (1UL << 1)
#define PTE_W (1UL << 2)
#define PTE_X (1UL << 3)
#define PTE_U (1UL << 4)   /* mandatory on G-stage leaves: all accesses are user-level */
#define PTE_A (1UL << 6)
#define PTE_D (1UL << 7)

#define HGATP_MODE_SV39X4  (8UL << 60)
#define HGATP_MODE_SV48X4  (9UL << 60)

struct gstage {
    uint64_t *root;    /* 2048 entries, 16 KiB aligned */
    int levels;        /* 3 for Sv39x4, 4 for Sv48x4 */
};

void     gstage_init(struct gstage *g, int levels);
int      gstage_map(struct gstage *g, uint64_t gpa, uint64_t hpa, uint64_t flags);
int      gstage_map_range(struct gstage *g, uint64_t gpa, uint64_t hpa,
                          size_t len, uint64_t flags);
int      gstage_unmap(struct gstage *g, uint64_t gpa);
uint64_t gstage_hgatp(const struct gstage *g);
uint64_t gstage_walk(const struct gstage *g, uint64_t gpa);  /* for tests + debug dump */
```

Other deltas from the reference:

- Set `PTE_A | PTE_D` on leaves. QEMU is permissive about A/D on G-stage; some hardware is not, and you do not want to write a fault handler for it.
- Support remap. The reference asserts `!entry.is_valid()`, which makes it impossible to convert a RAM page into an MMIO trap page later. You will want that for device hot-plug and for testing.
- Use 2 MiB superpages for the guest RAM mapping. 64 MiB at 4 KiB granularity is 16384 leaf entries and 33 tables; at 2 MiB it is 32 entries. Under TCG this is a visible boot-time difference, and it makes the debug dump readable.
- **Call `hfence_gvma_all()` after building the tables and before the first `sret`.** The reference never does. It works on TCG because the TLB starts empty; it will not work on hardware, and it will bite you the moment you add `gstage_unmap`.
- Start with Sv39x4 (`levels = 3`). It covers 512 GiB of GPA, which is more than enough, it is one level shallower to debug, and switching to Sv48x4 later is a constant change.

Write `gstage_walk()` first and unit-test it natively against a table your test builds. Everything downstream depends on this being right.

### 4.3 `guest_memory.rs` → linker-script regions + `vmm/loader.c`

Do not put 64 MiB of guest RAM in `.bss`. Carve it in the linker script as a `NOLOAD` region so that (a) it appears in the `.map` file, (b) `clear_bss` does not spend real time zeroing 64 MiB under emulation, and (c) the memory map is described in one place.

```ld
OUTPUT_ARCH(riscv)
ENTRY(boot)

SECTIONS {
    . = 0x80200000;

    .text : ALIGN(4K) {
        KEEP(*(.text.boot))
        *(.text .text.*)
    }

    .rodata : ALIGN(4K) {
        *(.rodata .rodata.*)
        . = ALIGN(8);
        __guest_image_start = .;
        KEEP(*(.guest_image))          /* .incbin of Image lands here */
        __guest_image_end = .;
    }

    .data : ALIGN(4K) { *(.data .data.*) }

    .bss : ALIGN(4K) {
        __bss = .;
        *(.sbss .sbss.* .bss .bss.* COMMON)
        . = ALIGN(4K);
        __bss_end = .;
    }

    . = ALIGN(16);
    . += 256K;  __stack_top = .;

    . = ALIGN(4K);
    __heap = .;      . += 32M;   __heap_end = .;

    . = ALIGN(2M);                      /* superpage-align guest RAM */
    .guest_ram (NOLOAD) : {
        __guest_ram = .;
        . += 64M;
        __guest_ram_end = .;
    }

    . = ALIGN(4K);
    __guest_dtb = .;  . += 64K;  __guest_dtb_end = .;

    /DISCARD/ : { *(.eh_frame) *(.comment) *(.note.*) }
}
```

Because the mapping is flat and contiguous, guest-physical to host-virtual is a subtraction, and that one fact simplifies the entire device model:

```c
static inline void *gpa_to_hva(uint64_t gpa) {
    if (gpa < GUEST_RAM_BASE || gpa >= GUEST_RAM_BASE + GUEST_RAM_SIZE)
        return NULL;
    return (void *)(__guest_ram + (gpa - GUEST_RAM_BASE));
}
```

Return `NULL` and check it. Virtqueue descriptors come from the guest and are not trustworthy; every descriptor address goes through this function, and the failure path is "kill the guest," not "dereference."

### 4.4 `linux_loader.rs` → `vmm/loader.c`

`include_bytes!` becomes `.incbin`:

```asm
/* guest/image.S */
.section .guest_image, "a"
.incbin "guest/Image"
```

Or `objcopy -I binary -O elf64-littleriscv --rename-section .data=.guest_image`. The `.incbin` route is fewer moving parts.

Header handling, with the fixes the reference skips:

```c
struct riscv_image_header {
    uint32_t code0, code1;
    uint64_t text_offset;    /* load offset from a 2 MiB-aligned base */
    uint64_t image_size;
    uint64_t flags;
    uint32_t version;
    uint32_t res1;
    uint64_t res2;
    uint64_t magic;          /* legacy "RISCV\0\0\0", zero on new kernels */
    uint32_t magic2;         /* "RSC\x05" = 0x05435352 */
    uint32_t res3;
};
```

- Validate `magic2`, not `magic`. The reference gets this right; keep it.
- **Honor `text_offset`.** The reference copies the image to offset 0 of guest RAM and ignores `text_offset` entirely. Copy to `GUEST_RAM_BASE + text_offset` and set the vCPU entry point to that address. Current kernels use `text_offset = 0x20_0000`, and a relocatable kernel papers over the difference, so this can silently "work" until it does not.
- Validate `image_size <= GUEST_RAM_SIZE - text_offset` and that the incbin length is at least `sizeof(header)`.
- **Move the DTB inside guest RAM.** `0x7000_0000` is below the declared `memory` node. Linux's early fixmap will still reach it, but it is outside the memory map it is being handed, which is a standard-violating setup you will eventually have to explain to a driver. Put it near the top of guest RAM (say `GUEST_RAM_BASE + GUEST_RAM_SIZE - 2 MiB`), 8-byte aligned, and add a memory reservation entry for it.

### 4.5 `linux_loader.rs::build_device_tree` → `vmm/fdt.c`

`vm_fdt` is the only real third-party dependency, and it needs replacing. Two options, and you should do both in order:

**Stage 1 (M5): write the `.dts`, compile it with `dtc`, `.incbin` the `.dtb`.** Zero code, and it decouples "does my hypervisor boot Linux" from "is my FDT writer correct." Use this to get to a kernel banner.

**Stage 2 (M6+): a ~200-line FDT writer.** You need it once memory size, hart count, or virtio device layout become runtime values. The format is small and entirely mechanical, all big-endian:

```
┌─ struct fdt_header (40 bytes, be32) ─────────────────────┐
│ magic = 0xd00dfeed                                        │
│ totalsize, off_dt_struct, off_dt_strings, off_mem_rsvmap  │
│ version = 17, last_comp_version = 16                      │
│ boot_cpuid_phys, size_dt_strings, size_dt_struct          │
└───────────────────────────────────────────────────────────┘
memory reservation block: (u64 addr, u64 size) pairs, 8-byte aligned,
                          terminated by (0, 0)
structure block, 4-byte aligned tokens:
  FDT_BEGIN_NODE (0x1) + NUL-terminated name, padded to 4
  FDT_PROP       (0x3) + be32 len + be32 nameoff + value, padded to 4
  FDT_END_NODE   (0x2)
  FDT_END        (0x9)
strings block: NUL-terminated property names, referenced by nameoff
```

API:

```c
struct fdt_writer { uint8_t *buf; size_t cap, cur, strings_cur; int depth; bool err; };

void   fdt_begin(struct fdt_writer *w, void *buf, size_t cap);
void   fdt_begin_node(struct fdt_writer *w, const char *name);
void   fdt_end_node(struct fdt_writer *w);
void   fdt_prop_null(struct fdt_writer *w, const char *name);
void   fdt_prop_u32(struct fdt_writer *w, const char *name, uint32_t v);
void   fdt_prop_u64(struct fdt_writer *w, const char *name, uint64_t v);
void   fdt_prop_reg(struct fdt_writer *w, uint64_t addr, uint64_t size);
void   fdt_prop_str(struct fdt_writer *w, const char *name, const char *v);
void   fdt_prop_cells(struct fdt_writer *w, const char *name,
                      const uint32_t *cells, size_t n);
size_t fdt_finish(struct fdt_writer *w);   /* 0 on error */
```

Use a sticky `err` flag instead of return codes on every call — it keeps the caller readable and gives one place to check. Dedupe strings with a linear search; there are under 30 property names.

Validate with `dtc -I dtb -O dts` on the generated blob during host unit tests. That single test catches nearly every FDT bug you will write.

Fix these while you are in here:

- `format!("memory@{}", GUEST_BASE_ADDR)` emits `memory@2147483648`. Unit addresses are hex without `0x`: `memory@80000000`.
- The PLIC `reg` size in the DT (`0x4000000`) does not match the trap range in `trap.rs` (`PLIC_ADDR + 0x400000`). A guest access in the gap panics the hypervisor. Pick one number and define it once.
- `interrupts-extended = <&intc 11 &intc 9>` exposes the M-mode external context to a guest that has no M-mode. Expose only the S-mode context: `<&intc 9>`, and number PLIC contexts accordingly in your model.
- Add `stdout-path` under `/chosen` once you have a real console.
- Add `riscv,isa` extensions only for things you actually implement. If you advertise `sstc`, Linux will use `vstimecmp` and stop calling SBI `set_timer`.

### 4.6 `vcpu.rs` + `trap.rs` → `arch/riscv/vcpu.c` + `entry.S` + `trap.c`

Two structural changes here, and they are the heart of the port.

**Change 1: `uint64_t x[32]` instead of 31 named fields.** The reference pays for named fields with two 32-arm `match` statements in the MMIO path. In C:

```c
static inline uint64_t gpr_read(struct vcpu *v, unsigned r) {
    return r == 0 ? 0 : v->x[r];
}
static inline void gpr_write(struct vcpu *v, unsigned r, uint64_t val) {
    if (r != 0) v->x[r] = val;
}
```

That deletes about 70 lines and removes an entire class of copy-paste bug. The assembly gets simpler too, because offsets become `n * 8`.

**Change 2: make guest entry a *call* that returns, not a tail call.** The reference's `handle_trap` is `-> !` and re-enters the guest from the bottom of the trap handler. That works, but it means the host can never hold state across an entry, cannot loop, cannot do "run guest until exit, then poll devices," and cannot ever shut down cleanly. Restructure to the KVM shape:

```c
enum vm_exit { EXIT_SBI, EXIT_MMIO, EXIT_INTERRUPT, EXIT_SHUTDOWN, EXIT_FAULT };

void vcpu_loop(struct vcpu *v) {
    for (;;) {
        uint64_t scause = vcpu_enter(v);       /* asm: sret ... returns after exit */
        switch (decode_exit(v, scause)) {
        case EXIT_SBI:       sbi_handle(v); v->sepc += 4;      break;
        case EXIT_MMIO:      mmio_handle(v);                   break;
        case EXIT_INTERRUPT: irq_handle(v);                    break;
        case EXIT_SHUTDOWN:  return;
        default:             vcpu_dump(v); panic("guest fault");
        }
        vm_deliver_interrupts(v);              /* update hvip from PLIC/timer state */
    }
}
```

`arch/riscv/entry.S`, with the host context save that makes the return possible:

```asm
#include "vcpu-offsets.h"

/* uint64_t vcpu_enter(struct vcpu *v); returns scause at guest exit */
.globl vcpu_enter
.align 4
vcpu_enter:
    addi    sp, sp, -HOST_CTX_SIZE
    sd      ra,  0(sp)
    sd      gp,  8(sp)          /* guest will clobber gp and tp */
    sd      tp, 16(sp)
    sd      s0, 24(sp)
    /* ... s1..s11 ... */
    sd      sp, VCPU_HOST_SP(a0)

    la      t0, guest_exit
    csrw    CSR_STVEC, t0
    csrw    CSR_SSCRATCH, a0

    ld      t0, VCPU_HSTATUS(a0);  csrw CSR_HSTATUS, t0
    ld      t0, VCPU_SSTATUS(a0);  csrw CSR_SSTATUS, t0
    ld      t0, VCPU_SEPC(a0);     csrw CSR_SEPC, t0

    ld      ra, VCPU_X(1)(a0)
    ld      sp, VCPU_X(2)(a0)
    /* ... x3..x9, x11..x31 ... */
    ld      a0, VCPU_X(10)(a0)     /* a0 last */
    sret

.align 4
guest_exit:
    csrrw   a0, CSR_SSCRATCH, a0   /* a0 = &vcpu, sscratch = guest a0 */
    sd      ra, VCPU_X(1)(a0)
    sd      sp, VCPU_X(2)(a0)
    /* ... everything except x10 ... */
    csrr    t0, CSR_SSCRATCH
    sd      t0, VCPU_X(10)(a0)
    csrw    CSR_SSCRATCH, a0

    csrr    t0, CSR_SEPC;     sd t0, VCPU_SEPC(a0)
    csrr    t0, CSR_SSTATUS;  sd t0, VCPU_SSTATUS(a0)

    la      t0, host_trap_vector
    csrw    CSR_STVEC, t0

    ld      sp, VCPU_HOST_SP(a0)
    ld      ra,  0(sp)
    ld      gp,  8(sp)
    ld      tp, 16(sp)
    ld      s0, 24(sp)
    /* ... s1..s11 ... */
    addi    sp, sp, HOST_CTX_SIZE
    csrr    a0, CSR_SCAUSE         /* return value */
    ret
```

Notes on that:

- Swapping `stvec` around the `sret` gives you a separate host trap vector. A trap while the host is running is a hypervisor bug, and you want it to print a register dump rather than be misparsed as a guest exit.
- `gp` and `tp` are not callee-saved by the ABI but the host expects them to survive. The guest will overwrite both. Save them.
- Read `scause`, `stval`, `htval`, `htinst` in C (`decode_exit`), not in asm. There is no reason to do it in assembly and they are much easier to log from C.

**Offset synchronization.** Rust's `offset_of!` in inline asm has no C equivalent, and the Linux `asm-offsets.c` generator is more build machinery than this project needs. Use a hand-written header plus compile-time verification:

```c
/* include/vcpu-offsets.h — included by both .c and .S */
#define VCPU_X(n)      ((n) * 8)
#define VCPU_SEPC      0x100
#define VCPU_SSTATUS   0x108
#define VCPU_HSTATUS   0x110
#define VCPU_HOST_SP   0x118
#define HOST_CTX_SIZE  0x80

/* in vcpu.c */
_Static_assert(offsetof(struct vcpu, x)       == 0,            "layout");
_Static_assert(offsetof(struct vcpu, sepc)    == VCPU_SEPC,    "layout");
_Static_assert(offsetof(struct vcpu, sstatus) == VCPU_SSTATUS, "layout");
_Static_assert(offsetof(struct vcpu, hstatus) == VCPU_HSTATUS, "layout");
_Static_assert(offsetof(struct vcpu, host_sp) == VCPU_HOST_SP, "layout");
```

Adding a field in the wrong place now fails the build instead of corrupting guest state.

**CSR setup at vCPU init**, with the reference's omissions filled in:

```c
#define HSTATUS_VSXL_64   (2UL << 32)
#define HSTATUS_SPV       (1UL << 7)
#define HSTATUS_SPVP      (1UL << 8)   /* hlv/hsv act with supervisor privilege */

#define SSTATUS_SPP       (1UL << 8)
#define SSTATUS_FS_INIT   (1UL << 13)  /* without this the guest's first FP insn traps */

v->hstatus = HSTATUS_VSXL_64 | HSTATUS_SPV | HSTATUS_SPVP;
v->sstatus = SSTATUS_SPP | SSTATUS_FS_INIT;
v->sepc    = guest_entry;

csr_write(CSR_HGATP, gstage_hgatp(&vm->gstage));
hfence_gvma_all();

csr_write(CSR_HEDELEG, (1<<0)|(1<<1)|(1<<2)|(1<<3)|(1<<4)|(1<<5)|
                       (1<<6)|(1<<7)|(1<<8)|(1<<12)|(1<<13)|(1<<15));
csr_write(CSR_HIDELEG, (1<<2)|(1<<6)|(1<<10));   /* VSSIP, VSTIP, VSEIP */
csr_write(CSR_HCOUNTEREN, 0x3);                   /* cycle, time */
csr_write(CSR_HTIMEDELTA, 0);
csr_write(CSR_HVIP, 0);
```

`hideleg` is the single most important missing line in the reference. Without it, no interrupt can ever reach the guest, which is why the reference stops at "set_timer is not implemented, ignoring."

The DT in the reference advertises `rv64imafdc` while `sstatus.FS` stays Off. The first floating-point instruction the guest executes traps as an illegal instruction and the trap handler panics. With one guest and a host that never touches FP, setting `FS=Initial` once and never saving/restoring the F/D register file is correct and free.

### 4.7 MMIO decode → `vmm/mmio.c`

Keep the reference's htval/stval reconstruction, which is right:

```c
uint64_t gpa = (csr_read(CSR_HTVAL) << 2) | (csr_read(CSR_STVAL) & 0x3);
```

And the `htinst` interpretation, which is also right but under-documented: `htinst` holds a *transformed* instruction. `rs1` and the immediate are stripped — the address comes from htval/stval, never from `htinst`. Bit 1 is forced to 1 for a standard 32-bit instruction and cleared for the 32-bit expansion of a compressed one, which is how you get the instruction length.

```c
struct mmio_op {
    bool     is_write;
    unsigned width;      /* 1, 2, 4, 8 */
    unsigned reg;        /* rd for loads, rs2 for stores */
    bool     sign_extend;
    unsigned ilen;       /* 2 or 4 */
};

static bool decode_mmio(uint64_t htinst, struct mmio_op *op) {
    unsigned opcode = htinst & 0x7f;
    unsigned funct3 = (htinst >> 12) & 0x7;

    op->ilen = (htinst & 0x2) ? 4 : 2;

    if (opcode == 0x03) {                 /* LOAD */
        op->is_write    = false;
        op->reg         = (htinst >> 7) & 0x1f;
        op->width       = 1u << (funct3 & 0x3);
        op->sign_extend = !(funct3 & 0x4);
        return (funct3 & 0x3) <= 3;
    }
    if (opcode == 0x23) {                 /* STORE */
        op->is_write    = true;
        op->reg         = (htinst >> 20) & 0x1f;
        op->width       = 1u << (funct3 & 0x3);
        op->sign_extend = false;
        return funct3 <= 3;
    }
    return false;                         /* AMO on MMIO, or something exotic */
}
```

Three improvements over the reference:

1. **Sign extension.** The reference ignores `lb` vs `lbu` and writes the raw value into the register. Sign-extend narrow loads.
2. **No `default: width = 4`.** Returning false and panicking with the actual `htinst` value tells you what the guest did; silently guessing 4 bytes produces corruption a hundred instructions later.
3. **`htinst == 0` fallback.** The reference asserts. QEMU almost always populates `htinst`, but real hardware is permitted to write 0, and so is QEMU in some paths. The fallback is to fetch the faulting instruction at the guest's `sepc` using `hlvx.hu` (hypervisor load-execute, which runs the full VS-stage plus G-stage translation with execute permission) and decode it yourself, including the compressed forms. Stub this as `panic("htinst == 0, unimplemented")` in M4 and implement it when you care about hardware.

MMIO dispatch becomes a bus instead of an address-range `match`:

```c
struct mmio_dev {
    uint64_t base, size;
    const char *name;
    bool (*read)(void *ctx, uint64_t off, unsigned width, uint64_t *out);
    bool (*write)(void *ctx, uint64_t off, unsigned width, uint64_t val);
    void *ctx;
};

bool mmio_register(struct vm *vm, const struct mmio_dev *dev);
bool mmio_dispatch(struct vm *vm, struct vcpu *v, uint64_t gpa, struct mmio_op *op);
```

An unmatched address logs the GPA, the PC, and the decoded operation, then kills the guest. It should not kill the hypervisor — you want the register dump and the exit-trace ring, not a hang.

### 4.8 `print.rs` → `lib/printf.c` + `vmm/sbi_call.c`

Host-side SBI client:

```c
struct sbiret { long error, value; };

static inline struct sbiret sbi_ecall(long eid, long fid,
                                      unsigned long a0, unsigned long a1,
                                      unsigned long a2, unsigned long a3) {
    register unsigned long r_a0 __asm__("a0") = a0;
    register unsigned long r_a1 __asm__("a1") = a1;
    register unsigned long r_a2 __asm__("a2") = a2;
    register unsigned long r_a3 __asm__("a3") = a3;
    register unsigned long r_a6 __asm__("a6") = fid;
    register unsigned long r_a7 __asm__("a7") = eid;
    __asm__ __volatile__ ("ecall"
        : "+r"(r_a0), "+r"(r_a1)
        : "r"(r_a2), "r"(r_a3), "r"(r_a6), "r"(r_a7)
        : "memory");
    return (struct sbiret){ (long)r_a0, (long)r_a1 };
}
```

`printf` needs `%s %c %d %u %x %lx %llx %p %%` and zero/width padding. About 120 lines. Do not pull in a third-party one; you want it to work when everything else is broken, and you want it to never allocate.

The reference buffers guest console output until a newline using an unbounded `Vec`. In C, use a fixed 256-byte buffer and flush on full as well as on `\n`. A guest that emits a long line without a newline should not be able to grow hypervisor memory.

### 4.9 `handle_sbi_call` → `vmm/sbi.c`

This is where the reference is thinnest, and there is a subtlety worth understanding before you change anything.

`get_spec_version` returns 0. Linux reads that as "SBI v0.1," which routes it to the legacy call path — which is exactly why `console=hvc earlycon=sbi` works with only legacy putchar (EID 0x01) implemented. That behavior is load-bearing. It also requires the guest kernel to be built with `CONFIG_RISCV_SBI_V01=y`, which is not the default on current kernels.

So decide explicitly:

**Option A (M5 shortcut):** report spec version 0, implement legacy putchar/getchar, and build the guest kernel with `CONFIG_RISCV_SBI_V01=y`. Fewest moving parts to a kernel banner.

**Option B (M7 target):** report spec version 1.0 (`0x0100_0000`) and implement the real extension surface. This is what you want if the guest kernel is not yours to configure.

Extension IDs for Option B:

| Extension | EID | Functions needed |
|---|---|---|
| Base | `0x10` | spec_version, impl_id, impl_version, probe_extension, mvendorid/marchid/mimpid |
| TIME | `0x54494D45` | `set_timer` |
| IPI | `0x735049` | `send_ipi` |
| RFENCE | `0x52464E43` | `remote_fence_i`, `remote_sfence_vma[_asid]` |
| HSM | `0x48534D` | `hart_start`, `hart_stop`, `hart_get_status` |
| SRST | `0x53525354` | `system_reset` |
| DBCN | `0x4442434E` | `debug_console_write`, `_read`, `_write_byte` |

Two calibration notes:

- `probe_extension` returns success with value 1 (supported) or 0 (unsupported). The reference returns an *error*, which is wrong even for v0.1 semantics.
- Implement **SRST first**, before anything else in this list. The guest DT says `panic=-1`, meaning a guest panic reboots — which is an SRST call — which the reference turns into a hypervisor panic with a confusing message. Implementing SRST turns "hypervisor exploded" into "guest requested reset," which is worth an hour of debugging on day one.

Structure the dispatcher as a table, not a nested switch:

```c
struct sbi_ext {
    uint64_t eid;
    const char *name;
    struct sbiret (*handle)(struct vcpu *v, uint64_t fid);
};
```

Unknown EID returns `SBI_ERR_NOT_SUPPORTED` (-2) and logs once per EID. Never panic on an unknown SBI call — Linux probes for things it does not need.

---

## 5. Interrupts and timers

The reference has none of this, and it is the difference between "kernel prints a banner" and "kernel runs."

**Timer, path A (Sstc).** If the platform has Sstc, set `henvcfg.STCE` (M-mode must have set `menvcfg.STCE` first; OpenSBI does this when the extension is present) and advertise `sstc` in the guest's `riscv,isa`. The guest then writes `vstimecmp` directly and the hypervisor is not involved in timekeeping at all. This is dramatically less code. Check availability by probing whether `henvcfg.STCE` sticks when you set it.

**Timer, path B (SBI forwarding).** The guest calls `set_timer`. The hypervisor:

1. clears `hvip.VSTIP`,
2. forwards the deadline to OpenSBI's TIME extension (`htimedelta` is 0, so no adjustment),
3. enables `sie.STIE`,
4. on taking an HS timer interrupt: sets `hvip.VSTIP` to inject into the guest, clears `sie.STIE` to stop the interrupt storm, returns to the guest.

With `hideleg[6]` set, the injected VSTIP is delivered to VS-mode without further HS involvement.

**External interrupts.** Device model raises a line → PLIC model recomputes pending/enabled/threshold for the guest's S-mode context → if an interrupt is claimable, hypervisor sets `hvip.VSEIP`, else clears it. Do this recompute in `vm_deliver_interrupts()` at the top of the run loop, in exactly one place.

**Host trap routing.** Once you enable `sie.STIE`, the trap handler sees interrupts that are the host's, not the guest's. `decode_exit` must branch on the sign bit of `scause` before anything else. The reference panics on every interrupt, which is fine only because it never enables any.

---

## 6. Devices

**PLIC** (`dev/plic.c`, ~150 lines). SiFive layout relative to base:

```
0x000000 + 4*irq            priority[irq]
0x001000 + 4*(irq/32)       pending bitmap (read-only)
0x002000 + 0x80*ctx         enable bits for context
0x200000 + 0x1000*ctx + 0   threshold
0x200000 + 0x1000*ctx + 4   claim (read) / complete (write)
```

Expose exactly one context (the guest's VS-external), and make sure the number you use matches the `interrupts-extended` entry in the DT. Claim returns the highest-priority pending enabled IRQ and clears its pending bit; complete re-arms it. This model is pure logic — unit-test it natively with a scripted sequence of raise/claim/complete.

**virtio-mmio + virtio-blk.** MMIO transport at `0x1000_1000`, IRQ 1. Modern (v2) is less work than legacy because there is no page-size negotiation. The virtqueue walker reads descriptor, avail, and used rings out of guest RAM through `gpa_to_hva()`, with a bounds check on every single descriptor address and length. Back it with a filesystem image via `.incbin`, or a RAM disk you generate at build time.

**Shortcut worth taking:** before writing any of this, build the guest kernel with a CPIO initramfs embedded (`CONFIG_INITRAMFS_SOURCE`) and boot to a busybox shell with no block device at all. Change `root=/dev/vda` to nothing and let `rdinit=/bin/sh` run. You get an interactive guest with only the console and timer working, which is a much better platform for developing the device model than a kernel that dies at `mount_root`.

---

## 7. Milestones

Each has a binary acceptance test. Do not move on without it.

| # | Deliverable | Accept when |
|---|---|---|
| M0 | `boot.S`, linker script, SBI putchar, `printf`, `panic` | banner + heap/map dump appears under OpenSBI |
| M1 | bump allocator, memory map constants, `_Static_assert` scaffolding | allocator self-test prints alignments; exhaustion panics cleanly |
| M2 | `gstage.c` + `gstage_walk`, `hgatp`, `hfence` | native unit test: 10k random GPA map/walk round-trips; on-target dump of the guest RAM mapping matches the linker map |
| M3 | `entry.S`, `vcpu_enter`/`guest_exit`, ecall path | a 6-instruction guest blob (`li a0,'A'; li a7,1; ecall; j .`) prints `A` through the hypervisor's SBI handler and returns to the host loop |
| M4 | MMIO bus + decode + dummy device | guest blob writes then reads back a value at a trapped address; test both `-march=rv64imac` and `rv64ima` builds of the blob to exercise the 2-byte and 4-byte `ilen` paths |
| M5 | loader + `.dtb` via `dtc`, SBI v0.1 console | Linux prints its banner and reaches the point of wanting a timer |
| M6 | `hideleg`, timer (Sstc or SBI forwarding), host interrupt routing | boot proceeds past `calibrate_delay`; `dmesg` timestamps advance sanely |
| M7 | SBI v1.0 surface: base/probe, DBCN in and out, SRST, HSM | guest console accepts input; `poweroff` in the guest exits the hypervisor cleanly |
| M8 | initramfs boot to shell, then PLIC + virtio-blk | busybox prompt; then `mount /dev/vda` |
| M9 | SMP: per-hart vCPU array, HSM `hart_start`, IPI, RFENCE | guest boots with `-smp 2` and both harts online |
| M10 | FDT writer replaces the static `.dtb` | generated blob round-trips through `dtc -I dtb -O dts`; guest boots identically |

M3 is the milestone people underestimate. Write the guest blob by hand in assembly, link it at `0x8000_0000`, `objcopy` it to a binary, and `.incbin` it. It exercises the page tables, the entry/exit path, `sscratch`, the trap decode, and `sepc` advancement with nothing else in the way. Every bug you find there would otherwise be found under a booting kernel with 40,000 exits already logged.

---

## 8. Debugging setup

Build this before M3, not after M6.

**Exit trace ring.** A 256-entry array of `{scause, sepc, stval, htval, htinst, count}` written on every exit, dumped on panic. Collapse consecutive identical entries into a count. This one thing will explain most guest hangs immediately.

**QEMU flags.**

```
-d int,guest_errors,unimp -D qemu.log      # every trap, both privilege levels
-d in_asm                                  # instruction trace; enormous, but decisive
-s -S                                      # gdb stub on :1234, halted at reset
-monitor telnet:127.0.0.1:5555,server,nowait
```

**gdb.** `target remote :1234`, then `add-symbol-file build/hypervisor.elf`. gdb sees the HS-mode view. To debug the guest, load `vmlinux` symbols separately and remember that guest virtual addresses go through the guest's own `satp`, which gdb does not know about. In practice, disassembling around the guest `sepc` by hand from `vmlinux` is faster than fighting gdb about it.

**Golden test in CI.**

```make
test: build/hypervisor.elf
	timeout 60 qemu-system-riscv64 -machine virt -cpu rv64,h=true -smp 1 \
	  -m 512M -nographic -bios default -kernel $< 2>&1 | tee /tmp/boot.log
	grep -q "Freeing unused kernel image" /tmp/boot.log
```

**Guest state dump.** `vcpu_dump()` printing all 32 GPRs, `sepc`, `scause` decoded to a string, `stval`, `htval`, `htinst`, `hstatus`, `hgatp`, plus the G-stage walk of `stval` and of `sepc`. Call it from `panic()` and from any unhandled exit. The reference's `scause_str` table is good; keep it.

---

## 9. What Rust was providing, and the C replacement

| Rust mechanism | Where it mattered | C replacement |
|---|---|---|
| `offset_of!` inside `naked_asm!` | trap save/restore correctness | offsets header + `_Static_assert` mirror in `vcpu.c` |
| Slice bounds checks | `slice[..src.len()]` in the loader, register index in MMIO | explicit `if (reg >= 32) panic()`, explicit length checks in the loader and FDT writer |
| `saturating_add` | allocator overflow | rewrite as `size > end - next`, so overflow is not expressible |
| Exhaustive `match` | scause and SBI dispatch | `switch` on an enum with `-Wswitch-enum -Werror`, plus an explicit `default: panic()` |
| `Option`/`Result` | allocator init, FDT errors | sticky error flag in the FDT writer; `NULL`-returning `gpa_to_hva` checked at every call |
| `!` return type | `handle_trap` never returns | `__attribute__((noreturn))` on `panic()`; the run loop restructure removes the need elsewhere |
| Type-level `const SIZE` | guest memory sizing | linker-script symbols + a single `memmap.h` with `_Static_assert` on the relationships |
| Borrow checker on `&mut VCpu` | aliasing the vCPU from asm and C | one `static struct vcpu vcpus[MAX_HARTS]`, never heap-allocated, never aliased |

Two additions that pay for themselves:

- **A debug build with UBSan.** `-fsanitize=undefined -fno-sanitize-recover=all` works freestanding if you provide the `__ubsan_handle_*` symbols yourself; each one can just call `panic()` with the source location it is handed. Roughly 30 lines, and it catches misaligned accesses and shift-out-of-range in the page table and FDT code.
- **`-Wall -Wextra -Werror` plus `-Wconversion` on the device model.** Width-truncation bugs in MMIO handlers are hard to see and easy to prevent.

---

## 10. Defect list carried over from the reference

Track these explicitly so the port fixes them rather than reproducing them.

1. Sv48x4 root table allocated at 4 KiB with a 9-bit top index; must be 16 KiB, 2048 entries, 16 KiB-aligned, 11-bit index. (§4.2)
2. No `hfence.gvma` after building the G-stage tables. (§4.2)
3. `hideleg` never written, so no interrupt can reach the guest. (§4.6)
4. `sstatus.FS` left Off while the DT advertises `imafdc`; first guest FP instruction traps. (§4.6)
5. SBI `set_timer` stubbed out. (§5)
6. `probe_extension` returns an error instead of success-with-0. (§4.9)
7. SRST unimplemented, so `panic=-1` in the guest becomes a hypervisor panic. (§4.9)
8. PLIC region size disagrees between the DT (`0x4000000`) and the trap range (`0x400000`). (§4.5)
9. PLIC `interrupts-extended` exposes an M-mode context to a guest with no M-mode. (§4.5)
10. `memory@` unit address emitted in decimal. (§4.5)
11. Guest DTB placed outside the declared guest RAM range. (§4.4)
12. `text_offset` from the Image header ignored. (§4.4)
13. MMIO loads not sign-extended; unknown encodings silently default to 4 bytes. (§4.7)
14. `htinst == 0` asserts instead of falling back to `hlvx` fetch and decode. (§4.7)
15. AMO to MMIO unhandled. (§4.7)
16. Console buffer is an unbounded `Vec` fed by the guest. (§4.8)
17. `gstage_map` cannot remap, so a mapping can never be changed. (§4.2)
18. 64 MiB of guest RAM sits in `.bss` and is zeroed at every boot. (§4.3)
19. Any host-directed interrupt panics the hypervisor. (§5)
20. Single hart hard-coded; no HSM, IPI, or RFENCE.

---

## 11. Open decisions to make before M0

- **Sv39x4 or Sv48x4.** Recommend Sv39x4 to start; one less level to debug, 512 GiB of GPA is plenty, and the change is a constant.
- **SBI v0.1 or v1.0.** Recommend v0.1 through M5 to shorten the path to a kernel banner, then move to v1.0 at M7. Note that this constrains the guest kernel config in the meantime.
- **Sstc or SBI timer forwarding.** Check whether `henvcfg.STCE` sticks on your QEMU. If it does, Sstc removes an entire subsystem.
- **Initramfs or virtio-blk first.** Recommend initramfs. An interactive guest shell is a far better development platform than a kernel that dies at `mount_root`.
- **One vCPU struct per hart, statically allocated, or heap-allocated per VM.** Recommend static, sized by `MAX_HARTS`. It keeps the asm offsets stable and makes the "who owns this pointer" question unaskable.
