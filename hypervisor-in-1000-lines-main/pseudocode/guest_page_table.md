# Guest Page Table - Pseudocode

Implements the two-stage address translation page table (hgatp) used by the
RISC-V hypervisor extension. Maps guest-physical addresses to host-physical
addresses using Sv48x4 paging (4-level page table).

## Constants

```c
#define PTE_V          (1 << 0)   // Valid
#define PTE_R          (1 << 1)   // Readable
#define PTE_W          (1 << 2)   // Writable
#define PTE_X          (1 << 3)   // Executable
#define PTE_U          (1 << 4)   // User-accessible

#define PPN_SHIFT      12
#define PTE_PPN_SHIFT  10
```

## Data Structures

### Page Table Entry

```c
// A single 64-bit page table entry
struct Entry {
    uint64_t value;
};

// Create a new entry from a physical address and flags
Entry entry_new(uint64_t paddr, uint64_t flags) {
    uint64_t ppn = paddr >> PPN_SHIFT;
    return (Entry){ .value = (ppn << PTE_PPN_SHIFT) | flags };
}

// Check if the entry is valid (present)
bool entry_is_valid(Entry *e) {
    return (e->value & PTE_V) != 0;
}

// Extract the physical address from the entry
uint64_t entry_paddr(Entry *e) {
    return (e->value >> PTE_PPN_SHIFT) << PPN_SHIFT;
}
```

### Page Table (one level)

```c
// A page table is an array of 512 entries (4096 bytes total)
struct Table {
    Entry entries[512];
};

// Allocate a new zeroed page table
Table *table_alloc(void) {
    return (Table *)alloc_pages(sizeof(Table));  // 4096 bytes, page-aligned
}

// Get the entry for a given guest address at a given level
// Extracts the 9-bit index for that level from the address
Entry *table_entry_by_addr(Table *t, uint64_t guest_paddr, int level) {
    uint64_t index = (guest_paddr >> (12 + 9 * level)) & 0x1FF;
    return &t->entries[index];
}
```

### Guest Page Table (top-level structure)

```c
// Top-level guest page table object
struct GuestPageTable {
    Table *root;  // Pointer to the root (level-3) table
};
```

## Functions

### GuestPageTable::new

```c
// Create a new guest page table with an allocated root table
GuestPageTable guest_page_table_new(void) {
    return (GuestPageTable){ .root = table_alloc() };
}
```

### GuestPageTable::hgatp

```c
// Compute the hgatp CSR value for this page table
// Format: mode (Sv48x4 = 9) in bits[63:60], PPN of root table
uint64_t guest_page_table_hgatp(GuestPageTable *self) {
    return (9ULL << 60) | ((uint64_t)self->root >> PPN_SHIFT);
}
```

### GuestPageTable::map

```c
// Map a single 4KB guest-physical page to a host-physical page
void guest_page_table_map(GuestPageTable *self, uint64_t guest_paddr,
                          uint64_t host_paddr, uint64_t flags) {
    Table *table = self->root;

    // Walk levels 3 -> 2 -> 1 (intermediate levels)
    for (int level = 3; level >= 1; level--) {
        Entry *entry = table_entry_by_addr(table, guest_paddr, level);

        if (!entry_is_valid(entry)) {
            // Allocate a new intermediate table and link it
            Table *new_table = table_alloc();
            *entry = entry_new((uint64_t)new_table, PTE_V);
        }

        // Descend to the next level
        table = (Table *)entry_paddr(entry);
    }

    // At level 0: install the leaf entry
    Entry *leaf = table_entry_by_addr(table, guest_paddr, 0);
    assert(!entry_is_valid(leaf));  // "already mapped"
    *leaf = entry_new(host_paddr, flags | PTE_V | PTE_U);
}
```
