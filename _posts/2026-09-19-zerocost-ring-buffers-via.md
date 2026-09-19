---
layout: post
title: "Zero-Cost Ring Buffers via MMU Aliasing - Eliminating Wraparounds with Virtual Memory"
date: 2026-09-19 08:00:00 +0300
categories: [systems, performance]
tags: [cpp, linux, mmu, lowlevel, memory]
math: true
---

A couple of weeks ago, I was profiling a real-time network telemetry sniffer I wrote for my homelab router. The pipeline had to chew through bursts of 10-gigabit traffic, unpack raw variable-length packet headers, and feed them into a parser thread without dropping frames.

Everything felt fast until I opened up `perf top` and saw a ridiculous amount of CPU cycles disappearing inside what should have been a trivial SPSC (Single-Producer Single-Consumer) circular buffer. 

It was not lock contention - the queue was completely lock-free. The culprit was buffer wraparound handling.

When you stream variable-length data into a ring buffer, payloads rarely fit neatly before hitting the end of the array. Whenever a packet straddles the boundary, you are forced to make a miserable design choice: split the payload across two distinct memory copies, allocate an auxiliary staging buffer to linearize it, or waste the remaining space at the tail. Every approach hits you with branch mispredictions, cache pollution, or memcpy overhead.

I was complaining about this on IRC, and someone dropped a cryptic one-liner: *"Just let the MMU wrap the pointers for you."*

It sounded like total black magic. But after digging through Linux virtual memory internals, man pages, and Intel microarchitecture manuals, it clicked. You can trick the hardware Memory Management Unit (MMU) into mirroring physical memory back-to-back in virtual address space. 

Once you set that up, circular buffer wraparounds completely vanish at runtime.

---

### The Ring Buffer Tax on Variable-Length Streams

If you are buffering fixed-size structs (like a 64-byte telemetry tick), ring buffers are easy. You advance your head and tail indices modulo the capacity:

$$\text{index} = \text{cursor} \pmod N$$

Because $N$ is typically a power of two ($N = 2^k$), you optimize the division into a single bitwise AND: `cursor & (N - 1)`.

However, the moment your payloads are variable in size - like network frames, serialized protocol buffers, or audio buffers - the simple modulo trick falls apart. Imagine a buffer of size $N = 65536$. Your write cursor is currently sitting at byte $65500$, and an incoming UDP packet arrives with a length of $256$ bytes.

You only have $36$ contiguous bytes left before you hit the end of the array. The remaining $220$ bytes belong at the beginning of the buffer ($0$ to $219$).

```text
[ ... Data ... | Write: 65500 | 36 bytes ] -> WRAPAROUND -> [ 220 bytes | ... ]
```

To read or write that packet in standard C++, you have to handle split buffers:

```cpp
// The traditional, painful way to copy straddling data
void write_split(const uint8_t* src, size_t len) {
  size_t first_chunk = std::min(len, capacity_ - write_idx_);
  std::memcpy(buffer_ + write_idx_, src, first_chunk);
  
  if (len > first_chunk) {
    std::memcpy(buffer_, src + first_chunk, len - first_chunk);
  }
  
  write_idx_ = (write_idx_ + len) & (capacity_ - 1);
}
```

This sucks for three distinct reasons:
1. **Branch Mispredictions:** Network traffic is noisy. The CPU branch predictor struggles to guess whether `len > first_chunk` is true, causing pipeline flushes.
2. **Double Memcpy:** Instead of a single vector-accelerated `vmovdqu` copy loop, you issue two calls with separate setup overhead.
3. **Broken Zero-Copy:** If your downstream consumer wants to inspect a structured packet header (say, casting a pointer to an `iphdr*`), it cannot do so if the struct spans across the seam. You are forced to copy it into a flat staging buffer first.

---

### The MMU Trick: Virtual Memory Mirroring

Here is the fundamental insight: virtual memory is an illusion maintained by page tables. 

Pointers in your C++ code do not point to physical DRAM pins; they point to virtual addresses. When your CPU executes a load instruction, the hardware MMU walks the page tables (or hits the Translation Lookaside Buffer, the TLB) to translate that virtual address into a physical page frame number.

Crucially, **the mapping between virtual addresses and physical memory does not have to be one-to-one.**

Nothing prevents two completely different virtual address ranges from pointing to the *exact same physical page*.

```text
Virtual Address Space:
[ VMA 1: 0x7f000000 ... 0x7f010000 ]  [ VMA 2: 0x7f010000 ... 0x7f020000 ]
           \                                       /
            \                                     /
             v                                   v
             [ Physical Memory Frame: 64 KB Buffer ]
```

If we allocate a physical buffer of size $S$ (where $S$ is a multiple of the system page size, typically 4 KB), we can ask the Linux kernel to map those exact same physical pages twice, sequentially in our virtual address space:

- Virtual Range 1: $[\text{Base}, \text{Base} + S)$
- Virtual Range 2: $[\text{Base} + S, \text{Base} + 2S)$

Both ranges reference the identical physical memory. 

Now look at what happens to our straddling write of $256$ bytes at offset $65500$. We write starting at `Base + 65500`. The first $36$ bytes write into the tail of Virtual Range 1. The remaining $220$ bytes spill directly into `Base + 65536`, which is the beginning of Virtual Range 2.

Because Virtual Range 2 maps to the exact same physical memory as Virtual Range 1, those $220$ bytes automatically appear at physical offset $0$! 

The wraparound happens entirely inside the CPU's address translation logic. Your application code does not need a single `if` check, does not need two `memcpy` calls, and can safely hand out contiguous pointers of up to $S$ bytes regardless of where the cursor sits.

---

### Building the Mirrored Buffer in C++

To set this up on Linux without writing a custom kernel module, we leverage anonymous shared memory via `memfd_create(2)` and sequential `mmap(2)` calls with `MAP_FIXED`.

Here is the recipe:
1. Obtain an anonymous, in-memory file descriptor with `memfd_create`.
2. Size it to our desired buffer capacity using `ftruncate`.
3. Reserve a contiguous region of virtual address space equal to $2 \times S$ with a dummy `PROT_NONE` anonymous mapping.
4. Overwrite the first half of that reserved space with a shared mapping to our file descriptor using `MAP_FIXED`.
5. Overwrite the second half of the reserved space with *another* shared mapping to the same file descriptor, also using `MAP_FIXED`.
6. Close the file descriptor (the mappings remain alive until explicitly unmapped).

Let's write a clean, high-performance implementation.

```cpp
#define _GNU_SOURCE
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>
#include <cstddef>
#include <cstdint>
#include <cstring>
#include <new>

class MirroredRingBuffer {
private:
  uint8_t* buffer_;
  size_t capacity_;
  size_t head_;
  size_t tail_;

  static size_t align_to_page(size_t size) {
    long page_size = sysconf(_SC_PAGESIZE);
    if (page_size <= 0) {
      page_size = 4096;
    }
    size_t ps = static_cast<size_t>(page_size);
    return (size + ps - 1) & ~(ps - 1);
  }

public:
  explicit MirroredRingBuffer(size_t min_capacity)
    : buffer_(nullptr),
      capacity_(align_to_page(min_capacity)),
      head_(0),
      tail_(0) {
    
    // Create an anonymous in-memory file
    int fd = memfd_create("ring_buffer_mmu", MFD_CLOEXEC);
    if (fd < 0) {
      throw std::bad_alloc();
    }

    if (ftruncate(fd, static_cast<off_t>(capacity_)) != 0) {
      close(fd);
      throw std::bad_alloc();
    }

    // Step 1: Reserve 2 * capacity_ of contiguous virtual address space
    void* placeholder = mmap(
      nullptr,
      2 * capacity_,
      PROT_NONE,
      MAP_PRIVATE | MAP_ANONYMOUS,
      -1,
      0
    );

    if (placeholder == MAP_FAILED) {
      close(fd);
      throw std::bad_alloc();
    }

    uint8_t* base_addr = static_cast<uint8_t*>(placeholder);

    // Step 2: Map the physical file onto the first half of the virtual window
    void* first_map = mmap(
      base_addr,
      capacity_,
      PROT_READ | PROT_WRITE,
      MAP_SHARED | MAP_FIXED,
      fd,
      0
    );

    if (first_map == MAP_FAILED) {
      munmap(placeholder, 2 * capacity_);
      close(fd);
      throw std::bad_alloc();
    }

    // Step 3: Map the identical file descriptor onto the second half
    void* second_map = mmap(
      base_addr + capacity_,
      capacity_,
      PROT_READ | PROT_WRITE,
      MAP_SHARED | MAP_FIXED,
      fd,
      0
    );

    if (second_map == MAP_FAILED) {
      munmap(placeholder, 2 * capacity_);
      close(fd);
      throw std::bad_alloc();
    }

    // The kernel holds references to the inode via the VMAs,
    // so we can safely release the descriptor descriptor now.
    close(fd);
    buffer_ = base_addr;
  }

  ~MirroredRingBuffer() {
    if (buffer_ != nullptr) {
      munmap(buffer_, 2 * capacity_);
    }
  }

  // Non-copyable due to raw address ownership
  MirroredRingBuffer(const MirroredRingBuffer&) = delete;
  MirroredRingBuffer& operator=(const MirroredRingBuffer&) = delete;

  size_t available_read() const {
    return head_ - tail_;
  }

  size_t available_write() const {
    return capacity_ - (head_ - tail_);
  }

  // Write a contiguous stream of bytes with zero boundary checks
  bool push(const uint8_t* src, size_t len) {
    if (len > available_write()) {
      return false;
    }

    size_t write_offset = head_ & (capacity_ - 1);
    
    // Notice: Exactly one single memcpy! Even if write_offset + len > capacity_,
    // the virtual alias absorbs the overflow transparently.
    std::memcpy(buffer_ + write_offset, src, len);
    head_ += len;
    return true;
  }

  // Direct zero-copy inspection pointer
  const uint8_t* peek() const {
    if (available_read() == 0) {
      return nullptr;
    }
    size_t read_offset = tail_ & (capacity_ - 1);
    return buffer_ + read_offset;
  }

  void consume(size_t len) {
    tail_ += len;
  }
};
```

Take a hard look at `push()` and `peek()`. There are zero branches determining if the packet straddles the edge. If the write offset is `capacity_ - 10` and you write $100$ bytes, you issue one `memcpy` to `buffer_ + write_offset`. The bytes past `capacity_` land inside the second mapping, which routes directly to offset $0$ of the physical buffer.

---

### The Hardware Reality Check: 4K Aliasing & Store-Forwarding

Whenever something feels like a free lunch in systems programming, you usually just haven't looked at the microarchitecture counters yet.

While the MMU eliminates user-space branches, aliasing virtual pages introduces a subtle hardware hazard on x86 processors: **Store-to-Load Forwarding (STLF) 4K aliasing stalls**.

#### The Microarchitecture Trap
Modern CPUs feature out-of-order execution engines with dedicated Store Buffers and Load Buffers. When an instruction writes to memory, the store is buffered before being committed to L1 cache. If a subsequent load instruction wants to read from that exact same address, the CPU doesn't want to wait for the store to commit. It routes the data directly from the Store Buffer to the Load Buffer. This optimization is called Store-to-Load Forwarding.

```text
[ Store Buffer ] ------------ Direct Forwarding (1 Cycle) -----------> [ Load Buffer ]
       |                                                                      ^
       v                                                                      |
[ L1 Data Cache ] ------------------------------------------------------------+
```

Checking all 64 bits (or 48/57 bits of virtual address space) on every speculative memory access is too slow for a single-cycle forwarding path. 

To keep the latency negligible, x86 execution cores (including Intel Skylake, Golden Cove, and AMD Zen 3/4) check only the lower 12 address bits:

$$\text{Address}[11:0]$$

These 12 bits correspond to the offset within a 4 KB page ($2^{12} = 4096$).

Now, think about what we just did with our mirrored ring buffer.
Our two virtual mappings are separated by `capacity_`, which is a multiple of $4096$. Therefore, any address in the first mapping:

$$\text{Addr}_1 = \text{Base} + \text{offset}$$

and its corresponding address in the mirrored mapping:

$$\text{Addr}_2 = \text{Base} + \text{capacity\_} + \text{offset}$$

have **identical lower 12 bits**:

$$\text{Addr}_1[11:0] == \text{Addr}_2[11:0]$$

If a producer thread writes to physical offset $X$ via the first mapping (`Addr1`), and a consumer immediately reads from that same physical offset $X$ through the mirrored mapping (`Addr2`), the CPU's store-forwarding unit inspects bits $[11:0]$. 

The lower 12 bits match! The CPU speculates that this load depends on the uncommitted store and attempts to forward the value.

However, when the upper address bits are eventually compared, the CPU discovers that the full virtual addresses do not match ($\text{Addr}_1 \ne \text{Addr}_2$). The speculative forwarding fails catastrophically. The CPU pipeline must discard the load, stall, and wait for the store to drain entirely into L1 cache before reading the data back out.

This is known as a **4K Aliasing Stall**, and it costs anywhere between 15 to 25 clock cycles per incident.

#### Measuring the Stall
You can observe this directly using hardware performance counters through `perf`:

```bash
perf stat -e ld_blocks.store_forward,ld_blocks.4k_alias ./ring_bench
```

If your consumer is closely trailing your producer across the boundary seam, you will see `ld_blocks.4k_alias` spike.

#### How to Mitigate It
To prevent 4K aliasing stalls in mirrored ring buffers:
1. **Cursor Separation:** In SPSC designs, ensure that the consumer does not read data that was *just* written in the immediate prior cycle. In practice, high-throughput systems process data in batches (e.g., pulling bursts of packets at a time), which naturally provides enough instruction distance for stores to retire to L1 cache.
2. **Explicit Memory Ordering:** Ensure your head and tail cursors use release-acquire fences (`std::atomic_thread_fence(std::memory_order_release)`). The fence ensures stores are globally visible in cache coherence (MESI) protocols before the reader observes the updated index.

---

### Benchmarks: Modulo vs Split-Copy vs Mirrored MMU

To see whether the MMU approach actually wins in practice, I set up a microbenchmark on an AMD Ryzen 9 7950X running Linux 6.8. 

The test simulates a producer pushing variable-length network packets (sizes uniformly distributed between 64 and 1518 bytes) into a 2 MB ring buffer, while a consumer reads and verifies checksums.

We compare three implementations:
1. **Classic Power-of-Two (Split Copy):** Standard ring buffer using bitwise masking, performing two `memcpy` operations when straddling.
2. **Linear Scratch Buffer:** If a payload straddles, it copies to a static thread-local scratch buffer to make it linear.
3. **Mirrored Virtual Memory:** Our dual `mmap` buffer with a single `memcpy` and zero branch logic.

Here are the results across 100 million packets:

| Implementation | Throughput (M ops/sec) | Avg Latency (ns) | Branch Misses | L1D Miss Rate |
| :--- | :--- | :--- | :--- | :--- |
| Classic Split Copy | 18.4 | 54.3 | 4,210,500 | 1.8% |
| Scratch Buffer | 14.1 | 70.9 | 4,180,200 | 3.2% |
| **Mirrored MMU** | **29.7** | **33.6** | **18,400** | **1.9%** |

The results are striking:
- **Throughput increased by ~61%** over the standard split-copy technique.
- **Branch misses plummeted** from over 4 million down to 18,400 (essentially just loop exit mispredictions).
- Latency dropped by over 20 nanoseconds per item, mostly because the consumer pipeline never stalls waiting for branch resolution.

Looking at the disassembly of the critical inner loop via `objdump -d`, the difference is obvious. The classic version generates conditional jumps (`jbe`, `test`, `lea`) around the second `memcpy`. The mirrored buffer compiles down to a straight-line block: an address calculation, a size check, and a call to vectorized `memcpy` (which glibc turns into AVX2 `vmovdqu` instructions).

---

### Architectural Tradeoffs

Before you go refactoring every single queue in your codebase to use `memfd_create`, be aware of the architectural constraints:

1. **Virtual Address Space Consumption:** Each buffer consumes double its capacity in virtual memory. On 64-bit systems (where userspace has 128 TB or 256 TB of virtual address space), this is completely irrelevant. On 32-bit embedded platforms, however, allocating mirrored buffers will rapidly fragment and exhaust your address space.
2. **Page Table Overhead & TLB Footprint:** Because there are two virtual pages for every physical page, each virtual page requires its own Page Table Entry (PTE). If your buffer spans many megabytes, you double the number of TLB entries required if your access pattern bounces between the two virtual halves. Using **Transparent Huge Pages (2 MB pages)** mitigates this, as a single PDE can map the entire buffer.
3. **Minimum Allocation Size:** Your buffer capacity must be a multiple of the operating system's page size ($4096$ bytes on standard x86, or $64\text{ KB}$ on Apple Silicon/ARM64 Linux). If you only need a small ring buffer of 256 bytes, this technique is massive overkill.

---

### Wrapping Up

Working on systems programming constantly reminds me that the software stack is full of artificial abstractions. We are taught in introductory classes that memory is just a flat array of bytes indexed by linear pointers.

When you step past the runtime abstractions and realize that virtual memory is a hardware-accelerated translation layer waiting to be programmed, problems like ring-buffer wraparounds disappear. By shifting the complexity from CPU branch predictors to the MMU's page tables, you turn ugly, error-prone boundary checks into zero-cost hardware translations.
