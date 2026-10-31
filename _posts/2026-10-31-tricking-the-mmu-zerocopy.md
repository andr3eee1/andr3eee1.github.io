---
layout: post
title: "Tricking the MMU - Zero-Copy Ring Buffers with Virtual Memory Aliasing in C++"
date: 2026-10-31 08:00:00 +0300
categories: [Systems Engineering, Low-Level C++]
tags: [cpp, linux, mmap, lock-free, memory-management]
math: true
---

If you have ever written a network packet parser, an audio DSP pipeline, or a high-frequency logging engine, you have definitely written a ring buffer. And if you wrote that ring buffer the standard way, you immediately ran into the most annoying edge case in systems programming: the wrap-around boundary.

Say you have a 64 KB buffer and your write head is sitting at byte $65,500$. You want to push a 128-byte packet, or maybe cast the next 64 bytes directly into a `struct PacketHeader*` so you can read the fields without copying anything. With a normal array, you cannot do that. The first 36 bytes live at the end of the array, and the remaining 92 bytes wrap around to index $0$. Suddenly, your "zero-copy" parser has to allocate a temporary scratch buffer, do two separate `memcpy` calls, stitch the split payload back together, and completely ruin your L1 instruction and data cache locality.

A few weeks ago, while debugging a throughput bottleneck in a custom UDP telemetry ingestion engine I was building, I got tired of writing branchy split-read logic. It turns out we can completely delete the wrap-around branch in software by abusing how the CPU's Memory Management Unit (MMU) and the Linux kernel handle virtual-to-physical page translations.

By mapping two adjacent virtual memory ranges to the exact same underlying physical pages, we can make the end of the ring buffer seamlessly flow into the beginning of the ring buffer in virtual address space.

Here is how the hardware mechanics actually work, how to build a production-grade implementation on Linux using `memfd_create` and `mmap`, and how to pair it with lock-free acquire/release atomics without shooting yourself in the foot via false sharing.

---

## The Problem with Modulo Arithmetic and Split Reads

Let's look at a naive ring buffer first. Usually, you have a fixed-size byte array on the stack or in a static segment, and you advance your `head` and `tail` indices modulo the capacity $N$.

First off, if $N$ is not a power of two, the compiler emits a `div` instruction for the modulo operator `%`. On modern x86-64 architectures (like Zen 4 or Golden Cove), a 64-bit integer division can still take anywhere from 10 to 18 cycles, compared to a single cycle for a bitwise AND:

$$
\text{index} = \text{offset} \bmod 2^k = \text{offset} \ \& \ (2^k - 1)
$$

Even when we force $N = 2^k$, the wrap-around memory layout problem remains. Consider a ring buffer of capacity $N$ where the current read offset is $R$. The number of contiguous bytes available before hitting the physical end of the array is:

$$
B_{\text{contig}} = N - (R \ \& \ (N - 1))
$$

If we want to read a message of length $L$ where $L > B_{\text{contig}}$, our code is forced to take a slow branch:

```cpp
void read_naive(const uint8_t* ring, size_t capacity, size_t read_pos, uint8_t* dest, size_t len) {
  size_t mask = capacity - 1;
  size_t offset = read_pos & mask;
  size_t first_chunk = capacity - offset;

  if (len > first_chunk) {
    // Awkward split across the boundary
    memcpy(dest, ring + offset, first_chunk);
    memcpy(dest + first_chunk, ring, len - first_chunk);
  } else {
    memcpy(dest, ring + offset, len);
  }
}
```

This is fine if you are already copying data out into a separate array (`dest`). Still, it is terrible if you want **in-place zero-copy access**—for example, passing a contiguous `const uint8_t*` slice directly to a SIMD AVX2 parsing routine (`_mm256_loadu_si256`) or casting a pointer to a protocol header struct. You cannot hand an AVX2 intrinsic a pointer that splits halfway across the end of an array and wraps back to index zero.

---

## Hardware Page Tables to the Rescue

Every pointer you touch in a user-space C++ program is a delusion created by the OS and the hardware MMU. On 64-bit x86 systems using standard 4-level paging (PML4, Page Directory Pointer, Page Directory, and Page Table), a 48-bit virtual address is carved up like this:

* **Bits 0–11 (12 bits):** The byte offset inside a $4096$-byte ($2^{12}$) page.
* **Bits 12–20 (9 bits):** Page Table Entry (PTE) index.
* **Bits 21–29 (9 bits):** Page Directory (PMD) index.
* **Bits 30–38 (9 bits):** Page Directory Pointer (PUD) index.
* **Bits 39–47 (9 bits):** Page Map Level 4 (PML4) index.

Nothing in the x86-64 architecture says that two different Page Table Entries cannot point to the exact same physical frame in RAM. In fact, shared libraries (`libc.so`) and inter-process shared memory rely on this all the time.

If we allocate a backing physical buffer of size $N$ (where $N$ is a multiple of the system page size, $4096$ bytes), we can ask the kernel for a contiguous virtual address space of size $2N$. Then, we map the first half $[V, V + N)$ to our physical pages, and we map the second half $[V + N, V + 2N)$ to the **exact same** physical pages.

```
Virtual Address Space (Contiguous 2N bytes):
+-----------------------------------+-----------------------------------+
| Virtual Region 1: [V, V + N)      | Virtual Region 2: [V + N, V + 2N) |
+-----------------------------------+-----------------------------------+
                  \                                   /
                   \                                 /
                    v                               v
                +---------------------------------------+
                | Physical RAM Backing (N bytes)        |
                +---------------------------------------+
```

Now look at what happens when a packet starts at offset $N - 16$ and is $64$ bytes long.
In virtual memory, we read from `ptr + (N - 16)` through `ptr + (N + 48)`.
To our C++ code and any SIMD instruction, those 64 bytes are completely linear and contiguous in memory. Under the hood, as soon as the pointer crosses `ptr + N`, the MMU translates virtual address $V + N + k$ to the exact same physical RAM cell as $V + k$.

---

## Dealing with Race Conditions During Mapping

If you try to allocate two adjacent virtual blocks by just calling `mmap(NULL, N, ...)` twice, the kernel's address space layout randomizer (ASLR) or existing heap allocations might stick them in totally different parts of your virtual address space, or another thread in your program might call `malloc` right between your two `mmap` calls and steal the address range right after the first block.

To make this rock-solid and atomic against concurrent allocations in our own process, we use a classic three-step dance:

1. **Create an anonymous in-memory file** using `memfd_create` and size it to $N$ bytes using `ftruncate`. This gives us a file descriptor representing our physical page backing without touching the disk.
2. **Reserve $2N$ bytes of contiguous virtual address space** using a single `mmap` call with `PROT_NONE` and `MAP_ANONYMOUS | MAP_PRIVATE`. Because this happens in one syscall, the kernel guarantees the entire $2N$ range is contiguous and belongs to us.
3. **Overwrite both halves** of that $2N$ reservation using `mmap` with the `MAP_FIXED` flag, pointing both halves `[base, base + N)` and `[base + N, base + 2N)` to our `memfd` file descriptor at offset $0$.

People online sometimes panic when they see `MAP_FIXED` because if you pass an arbitrary address to `MAP_FIXED`, you will silently clobber whatever mappings already existed there. However, when you pass `MAP_FIXED` onto a region you *just reserved yourself* via an anonymous `mmap`, it is completely safe and is actually the POSIX-sanctioned way to atomically replace reservations.

Let's write the low-level initialization logic in C++.

```cpp
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdint.h>
#include <stddef.h>
#include <string.h>
#include <atomic>
#include <stdio.h>

struct MirroredArena {
  uint8_t* base_ptr;
  size_t capacity;
  int fd;
};

bool init_mirrored_arena(MirroredArena* arena, size_t requested_capacity) {
  long page_size_raw = sysconf(_SC_PAGESIZE);
  if (page_size_raw <= 0) {
    return false;
  }
  size_t page_size = static_cast<size_t>(page_size_raw);

  // Capacity must be a power of 2 AND a multiple of the OS page size (usually 4096)
  if ((requested_capacity & (requested_capacity - 1)) != 0) {
    return false;
  }
  if (requested_capacity < page_size) {
    return false;
  }

  // 1. Create an anonymous physical backing file in RAM
  int fd = memfd_create("ring_buffer_backing", MFD_CLOEXEC);
  if (fd < 0) {
    return false;
  }

  if (ftruncate(fd, static_cast<off_t>(requested_capacity)) != 0) {
    close(fd);
    return false;
  }

  // 2. Reserve a contiguous 2 * capacity virtual address range
  void* reservation = mmap(
    nullptr,
    requested_capacity * 2,
    PROT_NONE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1,
    0
  );

  if (reservation == MAP_FAILED) {
    close(fd);
    return false;
  }

  uint8_t* base = static_cast<uint8_t*>(reservation);

  // 3. Map the first half [base, base + N) to the physical backing
  void* first_half = mmap(
    base,
    requested_capacity,
    PROT_READ | PROT_WRITE,
    MAP_SHARED | MAP_FIXED,
    fd,
    0
  );

  if (first_half == MAP_FAILED) {
    munmap(base, requested_capacity * 2);
    close(fd);
    return false;
  }

  // 4. Map the second half [base + N, base + 2N) to the EXACT SAME backing
  void* second_half = mmap(
    base + requested_capacity,
    requested_capacity,
    PROT_READ | PROT_WRITE,
    MAP_SHARED | MAP_FIXED,
    fd,
    0
  );

  if (second_half == MAP_FAILED) {
    munmap(base, requested_capacity * 2);
    close(fd);
    return false;
  }

  arena->base_ptr = base;
  arena->capacity = requested_capacity;
  arena->fd = fd;
  return true;
}

void destroy_mirrored_arena(MirroredArena* arena) {
  if (arena->base_ptr != nullptr) {
    munmap(arena->base_ptr, arena->capacity * 2);
    arena->base_ptr = nullptr;
  }
  if (arena->fd >= 0) {
    close(arena->fd);
    arena->fd = -1;
  }
}
```

Notice that as soon as both `mmap` calls finish, you could technically `close(fd)` right inside `init_mirrored_arena` and the mappings would stay alive until `munmap`. Keeping the file descriptor around is only necessary if you later want to pass it over a Unix domain socket (`SCM_RIGHTS`) to another process for zero-copy IPC.

---

## Designing the Lock-Free SPSC Ring Buffer

Now that our memory wraps around automatically in hardware, we want to build a Single-Producer Single-Consumer (SPSC) lock-free queue on top of it. Even with a mirrored virtual buffer, it is surprisingly easy to tank performance if you ignore CPU cache coherency protocols (MESI / MOESI).

### 1. Monotonically Increasing Indices

A common beginner mistake in ring buffers is wrapping the `head` and `tail` variables themselves back to `0` every time they cross `capacity`. If you do that, when `head == tail`, you cannot tell if the buffer is completely empty or completely full without wasting a slot or keeping a separate atomic counter.

Instead, we store `write_pos` and `read_pos` as unsigned 64-bit integers (`uint64_t`) that **only ever increment**.
At 10 Gigabits per second, a `uint64_t` byte counter will take over 460 years to overflow. And even if it does overflow, unsigned arithmetic in C++ is defined modulo $2^{64}$, which is cleanly divisible by our power-of-two capacity $2^k$.

Thus, at any point in time:
* **Bytes currently in the buffer:** $\text{size} = \text{write\_pos} - \text{read\_pos}$
* **Free space remaining:** $\text{free} = \text{capacity} - (\text{write\_pos} - \text{read\_pos})$
* **Pointer to read/write:** $\text{base\_ptr} + (\text{pos} \ \& \ (\text{capacity} - 1))$

Because the second half of our $2N$ virtual space mirrors the first half, as long as we never try to read or write a single contiguous chunk larger than `capacity`, any pointer offset $\text{pos} \ \& \ (\text{capacity} - 1)$ can safely read up to `capacity` contiguous bytes forward without ever checking for the end of the buffer!

### 2. Eliminating False Sharing and Redundant Cache Traffic

On modern x86-64 and ARM64 CPUs, cache lines are 64 bytes wide. If `write_pos` and `read_pos` sit next to each other in the same struct:

```cpp
struct BadLayout {
  std::atomic<uint64_t> write_pos;
  std::atomic<uint64_t> read_pos;
};
```

Every time the producer thread updates `write_pos`, its CPU core has to invalidate that 64-byte cache line in the consumer core's L1/L2 cache (transitioning the line to the Modified/Invalid state in MESI). When the consumer tries to read `read_pos` a nanosecond later, it stalls waiting for a cross-core cache line transfer—even though `read_pos` did not even change!

Even worse: the producer needs to read `read_pos` to check if the buffer is full, and the consumer needs to read `write_pos` to check if data is available. If the producer checks the atomic `read_pos` on every single push, it constantly pulls the consumer's cache line across the interconnect.

The fix is two-fold:
1. Pad the producer and consumer state onto separate 64-byte cache lines (`alignas(64)`).
2. Give the producer a local, non-atomic **cached shadow copy** of `read_pos` (`cached_read_pos`), and give the consumer a local cached shadow copy of `write_pos` (`cached_write_pos`).

The producer only touches the real atomic `read_pos` when its local `cached_read_pos` says the ring buffer looks full! Most of the time, the buffer isn't full, so the producer runs completely inside its own private L1 cache line with zero cross-core atomic traffic.

```cpp
struct alignas(64) ZeroCopyRing {
  uint8_t* base_ptr;
  size_t capacity;
  size_t mask;
  int fd;

  // Producer cache line
  alignas(64) std::atomic<uint64_t> write_pos;
  uint64_t cached_read_pos;

  // Consumer cache line
  alignas(64) std::atomic<uint64_t> read_pos;
  uint64_t cached_write_pos;
};
```

Let's write out the zero-copy claim/commit API. Instead of copying bytes inside the ring buffer methods, we expose `write_acquire` (which returns a raw `uint8_t*` directly into the ring buffer where the caller can write or `snprintf` or DMA straight into) and `write_commit` (which publishes the bytes to the consumer).

```cpp
bool ring_init(ZeroCopyRing* ring, size_t capacity) {
  MirroredArena arena;
  if (!init_mirrored_arena(&arena, capacity)) {
    return false;
  }

  ring->base_ptr = arena.base_ptr;
  ring->capacity = arena.capacity;
  ring->mask = arena.capacity - 1;
  ring->fd = arena.fd;

  ring->write_pos.store(0, std::memory_order_relaxed);
  ring->cached_read_pos = 0;

  ring->read_pos.store(0, std::memory_order_relaxed);
  ring->cached_write_pos = 0;
  return true;
}

void ring_destroy(ZeroCopyRing* ring) {
  MirroredArena arena;
  arena.base_ptr = ring->base_ptr;
  arena.capacity = ring->capacity;
  arena.fd = ring->fd;
  destroy_mirrored_arena(&arena);
}

// Returns a contiguous pointer with at least `len` bytes available to write,
// or nullptr if the ring buffer does not have enough space.
uint8_t* ring_write_acquire(ZeroCopyRing* ring, size_t len) {
  if (len > ring->capacity) {
    return nullptr;
  }

  uint64_t w = ring->write_pos.load(std::memory_order_relaxed);
  uint64_t available_space = ring->capacity - (w - ring->cached_read_pos);

  if (available_space < len) {
    // Only reload the consumer's atomic read_pos when our cached value says we are full
    ring->cached_read_pos = ring->read_pos.load(std::memory_order_acquire);
    available_space = ring->capacity - (w - ring->cached_read_pos);
    if (available_space < len) {
      return nullptr;
    }
  }

  return ring->base_ptr + (w & ring->mask);
}

// Publishes `len` written bytes so the consumer can see them.
void ring_write_commit(ZeroCopyRing* ring, size_t len) {
  uint64_t w = ring->write_pos.load(std::memory_order_relaxed);
  ring->write_pos.store(w + len, std::memory_order_release);
}

// Returns a contiguous pointer to at least `len` readable bytes,
// or nullptr if fewer than `len` bytes are currently available.
const uint8_t* ring_read_acquire(ZeroCopyRing* ring, size_t len) {
  uint64_t r = ring->read_pos.load(std::memory_order_relaxed);
  uint64_t available_data = ring->cached_write_pos - r;

  if (available_data < len) {
    // Only reload the producer's atomic write_pos when our cached value isn't enough
    ring->cached_write_pos = ring->write_pos.load(std::memory_order_acquire);
    available_data = ring->cached_write_pos - r;
    if (available_data < len) {
      return nullptr;
    }
  }

  return ring->base_ptr + (r & ring->mask);
}

// Marks `len` bytes as consumed, freeing space for the producer.
void ring_read_commit(ZeroCopyRing* ring, size_t len) {
  uint64_t r = ring->read_pos.load(std::memory_order_relaxed);
  ring->read_pos.store(r + len, std::memory_order_release);
}
```

Look closely at the memory orderings in `ring_write_commit` and `ring_read_acquire`:
* When the producer writes payload bytes into the pointer returned by `ring_write_acquire`, those are normal non-atomic stores to the ring buffer.
* `ring_write_commit` updates `write_pos` using `std::memory_order_release`. This guarantees that neither the compiler nor the CPU (especially on weakly-ordered architectures like ARM64) can reorder the payload writes *after* the update to `write_pos`.
* `ring_read_acquire` reads `write_pos` using `std::memory_order_acquire`, creating a happens-before edge. However, reading its *own* position (`w` for the producer, `r` for the consumer) only needs `std::memory_order_relaxed` because no other thread ever mutates that variable.

---

## Wait—What About Virtual Aliasing and the TLB?

Whenever I show this trick to people, the first question is usually: *"Doesn't having two virtual addresses point to the same physical address break CPU caches or trash the TLB?"*

This is where knowing how modern CPU caches are built really pays off.

### VIPT vs. PIPT Caches

Modern x86-64 CPUs use **VIPT (Virtually Indexed, Physically Tagged)** L1 data caches, and **PIPT (Physically Indexed, Physically Tagged)** L2 and L3 caches.

In a PIPT cache (L2/L3), the cache hardware looks up lines using the *physical* address after translation. Whether you access byte $0$ via virtual address $V$ or virtual address $V + N$, both resolve to physical address $P$, so they hit the exact same L2/L3 cache line.

In a VIPT L1 cache, the index bits are extracted from the *virtual* address in parallel with the TLB lookup so the CPU doesn't have to wait for address translation before indexing the cache SRAM. If bits outside the 12-bit page offset ($0\text{--}11$) were used to index the cache and differed between $V$ and $V + N$, you could get **virtual cache synonyms** (two L1 entries holding the same physical memory).

However, on virtually all modern x86-64 processors (Intel since Nehalem, AMD Zen series), the L1 data cache is designed such that either:
1. Each way is $4\text{ KB}$ (e.g., $32\text{ KB}$ 8-way associative: $32768 / 8 = 4096$), meaning the cache index bits come *strictly* from bits $0\text{--}11$ of the address—which are identical between virtual and physical addresses!
2. Or, for $48\text{ KB}$ 12-way caches, the hardware coerces synonyms in hardware without software intervention.

What if our ring buffer capacity $N$ is a multiple of a larger power of two, say $64\text{ KB}$? Then bits $0\text{--}15$ of $V$ and $V + N$ are identical anyway, avoiding even theoretical VIPT index mismatches on exotic architectures.

### The Real Cost: Extra TLB Entries

There is one tiny hardware cost you *do* pay: Translation Lookaside Buffer (TLB) entries.
The TLB caches virtual-to-physical page translations. Because $V$ and $V + N$ are distinct virtual pages, when a write wraps across the boundary at $V + N$, the CPU has to populate a TLB entry for the first page of the second mapping $[V + N, V + N + 4096)$, even though the entry for $[V, V + 4096)$ is already in the L1 dTLB.

In practice, only reads or writes that *actually straddle* the exact wrap boundary ever touch the first page of the second mapping $[V + N, V + N + 4096)$. So at any given moment, this trick consumes at most **one extra 4 KB dTLB entry** out of the ~64 entries in an L1 dTLB (and ~1,536+ in the L2 STLB). Trading a single TLB slot to eliminate every wrap-around branch and temporary scratch copy in your hot path is a trade I will take every single day.

---

## Proving Zero-Copy In-Place Parsing Works

To see why this makes protocol code so much cleaner, imagine we have a binary wire protocol where every packet starts with an 8-byte header followed by a variable-length payload, and we want to compute a fast checksum over the packet in-place right across the boundary of a 4096-byte ring buffer.

```cpp
struct WireHeader {
  uint32_t magic;
  uint16_t payload_len;
  uint16_t sequence;
};

int main() {
  ZeroCopyRing ring;
  if (!ring_init(&ring, 4096)) {
    return 1;
  }

  // Advance the ring buffer to byte 4090 (6 bytes before the 4096 physical end!)
  uint8_t* dummy = ring_write_acquire(&ring, 4090);
  ring_write_commit(&ring, 4090);
  ring_read_commit(&ring, 4090);

  // Now write a 32-byte packet starting at offset 4090.
  // The WireHeader itself (8 bytes) will split right across byte 4096!
  size_t packet_size = sizeof(WireHeader) + 24;
  uint8_t* out_ptr = ring_write_acquire(&ring, packet_size);
  if (out_ptr != nullptr) {
    WireHeader* hdr = reinterpret_cast<WireHeader*>(out_ptr);
    hdr->magic = 0xDEADBEEF;
    hdr->payload_len = 24;
    hdr->sequence = 42;

    char* payload = reinterpret_cast<char*>(out_ptr + sizeof(WireHeader));
    memcpy(payload, "Hello across the page!!", 24);

    ring_write_commit(&ring, packet_size);
  }

  // Consumer reads the packet directly as a contiguous struct pointer
  // without caring that it crossed the 4096-byte physical boundary.
  const uint8_t* in_ptr = ring_read_acquire(&ring, sizeof(WireHeader));
  if (in_ptr != nullptr) {
    const WireHeader* hdr = reinterpret_cast<const WireHeader*>(in_ptr);
    size_t total_len = sizeof(WireHeader) + hdr->payload_len;

    const uint8_t* full_pkt = ring_read_acquire(&ring, total_len);
    if (full_pkt != nullptr) {
      const char* msg = reinterpret_cast<const char*>(full_pkt + sizeof(WireHeader));
      printf("Seq %u | Magic 0x%X | Payload - %s\n", hdr->sequence, hdr->magic, msg);
      ring_read_commit(&ring, total_len);
    }
  }

  ring_destroy(&ring);
  return 0;
}
```

Look at what happened at offset `4090`:
The 8-byte `WireHeader` started at `base_ptr + 4090` and ended at `base_ptr + 4098`.
Specifically, `hdr->magic` (bytes 4090–4093) and `hdr->payload_len` (bytes 4094–4095) lived at the very end of the physical page, while `hdr->sequence` (bytes 4096–4097) automatically wrapped into the first 2 bytes of the physical page via the second virtual mapping.
We dereferenced `hdr->sequence` directly through a raw struct pointer with zero branches, zero `memcpy` calls, and zero scratch buffers.

When you stop fighting the hardware and start using the MMU the way it was designed to work, a lot of "mandatory" software complexity just vanishes.
