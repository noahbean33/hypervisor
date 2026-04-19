# Main - Pseudocode

Entry point and initialization for the RISC-V Type-1 hypervisor.
Boots from OpenSBI, initializes the heap, loads a Linux kernel image,
creates a virtual CPU, and enters guest mode.

## Linker Symbols

```c
// Defined by the linker script (hypervisor.ld)
extern uint8_t __bss;       // Start of BSS section
extern uint8_t __bss_end;   // End of BSS section
extern uint8_t __heap;      // Start of heap region
extern uint8_t __heap_end;  // End of heap region (100MB after __heap)
```

## Functions

### boot (entry point)

```c
// Placed at .text.boot so the linker puts it first at 0x80200000.
// OpenSBI jumps here after firmware initialization.
__attribute__((section(".text.boot")))
__attribute__((noreturn))
void boot(void) {
    // Set up stack pointer to linker-defined stack top
    asm("la sp, __stack_top");

    // Jump to main
    main();
}
```

### main

```c
__attribute__((noreturn))
void main(void) {
    // 1. Zero out the BSS section
    size_t bss_size = &__bss_end - &__bss;
    memset(&__bss, 0, bss_size);

    // 2. Set the trap vector (stvec) to our trap handler
    csr_write(stvec, (uint64_t)trap_handler);

    printf("\nBooting hypervisor...\n");

    // 3. Initialize the bump allocator with the heap region
    bump_allocator_init(&GLOBAL_ALLOCATOR, &__heap, &__heap_end);

    // 4. Load the Linux kernel image (embedded at compile time)
    const uint8_t *kernel_image = EMBEDDED_FILE("../linux/Image");
    GuestPageTable table = guest_page_table_new();
    load_linux_kernel(&table, kernel_image, sizeof(kernel_image));

    // 5. Create a virtual CPU targeting the kernel entry point
    VCpu vcpu = vcpu_new(&table, GUEST_BASE_ADDR);
    vcpu.a0 = 0;               // Hart ID = 0
    vcpu.a1 = GUEST_DTB_ADDR;  // Pointer to device tree blob

    // 6. Enter guest mode (never returns)
    vcpu_run(&vcpu);
}
```

### panic_handler

```c
// Called on any panic (assertion failure, unwrap on None, etc.)
__attribute__((noreturn))
void panic_handler(const char *info) {
    printf("panic: %s\n", info);
    while (1) {
        asm("wfi");  // Wait for interrupt (idle loop)
    }
}
```
