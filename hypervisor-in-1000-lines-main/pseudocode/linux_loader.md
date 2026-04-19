# Linux Loader - Pseudocode

Loads a Linux kernel image into guest memory and constructs the Flattened
Device Tree (FDT/DTB) that describes the virtual hardware to the guest OS.

## Constants

```c
#define GUEST_DTB_ADDR   0x70000000
#define GUEST_BASE_ADDR  0x80000000
#define PLIC_ADDR        0x0C000000
#define PLIC_END         (PLIC_ADDR + 0x400000)
#define MEMORY_SIZE      (64 * 1024 * 1024)  // 64 MB
```

## Data Structures

```c
// RISC-V Linux kernel image header (at the start of the Image file)
struct RiscvImageHeader {
    uint32_t code0;          // Executable code
    uint32_t code1;          // Executable code
    uint64_t text_offset;    // Image load offset
    uint64_t image_size;     // Effective image size
    uint64_t flags;          // Kernel flags
    uint32_t version;        // Version of this header
    uint32_t reserved1;
    uint64_t reserved2;
    uint64_t magic;          // Magic number
    uint32_t magic2;         // Magic number 2 (0x05435352 = "RSC\x05")
    uint32_t reserved3;
};
```

## Functions

### load_linux_kernel

```c
// Load a Linux kernel image into guest memory and set up the device tree
void load_linux_kernel(GuestPageTable *table, const uint8_t *image, size_t image_len) {
    // Validate header
    assert(image_len >= sizeof(RiscvImageHeader));
    RiscvImageHeader *header = (RiscvImageHeader *)image;
    assert(le32_to_cpu(header->magic2) == 0x05435352);  // "invalid magic"

    uint64_t kernel_size = le64_to_cpu(header->image_size);
    assert(image_len <= MEMORY_SIZE);

    // Copy kernel image into guest memory and map it as RWX
    guest_memory_write_bytes(&GUEST_MEMORY, table, image, image_len,
                             PTE_R | PTE_W | PTE_X);

    // Build and load the device tree into DTB memory (read-only)
    uint8_t *dtb = build_device_tree(&dtb_len);
    guest_memory_write_bytes(&DTB_MEMORY, table, dtb, dtb_len, PTE_R);

    printf("loaded kernel: size=%lluKB\n", kernel_size / 1024);
}
```

### build_device_tree

```c
// Construct a Flattened Device Tree describing the virtual machine
uint8_t *build_device_tree(size_t *out_len) {
    FdtWriter fdt;
    fdt_writer_init(&fdt);

    // Root node
    fdt_begin_node(&fdt, "");
    fdt_property_string(&fdt, "compatible", "riscv-virtio");
    fdt_property_u32(&fdt, "#address-cells", 2);
    fdt_property_u32(&fdt, "#size-cells", 2);

    // /chosen - boot arguments
    fdt_begin_node(&fdt, "chosen");
    fdt_property_string(&fdt, "bootargs",
                        "console=hvc earlycon=sbi panic=-1 root=/dev/vda");
    fdt_end_node(&fdt);

    // /memory@80000000 - RAM description
    fdt_begin_node(&fdt, "memory@80000000");
    fdt_property_string(&fdt, "device_type", "memory");
    fdt_property_u64_array(&fdt, "reg", (uint64_t[]){GUEST_BASE_ADDR, MEMORY_SIZE}, 2);
    fdt_end_node(&fdt);

    // /cpus
    fdt_begin_node(&fdt, "cpus");
    fdt_property_u32(&fdt, "#address-cells", 1);
    fdt_property_u32(&fdt, "#size-cells", 0);
    fdt_property_u32(&fdt, "timebase-frequency", 10000000);

    // /cpus/cpu@0
    fdt_begin_node(&fdt, "cpu@0");
    fdt_property_string(&fdt, "device_type", "cpu");
    fdt_property_string(&fdt, "compatible", "riscv");
    fdt_property_u32(&fdt, "reg", 0);
    fdt_property_string(&fdt, "status", "okay");
    fdt_property_string(&fdt, "mmu-type", "riscv,sv48");
    fdt_property_string(&fdt, "riscv,isa", "rv64imafdc");

    // /cpus/cpu@0/interrupt-controller
    fdt_begin_node(&fdt, "interrupt-controller");
    fdt_property_u32(&fdt, "#interrupt-cells", 1);
    fdt_property_null(&fdt, "interrupt-controller");
    fdt_property_string(&fdt, "compatible", "riscv,cpu-intc");
    fdt_property_phandle(&fdt, 1);
    fdt_end_node(&fdt);  // interrupt-controller

    fdt_end_node(&fdt);  // cpu@0
    fdt_end_node(&fdt);  // cpus

    // /plic@c000000 - Platform-Level Interrupt Controller
    fdt_begin_node(&fdt, "plic@c000000");
    fdt_property_string(&fdt, "compatible", "riscv,plic0");
    fdt_property_u32(&fdt, "#interrupt-cells", 1);
    fdt_property_null(&fdt, "interrupt-controller");
    fdt_property_u64_array(&fdt, "reg", (uint64_t[]){PLIC_ADDR, 0x4000000}, 2);
    fdt_property_u32(&fdt, "riscv,ndev", 3);
    fdt_property_u32_array(&fdt, "interrupts-extended", (uint32_t[]){1, 11, 1, 9}, 4);
    fdt_property_phandle(&fdt, 2);
    fdt_end_node(&fdt);  // plic

    fdt_end_node(&fdt);  // root

    return fdt_finish(&fdt, out_len);
}
```
