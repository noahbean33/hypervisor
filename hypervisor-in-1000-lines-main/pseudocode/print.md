# Print - Pseudocode

Serial output via SBI (Supervisor Binary Interface) console putchar extension.
The hypervisor itself runs in HS-mode and uses SBI calls to OpenSBI (M-mode)
to print characters to the UART.

## Functions

### sbi_putchar

```c
// Print a single character by calling SBI Console Putchar (extension ID=1)
// This triggers an ecall from HS-mode to M-mode (OpenSBI handles it)
void sbi_putchar(uint8_t ch) {
    register uint64_t a0 asm("a0") = ch;   // Argument: character
    register uint64_t a6 asm("a6") = 0;    // Function ID = 0
    register uint64_t a7 asm("a7") = 1;    // Extension ID = 1 (Console Putchar)
    asm volatile("ecall"
        : "+r"(a0)
        : "r"(a6), "r"(a7)
        : "a1");
}
```

### println (macro)

```c
// Print a formatted string followed by a newline
// Each byte of the formatted output is sent via sbi_putchar
#define println(fmt, ...) do {                          \
    char buf[256];                                      \
    int len = snprintf(buf, sizeof(buf), fmt "\n", ##__VA_ARGS__); \
    for (int i = 0; i < len; i++) {                    \
        sbi_putchar((uint8_t)buf[i]);                  \
    }                                                  \
} while(0)
```
