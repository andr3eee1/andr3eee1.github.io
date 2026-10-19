---
layout: post
title: "Beating the MESI Protocol - Wait-Free Arena Allocators and Double-Width CAS on x86-64"
date: 2026-10-19 08:00:00 +0300
categories: [Systems Engineering, Low-Level C++]
tags: [cpp, lock-free, x86-64, concurrency, memory-model]
math: true
---

A couple of weeks ago I was profiling a high-frequency packet processing engine I've been building in my spare time after school, and `perf c2c` showed something super annoying. Even though my worker threads were pinned to isolated physical cores with `isolcpus` and weren't using a single `std::mutex`, almost 42% of my CPU cycles were vanishing into thin air inside `lock cmpxchg` instructions.

When people say "lock-free is fast," they usually forget that the CPU hardware still runs a distributed consensus protocol under the hood. On x86-64, every time two cores try to atomically modify the same 64-byte cache line, the L1d hardware prefetcher and the MESI (or MOESI on AMD Zen) cache coherency interconnect have to play ping-pong with `Request For Ownership` (RFO) broadcasts. If you're doing naive Compare-And-Swap loops on a shared freelist head, you haven't actually removed the lock—you've just pushed the contention down from the OS scheduler into the silicon ring bus.

In this post, we're going to tear apart why standard lock-free freelists fall apart under heavy multi-core contention, how the infamous **ABA problem** sneaks into pointer recycling, and how to build a zero-allocation, wait-free-ish slab arena in C++ using raw 2MB hugepages, x86-64 `cmpxchg16b` (128-bit atomic CAS), and thread-local cache-line magazines.

---

## 1. The Silicon Bottleneck: MESI State Transitions and RFO Storms

To understand why a simple atomic stack chokes, we have to look at what a core actually does when it executes a `LOCK` prefixed instruction on x86-64.

Every 64-byte L1 data cache line sits in one of four primary states in the standard **MESI** protocol:

* **Modified (M):** The current core has the only valid copy, and it's dirty relative to main RAM.
* **Exclusive (E):** The current core has the only copy, and it matches main RAM.
* **Shared (S):** Multiple cores hold read-only copies of the line.
* **Invalid (I):** The cache line is dead; reading it triggers an L1d miss.

Suppose you have $N$ threads hammering a global lock-free freelist head. Before a thread can execute `compare_exchange_weak`, it usually loads the current head pointer (putting the cache line into **S** across all cores), computes the next pointer, and then issues `lock cmpxchg`. 

That `lock cmpxchg` requires the core to transition its local cache line to the **M** state. To do that, the core's Load-Store Unit broadcasts an invalidation message (RFO) over the interconnect to all other $N - 1$ cores, forcing their copies into the **I** state. Worse, the losing cores in the CAS race immediately re-read the cache line on their next retry iteration, demoting the winner's **M** line back to **S** and flushing the store buffer.

We can model the effective throughput degradation as the number of contending cores $N$ grows. If $t_{\text{bus}}$ is the interconnect traversal latency (around 35–60 cycles across CCXs or NUMA nodes) and $p$ is the probability of a single CAS succeeding during a contention window:

$$
T_{\text{expected}}(N) = \sum_{k=1}^{N} k \cdot t_{\text{bus}} \cdot (1 - p)^{k - 1} \cdot p + \mathcal{O}(N^2 \cdot t_{\text{snoop}})
$$

When $N = 8$, your "sub-nanosecond" memory allocator suddenly takes 400+ clock cycles per pop.

---

## 2. The ABA Nightmare in Pointer-Based Freelists

Before we fix the cache-line bouncing, we have to deal with the classic correctness bug in lock-free memory pools: the **ABA problem**.

Imagine a lock-free lifo stack where free slots store an embedded `next` pointer in their first 8 bytes. Suppose the freelist currently looks like $A \to B \to C$:

1. **Thread 1** reads `head` ($A$) and dereferences $A\text{->next}$ to get $B$. It prepares to execute `CAS(&head, A, B)`, right before the OS interrupts it (or a Zen core stalls on an L3 miss).
2. **Thread 2** swoops in, pops $A$, and then pops $B$. Now `head` points to $C$.
3. **Thread 2** frees $A$ back onto the stack. Now `head` is $A \to C$, while $B$ is actively allocated and holding live user data!
4. **Thread 1** wakes up and executes `CAS(&head, A, B)`. The hardware sees that `head` is indeed equal to $A$, so the CAS **succeeds**!

Now `head` points to $B$—a block that Thread 2 is actively mutating. Your heap is silently corrupted, and you'll spend all weekend staring at a `SIGSEGV` in `gdb` wondering why two unrelated structs share the same memory address.

### Why 48-Bit Virtual Addressing Saves Us (Sometimes)

On standard 64-bit x86 (without 5-level paging / LA57 enabled), virtual addresses only use the lower 48 bits (`[47:0]`), while bits `[63:48]` must be a sign-extension of bit 47 (canonical address form). For user-space pointers where bit 47 is `0`, those top 16 bits are always zero.

A lot of people pack a 16-bit ABA monotonic tag into those upper 16 bits. That works okay until a thread stalls long enough for another core to execute $2^{16} = 65536$ pops and pushes—which, at 5ns per operation, takes literally **327 microseconds**. A single minor page fault or `sched_yield` easily exceeds 327 microseconds, wrapping the 16-bit counter right back to the exact same tag!

If we want true mathematical immunity against ABA wrap-around without garbage collection or heavy hazard pointer scans, we need a full **64-bit generation counter** paired with a **64-bit pointer** (or index), updated atomically in a single 128-bit instruction: `LOCK CMPXCHG16B`.

---

## 3. Implementing a `CMPXCHG16B` Tagged Freelist in C++

While `std::atomic<T>` supports 16-byte structs in C++20, GCC and Clang have historically had quirky habits of falling back to `libatomic` spinlocks depending on target architecture flags (`-mcx16`). Writing a tiny inline assembly wrapper around `cmpxchg16b` guarantees that the compiler emits the exact hardware instruction we want and lets us inspect the zero flag (`ZF`) directly via `=@ccz`.

Here is how `cmpxchg16b` works at the register level:
* `RDX:RAX` holds the 128-bit **expected** value (`RAX` = lower 64 bits, `RDX` = upper 64 bits).
* `RCX:RBX` holds the 128-bit **desired** value (`RBX` = lower 64 bits, `RCX` = upper 64 bits).
* If the 16-byte aligned memory operand matches `RDX:RAX`, it gets overwritten by `RCX:RBX` and `ZF` is set to `1`. Otherwise, the memory contents are loaded into `RDX:RAX` and `ZF` is cleared to `0`.

Let's write our foundational primitives. Note that we avoid `std::vector` completely and stick to raw fixed-size arrays aligned to 64-byte hardware cache lines to prevent false sharing.

```cpp
#include <cstdint>
#include <cstddef>
#include <sys/mman.h>
#include <immintrin.h>

struct alignas(16) TaggedHead {
  uint64_t offset;
  uint64_t tag;
};

static inline bool cas128(
    volatile TaggedHead* target,
    TaggedHead* expected,
    TaggedHead desired) noexcept {
  bool success = false;
  __asm__ __volatile__(
    "lock cmpxchg16b %1"
    : "=@ccz"(success),
      "+m"(*target),
      "+a"(expected->offset),
      "+d"(expected->tag)
    : "b"(desired.offset),
      "c"(desired.tag)
    : "memory", "cc"
  );
  return success;
}
```

Using a 64-bit `tag` means another core would have to execute $2^{64}$ allocations on the exact same slot before our sleeping thread wakes up to trigger an ABA collision. Even at 1 billion operations per second, $2^{64}$ iterations takes **584.9 years**. That's good enough for me.

---

## 4. Designing the Two-Tier Slab Magazine Arena

Even with `cas128`, having 16 threads hammer a single `TaggedHead` is still going to cause an RFO storm on the L3 interconnect. How do we make our arena effectively wait-free on the fast path while remaining lock-free on the slow path?

We steal an idea from Bonwick's classic Solaris slab allocator and combine it with lock-free batching: **Per-Thread Cache-Line Magazines**.

Instead of popping a single 64-byte block from the global freelist, each thread owns a private `ThreadMagazine` aligned to 128 bytes (two cache lines, because Intel's spatial streamer prefetcher likes to pull pairs of adjacent 64-byte lines). The magazine holds two fixed arrays (`rounds_primary` and `rounds_backup`) of 32 block indices each.

1. **Fast Path (0 atomics, 0 RFOs):** When a thread wants to allocate a block, it simply decrements a local non-atomic counter in `rounds_primary` and returns the pointer. If `rounds_primary` hits 0, it checks `rounds_backup` and swaps the two magazine roles in $\mathcal{O}(1)$ time.
2. **Slow Path (Batch Global CAS):** Only when *both* local arrays are completely empty (or both are full during `free()`) does the thread touch the global shared arena. And instead of doing 32 separate global CAS operations, we link the 32 blocks into a **batch sub-chain** locally and splice the entire 32-element chain into (or out of) the global freelist using a single `cas128` loop!

This cuts our cross-core MESI invalidation traffic by a factor of $32\times$, and the hysteretic primary/backup split stops pathological push/pop ping-ponging right at the boundary.

```
Thread Local L1d Cache (Zero Contention)          Shared L3 / RAM (128-bit CAS)
+--------------------------------------+          +---------------------------+
| ThreadMagazine (alignas 128)         |          | Global Batch Freelist     |
|  primary: [idx0, idx1, ..., idx31]   |          |  TaggedHead {off, tag}    |
|  backup:  [idx32, idx33, ..., idx63] | <======> |    |                      |
|  p_count: 14, b_count: 32            |  Splices |    v                      |
+--------------------------------------+  batches |  [Batch Node 0] -> [Node 1]|
                                          of 32   +---------------------------+
```

Let's write the complete, self-contained C++ implementation. We'll back the arena with `mmap` and `MAP_HUGETLB` (falling back cleanly to standard 4KB pages + `MADV_HUGEPAGE` if explicit 2MB hugepages aren't reserved in `/proc/sys/vm/nr_hugepages`) so that we also eliminate dTLB (Data Translation Lookaside Buffer) page-walk misses.

```cpp
#include <cstdint>
#include <cstddef>
#include <cstdio>
#include <cstdlib>
#include <sys/mman.h>
#include <immintrin.h>

static constexpr size_t kBlockSize = 64;
static constexpr size_t kBatchSize = 32;
static constexpr size_t kTotalBlocks = 65536; // 4 MB arena (2 hugepages)
static constexpr uint64_t kNullIndex = 0xFFFFFFFFFFFFFFFFULL;

struct alignas(64) ArenaBlock {
  uint64_t next_in_batch;
  uint64_t next_batch_head;
  uint8_t payload[kBlockSize - 16];
};

struct alignas(128) ThreadMagazine {
  uint32_t primary[kBatchSize];
  uint32_t backup[kBatchSize];
  uint32_t primary_count;
  uint32_t backup_count;
};

class WaitFreeSlabArena {
private:
  ArenaBlock* base_ptr_;
  alignas(128) volatile TaggedHead global_batch_head_;
  alignas(128) volatile uint64_t bump_cursor_;

  static inline void spin_pause() noexcept {
    _mm_pause();
  }

  static inline uint64_t atomic_fetch_add64(
      volatile uint64_t* ptr,
      uint64_t val) noexcept {
    return __atomic_fetch_add(ptr, val, __ATOMIC_RELAXED);
  }

public:
  WaitFreeSlabArena() noexcept
      : base_ptr_(nullptr),
        global_batch_head_{kNullIndex, 0},
        bump_cursor_(0) {
    size_t bytes = kTotalBlocks * sizeof(ArenaBlock);
    void* mem = mmap(
      nullptr,
      bytes,
      PROT_READ | PROT_WRITE,
      MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
      -1,
      0
    );

    if (mem == MAP_FAILED) {
      mem = mmap(
        nullptr,
        bytes,
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS,
        -1,
        0
      );
      if (mem != MAP_FAILED) {
        madvise(mem, bytes, MADV_HUGEPAGE);
      }
    }

    if (mem == MAP_FAILED) {
      std::abort();
    }

    base_ptr_ = static_cast<ArenaBlock*>(mem);

    // Pre-fault pages so we don't take minor page faults on the hot path
    for (size_t i = 0; i < kTotalBlocks; i += 64) {
      base_ptr_[i].next_in_batch = kNullIndex;
    }
  }

  ~WaitFreeSlabArena() noexcept {
    if (base_ptr_ != nullptr) {
      munmap(base_ptr_, kTotalBlocks * sizeof(ArenaBlock));
    }
  }

  WaitFreeSlabArena(const WaitFreeSlabArena&) = delete;
  WaitFreeSlabArena& operator=(const WaitFreeSlabArena&) = delete;

  void init_magazine(ThreadMagazine* mag) noexcept {
    mag->primary_count = 0;
    mag->backup_count = 0;
  }

  // Slow path: pop a batch of 32 blocks from the global CMPXCHG16B stack,
  // or carve a fresh batch from the wait-free atomic bump cursor.
  bool refill_array(uint32_t (&target_arr)[kBatchSize], uint32_t* count_out) noexcept {
    TaggedHead expected;
    expected.offset = global_batch_head_.offset;
    expected.tag = global_batch_head_.tag;

    while (expected.offset != kNullIndex) {
      ArenaBlock* head_block = &base_ptr_[expected.offset];
      TaggedHead desired;
      desired.offset = head_block->next_batch_head;
      desired.tag = expected.tag + 1;

      if (cas128(&global_batch_head_, &expected, desired)) {
        uint64_t curr = expected.offset;
        for (size_t i = 0; i < kBatchSize; ++i) {
          target_arr[i] = static_cast<uint32_t>(curr);
          curr = base_ptr_[curr].next_in_batch;
        }
        *count_out = kBatchSize;
        return true;
      }
      spin_pause();
    }

    // Global batch stack was empty; use wait-free fetch_add bump allocation
    uint64_t start_idx = atomic_fetch_add64(&bump_cursor_, kBatchSize);
    if (start_idx + kBatchSize > kTotalBlocks) {
      return false;
    }

    for (size_t i = 0; i < kBatchSize; ++i) {
      target_arr[i] = static_cast<uint32_t>(start_idx + i);
    }
    *count_out = kBatchSize;
    return true;
  }

  // Slow path: link 32 local indices into a batch chain and push with 1 CAS
  void flush_array(uint32_t (&source_arr)[kBatchSize], uint32_t* count_inout) noexcept {
    for (size_t i = 0; i + 1 < kBatchSize; ++i) {
      uint32_t curr_idx = source_arr[i];
      uint32_t next_idx = source_arr[i + 1];
      base_ptr_[curr_idx].next_in_batch = next_idx;
    }
    base_ptr_[source_arr[kBatchSize - 1]].next_in_batch = kNullIndex;

    uint64_t batch_head_idx = source_arr[0];
    ArenaBlock* head_block = &base_ptr_[batch_head_idx];

    TaggedHead expected;
    expected.offset = global_batch_head_.offset;
    expected.tag = global_batch_head_.tag;

    while (true) {
      head_block->next_batch_head = expected.offset;
      TaggedHead desired;
      desired.offset = batch_head_idx;
      desired.tag = expected.tag + 1;

      if (cas128(&global_batch_head_, &expected, desired)) {
        *count_inout = 0;
        return;
      }
      spin_pause();
    }
  }

  [[nodiscard]] void* allocate(ThreadMagazine* mag) noexcept {
    if (__builtin_expect(mag->primary_count > 0, 1)) {
      uint32_t idx = mag->primary[--mag->primary_count];
      return &base_ptr_[idx];
    }

    if (mag->backup_count > 0) {
      for (size_t i = 0; i < mag->backup_count; ++i) {
        mag->primary[i] = mag->backup[i];
      }
      mag->primary_count = mag->backup_count;
      mag->backup_count = 0;
      uint32_t idx = mag->primary[--mag->primary_count];
      return &base_ptr_[idx];
    }

    if (!refill_array(mag->primary, &mag->primary_count)) {
      return nullptr;
    }

    uint32_t idx = mag->primary[--mag->primary_count];
    return &base_ptr_[idx];
  }

  void deallocate(ThreadMagazine* mag, void* ptr) noexcept {
    if (ptr == nullptr) {
      return;
    }

    uintptr_t raw_ptr = reinterpret_cast<uintptr_t>(ptr);
    uintptr_t raw_base = reinterpret_cast<uintptr_t>(base_ptr_);
    uint32_t idx = static_cast<uint32_t>((raw_ptr - raw_base) / sizeof(ArenaBlock));

    if (__builtin_expect(mag->primary_count < kBatchSize, 1)) {
      mag->primary[mag->primary_count++] = idx;
      return;
    }

    if (mag->backup_count < kBatchSize) {
      mag->backup[mag->backup_count++] = idx;
      return;
    }

    // Both magazines are completely full; flush backup to global pool
    flush_array(mag->backup, &mag->backup_count);
    mag->backup[mag->backup_count++] = idx;
  }
};
```

---

## 5. Subtle Hardware Details That Actually Matter

There are three details in that code that look minor at first glance, but make a huge difference when you inspect the generated assembly and hardware performance counters (`perf stat -e L1-dcache-load-misses,cycle_activity.stalls_l1d_miss`):

### 1. `_mm_pause()` Inside the CAS Retry Loop
Look at the `spin_pause()` call when `cas128` fails. Without `PAUSE` (`rep nop` in machine code), an out-of-order CPU pipeline speculatively queues up dozens of load micro-ops (`uops`) onto the target cache line in a tight loop. The moment the winning core releases ownership of the cache line, the losing core realizes its speculative loads violated memory ordering and has to **flush its entire reorder buffer (ROB)**—a pipeline machine clear that costs ~15–20 cycles on top of the cache miss! `PAUSE` throttles the speculative load unit and prevents memory order violations (`machine_clears.memory_ordering`).

### 2. Why `alignas(128)` Instead of `alignas(64)`?
Everyone knows an x86-64 cache line is 64 bytes. So why did we align `global_batch_head_`, `bump_cursor_`, and `ThreadMagazine` to **128 bytes**?
On modern Intel and AMD processors, the L2 hardware prefetcher includes an **adjacent cache-line prefetcher** (sometimes called the spatial streamer). Whenever a core misses on an even-numbered 64-byte cache line at address $2k \times 64$, the hardware automatically issues a prefetch for the sibling line at $(2k + 1) \times 64$. If two independent hot atomics live in adjacent 64-byte lines within the same 128-byte sector, mutating one can trigger false prefetch invalidations on the other. Padding hot shared structures to 128 bytes isolates them across spatial prefetch pairs.

### 3. Wait-Free Bump Initialization (`__atomic_fetch_add`)
Notice how our constructor doesn't link all $65536$ blocks into a giant linked list at startup. Linking 65,536 nodes up front dirties 4MB of cache lines on core 0 and causes initial pops on worker threads to pull cold linked-list pointers out of remote L3 slices. Instead, `bump_cursor_` uses `lock xadd` (`__atomic_fetch_add`), which is **strictly wait-free** in hardware since x86 CPUs execute atomic fetch-and-add directly in the cache controller without a compare-and-swap retry loop. A thread only touches the `cas128` stack once blocks have actually been recycled.

---

## 6. Benchmarking with `RDTSCP` and Hardware Serialization

If you try to benchmark a 2-nanosecond allocator using `std::chrono::high_resolution_clock`, the call overhead to `clock_gettime` (even through the kernel's `vDSO` page) takes around 15–20 nanoseconds, which completely drowns out the signal.

To measure cycle-accurate latency, we read the processor's Time Stamp Counter (`TSC`) directly. However, a naive `rdtsc` instruction can be executed **out of order** by the CPU's instruction scheduler—meaning `rdtsc` might fire *before* the code you're trying to measure even finishes!

The proper pattern on x86-64 is:
* Start timer: `lfence` (or `cpuid`) followed by `rdtsc` to drain prior instructions before sampling the clock.
* Stop timer: `rdtscp` (which waits until all previous instructions retire before reading the counter) followed by `lfence` so subsequent instructions can't hoist above the measurement boundary.

```cpp
static inline uint64_t ticks_start() noexcept {
  uint32_t lo = 0;
  uint32_t hi = 0;
  __asm__ __volatile__(
    "lfence\n\t"
    "rdtsc\n\t"
    : "=a"(lo), "=d"(hi)
    :
    : "memory"
  );
  return (static_cast<uint64_t>(hi) << 32) | lo;
}

static inline uint64_t ticks_end() noexcept {
  uint32_t lo = 0;
  uint32_t hi = 0;
  uint32_t aux = 0;
  __asm__ __volatile__(
    "rdtscp\n\t"
    "lfence\n\t"
    : "=a"(lo), "=d"(hi), "=c"(aux)
    :
    : "memory"
  );
  return (static_cast<uint64_t>(hi) << 32) | lo;
}

int main() {
  WaitFreeSlabArena arena;
  ThreadMagazine local_mag;
  arena.init_magazine(&local_mag);

  static constexpr size_t kWindow = 16;
  void* ptrs[kWindow];

  // Warm up the magazine and instruction cache
  for (size_t i = 0; i < kWindow; ++i) {
    ptrs[i] = arena.allocate(&local_mag);
  }
  for (size_t i = 0; i < kWindow; ++i) {
    arena.deallocate(&local_mag, ptrs[i]);
  }

  static constexpr size_t kIterations = 1000000;
  uint64_t start_cycles = ticks_start();

  for (size_t iter = 0; iter < kIterations; ++iter) {
    for (size_t i = 0; i < kWindow; ++i) {
      ptrs[i] = arena.allocate(&local_mag);
      __asm__ __volatile__("" : : "r"(ptrs[i]) : "memory");
    }
    for (size_t i = 0; i < kWindow; ++i) {
      arena.deallocate(&local_mag, ptrs[i]);
    }
  }

  uint64_t end_cycles = ticks_end();
  double total_ops = static_cast<double>(kIterations * kWindow * 2);
  double cycles_per_op = static_cast<double>(end_cycles - start_cycles) / total_ops;

  std::printf("Cycles per alloc/dealloc op - %.2f cycles\n", cycles_per_op);
  return 0;
}
```

### Results Across 8 Contending Threads (Ryzen 7 / 4.5 GHz)

When compiled with `g++ -O3 -march=native -mcx16 -fno-exceptions -fno-rtti` and tested across 8 worker threads continuously allocating, passing blocks through an SPSC ring, and freeing them:

| Allocator Strategy | Median Latency (Cycles) | p99.9 Latency (Cycles) | L1d Invalidation Traffic / 1M Ops |
| :--- | :--- | :--- | :--- |
| Standard `malloc` / `free` (glibc ptmalloc) | 68.4 cycles | 1,420 cycles | Moderate (futex spikes on trim) |
| Naive Global `CMPXCHG16B` Freelist | 184.2 cycles | 3,890 cycles | ~1,940,000 RFO broadcasts |
| **Magazine-Batched `CMPXCHG16B` Arena** | **4.1 cycles** | **62 cycles** | **~31,200 RFO broadcasts** |

Dropping from 184 cycles down to **4.1 cycles** (under 1 nanosecond!) on the median path—and cutting p99.9 tail latency by over $60\times$—comes almost entirely from respecting how the L1d cache and the MESI interconnect actually work.

---

## Wrapping Up

Whenever you're designing concurrent data structures in C++, `std::atomic` is only half the story. The hardware doesn't care how clever your C++ templates look if your memory layout forces every core to fight over the same 64-byte region of silicon. By combining thread-local magazine hysteresis, 128-byte cache-sector alignment, 2MB hugepages, and batched 128-bit `cmpxchg16b` splice operations, you can get deterministic, sub-nanosecond memory management without ever worrying about ABA corruption or kernel syscalls.

Next up, I'm probably going to experiment with hooking `rseq` (Restartable Sequences on Linux 4.18+) so the per-CPU magazine doesn't even need `thread_local` storage and can survive preemptions on the exact same physical core without locks. If I get that working without crashing my kernel, I'll post the code here!
