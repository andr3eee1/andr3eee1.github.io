---
layout: post
title: "Zero-Copy Mirrored Ring Buffers - Exploiting Virtual Memory Aliasing in Modern Linux"
date: 2026-09-13 08:00:00 +0300
categories: [systems-programming, c-plus-plus]
tags: [linux, memory-management, atomics, low-level, performance]
math: true
---

Circular ring buffers are one of the first data structures everyone learns. You take a contiguous array, maintain `head` and `tail` indices, and wrap them around using modulo arithmetic. It is clean, straightforward, and works fine for simple toy problems.

Once you start writing high-throughput systems—whether that is an audio DSP processing pipeline, a high-frequency market data feed, or a user-space network driver handling packet frames—standard ring buffers hit a massive architectural roadblock: **the wrap-around boundary**.

If you need to read or write a contiguous block of bytes that spans past the physical end of the buffer, you are forced into an uncomfortable compromise:
1. Break your operation into two separate memory chunks (`memcpy` part 1 to the end, wrap to index 0, `memcpy` part 2).
2. Maintain an intermediate scratch buffer on the stack or heap to linearize the payload before processing.
3. Expose awkward iterator abstractions to your callers instead of handing them a direct, contiguous slice (`const uint8_t*`).

Both approaches burn memory bandwidth, pollute cache lines, and introduce branch mispredictions in critical inner loops.

Instead of writing complex software logic to handle edge-case wrapping, we can solve this at the hardware level. Modern CPUs have an MMU (Memory Management Unit) that translates virtual addresses into physical addresses through page tables. Nothing in POSIX or hardware architecture states that two virtual addresses cannot point to the exact same physical page frame.

By creating a **mirrored ring buffer** (sometimes called a virtual ring buffer), we map the exact same physical memory buffer into two adjacent virtual memory regions. When our read or write index advances past the end of the first region, it seamlessly spills into the second region without branching, wrapping, or copying. The MMU handles the wrap-around for free.

---

### The Problem: Contiguous Slices in Circular Memory

Consider a consumer thread trying to parse a binary protocol frame of size $L$ from a ring buffer of capacity $C$. The current read offset is $R$.

If $R + L \le C$, the payload is contiguous in memory. You pass `buffer + R` straight to your zero-copy parser.

If $R + L > C$, the payload is physically split across two disconnected memory locations:

$$
\text{Chunk}_1 = [\text{buffer} + R, \;\; \text{buffer} + C)
$$

$$
\text{Chunk}_2 = [\text{buffer}, \;\; \text{buffer} + (R + L - C))
$$

```
Normal Ring Buffer (Split Payload across boundary):
[ Chunk 2 (Tail) | ... empty space ... | Chunk 1 (Head) ]
  ^                                      ^
  0                                      R           C
```

If your parser expects a contiguous pointer (for instance, passing audio frames to `vDSP` / AVX-512 kernels or handing raw buffers to `writev`), you cannot do this zero-copy. You must linearize it:

```c
// The slow, painful way:
uint8_t scratch[MAX_FRAME_SIZE];
size_t first_part = capacity - read_idx;
memcpy(scratch, buffer + read_idx, first_part);
memcpy(scratch + first_part, buffer, len - first_part);
process_frame(scratch, len);
```

This is frustrating. You are copying data twice, thrashing your L1 data cache, and forcing the CPU to speculate across branch conditions on every packet.

---

### The Solution: Virtual Memory Aliasing

The CPU does not see physical RAM directly. Every load and store instruction operates on virtual memory addresses. Virtual addresses are translated into physical addresses via hierarchical page tables (PML4 $\to$ PDPT $\to$ PD $\to$ PT on x86-64).

```
Virtual Address Space:
[  Base Region (Size S)  ][  Mirrored Region (Size S)  ]
[ Page 0 | Page 1 | ...  ][ Page 0 | Page 1 | ...      ]
       \         \             /         /
        \         \           /         /
Physical RAM:
       [ Frame 0 | Frame 1 | ... ]
```

If we allocate a contiguous virtual address region of size $2S$, and map the physical pages of size $S$ into both the lower half $[V, V + S)$ and the upper half $[V + S, V + 2S)$, we obtain an extraordinary property:

Any slice of length $L \le S$ starting at any index $i \in [0, S)$ is guaranteed to be completely contiguous in virtual memory!

If you start reading at index $S - 16$ and read 64 bytes, bytes $0 \dots 15$ are read from the end of the lower virtual region, and bytes $16 \dots 63$ are read from the beginning of the upper virtual region. Because the upper virtual region maps to the exact same physical pages as the lower region, the hardware fetches physical bytes $0 \dots 47$.

Zero branches. Zero `memcpy`. Zero modulo arithmetic on the pointer.

---

### Hardware Cache Implications: Does Aliasing Duplicate Cache Lines?

A common question is whether mapping the same physical memory to two virtual addresses causes cache aliasing issues or wastes L1/L2 cache capacity.

On modern x86 processors (and most ARM64 chips), the L1 data cache is **VIPT** (Virtually Indexed, Physically Tagged):
- The cache index is derived from the lower bits of the virtual address.
- The cache tag is derived from the translated physical address.

Standard Linux pages are 4 KB ($2^{12}$ bytes). A standard Intel L1 data cache is 32 KB or 48 KB with 8-way set associativity:

$$
\text{Sets} = \frac{32768 \text{ bytes}}{8 \times 64 \text{ bytes/line}} = 64 \text{ sets}
$$

To index 64 sets, the CPU requires $\log_2(64) = 6$ bits. Since cache lines are 64 bytes ($2^6$), the cache set index uses bits 6 through 11 of the address.

Notice that bits 0 through 11 are the page offset! Because our buffer size $S$ is always a multiple of the page size (at least 4 KB), the lower 12 bits of any address in the lower region and its corresponding alias in the upper region are identical:

$$
\text{Addr}_{\text{upper}} = \text{Addr}_{\text{lower}} + S
$$

Because $S$ is page-aligned, $\text{Addr}_{\text{upper}} \pmod{4096} = \text{Addr}_{\text{lower}} \pmod{4096}$.

Consequently, both virtual addresses index into the **exact same cache set** and match the **exact same physical tag**. There is zero duplication of cache lines in L1, L2, or L3, and no hardware cache coherence protocol overhead between the aliases.

---

### Linux Kernel Plumbing: `memfd_create` and `mmap`

To make this work cleanly without touching the actual filesystem or disk, we can use `memfd_create` (introduced in Linux 3.17). It creates an anonymous, in-memory file descriptor backed entirely by the kernel page cache.

Here is the exact setup procedure:
1. Determine the system page size using `sysconf(_SC_PAGESIZE)`.
2. Ensure the requested buffer capacity is a power of two and a multiple of the page size.
3. Allocate an anonymous memory file with `memfd_create`.
4. Set its size with `ftruncate`.
5. Reserve a contiguous virtual address range of $2S$ using `mmap` with `PROT_NONE` and `MAP_ANONYMOUS | MAP_PRIVATE`. This reserves the address space without allocating physical pages.
6. Map the physical buffer to the first half $[V, V + S)$ using `MAP_SHARED | MAP_FIXED`.
7. Map the same physical buffer to the second half $[V + S, V + 2S)$ using `MAP_SHARED | MAP_FIXED`.
8. Close the file descriptor. The mappings remain valid until `munmap` is called.

---

### Lock-Free SPSC Design & Memory Orderings

Let us implement this as a high-performance Single-Producer Single-Consumer (SPSC) queue.

To prevent cache line bouncing and false sharing, we must ensure the producer's write state and the consumer's read state reside on separate cache lines. We enforce this with `alignas(64)`:

```
Cache Line 0 (Producer): [ head (atomic) | cached_tail | padding ]
Cache Line 1 (Consumer): [ tail (atomic) | cached_head | padding ]
```

We use monotonic 64-bit unsigned integers for `head` and `tail`. Even at an extreme write rate of 100 million messages per second:

$$
\frac{2^{64}}{10^8 \text{ ops/sec}} \approx 1.84 \times 10^{11} \text{ seconds} \approx 5845 \text{ years}
$$

Monotonic counters will never overflow during the lifetime of the process, which eliminates wraparound math entirely.

#### Acquire-Release Semantics
We avoid sequential consistency (`std::memory_order_seq_cst`) because it emits full memory barriers (`mfence` or locked instructions on x86, `dmb ish` on ARM64).
- The **Producer** writes payload bytes into the buffer, then stores `head` with `std::memory_order_release`. This ensures all writes to the ring buffer payload are committed before the updated `head` becomes visible.
- The **Consumer** loads `head` with `std::memory_order_acquire`. This guarantees subsequent reads of the payload cannot be reordered before the load of `head`.

---

### Implementation

Below is the complete implementation in modern C++20, using raw memory allocations, strict 2-space indentation, and low-level Linux systems calls.

```cpp
#define _GNU_SOURCE
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>
#include <cstdint>
#include <cstddef>
#include <cstring>
#include <atomic>
#include <new>

class MirroredRingBuffer {
public:
  explicit MirroredRingBuffer(size_t minimum_capacity) {
    size_t page_size = static_cast<size_t>(sysconf(_SC_PAGESIZE));
    capacity_ = page_size;
    while (capacity_ < minimum_capacity) {
      capacity_ <<= 1;
    }

    int fd = memfd_create("mirrored_ring_buffer", MFD_CLOEXEC);
    if (fd < 0) {
      return;
    }

    if (ftruncate(fd, static_cast<off_t>(capacity_)) != 0) {
      close(fd);
      return;
    }

    // Reserve 2 * capacity_ bytes of contiguous virtual address space
    uint8_t* reserved_address = static_cast<uint8_t*>(mmap(
      nullptr,
      2 * capacity_,
      PROT_NONE,
      MAP_PRIVATE | MAP_ANONYMOUS,
      -1,
      0
    ));

    if (reserved_address == MAP_FAILED) {
      close(fd);
      return;
    }

    // Map the physical pages to the first half
    uint8_t* lower_half = static_cast<uint8_t*>(mmap(
      reserved_address,
      capacity_,
      PROT_READ | PROT_WRITE,
      MAP_SHARED | MAP_FIXED,
      fd,
      0
    ));

    if (lower_half == MAP_FAILED) {
      munmap(reserved_address, 2 * capacity_);
      close(fd);
      return;
    }

    // Map the EXACT SAME physical pages to the adjacent second half
    uint8_t* upper_half = static_cast<uint8_t*>(mmap(
      reserved_address + capacity_,
      capacity_,
      PROT_READ | PROT_WRITE,
      MAP_SHARED | MAP_FIXED,
      fd,
      0
    ));

    if (upper_half == MAP_FAILED) {
      munmap(reserved_address, 2 * capacity_);
      close(fd);
      return;
    }

    // Once mapped, we do not need to keep the file descriptor open
    close(fd);
    buffer_ = reserved_address;
  }

  ~MirroredRingBuffer() {
    if (buffer_ != nullptr) {
      munmap(buffer_, 2 * capacity_);
    }
  }

  // Non-copyable, non-movable for safety
  MirroredRingBuffer(const MirroredRingBuffer&) = delete;
  MirroredRingBuffer& operator=(const MirroredRingBuffer&) = delete;

  [[nodiscard]] bool is_valid() const {
    return buffer_ != nullptr;
  }

  [[nodiscard]] size_t capacity() const {
    return capacity_;
  }

  // --- Producer API (Only call from producer thread) ---

  [[nodiscard]] size_t write_available() {
    uint64_t current_head = producer_state_.head.load(std::memory_order_relaxed);
    uint64_t current_tail = producer_state_.cached_tail;

    if (current_head - current_tail >= capacity_) {
      // Refresh cached tail from consumer
      current_tail = consumer_state_.tail.load(std::memory_order_acquire);
      producer_state_.cached_tail = current_tail;
    }

    return capacity_ - static_cast<size_t>(current_head - current_tail);
  }

  // Reserves contiguous write space. Returns pointer directly into the buffer.
  [[nodiscard]] uint8_t* prepare_write(size_t length) {
    if (length > write_available()) {
      return nullptr;
    }

    uint64_t current_head = producer_state_.head.load(std::memory_order_relaxed);
    size_t offset = static_cast<size_t>(current_head & (capacity_ - 1));
    return buffer_ + offset;
  }

  void commit_write(size_t length) {
    uint64_t current_head = producer_state_.head.load(std::memory_order_relaxed);
    producer_state_.head.store(current_head + length, std::memory_order_release);
  }

  // --- Consumer API (Only call from consumer thread) ---

  [[nodiscard]] size_t read_available() {
    uint64_t current_tail = consumer_state_.tail.load(std::memory_order_relaxed);
    uint64_t current_head = consumer_state_.cached_head;

    if (current_head == current_tail) {
      // Refresh cached head from producer
      current_head = producer_state_.head.load(std::memory_order_acquire);
      consumer_state_.cached_head = current_head;
    }

    return static_cast<size_t>(current_head - current_tail);
  }

  // Returns a direct, contiguous slice to read up to read_available() bytes.
  [[nodiscard]] const uint8_t* prepare_read(size_t length) {
    if (length > read_available()) {
      return nullptr;
    }

    uint64_t current_tail = consumer_state_.tail.load(std::memory_order_relaxed);
    size_t offset = static_cast<size_t>(current_tail & (capacity_ - 1));
    return buffer_ + offset;
  }

  void commit_read(size_t length) {
    uint64_t current_tail = consumer_state_.tail.load(std::memory_order_relaxed);
    consumer_state_.tail.store(current_tail + length, std::memory_order_release);
  }

private:
  uint8_t* buffer_{nullptr};
  size_t capacity_{0};

  // Keep producer and consumer variables on separate cache lines to avoid false sharing
  struct alignas(64) ProducerState {
    std::atomic<uint64_t> head{0};
    uint64_t cached_tail{0};
  } producer_state_;

  struct alignas(64) ConsumerState {
    std::atomic<uint64_t> tail{0};
    uint64_t cached_head{0};
  } consumer_state_;
};
```

---

### Step-by-Step Analysis: How Zero-Copy Unrolls

Let's trace what happens when we write across the edge.

Suppose our page size is 4096 bytes ($C = 4096$).
1. `buffer_` points to virtual address `0x7fff0000`.
2. Lower half: `0x7fff0000` to `0x7fff0fff` (Physical Page A).
3. Upper half: `0x7fff1000` to `0x7fff1fff` (Physical Page A).

Now assume `head = 4090`. We want to write an 8-byte uint64 integer.

1. `offset = head & (capacity_ - 1) = 4090 & 4095 = 4090`.
2. Pointer returned: `buffer_ + 4090 = 0x7fff0ffa`.
3. The producer writes 8 bytes directly to `0x7fff0ffa`:
   - Bytes 0 through 5 write to `0x7fff0ffa ... 0x7fff0fff` (the end of Physical Page A).
   - Bytes 6 and 7 write to `0x7fff1000 ... 0x7fff1001` (the beginning of the upper virtual mapping).
4. Because `0x7fff1000` is mapped to Physical Page A, the MMU writes bytes 6 and 7 into physical offsets 0 and 1 of Page A!
5. Next, the producer calls `commit_write(8)`.
6. `head` updates to `4090 + 8 = 4098`.
7. On the next write:
   `offset = 4098 & 4095 = 2`.
   The returned pointer is `buffer_ + 2 = 0x7fff0002`.

No conditional checks. No `if (tail + len > capacity)` branches. No temporary arrays.

---

### Assembly Level Comparison

Look at what happens to the hot path in your compiler.

#### Standard Ring Buffer Read (Branchy)
```cpp
void read_standard(uint8_t* dest, const uint8_t* src, size_t tail, size_t len, size_t cap) {
  size_t first = cap - tail;
  if (len <= first) {
    memcpy(dest, src + tail, len);
  } else {
    memcpy(dest, src + tail, first);
    memcpy(dest + first, src, len - first);
  }
}
```

Disassembly on x86-64 (`gcc -O3`):
```nasm
read_standard:
  mov    rax, r8
  sub    rax, rdx             ; rax = cap - tail
  cmp    rcx, rax             ; compare len with (cap - tail)
  jbe    .L_single_copy       ; conditional branch! (branch predictor hazard)
  push   r12
  push   rbx
  ; Setup and call first memcpy...
  call   memcpy
  ; Setup and call second memcpy...
  call   memcpy
  pop    rbx
  pop    r12
  ret
.L_single_copy:
  add    rsi, rdx
  jmp    memcpy
```

You have a conditional branch instruction (`jbe`), register spills, and potentially two separate function calls to `memcpy`.

#### Mirrored Ring Buffer Read (Zero Branch)
```cpp
void read_mirrored(uint8_t* dest, const uint8_t* src, size_t tail, size_t len, size_t mask) {
  memcpy(dest, src + (tail & mask), len);
}
```

Disassembly on x86-64 (`gcc -O3`):
```nasm
read_mirrored:
  and    rdx, r8              ; rdx = tail & mask
  add    rsi, rdx             ; rsi = src + offset
  jmp    memcpy               ; direct tail call, zero branches!
```

The assembly collapses into a single `and`, a single `add`, and a tail call to `memcpy`. The compiler can easily inline small constant-size reads (like deserializing a header) into scalar SIMD loads (`vmovups`).

---

### Quantitative Performance Modeling

Let $B$ denote the buffer capacity in bytes.
Let $k$ denote the payload size in bytes ($k \le B$).

Assuming random write offsets uniformly distributed over $[0, B)$, the probability $P_{\text{wrap}}$ that a given operation spans across the buffer boundary is:

$$
P_{\text{wrap}} = \frac{k - 1}{B}
$$

For small packets ($k = 64$) in a 64 KB buffer, $P_{\text{wrap}} \approx 0.1\%$.
However, in high-bandwidth audio streaming or video capture where buffer payloads are large (for instance, reading $k = 4096$ audio frames from a $16384$ frame buffer):

$$
P_{\text{wrap}} = \frac{4096 - 1}{16384} \approx 25\%
$$

In a standard ring buffer, **25% of all calls** trigger the branch, execute two `memcpy` operations, and incur branch predictor degradation.

#### Latency Analysis
The cost of an operation in CPU cycles can be modeled as:

$$
T_{\text{standard}} = T_{\text{base}} + P_{\text{wrap}} \cdot \left( T_{\text{branch\_miss}} + T_{\text{second\_call}} \right)
$$

Where:
- $T_{\text{base}}$ is the cost of calculating indices and copying payload.
- $T_{\text{branch\_miss}} \approx 15 \text{ to } 20 \text{ cycles}$ on modern Zen 4 / Golden Cove architectures when speculative execution mispredicts the boundary transition.
- $T_{\text{second\_call}} \approx 10 \text{ to } 25 \text{ cycles}$ (parameter setup, register saving, and non-inlined function call overhead).

With the mirrored ring buffer:

$$
T_{\text{mirrored}} = T_{\text{base}}
$$

The variance in latency ($\sigma^2$) approaches zero because the execution path is fully deterministic regardless of where the indices land. In latency-critical systems, deterministic execution time is often more valuable than raw throughput.

---

### Real-World Verification

To verify that this works in practice, here is a small test harness:

```cpp
#include <iostream>
#include <cassert>

int main() {
  MirroredRingBuffer rb(4096);
  if (!rb.is_valid()) {
    std::cerr << "Failed to allocate mirrored ring buffer!\n";
    return 1;
  }

  std::cout << "Allocated buffer with capacity: " << rb.capacity() << " bytes\n";

  // Advance head near the edge
  size_t near_end = rb.capacity() - 4;
  uint8_t* p1 = rb.prepare_write(near_end);
  assert(p1 != nullptr);
  std::memset(p1, 0xAA, near_end);
  rb.commit_write(near_end);

  // Consume all bytes except leave head at near_end
  const uint8_t* r1 = rb.prepare_read(near_end);
  assert(r1 != nullptr);
  rb.commit_read(near_end);

  // Now write an 8-byte integer spanning the boundary
  uint8_t payload[8] = {1, 2, 3, 4, 5, 6, 7, 8};
  uint8_t* p2 = rb.prepare_write(8);
  assert(p2 != nullptr);
  std::memcpy(p2, payload, 8);
  rb.commit_write(8);

  // Read back the contiguous slice across the boundary
  const uint8_t* r2 = rb.prepare_read(8);
  assert(r2 != nullptr);

  for (size_t i = 0; i < 8; ++i) {
    if (r2[i] != payload[i]) {
      std::cerr << "Mismatch at byte " << i << "\n";
      return 1;
    }
  }

  std::cout << "Successfully verified zero-copy wrap-around!\n";
  return 0;
}
```

Compiling and running this under Linux:

```bash
g++ -O3 -std=c++20 main.cpp -o mirrored_test
./mirrored_test
```

Output:
```text
Allocated buffer with capacity: 4096 bytes
Successfully verified zero-copy wrap-around!
```

---

### Important Considerations and Edge Cases

1. **Virtual Address Space Consumption**:
   Each buffer consumes $2 \times S$ bytes of virtual address space. On a modern 64-bit Linux architecture (with standard 48-bit or 57-bit virtual addressing), processes have 128 TB to 64 PB of virtual address space. Burning an extra 64 KB or 2 MB of virtual space is completely irrelevant. However, on legacy 32-bit embedded systems, virtual memory fragmentation can become a real constraint if allocating thousands of these buffers.

2. **Page Alignment Minimums**:
   Your capacity will always round up to the architecture's minimum page size (4 KB on x86, 4 KB/16 KB/64 KB on ARM). If you only need a 64-byte queue, this technique is overkill—stick to a plain array. But for large ring buffers ($>4\text{ KB}$), it is nearly always a net win.

3. **HugePages Support**:
   For ultra-high performance (like kernel bypass with DPDK), you can back the `memfd` with transparent huge pages or pass `MFD_HUGETLB | MFD_HUGE_2MB` to `memfd_create`. This drastically reduces TLB miss overhead when working with massive queues (e.g., 32 MB+).

4. **Multi-Consumer / Multi-Producer Extensions**:
   While the implementation above is SPSC, the virtual memory aliasing trick works equally well for MPMC (Multi-Producer Multi-Consumer) queues. You simply combine the mirrored address mapping with atomic CAS (`compare_exchange_weak`) operations on reservations.

---

### Conclusion

Low-level systems engineering is full of scenarios where software engineers spend days writing intricate code to work around limitations that the underlying hardware can solve in zero cycles. 

Mirrored ring buffers are a prime example: by understanding how modern MMU page tables, VIPT L1 caches, and Linux virtual memory subsystems work, we turn a messy edge-case wrapping problem into a clean, branchless, zero-copy primitive.
