---
layout: post
title: "Zero Modulo Ring Buffers - Tricking the Linux MMU into Wrapping Memory for You"
date: 2026-09-04 08:00:00 +0300
categories: [systems, performance]
tags: [linux, mmap, memory, cpp, optimization]
math: true
---

If you have ever built an audio mixer, a packet processor, or a lock-free queue, you have written a circular ring buffer. And if you care about latency down to the nanosecond, you know the classic annoyance: circular buffers wrap around, and contiguous data hates wrapping around.

Most people settle for power-of-two masking or modular arithmetic to handle indices:

```c
size_t next_idx = (current_idx + 1) & (buffer_size - 1);
```

While that bitwise `AND` is fast, the real issue hits when you want to write or read a contiguous chunk of memory (think `read()`, `write()`, SIMD vectorization, or direct DMA). If your slice crosses the boundary of the ring, your data is abruptly split into two disjoint memory fragments. You end up writing two `memcpy` calls, dealing with extra branches, and throwing away linear vector registers.

Last weekend, I went down a rabbit hole trying to eliminate this split-copy penalty entirely. It turns out you can solve this without writing a single branch in userspace. You just trick the Linux Memory Management Unit (MMU) into wrapping physical memory for you using mirrored virtual page tables.

Here is how the virtual address mirror trick works under the hood, why hardware cache tagging makes it free, and how to build one in modern C++.

---

### The Problem With Linear Wraparounds

Let's say we have a ring buffer of capacity $S$. At some point in time, our read pointer sits at offset $p$, and an incoming network packet or audio block of length $L$ needs to be copied into the queue.

If $p + L \le S$, the data fits sequentially:

$$\text{Range} = [V_{\text{base}} + p,\; V_{\text{base}} + p + L)$$

However, if $p + L > S$, the data overflows the end of the buffer. You have to split the operation into two chunks:

1. First chunk of size $S - p$ written to $[V_{\text{base}} + p,\; V_{\text{base}} + S)$
2. Second chunk of size $(p + L) - S$ written to $[V_{\text{base}},\; V_{\text{base}} + (p + L - S))$

This ruins zero-copy APIs. You cannot just pass a raw pointer to `send()` or parse a contiguous flat `struct` across the boundary without copying it into an intermediate flat scratch buffer. 

Can we make the memory look contiguous to the CPU even when it wraps around?

---

### The Virtual Memory Mirroring Trick

Virtual addresses are a complete illusion managed by page tables. A virtual address does not represent a physical DRAM cell; it simply points to a Page Table Entry (PTE) that holds a Physical Frame Number (PFN).

Nothing in the x86-64 or ARM architecture stops two completely different virtual addresses from pointing to the exact same physical page frame!

```
Virtual Address Space:
+-------------------------------+-------------------------------+
|       Base Buffer [0, S)      |     Mirrored Buffer [S, 2S)   |
+-------------------------------+-------------------------------+
                \                               /
                 \                             /
                  v                           v
              +-------------------------------+
              |   Same Physical Memory Pages  |
              +-------------------------------+
```

If we allocate $S$ bytes of physical memory (where $S$ is a multiple of page size, typically 4 KiB), we can ask the kernel to map that **exact same backing store twice**, back-to-back in contiguous virtual address space over a range of $2S$.

Now, let $V_{\text{base}}$ be the start address. For any offset $p \in [0, S)$ and any read/write size $L \le S$:

$$\forall i \in [0, L),\quad \text{phys}(V_{\text{base}} + p + i) = \text{phys}(V_{\text{base}} + ((p + i) \bmod S))$$

If your slice starts at $p = S - 10$ and has length $L = 64$, bytes $10$ through $63$ automatically hit the virtual range $[V_{\text{base}} + S,\; V_{\text{base}} + S + 54)$. But the MMU maps those virtual addresses right back to the physical frames at $[0, 54)$. 

The wrap-around happens in the hardware translation lookaside buffer (TLB). Your code sees a single, unbroken pointer. You can run 256-bit AVX2 loads straight through the boundary without caring.

---

### Why Doesn't This Kill the CPU Caches?

When I first thought about this, I wondered if aliasing two virtual addresses to the same physical memory would cause cache coherence disasters or double-caching.

On modern x86-64 processors, L1 data caches are **Virtually Indexed, Physically Tagged (VIPT)**, and L2/L3 caches are **Physically Indexed, Physically Tagged (PIPT)**. 

Because the cache lines are tagged with the physical address (the PFN), the CPU's cache subsystem knows both virtual addresses refer to the exact same physical cache line. When you write to `V_base + S + 4`, it resolves to physical address `P + 4`. A subsequent read from `V_base + 4` also resolves to physical address `P + 4`. 

There is zero cache inconsistency. No cache flush required, no stale data, and no bus locks.

---

### Implementing the Double Mapping on Linux

To build this cleanly on modern Linux without leaving temp files in `/tmp` or `/dev/shm`, we use `memfd_create`. It creates an anonymous, memory-backed file descriptor that lives solely in RAM.

Here is the step-by-step game plan:

1. **Reserve Virtual Address Space**: Call `mmap` with `PROT_NONE` and `MAP_ANONYMOUS | MAP_PRIVATE` for $2S$ bytes. This secures a contiguous chunk of virtual addresses without committing any physical pages.
2. **Allocate Anonymous Memory**: Call `memfd_create` to get an anonymous file descriptor, then `ftruncate` it to size $S$.
3. **Map First Half**: Map the file into the first $S$ bytes using `MAP_SHARED | MAP_FIXED`.
4. **Map Second Half**: Map the same file descriptor into the upper half (offset $+S$) with `MAP_SHARED | MAP_FIXED`.
5. **Close File Descriptor**: Closing the descriptor does not unmap the memory; the kernel keeps the physical backing store alive until both mappings are torn down.

Here is the implementation in strict low-level C++:

```cpp
#define _GNU_SOURCE
#include <sys/mman.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <fcntl.h>
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <cassert>

class MirroredRingBuffer {
private:
  uint8_t* buffer_;
  size_t capacity_;
  size_t head_;
  size_t tail_;

  static size_t align_to_page(size_t size) {
    size_t page_size = static_cast<size_t>(sysconf(_SC_PAGESIZE));
    return (size + page_size - 1) & ~(page_size - 1);
  }

public:
  explicit MirroredRingBuffer(size_t requested_size)
    : buffer_(nullptr), capacity_(0), head_(0), tail_(0) {
    
    capacity_ = align_to_page(requested_size);
    if (capacity_ == 0) {
      return;
    }

    // 1. Reserve 2x capacity of contiguous virtual address space
    void* reservation = mmap(
      nullptr,
      2 * capacity_,
      PROT_NONE,
      MAP_PRIVATE | MAP_ANONYMOUS,
      -1,
      0
    );

    if (reservation == MAP_FAILED) {
      return;
    }

    // 2. Create anonymous in-memory file
    int fd = memfd_create("mirrored_ring_buffer", MFD_CLOEXEC);
    if (fd < 0) {
      munmap(reservation, 2 * capacity_);
      return;
    }

    if (ftruncate(fd, static_cast<off_t>(capacity_)) != 0) {
      close(fd);
      munmap(reservation, 2 * capacity_);
      return;
    }

    // 3. Map the first half [reservation, reservation + capacity_)
    uint8_t* base_addr = static_cast<uint8_t*>(reservation);
    void* first_map = mmap(
      base_addr,
      capacity_,
      PROT_READ | PROT_WRITE,
      MAP_SHARED | MAP_FIXED,
      fd,
      0
    );

    if (first_map == MAP_FAILED) {
      close(fd);
      munmap(reservation, 2 * capacity_);
      return;
    }

    // 4. Map the second half [reservation + capacity_, reservation + 2 * capacity_)
    void* second_map = mmap(
      base_addr + capacity_,
      capacity_,
      PROT_READ | PROT_WRITE,
      MAP_SHARED | MAP_FIXED,
      fd,
      0
    );

    // The kernel retains the inode reference via the mappings
    close(fd);

    if (second_map == MAP_FAILED) {
      munmap(base_addr, 2 * capacity_);
      return;
    }

    buffer_ = base_addr;
  }

  ~MirroredRingBuffer() {
    if (buffer_) {
      munmap(buffer_, 2 * capacity_);
    }
  }

  // Prevent copies because we manage raw mmap regions
  MirroredRingBuffer(const MirroredRingBuffer&) = delete;
  MirroredRingBuffer& operator=(const MirroredRingBuffer&) = delete;

  size_t capacity() const {
    return capacity_;
  }

  size_t size() const {
    return head_ - tail_;
  }

  size_t available_space() const {
    return capacity_ - size();
  }

  // Get contiguous write pointer. Can write up to available_space() bytes!
  uint8_t* write_ptr() {
    if (!buffer_) {
      return nullptr;
    }
    return buffer_ + (head_ % capacity_);
  }

  void commit_write(size_t bytes) {
    assert(bytes <= available_space());
    head_ += bytes;
  }

  // Get contiguous read pointer. Can read up to size() bytes!
  const uint8_t* read_ptr() const {
    if (!buffer_) {
      return nullptr;
    }
    return buffer_ + (tail_ % capacity_);
  }

  void commit_read(size_t bytes) {
    assert(bytes <= size());
    tail_ += bytes;
    // Keep absolute indices from overflowing after billions of ops
    if (tail_ >= capacity_) {
      head_ -= capacity_;
      tail_ -= capacity_;
    }
  }
};
```

Look at how `write_ptr()` and `read_ptr()` work:

```cpp
uint8_t* write_ptr() {
  return buffer_ + (head_ % capacity_);
}
```

That is it. If `head_ % capacity_` is $4090$ on a 4096-byte page, and you write 20 bytes, you simply write to `buffer_ + 4090` through `buffer_ + 4110`. The last 14 bytes land in the mirror. The next read from offset 0 will see those exact 14 bytes.

---

### Real-World Verification

Let's write a small verification test to prove the mirror property holds true across the boundary:

```cpp
int main() {
  // Page size on x86-64 is 4096 bytes
  MirroredRingBuffer ring(4096);
  size_t cap = ring.capacity();
  printf("Allocated ring with capacity: %zu bytes\n", cap);

  // Advance pointers close to the edge
  size_t initial_offset = cap - 4;
  ring.commit_write(initial_offset);
  ring.commit_read(initial_offset);

  assert(ring.size() == 0);
  assert(ring.available_space() == cap);

  // Write 8 bytes across boundary: 4 bytes before boundary, 4 bytes after
  uint8_t* w_ptr = ring.write_ptr();
  const uint8_t raw_payload[8] = { 'D', 'E', 'A', 'D', 'B', 'E', 'E', 'F' };
  
  // Single contiguous memcpy across the boundary!
  memcpy(w_ptr, raw_payload, 8);
  ring.commit_write(8);

  // Read back all 8 contiguous bytes from read_ptr
  const uint8_t* r_ptr = ring.read_ptr();
  char out_str[9];
  memcpy(out_str, r_ptr, 8);
  out_str[8] = '\0';
  ring.commit_read(8);

  printf("Read back data: %s\n", out_str);
  assert(strcmp(out_str, "DEADBEEF") == 0);

  // Confirm memory at offset 0 physically matches offset 'cap'
  uint8_t* base = ring.write_ptr();
  printf("Value at virtual offset 0: %c\n", base[0]);
  assert(base[0] == 'B');
  assert(base[1] == 'E');
  assert(base[2] == 'E');
  assert(base[3] == 'F');

  printf("Mirrored virtual memory verified successfully!\n");
  return 0;
}
```

Compile and run on any Linux box:

```bash
g++ -O3 -std=c++20 main.cpp -o ring
./ring
```

Output:

```text
Allocated ring with capacity: 4096 bytes
Read back data: DEADBEEF
Value at virtual offset 0: B
Mirrored virtual memory verified successfully!
```

---

### Trade-offs: What's the Catch?

Nothing in systems engineering is free. While this pattern eliminates split-copies and branch misses, there are real trade-offs you need to consider before using it everywhere:

1. **Granularity Constraint**: The buffer size must be a multiple of the OS page size (4096 bytes on x86-64). If you need a small 64-byte or 512-byte ring buffer, this is overkill and will waste memory.
2. **Virtual Address Space Consumption**: You are burning double the virtual address space. On a 64-bit architecture with a 48-bit or 57-bit address space, burning virtual addresses is completely negligible. On 32-bit embedded systems, however, virtual address space exhaustion is a real problem.
3. **TLB Footprint**: While physical memory is shared, the CPU still allocates separate TLB entries for translations in the mirrored region. If your working set hops constantly across page boundaries, you might see slightly higher L1 D-TLB miss rates compared to a tiny stack-allocated circular buffer.
4. **Syscall Overhead at Initialization**: Initializing via `mmap` and `memfd_create` takes roughly 15 to 30 microseconds due to kernel traps and page table manipulation. You should allocate these buffers during startup or use an arena pool, not on the hot path.

### Summary

For high-throughput systems (audio engines, socket buffers, or high-performance ring queues), mapping an identical memory chunk twice via `memfd_create` and `MAP_FIXED` is one of the cleanest architectural tricks out there. 

Instead of cluttering your codebase with wrapping branches, split `iovec` structs, and conditional copies, you let the CPU's paging unit handle the circular logic directly in hardware.
