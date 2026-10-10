---
layout: post
title: "Building a Custom Slab Allocator - Destroying Malloc Overhead"
date: 2026-10-10 08:00:00 +0300
categories: [Systems, Programming]
tags: [cpp, memory, optimization, gamedev, engine]
math: true
---

So I was profiling my custom voxel engine the other day, and noticed something super annoying. I was getting these weird micro-stutters every time I spawned a bunch of particles. I cracked open `perf` on Linux and saw that standard library `malloc` and `free` were eating up a terrifying chunk of my CPU cycles. 

It turns out that dynamically allocating tons of tiny objects every frame is just a really bad idea. The OS memory allocator is great for general-purpose stuff, but it has overhead. It has to do bookkeeping, handle thread synchronization, and fight memory fragmentation. 

I definitely didn't want to use standard library containers like `std::vector` here. Heap allocations are exactly what I'm trying to avoid, and a primitive array works just fine for fixed-size memory pools. So, I spent the weekend writing a custom slab allocator. Here is how I did it and why it's so much faster.

### The Problem with Malloc

When you call `new` or `malloc`, the system has to search for a free block of memory that fits your requested size. If your memory is highly fragmented, this search can take a while. 

We can represent the rough latency of standard allocation like this:

$$ \text{latency} \approx \text{syscall overhead} + \text{fragmentation search...} $$

Notice the `...` at the end? Yeah, it gets worse if the OS has to map new pages. We want to drop all of that to a guaranteed $$ O(1) $$ operation.

### The Slab Allocator Concept

A slab allocator pre-allocates a massive chunk of memory (a "slab") upfront and divides it into fixed-size blocks. When you need memory, the allocator just gives you a pointer to the next free block. When you free it, it puts the block back on an internal "free list".

Since the blocks are all the exact same size, there's zero fragmentation. And allocation is literally just popping a pointer off a stack. 

### The Implementation

Here is my implementation of a basic intrusive free list in C++. It's "intrusive" because we embed the `next` pointer directly inside the unallocated memory blocks. This means we don't even need extra memory for the bookkeeping!

```cpp
#include <iostream>
#include <cstdint>

// This struct acts as a node in our free list.
// We only use it when the memory block is NOT active.
struct Block {
  Block* next;
};

class SlabAllocator {
private:
  static const int MAX_BLOCKS = 4096;
  static const int BLOCK_SIZE = 32;

  // We strictly avoid std::vector here to prevent heap allocations.
  // Using a raw primitive array guarantees the memory is contiguous 
  // and sits directly inside the allocator instance.
  alignas(8) uint8_t memory_pool[MAX_BLOCKS * BLOCK_SIZE];
  Block* free_list_head;

public:
  SlabAllocator() {
    free_list_head = nullptr;

    // Initialize the free list by linking all blocks together.
    for (int i = 0; i < MAX_BLOCKS; ++i) {
      Block* block = reinterpret_cast<Block*>(&memory_pool[i * BLOCK_SIZE]);
      block->next = free_list_head;
      free_list_head = block;
    }
  }

  void* allocate() {
    if (free_list_head == nullptr) {
      // Out of memory! In a real engine, we'd allocate another slab.
      return nullptr; 
    }
    
    // Pop the head off the free list
    Block* block = free_list_head;
    free_list_head = block->next;
    
    return block;
  }

  void deallocate(void* ptr) {
    if (ptr == nullptr) {
      return;
    }

    // Cast the dead memory back into a block and push it onto the list
    Block* block = static_cast<Block*>(ptr);
    block->next = free_list_head;
    free_list_head = block;
  }
};
```

Notice how we're casting the raw `uint8_t` memory into `Block*`. Since the block isn't being used by the game engine yet, we can safely overwrite the first 8 bytes (on a 64-bit system) to store our `next` pointer. Once we hand the memory back to the application via `allocate()`, they can overwrite those 8 bytes with their own data.

### Why this is awesome

1. **Blazing Speed**: We are doing zero syscalls during `allocate` or `deallocate`. It's just a couple of pointer swaps.
2. **Cache Locality**: All our objects are packed tightly into a single contiguous array (`memory_pool`). The CPU prefetcher is going to love this because the data is perfectly predictable.

When I swapped my particle system to use this instead of standard `new`/`delete`, my frame rate stopped dropping completely, and the memory footprint stayed perfectly flat. 

If you're writing an engine, a high-frequency trading bot, or anything that needs raw speed, you basically *have* to write custom allocators. It took me a bit of debugging to get the pointer math right (segfaults are fun), but honestly, it was completely worth it.
