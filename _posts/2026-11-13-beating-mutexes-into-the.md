---
layout: post
title: "Beating Mutexes into the Ground - Designing a Wait-Free SeqLock Ring Buffer in C++"
date: 2026-11-13 08:00:00 +0300
categories: [Systems Engineering, Low-Level C++]
tags: [cpp, concurrency, lock-free, atomics, optimization]
math: true
---

Last weekend I was profiling a market-data replay engine I've been building in my spare time, and I hit a wall that honestly annoyed me more than it should have. My parser was chewing through raw PCAP packets in around 18 nanoseconds, which was pretty sweet, except the thread handoff between the network poller and my strategy evaluator was spiking to 4 microsecond tail latencies. Four microseconds! That's basically an eternity when you're trying to keep everything inside the L1/L2 cache hierarchy.

At first I was using a standard single-producer single-consumer (SPSC) queue with `std::atomic` head and tail pointers. That works fine when you have exactly one reader. The problem started when I wanted three separate consumer threads—a logger, a risk checker, and the main execution loop—reading the exact same telemetry stream simultaneously without slowing down the writer. If you slap a `std::mutex` or even a `std::shared_mutex` on that, your cache lines start ping-ponging across cores via the MESI protocol, and your throughput falls off a cliff. Even traditional lock-free multi-producer multi-consumer (MPMC) queues force readers to perform Compare-And-Swap (`CAS` / `lock cmpxchg` on x86) operations on a shared read index. That means readers actually fight *each other* for exclusive cache line ownership just to read data.

I didn't want readers mutating shared state at all. If a reader crashes or gets preempted by the Linux kernel scheduler in the middle of a read, the writer shouldn't care, and the other readers shouldn't care either.

The solution I ended up building is an intrusive, zero-allocation **SeqLock (Sequence Lock) broadcast ring buffer** backed by raw aligned arrays and careful C++20 memory ordering. Let's tear down how it works, why the CPU hardware acts the way it does, and how to write one without invoking Undefined Behavior (which is surprisingly tricky in C++).

---

## Why Reader-Writer Locks Lie to You

When people first hear "one writer, multiple readers," they immediately reach for `pthread_rwlock_t` or `std::shared_mutex`. Here is the dirty secret of reader-writer locks: **reading still requires writing to memory**.

To acquire a read lock, a thread has to atomically increment an internal `reader_count` integer inside the lock structure. On x86_64, that compiles down to a `lock xadd` instruction. Under the hood, your CPU core has to broadcast an RFO (Request For Ownership) across the interconnect, invalidating that cache line in every other core's L1 cache.

If we model the coherence traffic latency when $N$ reader cores constantly poll a shared lock structure, the serialization bottleneck scales horribly:

$$
T_{\text{coherence} } \approx N \cdot \left( t_{\text{L3\_hit} } + t_{\text{RFO\_invalidate} } \right)
$$

Even worse, if a new burst of data comes in while five readers are holding the shared lock, the writer is stuck spinning until `reader_count` drops to zero. In a real-time telemetry or trading pipeline, blocking the producer because a background logging thread got context-switched out is completely unacceptable. We need the writer to be strictly **wait-free**—meaning it finishes its write in a bounded number of instructions, every single time, regardless of what the readers are doing.

---

## Enter the SeqLock (Sequence Lock)

The Linux kernel has used SeqLocks for decades (originally to update the system `jiffies` and wall-clock time in the `vDSO` page so `clock_gettime()` doesn't need a syscall). The idea is almost stupidly simple:

1. Maintain a monotonically increasing 64-bit sequence counter `seq`, initialized to `0`.
2. Before the writer touches the payload, it increments `seq` by 1. Now `seq` is **odd** (indicating a write is currently in progress).
3. The writer copies the new payload into the slot.
4. The writer increments `seq` by 1 again. Now `seq` is **even** (indicating the data is stable and consistent).

Now look at what the reader does:
1. Read `seq1 = seq`. If `seq1` is odd (`seq1 & 1`), a write is actively happening right now. Spin and retry.
2. Copy the payload into a local stack variable.
3. Issue an acquire/load memory fence so the payload reads cannot be reordered *after* step 4.
4. Read `seq2 = seq`. If `seq1 == seq2`, nobody touched the slot while we were reading it! Return the local copy. Otherwise, our read was torn, so loop back to step 1.

Notice the magic here: **readers never write to shared memory**. Their cache lines stay in the **Shared (S)** state in the MESI protocol. Fifty cores can read the same slot simultaneously at full L1 cache bandwidth with zero bus lock contention.

---

## The C++ Memory Model Catch (Torn Reads Are Undefined Behavior!)

Okay, if you try to write a naive SeqLock in standard C++ by just reading a plain struct while the writer writes to that struct, someone on a language-lawyer forum is going to yell at you. And technically, they're right.

According to the C++ standard (`[intro.races]`), if one thread writes to a non-atomic memory location while another thread reads from it without synchronization, it is a **data race**, which is instant Undefined Behavior (UB). It doesn't matter that you throw away the corrupted struct when `seq1 != seq2`! If the compiler sees a non-atomic read, it is allowed to assume the value never changes mid-read. For instance, it could reload a field from memory twice instead of caching it in a register, or if your struct has a jump-table index, a torn read might cause an out-of-bounds jump before you even get to the `seq1 == seq2` check.

To make a SeqLock 100% legal in C++ without sacrificing speed, we have to copy the payload in 8-byte chunks using `std::atomic<uint64_t>` with `std::memory_order_relaxed`. On x86_64 and ARM64, a relaxed 64-bit aligned atomic load/store compiles down to a plain `mov` or `ldr`/`str` instruction—literally zero overhead compared to `memcpy`, while telling the compiler: *"Hey, another thread might be mutating these bytes right now, don't do weird register-reloading optimizations."*

Let's also talk about cache line geometry. Modern CPUs fetch memory in 64-byte chunks (or 128 bytes on Apple Silicon, and effectively 128 bytes on Intel when the spatial hardware prefetcher is enabled). If two slots in a ring buffer share a cache line, the writer updating slot $i$ will invalidate the cache line for a reader currently inspecting slot $i-1$. That's **false sharing**, and it will silently wreck your benchmarks.

---

## Building the Cache-Aligned SeqLock Slot

Let's write the core slot structure. Notice how I'm avoiding `std::vector` or dynamic heap allocations here—everything is statically sized, aligned to 64-byte cache lines, and uses raw primitive arrays so the layout is completely predictable in memory.

```cpp
#include <atomic>
#include <cstdint>
#include <cstring>
#include <new>
#include <type_traits>

#if defined(__x86_64__) || defined(_M_X64)
#include <immintrin.h>
#define CPU_PAUSE() _mm_pause()
#elif defined(__aarch64__)
#define CPU_PAUSE() asm volatile("yield" ::: "memory")
#else
#define CPU_PAUSE()
#endif

constexpr size_t kCacheLineSize = 64;

template <typename T>
struct alignas(kCacheLineSize) SeqLockSlot {
  static_assert(std::is_trivially_copyable_v<T>,
                "Payload T must be trivially copyable for tear-safe atomic words.");

  static constexpr size_t kWordCount = (sizeof(T) + sizeof(uint64_t) - 1) / sizeof(uint64_t);

  std::atomic<uint64_t> seq{0};
  std::atomic<uint64_t> words[kWordCount];

  void write(const T& item) noexcept {
    const uint64_t s = seq.load(std::memory_order_relaxed);
    
    // Step 1 - Mark slot as odd (write in progress).
    // Release fence ensures the odd sequence update is visible before payload stores.
    seq.store(s + 1, std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_release);

    // Step 2 - Copy payload into 64-bit raw buffer, then store atomically.
    alignas(uint64_t) uint64_t raw_buf[kWordCount];
    if constexpr (sizeof(T) % sizeof(uint64_t) != 0) {
      raw_buf[kWordCount - 1] = 0;
    }
    std::memcpy(raw_buf, &item, sizeof(T));

    for (size_t i = 0; i < kWordCount; ++i) {
      words[i].store(raw_buf[i], std::memory_order_relaxed);
    }

    // Step 3 - Publish even sequence number with release semantics.
    seq.store(s + 2, std::memory_order_release);
  }

  bool try_read(T& out, uint64_t expected_seq) const noexcept {
    const uint64_t s1 = seq.load(std::memory_order_acquire);
    if (s1 != expected_seq) {
      return false;
    }

    alignas(uint64_t) uint64_t raw_buf[kWordCount];
    for (size_t i = 0; i < kWordCount; ++i) {
      raw_buf[i] = words[i].load(std::memory_order_relaxed);
    }

    // Crucial fence - prevents the compiler or CPU from hoisting the second
    // seq load above the relaxed payload loads!
    std::atomic_thread_fence(std::memory_order_acquire);

    const uint64_t s2 = seq.load(std::memory_order_relaxed);
    if (s1 != s2) {
      return false;
    }

    std::memcpy(&out, raw_buf, sizeof(T));
    return true;
  }
};
```

Look closely at `write()` and `try_read()`. There is a subtle detail in `write()` that trips up a lot of people when they first mess with `std::memory_order`.

Why did I do `seq.store(s + 1, std::memory_order_relaxed)` followed by `std::atomic_thread_fence(std::memory_order_release)` instead of just `seq.store(s + 1, std::memory_order_release)`?

Because of how **release semantics** are defined in the C++ memory model! A `store(memory_order_release)` prevents memory operations *before* the store from being reordered *after* it. It does **not** prevent stores *after* it (like our `words[i].store(...)` calls) from floating *up* above `seq.store(s + 1)`! If the compiler or CPU reordered `words[0].store()` to happen before `seq` became odd, a reader could read a mutated `words[0]` while still seeing the old even sequence number!

Putting `std::atomic_thread_fence(std::memory_order_release)` *after* `seq.store(s + 1)` forces `seq.store(s + 1)` to commit before any subsequent `words[i].store()` can become visible to other cores. On x86_64 (Total Store Order, or TSO), stores are never reordered with other stores in hardware, so that fence compiles to a zero-cost compiler barrier (`asm volatile("" ::: "memory")`). On weakly-ordered architectures like ARM64, it emits a `dmb ishst` (Data Memory Barrier - Store/Store), which is still way cheaper than a full barrier.

---

## Designing the Broadcast Ring Buffer Without Head/Tail Pointers

In a traditional queue, you have a `write_idx` and a `read_idx`. In a broadcast ring buffer with $K$ independent readers and a power-of-two capacity $C = 2^m$, we don't even need a shared `write_idx` if we encode the lap number directly into the sequence counter of each slot!

Wait—think about how cool that is. Suppose our ring buffer has capacity $C = 1024$ slots. Every time the writer writes its $k$-th message (where $k = 0, 1, 2, ...$), which slot does it hit?

$$
\text{slot\_index}(k) = k \bmod C = k \ \& \ (C - 1)
$$

And what should the sequence counter inside that specific slot be after message $k$ is written?

Instead of incrementing each slot's sequence number by 2 (`0 -> 2 -> 4`), what if we set the slot's sequence state directly from the global ticket $k$?
- While writing message $k$, the slot's `seq` is set to $(k \cdot 2) + 1$ (odd).
- Once message $k$ is finished, the slot's `seq` is set to $(k + 1) \cdot 2$ (even).

Let's trace that! Suppose $C = 4$:
- Slot 0 starts at `seq = 0`.
- Writer writes message $k = 0$ into Slot 0: `seq` becomes $1$ (odd), then $2$ (even).
- Writer writes message $k = 1$ into Slot 1: `seq` becomes $3$ (odd), then $4$ (even).
- Writer writes message $k = 2$ into Slot 2: `seq` becomes $5$ (odd), then $6$ (even).
- Writer writes message $k = 3$ into Slot 3: `seq` becomes $7$ (odd), then $8$ (even).
- Now the writer wraps around! It writes message $k = 4$ into Slot 0: `seq` becomes $9$ (odd), then $10$ (even).

Let's verify what this gives a reader waiting for message $k$:
1. The reader looks directly at `slots[k & (C - 1)]`. It doesn't even have to read a central `write_cursor` cache line first! That eliminates an entire cache line bounce per message!
2. For message $k$, the expected completed sequence number in that slot is:

$$
S_{\text{expected} }(k) = (k + 1) \cdot 2
$$

3. If the reader sees `s1 < S_expected(k)` (and `s1` is even), message $k$ hasn't been written yet! The reader knows immediately to wait.
4. If `s1 == S_expected(k)`, message $k$ is sitting right there in the slot! The reader pulls the words, checks `s1 == s2`, and advances its local `next_ticket` to $k + 1$.
5. What if the reader was too slow (e.g., preempted by the OS) and the writer lapped the entire ring buffer? Then `s1 > S_expected(k)` (or `s1 == (k + n*C) * 2 + 1`). The reader immediately detects the overrun, calculates the latest available ticket right from `s1`, and jumps forward without corrupting anything!

Let's check the math for recovering the writer's ticket when a slow reader gets lapped at slot $idx = k \ \& \ (C - 1)$. If the slot currently holds an even sequence value $s_1 > (k + 1) \cdot 2$, then the message currently sitting in that slot is:

$$
k_{\text{latest} } = \frac{s_1}{2} - 1
$$

Because $s_1 = (k_{\text{latest} } + 1) \cdot 2$. That's literally a single right-shift (`(s1 >> 1) - 1`). No divisions, no branches, no shared state.

---

## The Full Implementation

Let's put the whole broadcast ring buffer together. Again, notice how we use a fixed-size primitive array `Slot slots_[Capacity]` instead of `std::vector` so the entire ring buffer can be mapped directly into a single `mmap` shared-memory segment (`MAP_SHARED | MAP_HUGETLB`) if we want to use it for inter-process communication (IPC).

```cpp
#include <atomic>
#include <cstddef>
#include <cstdint>
#include <cstring>
#include <type_traits>

enum class ReadResult : uint8_t {
  kSuccess = 0,
  kEmpty = 1,
  kOverrun = 2
};

template <typename T, size_t Capacity>
class SeqLockRingBuffer {
  static_assert((Capacity != 0) && ((Capacity & (Capacity - 1)) == 0),
                "Capacity must be a power of 2 for bitwise modulo masking.");
  static_assert(std::is_trivially_copyable_v<T>,
                "Payload T must be trivially copyable.");

private:
  static constexpr size_t kMask = Capacity - 1;
  static constexpr size_t kWordCount = (sizeof(T) + sizeof(uint64_t) - 1) / sizeof(uint64_t);

  struct alignas(64) Slot {
    std::atomic<uint64_t> seq{0};
    std::atomic<uint64_t> words[kWordCount];
  };

  alignas(64) uint64_t write_ticket_{0};
  alignas(64) Slot slots_[Capacity];

public:
  constexpr SeqLockRingBuffer() noexcept = default;

  // Single producer push - strictly wait-free, zero CAS loops.
  void publish(const T& item) noexcept {
    const uint64_t ticket = write_ticket_++;
    Slot& slot = slots_[ticket & kMask];

    // Odd sequence indicates active write on this ticket.
    slot.seq.store((ticket << 1) | 1ULL, std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_release);

    alignas(uint64_t) uint64_t raw_buf[kWordCount];
    if constexpr (sizeof(T) % sizeof(uint64_t) != 0) {
      raw_buf[kWordCount - 1] = 0;
    }
    std::memcpy(raw_buf, &item, sizeof(T));

    for (size_t i = 0; i < kWordCount; ++i) {
      slot.words[i].store(raw_buf[i], std::memory_order_relaxed);
    }

    // Even sequence indicates ticket is published and stable.
    slot.seq.store((ticket + 1ULL) << 1, std::memory_order_release);
  }

  // Non-blocking reader poll. Pass 'cursor' by reference; it increments on success
  // or snaps forward to the newest valid ticket on overrun.
  ReadResult try_consume(uint64_t& cursor, T& out) const noexcept {
    const Slot& slot = slots_[cursor & kMask];
    const uint64_t expected_seq = (cursor + 1ULL) << 1;

    while (true) {
      const uint64_t s1 = slot.seq.load(std::memory_order_acquire);

      if (s1 < expected_seq) {
        // If s1 == (cursor << 1) | 1ULL, the writer is actively writing our exact
        // ticket right now! Spin briefly instead of returning kEmpty immediately.
        if (s1 == ((cursor << 1) | 1ULL)) {
          CPU_PAUSE();
          continue;
        }
        return ReadResult::kEmpty;
      }

      if (s1 > expected_seq) {
        // The writer lapped us! If s1 is odd, a lap write is in progress right now.
        if ((s1 & 1ULL) != 0) {
          // Snap cursor to the ticket currently being written and spin on it.
          cursor = s1 >> 1;
          return ReadResult::kOverrun;
        }
        // s1 is even and newer than expected_seq; snap cursor to the ticket in this slot.
        cursor = (s1 >> 1) - 1ULL;
        return ReadResult::kOverrun;
      }

      // s1 == expected_seq! Copy the payload words optimistically.
      alignas(uint64_t) uint64_t raw_buf[kWordCount];
      for (size_t i = 0; i < kWordCount; ++i) {
        raw_buf[i] = slot.words[i].load(std::memory_order_relaxed);
      }

      std::atomic_thread_fence(std::memory_order_acquire);
      const uint64_t s2 = slot.seq.load(std::memory_order_relaxed);

      if (s1 == s2) {
        std::memcpy(&out, raw_buf, sizeof(T));
        ++cursor;
        return ReadResult::kSuccess;
      }

      // If s1 != s2, the writer wrapped all the way around Capacity slots and
      // overwrote our slot mid-read. Loop around so the overrun logic handles it.
      CPU_PAUSE();
    }
  }
};
```

---

## Inspecting the Generated Assembly

Whenever you write low-level concurrency code in C++, you shouldn't trust your intuition until you actually check the assembly output from `g++ -O3 -march=native`.

Suppose we instantiate `SeqLockRingBuffer<MarketTick, 1024>` where `MarketTick` is a 32-byte struct (four 64-bit words):

```cpp
struct MarketTick {
  uint64_t timestamp_ns;
  uint64_t instrument_id;
  int64_t price_mantissa;
  uint64_t quantity;
};
```

Look at what `publish()` compiles down to on x86_64:

```nasm
; rdi = this, rsi = const MarketTick& item
mov     rax, QWORD PTR [rdi]          ; load write_ticket_
lea     rdx, [rax+1]
mov     QWORD PTR [rdi], rdx          ; write_ticket_++
mov     rcx, rax
and      ecx, 1023                     ; slot index = ticket & 1023
shl     rcx, 6                        ; multiply by 64 (sizeof(Slot))
add     rdi, rcx
lea     rcx, [rax+rax+1]              ; (ticket << 1) | 1
mov     QWORD PTR [rdi+64], rcx       ; slot.seq = odd (mark write in progress)
mov     rcx, QWORD PTR [rsi]          ; copy 4 words of MarketTick
mov     QWORD PTR [rdi+72], rcx
mov     rcx, QWORD PTR [rsi+8]
mov     QWORD PTR [rdi+80], rcx
mov     rcx, QWORD PTR [rsi+16]
mov     QWORD PTR [rdi+88], rcx
mov     rcx, QWORD PTR [rsi+24]
mov     QWORD PTR [rdi+96], rcx
shl     rdx, 1                        ; (ticket + 1) << 1
mov     QWORD PTR [rdi+64], rdx       ; slot.seq = even (publish!)
ret
```

Wait—look at that assembly! There is **not a single `lock` prefix or `mfence` instruction anywhere**!

Why? Because on x86_64, hardware Total Store Order (TSO) guarantees that stores are visible to other cores in the exact order they were issued. Every store on x86 naturally has release semantics, and every load naturally has acquire semantics (as long as you don't need Store-Load ordering, which would require `mfence`). By carefully structuring our SeqLock so that the writer only does Store-Store ordering (`seq = odd` $\to$ `words = ...` $\to$ `seq = even`) and the reader only does Load-Load ordering (`s1 = seq` $\to$ `... = words` $\to$ `s2 = seq`), all of our C++20 acquire/release fences vanish into pure compiler barriers at compile time!

---

## Hardware Prefetcher Pitfalls and Stride Tuning

When I first benchmarked this on my Ryzen 7 machine (pinning the producer to Core 2 and three readers to Cores 3, 4, and 5 via `pthread_setaffinity_np`), I noticed something weird. Latency was around **28 nanoseconds** per hop when the writer published every 10 microseconds, which is basically the hardware interconnect speed between CCXs/cores.

However, when I cranked the producer up to an artificial stress test of 50 million messages per second, tail latency occasionally spiked to 190 nanoseconds.

I ran `perf stat -e cache-misses,L1-dcache-load-misses` and figured out why: **the L2 spatial hardware prefetcher**.

On modern Intel and AMD CPUs, the L2 streamer/spatial prefetcher watches sequential memory accesses and automatically pulls the *next* cache line ($N + 1$) into L2. While a reader was spinning on `slots_[i]` waiting for `seq` to turn even, the writer was finishing `slots_[i]` and immediately dirtying `slots_[i + 1]`. Because `slots_[i]` and `slots_[i + 1]` were adjacent 64-byte cache lines inside the same 128-byte aligned pair, the hardware prefetcher on the reader core was speculatively pulling `slots_[i + 1]` into the Shared state right as the writer wanted Exclusive ownership of it!

The fix is super simple: change `alignas(64)` on `Slot` to `alignas(128)`.

Yes, padding each slot to 128 bytes wastes 88 bytes per slot if your struct is only 32 bytes (plus the 8-byte `seq` counter). For a 1024-slot ring buffer, that bumps the total footprint from 64 KB to 128 KB—which still easily fits inside a single core's 512 KB or 1 MB L2 cache, while completely eliminating adjacent-line prefetcher collisions.

Let's compare the benchmark numbers across 10 million messages with 1 producer thread and 3 concurrent reader threads pinned to isolated cores (`isolcpus` kernel boot parameter enabled):

| Synchronization Mechanism | Median Latency (p50) | 99.9th Percentile (p99.9) | Max Latency (p99.999) |
| :--- | :--- | :--- | :--- |
| `std::mutex` + Deque | 1,420 ns | 14,800 ns | 68,400 ns |
| `std::shared_mutex` (RWLock) | 890 ns | 9,210 ns | 41,000 ns |
| Lock-Free CAS MPMC Queue | 145 ns | 620 ns | 3,150 ns |
| **SeqLock Ring Buffer (64B align)** | **31 ns** | **185 ns** | **410 ns** |
| **SeqLock Ring Buffer (128B align)** | **26 ns** | **48 ns** | **112 ns** |

26 nanoseconds median and 48 nanoseconds at the 99.9th percentile across three concurrent readers! And the writer loop never slows down even if you send `SIGSTOP` to one of the reader processes.

---

## When Should You *Not* Use a SeqLock?

As awesome as SeqLocks are, they aren't a silver bullet. You have to keep three constraints in mind before dropping this into a codebase:

1. **Payloads must be `trivially_copyable` (Plain Old Data).** You cannot put a `std::string` or `std::shared_ptr` inside a SeqLock slot. Why? Because during a torn read, the reader might read half of a pointer before the `s1 == s2` check fails. If a destructor or copy constructor runs on that half-written pointer, your program segfaults immediately. Stick to flat structs with fixed-size arrays (`char symbol[8]`) and primitive integers/floats.
2. **Single Writer Only (unless you wrap the writer side).** A pure SeqLock assumes only one writer is incrementing `seq`. If you have multiple writers, they have to coordinate via a spinlock or a CAS ticket before writing, which brings back writer-to-writer contention (though readers remain completely wait-free).
3. **Overruns are lossy by design.** If a reader falls behind by more than `Capacity` messages, its slot gets overwritten. For real-time sensor fusion, high-frequency market data book snapshots, or telemetry, dropping stale updates and snapping to the latest state is usually *preferred* over applying backpressure and stalling the producer. If you need guaranteed delivery of every single historical event without loss, either size the ring buffer large enough to absorb your worst-case OS jitter, or pair the fast path with a fallback disk journal.

Writing lock-free code that works with the CPU cache coherence protocol instead of fighting it is honestly one of the most satisfying parts of systems programming. Once you stop thinking of memory as a flat array of bytes and start seeing it as 64-byte state machines trading MESI messages over a silicon ring bus, C++ becomes a completely different game.
