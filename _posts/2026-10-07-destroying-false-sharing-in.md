---
layout: post
title: "Destroying False Sharing in Lock-Free Ring Buffers - A Deep Dive"
date: 2026-10-07 08:00:00 +0300
categories: [systems, cpp]
tags: [optimization, low-level, threading, memory]
math: true
---

So I was putting off my AP Calc homework the other night, and tbh I went down this insane rabbit hole on lock-free data structures. Ngl, most of the tutorials out there kinda miss the actual hardware details. Like, they show you the C++ code for an SPSC (Single-Producer Single-Consumer) queue and call it a day. But if you actually benchmark it on a multi-core setup, it runs like absolute garbage. 

Why? Because of this sneaky little hardware quirk called *false sharing*. It is literally the bane of multithreaded systems engineering. Let's break it down.

### The Problem with Naive Queues

When you're writing a high-performance lock-free queue, you generally want to avoid heap-allocated things like `std::vector`. They introduce unnecessary indirection and allocation overhead. We are writing systems-level code here, so we're just going to use a raw primitive array for maximum performance. No dynamic sizing, just pure speed.

Here is how you might intuitively write a basic lock-free ring buffer:

```cpp
#include <atomic>
#include <iostream>
#include <thread>

const int RING_SIZE = 1000000;

struct NaiveQueue {
  int buffer[RING_SIZE]; // raw primitive array, zero heap allocation
  std::atomic<int> head;
  std::atomic<int> tail;

  NaiveQueue() {
    head.store(0);
    tail.store(0);
  }

  bool push(int val) {
    int curr = tail.load(std::memory_order_relaxed);
    int next = (curr + 1) % RING_SIZE;
    if (next == head.load(std::memory_order_acquire)) {
      return false; // queue is full
    }
    buffer[curr] = val;
    tail.store(next, std::memory_order_release);
    return true;
  }

  bool pop(int& val) {
    int curr = head.load(std::memory_order_relaxed);
    if (curr == tail.load(std::memory_order_acquire)) {
      return false; // queue is empty
    }
    val = buffer[curr];
    head.store((curr + 1) % RING_SIZE, std::memory_order_release);
    return true;
  }
};
```

Looks totally fine, right? The logic is sound. We have a producer pushing to the `tail` and a consumer popping from the `head`. We're even using proper memory ordering (`acquire`/`release`) instead of sequential consistency to save cycles. 

But if you run this with the producer on CPU Core 0 and the consumer on CPU Core 1, the performance will totally tank.

### Enter the Cache Line

Your CPU doesn't fetch memory one byte at a time. It's way too slow to talk to main memory that often. Instead, the CPU fetches memory in chunks called *cache lines*. On almost all modern x86_64 and ARM processors, a cache line is exactly 64 bytes.

Look at our `NaiveQueue` struct again. The `head` and `tail` atomic variables are just 4 bytes each (standard `int` size), and they are declared right next to each other in memory. 

This means `head` and `tail` almost certainly live on the **exact same 64-byte cache line**.

When Core 0 (producer) updates `tail`, the CPU's cache coherence protocol (usually MESI) goes into overdrive. It has to invalidate that entire cache line across all other cores to ensure data consistency. 
So when Core 1 (consumer) tries to read `head`, it gets a cache miss because Core 0 invalidated the line! Core 1 is forced to fetch the line from L3 cache or main memory again. 

Then Core 1 updates `head`, which invalidates the line for Core 0. They are just endlessly playing ping-pong with this poor cache line. This is *false sharing*. They aren't even writing to the same variable, but because they share a cache line, they lock each other up.

Let's look at the math for the latency penalty:

$$ 
T_{\text{latency}} = (N_{\text{hits}} \times t_{\text{L1}}) + (N_{\text{misses}} \times t_{\text{RAM wait...}}) 
$$

A typical L1 cache hit is like ~1 ns. A main memory fetch because of an invalidated cache line can be ~100 ns. If your loop is tight, you are multiplying your execution time by 100x just because two variables are sitting too close to each other on the silicon.

### The Chad Solution: Padding

The fix is actually mind-blowingly simple. We just force the compiler to put `head` and `tail` on separate cache lines. We can do this in C++11 and later using the `alignas` keyword.

```cpp
struct BasedQueue {
  int buffer[RING_SIZE];
  
  // Align head to its own 64-byte cache line
  alignas(64) std::atomic<int> head;
  
  // Align tail to the NEXT 64-byte cache line
  alignas(64) std::atomic<int> tail;

  BasedQueue() {
    head.store(0);
    tail.store(0);
  }

  bool push(int val) {
    int curr = tail.load(std::memory_order_relaxed);
    int next = (curr + 1) % RING_SIZE;
    if (next == head.load(std::memory_order_acquire)) {
      return false;
    }
    buffer[curr] = val;
    tail.store(next, std::memory_order_release);
    return true;
  }

  bool pop(int& val) {
    int curr = head.load(std::memory_order_relaxed);
    if (curr == tail.load(std::memory_order_acquire)) {
      return false;
    }
    val = buffer[curr];
    head.store((curr + 1) % RING_SIZE, std::memory_order_release);
    return true;
  }
};
```

By slapping `alignas(64)` on those atomics, we pad out the struct. Yes, we are technically wasting a few bytes of memory between `head` and `tail`, but RAM is insanely cheap and CPU cycles are not. 

When you run `BasedQueue`, Core 0 modifies `tail` on Cache Line A, and Core 1 modifies `head` on Cache Line B. The MESI protocol doesn't trip, no false sharing happens, and your throughput skyrockets. 

If you want to test it yourself, throw it in a simple benchmark loop:

```cpp
void run_producer(BasedQueue* q) {
  for (int i = 0; i < 10000000; ++i) {
    while (!q->push(i)) {
      std::this_thread::yield();
    }
  }
}

void run_consumer(BasedQueue* q) {
  int val = 0;
  for (int i = 0; i < 10000000; ++i) {
    while (!q->pop(val)) {
      std::this_thread::yield();
    }
  }
}
```

Try swapping out `BasedQueue` for `NaiveQueue` and timing the threads with `std::chrono`. The difference is literally night and day.

Anyway, that's enough systems programming for one night. I genuinely gotta go finish this Calculus worksheet before first period tomorrow. Peace!
