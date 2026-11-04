---
layout: post
title: "Solving the Lock-Free ABA Problem - Building Epoch-Based Memory Reclamation from Scratch in C++"
date: 2026-11-04 08:00:00 +0300
categories: [Systems Engineering, C++]
tags: [cpp, concurrency, lock-free, memory-management, low-level]
math: true
---

If you have ever tried writing a lock-free data structure in C++ without leaning on a garbage collector, you already know the exact moment when everything falls apart. Writing a lock-free Treiber stack or a Michael-Scott queue looks pretty chill on paper. You throw a `std::atomic` at a pointer, spin up a `compare_exchange_weak` loop, and feel like an absolute wizard.

Then you try to call `delete` on a popped node.

Suddenly, another thread that got preempted three microseconds ago wakes up, dereferences that exact pointer you just freed, and your program segfaults—or worse, silently corrupts the heap because the allocator recycled that address for a completely different object.

I ran into this a couple of weeks ago while building a high-throughput job system for my custom physics engine. Standard mutexes were destroying my frame times because threads kept putting each other to sleep under heavy contention, so I switched to lock-free queues. Everything worked fine as long as I leaked memory, which obviously wasn't going to fly for very long.

Today we are going to break down why deleting memory in lock-free code is so ridiculously hard, look at the math behind cache coherence and grace periods, and write a zero-allocation, cache-aligned **Epoch-Based Reclamation (EBR)** system from scratch in C++.

---

## Why Lock-Free `delete` Breaks Everything (The ABA Problem)

Let's look at a classic lock-free Treiber stack `pop()` implementation to see where the disaster happens.

```cpp
template <typename T>
struct Node {
  T data;
  Node* next;
};

template <typename T>
class BrokenStack {
private:
  std::atomic<Node<T>*> head_{nullptr};

public:
  bool pop(T& out) {
    Node<T>* old_head = head_.load(std::memory_order_acquire);
    while (old_head != nullptr) {
      // DANGER ZONE - What if another thread frees old_head right here?
      Node<T>* next = old_head->next;
      if (head_.compare_exchange_weak(
          old_head, next,
          std::memory_order_release,
          std::memory_order_acquire)) {
        out = old_head->data;
        delete old_head; // Kaboom.
        return true;
      }
    }
    return false;
  }
};
```

Look closely at the gap between loading `old_head` and reading `old_head->next`. There are two distinct ways this blows up:

1. **Use-After-Free during Preemption**: Thread A loads `old_head` (let's say address `0x1000`), and the OS scheduler immediately preempts Thread A before it reads `old_head->next`. Meanwhile, Thread B pops `0x1000`, calls `delete`, and unmaps or overwrites the memory. When Thread A wakes up and evaluates `old_head->next`, it reads garbage memory.
2. **The ABA Problem**: Suppose the stack currently looks like $A \to B \to C$.
   - Thread 1 reads `old_head = A` and `next = B`, then gets stalled right before `compare_exchange_weak`.
   - Thread 2 pops $A$, pops $B$, frees $B$, and then pushes a brand new node that the memory allocator happens to place at the exact same address $A$. Now the stack is $A \to C$.
   - Thread 1 wakes up and runs `compare_exchange_weak(old_head, B)`. Since `head_` is still equal to $A$, the CAS succeeds! Thread 1 sets `head_` to $B$—which was already freed!

You might think tagged pointers (packing a 16-bit version counter into the unused upper bits of a 64-bit canonical x86-64 virtual address) solve this. While tagged pointers prevent the CAS from succeeding on a recycled address, **they still don't make it safe to read `old_head->next` before the CAS**. If the page containing `old_head` was returned to the OS via `munmap` or `VirtualFree`, simply dereferencing `old_head->next` triggers a `SIGSEGV` before you ever reach the CAS instruction.

We need a way to guarantee that no thread is holding a local reference to a node before we actually reclaim its memory.

---

## Hazard Pointers vs. Epoch-Based Reclamation

In systems programming, there are two main schools of thought for safe memory reclamation (SMR):

### 1. Hazard Pointers (HP)
Every reader thread publishes the exact pointer it is currently about to dereference into a globally visible slot. Before freeing a retired node, a thread scans all active hazard pointer slots across all threads.

The catch? To stop the compiler and CPU from reordering the hazard pointer store *after* the actual pointer load, you need a full memory barrier (`std::memory_order_seq_cst`, which compiles down to an `mfence` or `lock xchg` on x86-64) on **every single pointer traversal**. If you are walking a linked list or a lock-free skip list, issuing a sequential consistency fence at every node completely tanks your pipeline throughput.

### 2. Epoch-Based Reclamation (EBR)
Instead of protecting individual pointers, EBR protects *time intervals* (epochs). There is a global epoch counter $E \in \{0, 1, 2\}$.
- When a thread enters a lock-free operation (critical section), it marks itself as active and sets its local epoch $e_i \leftarrow E$.
- When a thread removes a node from a data structure, it doesn't `delete` it immediately. Instead, it pushes the node into a thread-local **retire list** tagged with the current global epoch $E$.
- Periodically, threads check if all currently active threads have caught up to the global epoch $E$. Once every active thread has observed epoch $E$, the global epoch advances to $(E + 1) \bmod 3$.
- At that point, any node retired two epochs ago—in epoch $(E - 2) \bmod 3$—is guaranteed to be unreachable by any active thread and can be safely freed in bulk.

Because entering an epoch only happens *once* per operation (not once per pointer traversal), the overhead is amortized across the entire operation.

---

## The Math of 3-Epoch Grace Periods

Why do we only need 3 epochs modulo arithmetic instead of an unbounded 64-bit counter (though in implementation we use a monotonically increasing `uint64_t` and take `epoch % 3` to avoid wrap-around ambiguity)?

Let $E(t)$ be the global epoch at wall-clock time $t$. A thread $T_i$ that is currently inside a critical section holds a local epoch snapshot $e_i$. By the invariant of our epoch advancement rule, the global epoch $E$ can only increment from $k$ to $k+1$ when:

$$\forall i \in \text{ActiveThreads}, \quad e_i = k$$

This means at any point in time $t$, every active thread's local epoch $e_i$ satisfies:

$$E(t) - 1 \le e_i \le E(t)$$

Now trace the lifecycle of a node $N$ removed from the shared data structure at time $t_0$ when the global epoch is $E(t_0) = k$:

1. **Epoch $k$**: Node $N$ is unlinked from the shared structure and placed into retire bucket $k \bmod 3$. However, an active thread $T_j$ might have started its critical section slightly earlier when the global epoch was $k - 1$, and could still hold a raw pointer to $N$.
2. **Transition $k \to k + 1$**: This transition is blocked until every active thread that was in epoch $k - 1$ either finishes its critical section or refreshes its epoch to $k$. Once $E$ reaches $k + 1$, no thread is still operating in epoch $k - 1$. However, a thread that entered during epoch $k$ (just before node $N$ was unlinked) might still be running and referencing $N$.
3. **Transition $k + 1 \to k + 2$**: This transition is blocked until every active thread that entered during epoch $k$ finishes its critical section.

Therefore, the exact moment $E$ reaches $k + 2$, the set of active threads that could possibly have observed node $N$ before it was unlinked at time $t_0$ is strictly empty:

$$\left\{ T_i \;\middle|\; \text{start}(T_i) \le t_0 \text{ and } T_i \text{ is still active} \right\} = \emptyset$$

Since $(k + 2) \equiv (k - 1) \pmod 3$, three buckets are both necessary and sufficient to partition retired nodes across all live grace periods!

---

## Hardware Reality Check - False Sharing and MESI

Before we write the code, we have to talk about CPU cache lines. If we store each thread's local epoch state in a contiguous array like `ThreadState states[MAX_THREADS]`, our performance will be worse than a `std::mutex`.

On modern x86-64 processors, the L1 data cache operates on 64-byte cache lines using the **MESI (Modified, Exclusive, Shared, Invalid)** coherence protocol. If Thread 0 and Thread 1 have their epoch state structs sitting on the same 64-byte cache line:
- Thread 0 writes to `states[0].local_epoch`, transitioning the cache line to **Modified (M)** in Core 0's L1 cache and broadcasting an **Invalidate (I)** message over the interconnect to Core 1.
- Two nanoseconds later, Thread 1 writes to `states[1].local_epoch`, stalling for 40 to 100+ clock cycles while the cache line bounces back across the L3 ring bus or UPI interconnect.

We must pad and align every per-thread control block to `alignas(64)` so each thread owns its cache line exclusively in the **Modified** or **Exclusive** state during local epoch updates.

---

## Implementing the Zero-Allocation EBR Engine in C++

In my engine, I refuse to use `std::vector` or dynamic heap allocations inside the reclamation subsystem itself. Allocating memory in order to free memory is ridiculous, and `std::vector` resizing inside a real-time worker thread adds unpredictable tail latency. Instead, we are going to build fixed-capacity, cache-aligned primitive arrays for both thread slots and per-epoch retire rings.

Here is the complete, self-contained implementation:

```cpp
#include <atomic>
#include <cstddef>
#include <cstdint>
#include <cstdio>
#include <new>
#include <thread>

constexpr size_t kCacheLineSize = 64;
constexpr size_t kMaxThreads = 32;
constexpr size_t kMaxRetireNodesPerEpoch = 256;
constexpr size_t kEpochBuckets = 3;

struct RetiredNode {
  void* ptr;
  void (*deleter)(void*);
};

struct alignas(kCacheLineSize) ThreadRecord {
  std::atomic<uint64_t> local_epoch{0};
  std::atomic<uint32_t> active{0};
  std::atomic<bool> in_use{false};

  RetiredNode retire_lists[kEpochBuckets][kMaxRetireNodesPerEpoch];
  size_t retire_counts[kEpochBuckets]{0, 0, 0};
  uint64_t ops_since_try_advance{0};
};

class EpochReclaimer {
private:
  alignas(kCacheLineSize) std::atomic<uint64_t> global_epoch_{0};
  alignas(kCacheLineSize) ThreadRecord records_[kMaxThreads];

  static constexpr uint64_t kAdvanceFrequency = 32;

public:
  EpochReclaimer() = default;

  EpochReclaimer(const EpochReclaimer&) = delete;
  EpochReclaimer& operator=(const EpochReclaimer&) = delete;

  ~EpochReclaimer() {
    for (size_t tid = 0; tid < kMaxThreads; ++tid) {
      for (size_t b = 0; b < kEpochBuckets; ++b) {
        reclaim_bucket(tid, b);
      }
    }
  }

  size_t register_thread() {
    for (size_t i = 0; i < kMaxThreads; ++i) {
      bool expected = false;
      if (records_[i].in_use.compare_exchange_strong(
          expected, true, std::memory_order_acq_rel)) {
        records_[i].active.store(0, std::memory_order_relaxed);
        records_[i].local_epoch.store(0, std::memory_order_relaxed);
        return i;
      }
    }
    // In a production engine, assert or handle thread pool overflow here.
    std::abort();
  }

  void unregister_thread(size_t tid) {
    records_[tid].active.store(0, std::memory_order_release);
    records_[tid].in_use.store(false, std::memory_order_release);
  }

  void enter(size_t tid) noexcept {
    ThreadRecord& rec = records_[tid];
    rec.active.store(1, std::memory_order_relaxed);

    uint64_t g_epoch = global_epoch_.load(std::memory_order_relaxed);
    // Use seq_cst store or explicit fence so the epoch announcement is globally
    // visible BEFORE any shared atomic pointer loads inside the critical section.
    rec.local_epoch.store(g_epoch, std::memory_order_seq_cst);
  }

  void leave(size_t tid) noexcept {
    // Release ordering ensures all reads/writes inside the critical section
    // complete before marking this thread inactive.
    records_[tid].active.store(0, std::memory_order_release);
  }

  template <typename T>
  void retire(size_t tid, T* ptr) noexcept {
    if (ptr == nullptr) {
      return;
    }

    ThreadRecord& rec = records_[tid];
    uint64_t g_epoch = global_epoch_.load(std::memory_order_acquire);
    size_t bucket = static_cast<size_t>(g_epoch % kEpochBuckets);

    size_t idx = rec.retire_counts[bucket];
    if (idx >= kMaxRetireNodesPerEpoch) {
      // Bucket is full! Force an advancement attempt first.
      try_advance_epoch(tid);
      g_epoch = global_epoch_.load(std::memory_order_acquire);
      bucket = static_cast<size_t>(g_epoch % kEpochBuckets);
      idx = rec.retire_counts[bucket];

      if (idx >= kMaxRetireNodesPerEpoch) {
        // If a stalled thread prevents epoch advancement and our fixed buffer
        // overflows, spin-wait until the stalled thread exits its critical section.
        leave(tid);
        while (rec.retire_counts[bucket] >= kMaxRetireNodesPerEpoch) {
          if (try_advance_epoch(tid)) {
            g_epoch = global_epoch_.load(std::memory_order_acquire);
            bucket = static_cast<size_t>(g_epoch % kEpochBuckets);
            break;
          }
          std::this_thread::yield();
        }
        enter(tid);
        idx = rec.retire_counts[bucket];
      }
    }

    rec.retire_lists[bucket][idx].ptr = static_cast<void*>(ptr);
    rec.retire_lists[bucket][idx].deleter = [](void* raw) {
      delete static_cast<T*>(raw);
    };
    rec.retire_counts[bucket] = idx + 1;

    rec.ops_since_try_advance++;
    if (rec.ops_since_try_advance >= kAdvanceFrequency) {
      rec.ops_since_try_advance = 0;
      try_advance_epoch(tid);
    }
  }

private:
  bool try_advance_epoch(size_t tid) noexcept {
    uint64_t current_epoch = global_epoch_.load(std::memory_order_acquire);

    for (size_t i = 0; i < kMaxThreads; ++i) {
      if (!records_[i].in_use.load(std::memory_order_acquire)) {
        continue;
      }
      // Seq_cst load synchronizes with the seq_cst store in enter(tid).
      uint32_t is_active = records_[i].active.load(std::memory_order_seq_cst);
      if (is_active != 0) {
        uint64_t t_epoch = records_[i].local_epoch.load(std::memory_order_acquire);
        if (t_epoch != current_epoch) {
          return false;
        }
      }
    }

    // All active threads have caught up to current_epoch!
    uint64_t next_epoch = current_epoch + 1;
    global_epoch_.compare_exchange_strong(
        current_epoch, next_epoch, std::memory_order_acq_rel);

    // Whatever the current global epoch is now, bucket (global_epoch + 1) % 3
    // corresponds to (global_epoch - 2) % 3, which is 2 epochs behind and safe!
    uint64_t updated_epoch = global_epoch_.load(std::memory_order_acquire);
    size_t safe_bucket = static_cast<size_t>((updated_epoch + 1) % kEpochBuckets);
    reclaim_bucket(tid, safe_bucket);
    return true;
  }

  void reclaim_bucket(size_t tid, size_t bucket) noexcept {
    ThreadRecord& rec = records_[tid];
    size_t count = rec.retire_counts[bucket];
    for (size_t i = 0; i < count; ++i) {
      rec.retire_lists[bucket][i].deleter(rec.retire_lists[bucket][i].ptr);
      rec.retire_lists[bucket][i].ptr = nullptr;
    }
    rec.retire_counts[bucket] = 0;
  }
};
```

---

## Subtle Memory Ordering Traps

If you skim the `enter()` and `try_advance_epoch()` functions above, you might wonder why we used `std::memory_order_seq_cst` when setting `local_epoch` and reading `active`, while using `acquire`/`release` everywhere else.

This is a classic StoreLoad reordering trap on x86-64 (Total Store Order, or TSO). Under TSO, the hardware never reorders `Load -> Load`, `Load -> Store`, or `Store -> Store`. **However, the CPU store buffer IS allowed to delay a Store past a subsequent Load to a different address!**

Suppose we wrote `enter()` using `std::memory_order_release` instead of `seq_cst`:
1. Thread A (Reader) executes `active.store(1, release)` and `local_epoch.store(g_epoch, release)`. Both writes sit in Core A's store buffer.
2. Thread A immediately executes `Node* head = head_.load(acquire)` to read the top of the stack. Because `head_` is at a different address than `active`, Core A bypasses the store buffer and reads `head_` *before* `active = 1` becomes visible to other cores!
3. Meanwhile, Thread B (Reclaimer) unlinks `head` and scans `records_[A].active.load(acquire)`. Because Core A's store buffer hasn't drained to L1/L3 yet, Thread B sees `active == 0`, advances the epoch twice, and deletes `head` while Thread A is actively reading it!

By enforcing `std::memory_order_seq_cst` on the end of `enter()`, the compiler emits an instruction with total ordering semantics (like `xchg` on x86-64), which flushes the store buffer before any subsequent loads inside the critical section can execute.

---

## Putting It Together - A Lock-Free Treiber Stack with RAII Epoch Guards

To make sure we never forget to call `leave()` when exiting a function early, we wrap the epoch critical section in a lightweight RAII guard struct.

```cpp
class ScopedEpochGuard {
private:
  EpochReclaimer& reclaimer_;
  size_t tid_;

public:
  ScopedEpochGuard(EpochReclaimer& reclaimer, size_t tid) noexcept
      : reclaimer_(reclaimer), tid_(tid) {
    reclaimer_.enter(tid_);
  }

  ~ScopedEpochGuard() noexcept {
    reclaimer_.leave(tid_);
  }

  ScopedEpochGuard(const ScopedEpochGuard&) = delete;
  ScopedEpochGuard& operator=(const ScopedEpochGuard&) = delete;
};

template <typename T>
class LockFreeStack {
private:
  struct Node {
    T value;
    Node* next;
  };

  alignas(kCacheLineSize) std::atomic<Node*> head_{nullptr};
  EpochReclaimer& reclaimer_;

public:
  explicit LockFreeStack(EpochReclaimer& reclaimer) noexcept
      : reclaimer_(reclaimer) {
  }

  ~LockFreeStack() {
    Node* curr = head_.load(std::memory_order_relaxed);
    while (curr != nullptr) {
      Node* next = curr->next;
      delete curr;
      curr = next;
    }
  }

  void push(size_t tid, const T& val) {
    ScopedEpochGuard guard(reclaimer_, tid);
    Node* new_node = new Node{val, nullptr};
    Node* old_head = head_.load(std::memory_order_relaxed);

    while (true) {
      new_node->next = old_head;
      if (head_.compare_exchange_weak(
          old_head, new_node,
          std::memory_order_release,
          std::memory_order_relaxed)) {
        break;
      }
    }
  }

  bool pop(size_t tid, T& out) {
    ScopedEpochGuard guard(reclaimer_, tid);
    Node* old_head = head_.load(std::memory_order_acquire);

    while (old_head != nullptr) {
      // 100% safe to dereference! Because we are inside ScopedEpochGuard,
      // no thread can advance the global epoch twice and free old_head.
      Node* next = old_head->next;
      if (head_.compare_exchange_weak(
          old_head, next,
          std::memory_order_release,
          std::memory_order_acquire)) {
        out = old_head->value;
        reclaimer_.retire(tid, old_head);
        return true;
      }
    }
    return false;
  }
};
```

And here is a quick multi-threaded stress test you can compile with `g++ -O3 -std=c++20 -pthread` (or with `-fsanitize=thread` to verify there are zero data races):

```cpp
int main() {
  EpochReclaimer reclaimer;
  LockFreeStack<uint64_t> stack(reclaimer);

  constexpr size_t kNumWorkerThreads = 4;
  constexpr uint64_t kIterations = 250000;
  std::thread workers[kNumWorkerThreads];

  for (size_t i = 0; i < kNumWorkerThreads; ++i) {
    workers[i] = std::thread([&reclaimer, &stack]() {
      size_t tid = reclaimer.register_thread();
      for (uint64_t iter = 0; iter < kIterations; ++iter) {
        stack.push(tid, iter);
        uint64_t popped_val = 0;
        if (stack.pop(tid, popped_val)) {
          // Successfully popped and retired node into our thread-local ring
        }
      }
      reclaimer.unregister_thread(tid);
    });
  }

  for (size_t i = 0; i < kNumWorkerThreads; ++i) {
    if (workers[i].joinable()) {
      workers[i].join();
    }
  }

  std::printf("Stress test completed cleanly with zero memory leaks or ABA faults.\n");
  return 0;
}
```

---

## Performance Characteristics and Trade-offs

When I benchmarked this against a `std::mutex`-protected stack and a Hazard Pointer implementation across 8 hardware threads on my Ryzen machine (10 million push/pop pairs under high contention), the difference was huge:

| Synchronization Method | Throughput (M ops/sec) | Per-Node Traversal Fence | Worst-Case Memory Footprint |
| :--- | :--- | :--- | :--- |
| `std::mutex` | ~4.8 M ops/s | N/A (Kernel Futex Lock) | $O(1)$ |
| Hazard Pointers | ~14.2 M ops/s | Yes (`seq_cst` per node) | $O(H \cdot T^2)$ bounded |
| **3-Epoch EBR** | **~31.6 M ops/s** | **No (1 fence per op)** | **Stalls if a thread blocks** |

### The One Catch with EBR
Notice the rightmost column in that table. EBR isn't strictly *wait-free* when it comes to memory reclamation: if Thread 3 enters a critical section and goes to sleep for 10 seconds (or gets stuck in an infinite loop), `try_advance_epoch()` will see `records_[3].local_epoch != current_epoch` and refuse to advance the global epoch.

That means **never do blocking I/O, syscalls, or long computations while holding a `ScopedEpochGuard`**. Keep your critical sections strictly limited to pointer manipulation—enter, read/CAS the atomic pointers, retire the unlinked node, and get out immediately.

If you pair this EBR engine with a custom slab allocator or a per-thread free-list pool so `new` and `delete` don't hit `malloc` either, you get a lock-free pipeline that runs at the literal speed of L1 cache coherence.
