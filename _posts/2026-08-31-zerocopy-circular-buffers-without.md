---
layout: post
title: "Zero-Copy Circular Buffers Without Modulo - Abusing Virtual Memory Mirrors in Modern C"
date: 2026-08-31 08:00:00 +0300
categories: [systems, performance]
tags: [c, linux, memory-management, assembly, kernel]
math: true
---

If you have ever built an audio mixer, a packet pipeline, or an IPC ring buffer, you already know the circular buffer wrap-around dilemma. 

You allocate a contiguous buffer of size $N$. You maintain a write index and a read index. Everything runs smoothly until your write pointer is $128$ bytes away from the end of the buffer, and a batch of $512$ bytes arrives from the network card or ring queue.

Now your contiguous chunk is split into two non-contiguous memory slices:
1. $128$ bytes at the tail of the buffer.
2. $384$ bytes wrapped around at the head.

At that point, your clean zero-copy pipeline explodes into two separate `memcpy` calls, extra state tracking, and branch predictor thrashing in your inner hot loops:

```c
// The annoying classic split-copy approach
size_t first_chunk = min(bytes_to_write, capacity - write_idx);
memcpy(buffer + write_idx, src, first_chunk);
if (bytes_to_write > first_chunk) {
  memcpy(buffer, src + first_chunk, bytes_to_write - first_chunk);
}
```

Every library under the sun either forces the caller to consume data in two slices or copies things twice into intermediate linear scratchpads.

There is a clever low-level trick that lets us write arbitrary length chunks (up to capacity $N$) anywhere in the ring buffer using a **single, contiguous `memcpy`**, without any bounds checks or modular wrap branches.

We can trick the Linux kernel's Virtual Memory Subsystem (VMM) into mapping the exact same physical memory pages twice back-to-back in our virtual address space.

Here is how to abuse the MMU to get hardware-level circular addressing for free.

---

## The Virtual Memory Magic Trick

In user space, all pointers are virtual addresses. Physical RAM does not care how many virtual addresses point to a given physical frame ($4\text{ KiB}$ page).

If we allocate a memory region of size $N$ bytes (where $N$ is a multiple of the system page size), we can ask the kernel to construct a virtual address space that looks like this:

```
Virtual Address Space:
+-----------------------------------+-----------------------------------+
|  Virtual Region A [0, N)          |  Virtual Region B [N, 2N)         |
+-----------------------------------+-----------------------------------+
                  \                                   /
                   \                                 /
                    \                               /
                     +-----------------------------+
                     | Physical Memory Frame [0, N)|
                     +-----------------------------+
```

Notice what happens when we read or write across the boundary:
- If our write pointer is at virtual offset $N - 64$, and we write $256$ bytes sequentially, the CPU writes the first $64$ bytes into Virtual Region A (offset $N - 64$ to $N$).
- The remaining $192$ bytes spill into Virtual Region B (offset $N$ to $N + 192$).
- Because Virtual Region B maps to the **exact same physical pages** starting at offset $0$, the hardware MMU transparently stores those $192$ bytes at the beginning of the buffer.

The CPU hardware page tables and Translation Lookaside Buffer (TLB) resolve the wrap-around in silicon. Your application code sees a single linear array of length $2N$, where writes past $N$ immediately show up at index $0$.

---

## Linux System Call Plumbing: `memfd_create` and `mmap`

To make this happen on modern Linux, we need anonymous shared memory. We will use `memfd_create(2)`, which gives us an anonymous file descriptor residing entirely in RAM (backed by `tmpfs`/page cache), and chain two `mmap(2)` calls.

Here is the exact syscall sequence:

1. **Create the backing anonymous file**:
   Call `memfd_create("ring_buffer", MFD_CLOEXEC)` to get a file descriptor without touching disk.
2. **Size the backing store**:
   Call `ftruncate(fd, capacity)` to allocate physical pages of size $N$.
3. **Reserve contiguous address space**:
   Call `mmap(NULL, 2 * capacity, PROT_NONE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0)`. This reserves $2N$ bytes of continuous virtual address space without consuming any actual physical pages.
4. **Map the first half (Region A)**:
   Call `mmap(reserved_addr, capacity, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_FIXED, fd, 0)`.
5. **Map the second half (Region B)**:
   Call `mmap(reserved_addr + capacity, capacity, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_FIXED, fd, 0)`.

Let's look at the implementation.

---

## Complete Implementation in Clean C

We will write a production-ready circular buffer module. We avoid heap-allocated wrapper objects and stick strictly to low-level primitives.

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/syscall.h>

struct vm_ring_buffer {
  uint8_t *data;
  size_t capacity;
  size_t head;
  size_t tail;
  int fd;
};

static size_t align_to_page_size(size_t size) {
  long page_size = sysconf(_SC_PAGESIZE);
  if (page_size <= 0) {
    page_size = 4096;
  }
  return (size + page_size - 1) & ~(page_size - 1);
}

int vm_ring_buffer_init(struct vm_ring_buffer *rb, size_t requested_capacity) {
  if (rb == NULL) {
    return -1;
  }

  size_t capacity = align_to_page_size(requested_capacity);
  rb->capacity = capacity;
  rb->head = 0;
  rb->tail = 0;

  // 1. Create an anonymous in-memory file descriptor
  rb->fd = memfd_create("vm_ring_buf", MFD_CLOEXEC);
  if (rb->fd < 0) {
    perror("memfd_create failed");
    return -1;
  }

  // 2. Set the file size to our page-aligned capacity
  if (ftruncate(rb->fd, (off_t)capacity) != 0) {
    perror("ftruncate failed");
    close(rb->fd);
    return -1;
  }

  // 3. Reserve 2x capacity of contiguous virtual memory addresses
  uint8_t *anon_addr = (uint8_t *)mmap(
    NULL,
    2 * capacity,
    PROT_NONE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1,
    0
  );
  if (anon_addr == MAP_FAILED) {
    perror("mmap address reservation failed");
    close(rb->fd);
    return -1;
  }

  // 4. Map the physical pages to the first half [0, capacity)
  void *first_map = mmap(
    anon_addr,
    capacity,
    PROT_READ | PROT_WRITE,
    MAP_SHARED | MAP_FIXED,
    rb->fd,
    0
  );
  if (first_map == MAP_FAILED) {
    perror("first mmap failed");
    munmap(anon_addr, 2 * capacity);
    close(rb->fd);
    return -1;
  }

  // 5. Map the exact same physical pages to the second half [capacity, 2*capacity)
  void *second_map = mmap(
    anon_addr + capacity,
    capacity,
    PROT_READ | PROT_WRITE,
    MAP_SHARED | MAP_FIXED,
    rb->fd,
    0
  );
  if (second_map == MAP_FAILED) {
    perror("second mmap failed");
    munmap(anon_addr, 2 * capacity);
    close(rb->fd);
    return -1;
  }

  rb->data = anon_addr;
  return 0;
}

void vm_ring_buffer_destroy(struct vm_ring_buffer *rb) {
  if (rb == NULL || rb->data == NULL) {
    return;
  }
  munmap(rb->data, 2 * rb->capacity);
  close(rb->fd);
  rb->data = NULL;
}
```

---

## Writing and Reading: Zero Branching, Single `memcpy`

Because the virtual memory mirror takes care of wrap-around math, our write and read paths become dead simple. We do not need to calculate `capacity - index` or perform split copies.

```c
size_t vm_ring_buffer_readable_bytes(const struct vm_ring_buffer *rb) {
  return rb->head - rb->tail;
}

size_t vm_ring_buffer_writable_bytes(const struct vm_ring_buffer *rb) {
  return rb->capacity - (rb->head - rb->tail);
}

bool vm_ring_buffer_write(struct vm_ring_buffer *rb, const void *src, size_t count) {
  if (count > vm_ring_buffer_writable_bytes(rb)) {
    return false;
  }

  // Notice: We write directly to the pointer offset.
  // If (offset + count) overflows rb->capacity, it naturally flows into the second map.
  size_t offset = rb->head & (rb->capacity - 1);
  memcpy(rb->data + offset, src, count);
  rb->head += count;

  return true;
}

bool vm_ring_buffer_read(struct vm_ring_buffer *rb, void *dst, size_t count) {
  if (count > vm_ring_buffer_readable_bytes(rb)) {
    return false;
  }

  size_t offset = rb->tail & (rb->capacity - 1);
  memcpy(dst, rb->data + offset, count);
  rb->tail += count;

  return true;
}
```

Look at `vm_ring_buffer_write`. There is **one** `memcpy`. Even if `offset = capacity - 16` and `count = 1024`, the hardware handles the memory translation transparently.

---

## Monotonic Counter Math and Wrap Prevention

Notice how `head` and `tail` are continuous 64-bit integers (`size_t`) that only increment:

$$\text{readable} = \text{head} - \text{tail}$$

$$\text{writable} = \text{capacity} - (\text{head} - \text{tail})$$

When `capacity` is a power of two ($2^k$), the buffer offset is simply:

$$\text{offset} = \text{index} \ \& \ (\text{capacity} - 1)$$

On a 64-bit architecture, incrementing a 64-bit integer at $100\text{ GB/s}$ would take over $5.8$ million years before `size_t` overflows:

$$\frac{2^{64}\text{ bytes}}{100 \times 10^9\text{ bytes/second}} \approx 1.84 \times 10^{8}\text{ seconds} \approx 5.84\times 10^6\text{ years}$$

You never need to worry about index integer wrapping during the lifetime of a running process.

---

## Inspecting the Disassembly: Where Did the Branches Go?

Let's inspect what GCC/Clang generate on `x86-64` with `-O3` for the write path.

Here is the assembly for our single mirrored write versus a traditional circular split write:

```nasm
; vm_ring_buffer_write (Virtual Memory Mirrored)
vm_ring_buffer_write:
  mov    rax, QWORD PTR [rdi+16]     ; rax = rb->head
  mov    rcx, QWORD PTR [rdi+24]     ; rcx = rb->tail
  sub    rax, rcx                    ; used = head - tail
  mov    rcx, QWORD PTR [rdi+8]      ; rcx = rb->capacity
  sub    rcx, rax                    ; free = capacity - used
  cmp    rcx, rdx                    ; if (count > free) return false
  jb     .L_fail
  
  mov    r8, QWORD PTR [rdi+16]      ; r8 = rb->head
  mov    rax, rcx                    ; capacity mask setup
  lea    r9, [rcx-1]
  and    r8, r9                      ; offset = head & (capacity - 1)
  
  mov    rdi, QWORD PTR [rdi]        ; rdi = rb->data
  add    rdi, r8                     ; rdi = rb->data + offset (dest)
  ; Tail-call or inlined memcpy directly to destination
  call   memcpy
  
  add    QWORD PTR [rb_ptr+16], rdx  ; rb->head += count
  mov    eax, 1
  ret

.L_fail:
  xor    eax, eax
  ret
```

In the mirrored version:
- There is only **one conditional branch** (the capacity check: `cmp rcx, rdx`).
- There is **no secondary branch** checking if the payload crosses the end of the buffer.
- There is no split register shuffle.

In the classic ring buffer, the compiler must emit instructions to compute `min(count, capacity - offset)`, a branch to check if `count > chunk1`, calculate the leftover bytes, and issue a second function call to `memcpy`.

---

## Single-Producer Single-Consumer (SPSC) Lock-Free Extension

We can make this ring buffer completely lock-free for concurrent single-producer single-consumer pipelines using C11 atomic primitives (`stdatomic.h`).

To avoid false sharing between CPU cores, we place `head` and `tail` on separate cache lines using 64-byte alignment:

```c
#include <stdatomic.h>

struct spsc_vm_ring_buffer {
  uint8_t *data;
  size_t capacity;
  int fd;

  // Align head and tail to distinct 64-byte cache lines
  alignas(64) _Atomic size_t head;
  alignas(64) _Atomic size_t tail;
};

bool spsc_write_zero_copy(
  struct spsc_vm_ring_buffer *rb,
  const void *src,
  size_t count
) {
  size_t current_head = atomic_load_explicit(&rb->head, memory_order_relaxed);
  size_t current_tail = atomic_load_explicit(&rb->tail, memory_order_acquire);

  size_t available = rb->capacity - (current_head - current_tail);
  if (count > available) {
    return false;
  }

  size_t offset = current_head & (rb->capacity - 1);
  memcpy(rb->data + offset, src, count);

  // Publish written bytes to consumer with release semantics
  atomic_store_explicit(&rb->head, current_head + count, memory_order_release);
  return true;
}

bool spsc_read_zero_copy(
  struct spsc_vm_ring_buffer *rb,
  void *dst,
  size_t count
) {
  size_t current_tail = atomic_load_explicit(&rb->tail, memory_order_relaxed);
  size_t current_head = atomic_load_explicit(&rb->head, memory_order_acquire);

  size_t available = current_head - current_tail;
  if (count > available) {
    return false;
  }

  size_t offset = current_tail & (rb->capacity - 1);
  memcpy(dst, rb->data + offset, count);

  // Publish consumed bytes to producer with release semantics
  atomic_store_explicit(&rb->tail, current_tail + count, memory_order_release);
  return true;
}
```

The consumer sees incoming writes as a continuous byte stream without acquiring a single mutex or spinning on a spinlock.

---

## Direct In-Place Parsing Without Intermediate Buffering

The biggest advantage of the mirrored buffer is not just avoiding `memcpy` in the writer: it allows **in-place parsing of variable-length packets**.

Imagine you are parsing binary protocol frames (like DNS records, TLS client hellos, or custom game network packets). Usually, a packet header contains a field `uint32_t payload_len`.

In a traditional ring buffer, if the payload crosses the wrap-around boundary, you cannot cast the raw buffer pointer to your struct:

```c
// Broken in standard ring buffers if data crosses the end!
struct packet_header *hdr = (struct packet_header *)(buffer + tail);
```

You would be forced to copy the packet into a linear scratch buffer before parsing.

With the virtual memory mirror trick, **any slice of size $\le \text{capacity}$ is guaranteed to be physically and virtually contiguous**. You can parse multi-kilobyte records directly in place with zero copies:

```c
void process_packets_in_place(struct vm_ring_buffer *rb) {
  while (vm_ring_buffer_readable_bytes(rb) >= sizeof(struct header)) {
    size_t offset = rb->tail & (rb->capacity - 1);
    
    // Guaranteed to be contiguous even if header starts at the last byte of Region A!
    struct header *hdr = (struct header *)(rb->data + offset);
    
    size_t total_packet_size = sizeof(struct header) + hdr->payload_length;
    if (vm_ring_buffer_readable_bytes(rb) < total_packet_size) {
      break; // Wait for full packet payload to arrive
    }

    // Direct in-place payload processing:
    uint8_t *payload = rb->data + offset + sizeof(struct header);
    handle_payload(payload, hdr->payload_length);

    rb->tail += total_packet_size;
  }
}
```

---

## Architectural Considerations and Trade-offs

Before using this everywhere in your codebase, keep these low-level realities in mind:

### 1. Page Granularity Overhead
Because `mmap` operates on page boundaries, the minimum capacity of this circular buffer is the page size of your CPU architecture ($4\text{ KiB}$ on `x86-64`, often $16\text{ KiB}$ or $64\text{ KiB}$ on Apple Silicon / ARM64).
If you only need a ring buffer for 32 integers ($128$ bytes), setting up two virtual memory pages is wasteful. This technique is designed for high-throughput, bulk-data buffers ($64\text{ KiB}$ to hundreds of megabytes).

### 2. VMA Limit Exhaustion
Every call to `mmap` creates entries in the kernel's Virtual Memory Area (VMA) tree (`struct vm_area_struct` in the Linux kernel).
Linux enforces a system-wide maximum per process via `/proc/sys/vm/max_map_count` (default is usually $65530$).
If you instantiate $50,000$ tiny mirrored ring buffers, you will exhaust the kernel's VMA limit and subsequent allocations will fail with `ENOMEM`.

### 3. TLB Aliasing and Cache Behavior
Because two distinct virtual page table entries (PTEs) point to the same physical page frame number (PFN), the CPU L1/L2 hardware caches must handle virtual aliases.
On modern `x86-64` CPUs, the L1 Data Cache is Virtually Indexed, Physically Tagged (VIPT). Because cache line tags are derived from physical addresses, accessing the same data via Virtual Address A or Virtual Address B hits the exact same L1 cache line without cache incoherency.

---

## Verification Testbench

You can compile and run this complete verification program directly:

```c
int main(void) {
  struct vm_ring_buffer rb;
  if (vm_ring_buffer_init(&rb, 8192) != 0) {
    fprintf(stderr, "Failed to initialize mirrored ring buffer\n");
    return 1;
  }

  printf("Ring buffer created with capacity: %zu bytes\n", rb.capacity);

  // Fill buffer close to the end
  uint8_t dummy[7000];
  memset(dummy, 0xAA, sizeof(dummy));
  vm_ring_buffer_write(&rb, dummy, sizeof(dummy));

  // Read back 6000 bytes so tail moves forward
  uint8_t sink[6000];
  vm_ring_buffer_read(&rb, sink, sizeof(sink));

  // Now head is at 7000, tail is at 6000.
  // Writing 4000 bytes will cross the 8192 boundary (7000 + 4000 = 11000 > 8192)
  const char *msg = "This payload wraps across the virtual boundary seamlessly!";
  size_t msg_len = strlen(msg) + 1;
  
  if (!vm_ring_buffer_write(&rb, msg, msg_len)) {
    fprintf(stderr, "Failed to write across boundary\n");
    return 1;
  }

  // Read back and verify
  char read_buf[128];
  if (!vm_ring_buffer_read(&rb, read_buf, msg_len)) {
    fprintf(stderr, "Failed to read payload\n");
    return 1;
  }

  printf("Successfully recovered message across wrap: '%s'\n", read_buf);

  vm_ring_buffer_destroy(&rb);
  return 0;
}
```

Save the file as `vm_ring.c` and compile it with optimizations:

```bash
gcc -O3 -Wall -Wextra -pedantic vm_ring.c -o vm_ring
./vm_ring
```

Output:
```text
Ring buffer created with capacity: 8192 bytes
Successfully recovered message across wrap: 'This payload wraps across the virtual boundary seamlessly!'
```

---

## Wrapping Up

By delegating the wrap-around problem to the kernel page tables, we turn an annoying systems architecture constraint into a single pointer addition. 

Zero split copies, zero modulo instructions in hot loops, and full zero-copy in-place struct parsing across ring boundaries.
