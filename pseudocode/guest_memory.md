# Guest Memory - Pseudocode

Statically allocated memory buffers that represent the guest's physical memory.
Each buffer is page-aligned and maps into the guest's address space via the
guest page table.

## Constants

```c
#define MEMORY_SIZE      (64 * 1024 * 1024)  // 64 MB
#define GUEST_BASE_ADDR  0x80000000
#define GUEST_DTB_ADDR   0x70000000
```

## Data Structures

```c
// A page-aligned memory region that backs guest physical memory
// SIZE is the compile-time fixed size of this region
struct GuestMemory {
    uint8_t  data[SIZE];       // Page-aligned (4096) backing storage
    uint64_t guest_base;       // Guest physical base address this maps to
};
```

## Global Instances

```c
// Main guest memory: 64MB starting at guest address 0x80000000
static GuestMemory GUEST_MEMORY = {
    .data = {0},
    .guest_base = GUEST_BASE_ADDR
};

// Device tree blob memory: 64KB starting at guest address 0x70000000
static GuestMemory DTB_MEMORY = {
    .data = {0},
    .guest_base = GUEST_DTB_ADDR
};
```

## Functions

### GuestMemory::write_bytes

```c
// Copy source data into the guest memory buffer and map all pages
// into the guest page table with the given permission flags.
void guest_memory_write_bytes(GuestMemory *self, GuestPageTable *table,
                              const uint8_t *src, size_t src_len, uint64_t flags) {
    // Copy source bytes into the backing buffer
    memcpy(self->data, src, src_len);

    // Map every 4KB page of this region into the guest page table
    for (uint64_t offset = 0; offset < SIZE; offset += 4096) {
        uint64_t guest_addr = self->guest_base + offset;
        uint64_t host_addr  = (uint64_t)&self->data[0] + offset;
        guest_page_table_map(table, guest_addr, host_addr, flags);
    }
}
```
