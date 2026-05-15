# Allocator - Pseudocode

A simple bump allocator that allocates memory linearly without freeing.
Used as the global memory allocator for the hypervisor.

## Data Structures

```c
// Internal mutable state protected by a spinlock
struct Mutable {
    uintptr_t next;  // Next available address
    uintptr_t end;   // End of allocatable region
};

// Bump allocator: allocates forward, never frees
struct BumpAllocator {
    spinlock_t lock;
    Mutable *state;  // NULL until initialized
};
```

## Global Instance

```c
// Single global allocator instance
static BumpAllocator GLOBAL_ALLOCATOR = { .state = NULL };
```

## Functions

### BumpAllocator::init

```c
// Initialize the allocator with a memory region [start, end)
void bump_allocator_init(BumpAllocator *self, void *start, void *end) {
    lock(&self->lock);
    self->state = &(Mutable){
        .next = (uintptr_t)start,
        .end  = (uintptr_t)end
    };
    unlock(&self->lock);
}
```

### BumpAllocator::alloc

```c
// Allocate memory with given size and alignment
void *bump_allocator_alloc(BumpAllocator *self, size_t size, size_t align) {
    lock(&self->lock);
    assert(self->state != NULL);  // "allocator not initialized"

    // Align the next pointer up to required alignment
    uintptr_t addr = align_up(self->state->next, align);

    // Check we have enough space
    assert(addr + size <= self->state->end);  // "out of memory"

    // Bump the pointer forward
    self->state->next = addr + size;

    unlock(&self->lock);
    return (void *)addr;
}
```

### BumpAllocator::dealloc

```c
// Deallocation is a no-op in a bump allocator
void bump_allocator_dealloc(BumpAllocator *self, void *ptr, size_t size) {
    // intentionally empty - memory is never freed
}
```

### alloc_pages

```c
// Allocate zeroed memory aligned to 4096-byte (page) boundary
void *alloc_pages(size_t len) {
    void *ptr = bump_allocator_alloc(&GLOBAL_ALLOCATOR, len, 4096);
    memset(ptr, 0, len);
    return ptr;
}
```
