---
layout: post
title: "Writing a Zero-Allocation x86-64 Fiber Scheduler From Scratch in C++"
date: 2026-10-28 08:00:00 +0300
categories: [Systems Engineering, C++]
tags: [cpp, assembly, concurrency, x86-64, low-level]
math: true
---

A couple of weeks ago I was profiling a hobby game server I've been building, and I noticed something kind of annoying in `perf`. Even though my actual packet processing logic took maybe a few hundred nanoseconds, the OS kernel was burning a ridiculous amount of CPU cycles just doing thread context switches and `futex` wakes whenever contention spiked.

Standard OS threads (`pthread` or `std::thread`) are great when you have long-running parallel work, because the Linux scheduler preempts them automatically. Wait too long on a socket? Boom, the kernel saves your registers, flushes TLB state if it jumps processes, runs `schedule()`, and swaps in another thread. The problem is that crossing the privilege boundary into ring 0 via a syscall and letting the kernel scheduler figure out life takes anywhere from $1.2\ \mu\text{s}$ to $3.5\ \mu\text{s}$ on modern x86-64 hardware, plus all the L1/L2 cache pollution that comes with it.

If you already know exact points where your code needs to yield—like waiting on an `io_uring` completion or a job dependency—you don't actually need the kernel to referee. You can do **cooperative multitasking** entirely in user space using fibers (stackful coroutines). And the crazy part is that an entire user-space context switch on x86-64 only needs to touch 6 general-purpose registers and the stack pointer. We're talking around $15\text{ ns}$ per switch.

Here is how I built a zero-allocation, fixed-memory fiber scheduler in C++ and raw x86-64 assembly without touching `std::vector`, heap allocations on the hot path, or `ucontext.h`.

---

## Why `ucontext.h` and `setjmp` Are Too Slow

When people first try writing coroutines in C, they usually reach for `<ucontext.h>` (`makecontext` / `swapcontext`) or `setjmp` / `longjmp`.

The issue with `swapcontext` is that it's designed to be completely generic, so it invokes the `rt_sigprocmask` system call on every single switch to save and restore the thread's POSIX signal mask. That means you're right back to paying the ring-0 syscall penalty you were trying to avoid in the first place!

We can calculate the throughput difference pretty easily. Suppose we have $N$ fibers yielding $K$ times per frame in a $60\text{ Hz}$ tick loop (so a frame budget of $16.67\text{ ms}$). The total scheduling overhead $T_{\text{sched}}$ is:

$$
T_{\text{sched}} = N \cdot K \cdot \left( t_{\text{switch}} + t_{\text{queue}} \right)
$$

If $N = 1000$ fibers and each yields $K = 10$ times per frame:
* With `swapcontext` ($t_{\text{switch}} \approx 850\text{ ns}$), $T_{\text{sched}} \approx 8.5\text{ ms}$, eating over half of our entire frame budget just swapping contexts.
* With a custom assembly switcher ($t_{\text{switch}} \approx 14\text{ ns}$), $T_{\text{sched}} \approx 0.14\text{ ms}$, which is basically noise.

---

## Understanding the System V AMD64 ABI

To switch execution from Fiber A to Fiber B inside a C++ function call, we don't need to save every single register on the CPU. Because a context switch happens via an explicit function call (`switch_context(&old_sp, new_sp)`), the compiler *already* assumes that all **caller-saved (volatile)** registers will get trashed by the function call anyway.

According to the System V AMD64 ABI (used on Linux, macOS, and BSD):

| Register Group | Registers | Responsibility |
| :--- | :--- | :--- |
| **Caller-Saved (Volatile)** | `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`–`r11`, `xmm0`–`xmm15` | Compiler saves them before calling `switch_context` if it still needs them. |
| **Callee-Saved (Non-Volatile)** | `rbx`, `rbp`, `r12`, `r13`, `r14`, `r15` | Our assembly routine **must** preserve these across the switch. |
| **Stack & Instruction** | `rsp`, `rip` | `rip` is pushed onto the stack automatically by the `call` instruction; `rsp` is saved to our fiber struct. |

This is super clean. When Fiber A calls `switch_context(uint64_t** old_sp, uint64_t* new_sp)`:
1. The CPU executes `call switch_context`, which pushes the return address (`rip`) onto Fiber A's stack.
2. Inside `switch_context`, we push the 6 callee-saved registers (`rbx`, `rbp`, `r12`–`r15`) onto Fiber A's stack.
3. We store Fiber A's current stack pointer (`rsp`) into `*old_sp` (which is passed in `rdi`).
4. We overwrite `rsp` with `new_sp` (which is passed in `rsi`), immediately tele-porting onto Fiber B's stack.
5. We pop the 6 callee-saved registers off Fiber B's stack.
6. We execute `ret`, which pops Fiber B's saved return address off Fiber B's stack into `rip`.

Just like that, the CPU is now running Fiber B as if `switch_context` simply returned.

---

## The 16-Byte Stack Alignment Trap

There is one massive footgun in the x86-64 ABI that broke my code with a mysterious `SIGSEGV` the first time my fiber tried calling `printf`.

The System V ABI mandates that **before a `call` instruction is executed, the stack pointer `rsp` must be 16-byte aligned** ($\text{rsp} \equiv 0 \pmod{16}$). When `call` executes, it pushes the 8-byte return address onto the stack, meaning that at the exact entry point of any C/C++ function, $\text{rsp} \equiv 8 \pmod{16}$.

Why does this matter? Because GCC and Clang emit SSE/AVX instructions like `movaps` (Move Aligned Packed Single-Precision Floating-Point Values) to zero out local structs or save vector registers on the stack. If `rsp` is off by 8 bytes when entering a C++ bootstrap function, `movaps` faults immediately with a General Protection Fault (`#GP`).

Let's trace the math when we initialize a brand-new fiber stack from a raw byte array. If the top of our stack buffer is aligned to 16 bytes:

$$
\text{Addr}_{\text{top}} \equiv 0 \pmod{16}
$$

When our fiber starts for the very first time, `switch_context` will pop 6 registers ($6 \times 8 = 48$ bytes) and then `ret` will pop the entry point address ($8$ bytes). If we only place the entry point address and the 6 registers at the top of the stack, right after `ret` executes, `rsp` will be at $\text{Addr}_{\text{top}}$. Wait—at the entry of a function, `rsp` is supposed to end in `8`, not `0`! Why? Because a normal function is entered via `call`, which pushes an 8-byte return address after a 16-byte boundary.

So when laying out a fake initial stack frame for a new fiber, we have to leave an 8-byte dummy slot (or a killer trap return address) above the entry point `rip` so that when `ret` pops the entry `rip`, the resulting `rsp` satisfies:

$$
\text{rsp}_{\text{entry}} = \text{Addr}_{\text{top}} - 8 \equiv 8 \pmod{16}
$$

---

## Writing the Context Switcher and Scheduler in C++

To keep memory deterministic and cache-friendly, we're not touching `std::vector` or `new` during scheduling. We pre-allocate a static arena of stacks and fiber control blocks inside our scheduler struct, along with a power-of-two ring buffer for the ready queue.

Here is the complete, self-contained implementation:

```cpp
#include <cstdint>
#include <cstdio>

// Configuration constants - tuned to fit comfortably in L2/L3 cache
constexpr uint32_t MAX_FIBERS = 64;
constexpr uint32_t STACK_SIZE = 32768; // 32 KB per fiber

enum class FiberState : uint8_t {
  INACTIVE = 0,
  READY = 1,
  RUNNING = 2,
  FINISHED = 3
};

using FiberEntryFn = void (*)(void*);

// Layout of the saved context on the top of a paused fiber's stack.
// Because x86 stacks grow downward, the last register pushed is at the lowest address.
struct SavedStackFrame {
  uint64_t r15;
  uint64_t r14;
  uint64_t r13;
  uint64_t r12;
  uint64_t rbp;
  uint64_t rbx;
  uint64_t rip;        // Popped by 'ret' inside switch_context
  uint64_t pad_return; // Ensures rsp % 16 == 8 upon entering 'rip'
};

struct Fiber {
  uint64_t* rsp;
  FiberEntryFn entry;
  void* user_data;
  uint32_t id;
  FiberState state;
};

// Raw naked assembly context switch for System V x86-64 ABI.
// rdi = uint64_t** old_sp
// rsi = uint64_t*  new_sp
extern "C" void switch_context(uint64_t** old_sp, uint64_t* new_sp);

__asm__(
  ".global switch_context\n"
  ".type switch_context, @function\n"
  "switch_context:\n"
  "  pushq %rbx\n"
  "  pushq %rbp\n"
  "  pushq %r12\n"
  "  pushq %r13\n"
  "  pushq %r14\n"
  "  pushq %r15\n"
  "  movq %rsp, (%rdi)\n"
  "  movq %rsi, %rsp\n"
  "  popq %r15\n"
  "  popq %r14\n"
  "  popq %r13\n"
  "  popq %r12\n"
  "  popq %rbp\n"
  "  popq %rbx\n"
  "  retq\n"
);

class FiberScheduler {
public:
  uint64_t* main_rsp;
  uint32_t current_fiber_idx;
  uint32_t active_count;

  // Fixed-size ring buffer queue (MAX_FIBERS is a power of 2)
  uint32_t ready_queue[MAX_FIBERS];
  uint32_t queue_head;
  uint32_t queue_tail;

  Fiber fibers[MAX_FIBERS];
  alignas(16) uint8_t stack_arena[MAX_FIBERS][STACK_SIZE];

  void init() {
    main_rsp = nullptr;
    current_fiber_idx = 0;
    active_count = 0;
    queue_head = 0;
    queue_tail = 0;

    for (uint32_t i = 0; i < MAX_FIBERS; ++i) {
      fibers[i].rsp = nullptr;
      fibers[i].entry = nullptr;
      fibers[i].user_data = nullptr;
      fibers[i].id = i;
      fibers[i].state = FiberState::INACTIVE;
    }
  }

  void enqueue_ready(uint32_t fiber_id) {
    uint32_t index = queue_tail & (MAX_FIBERS - 1);
    ready_queue[index] = fiber_id;
    queue_tail = queue_tail + 1;
  }

  bool dequeue_ready(uint32_t* out_fiber_id) {
    if (queue_head == queue_tail) {
      return false;
    }
    uint32_t index = queue_head & (MAX_FIBERS - 1);
    *out_fiber_id = ready_queue[index];
    queue_head = queue_head + 1;
    return true;
  }

  int32_t spawn(FiberEntryFn fn, void* arg) {
    int32_t slot = -1;
    for (uint32_t i = 0; i < MAX_FIBERS; ++i) {
      if (fibers[i].state == FiberState::INACTIVE ||
          fibers[i].state == FiberState::FINISHED) {
        slot = static_cast<int32_t>(i);
        break;
      }
    }

    if (slot < 0) {
      return -1;
    }

    Fiber* f = &fibers[slot];
    f->entry = fn;
    f->user_data = arg;
    f->state = FiberState::READY;

    // Carve out the initial stack frame at the top of the 16-byte aligned buffer
    uint8_t* stack_top = &stack_arena[slot][0] + STACK_SIZE;
    SavedStackFrame* frame = reinterpret_cast<SavedStackFrame*>(
      stack_top - sizeof(SavedStackFrame)
    );

    frame->r15 = 0;
    frame->r14 = 0;
    frame->r13 = 0;
    frame->r12 = 0;
    frame->rbp = 0;
    frame->rbx = 0;
    frame->rip = reinterpret_cast<uint64_t>(&FiberScheduler::fiber_trampoline);
    frame->pad_return = 0xDEADBEEFCAFEBABEULL;

    f->rsp = reinterpret_cast<uint64_t*>(frame);

    active_count = active_count + 1;
    enqueue_ready(static_cast<uint32_t>(slot));
    return slot;
  }

  void yield() {
    uint32_t prev_idx = current_fiber_idx;
    Fiber* prev_fiber = &fibers[prev_idx];

    if (prev_fiber->state == FiberState::RUNNING) {
      prev_fiber->state = FiberState::READY;
      enqueue_ready(prev_idx);
    }

    uint32_t next_idx = 0;
    if (dequeue_ready(&next_idx)) {
      current_fiber_idx = next_idx;
      Fiber* next_fiber = &fibers[next_idx];
      next_fiber->state = FiberState::RUNNING;

      if (next_idx != prev_idx) {
        switch_context(&prev_fiber->rsp, next_fiber->rsp);
      }
    } else {
      // No other fibers ready; return control to the main OS thread
      switch_context(&prev_fiber->rsp, main_rsp);
    }
  }

  void run_until_complete() {
    while (active_count > 0) {
      uint32_t next_idx = 0;
      if (!dequeue_ready(&next_idx)) {
        break;
      }

      current_fiber_idx = next_idx;
      Fiber* f = &fibers[next_idx];
      f->state = FiberState::RUNNING;

      switch_context(&main_rsp, f->rsp);
    }
  }

  static void fiber_trampoline() {
    // Execute the user callback on the active fiber
    Fiber* current = &g_scheduler.fibers[g_scheduler.current_fiber_idx];
    current->entry(current->user_data);

    // Mark finished and jump to the next ready fiber (or back to main)
    current->state = FiberState::FINISHED;
    g_scheduler.active_count = g_scheduler.active_count - 1;
    g_scheduler.yield();
  }

  static FiberScheduler g_scheduler;
};

FiberScheduler FiberScheduler::g_scheduler;
```

---

## Walking Through the Trampoline Trick

Look closely at `FiberScheduler::spawn`. Notice that we don't set `frame->rip` directly to the user's function `fn`. Instead, we point `frame->rip` to `FiberScheduler::fiber_trampoline`.

There are two reasons why jumping directly to `fn` wouldn't work:
1. **Passing Arguments:** In the System V ABI, the first argument to a function is passed in the `rdi` register. Our `switch_context` assembly routine only pops `r15` through `rbx`—it doesn't touch `rdi` because adding `pushq %rdi` / `popq %rdi` to `switch_context` would slightly slow down *every single yield* just to help the initial startup.
2. **Graceful Exit:** What happens when the user's function hits its closing brace `}` and executes `ret`? If there isn't a caller underneath it on the stack, `ret` will pop our `0xDEADBEEFCAFEBABE` padding value into `rip` and immediately segfault.

By routing execution through `fiber_trampoline()`, the trampoline looks up the currently running fiber from the scheduler, loads `current->user_data` into `rdi` through a normal C++ function call `current->entry(current->user_data)`, and once the user function returns, marks the fiber as `FINISHED` and yields one final time.

Let's test it out with a quick workload that bounces execution back and forth between fibers:

```cpp
struct TaskPayload {
  int32_t task_id;
  int32_t iterations;
};

void worker_fiber(void* raw_arg) {
  TaskPayload* payload = static_cast<TaskPayload*>(raw_arg);
  for (int32_t step = 0; step < payload->iterations; ++step) {
    std::printf(
      "[Fiber %d] executing step %d of %d\n",
      payload->task_id,
      step + 1,
      payload->iterations
    );
    FiberScheduler::g_scheduler.yield();
  }
}

int main() {
  FiberScheduler::g_scheduler.init();

  TaskPayload payloads[3];
  for (int32_t i = 0; i < 3; ++i) {
    payloads[i].task_id = i + 100;
    payloads[i].iterations = 3;
    FiberScheduler::g_scheduler.spawn(worker_fiber, &payloads[i]);
  }

  std::printf("--- Starting Fiber Scheduler ---\n");
  FiberScheduler::g_scheduler.run_until_complete();
  std::printf("--- All Fibers Completed ---\n");

  return 0;
}
```

Wait—Look at what happens in `yield()` when `prev_idx != next_idx`. Suppose Fiber 0 yields to Fiber 1, and Fiber 1 yields to Fiber 2, and Fiber 2 yields back to Fiber 0. We don't even bounce back to `run_until_complete()` on `main_rsp`! Fiber 0 switches *directly* to Fiber 1 (`switch_context(&prev_fiber->rsp, next_fiber->rsp)`), cutting the number of context switches in half compared to a hub-and-spoke dispatcher where every yield returns to a central loop first.

---

## Subtle Hardware Details You Have to Watch Out For

When I started stress-testing this scheduler under `-O3`, a couple of interesting hardware details popped up that are worth keeping in mind if you build something similar.

### 1. The x86-64 Red Zone
The System V ABI defines a 128-byte "red zone" directly below `rsp` ($\text{rsp} - 128$ to $\text{rsp}$). Leaf functions (functions that don't call any other functions) are allowed to use this memory for local variables without actually decrementing `rsp`.
You might wonder: does our context switch corrupt the red zone when pushing `rbx`..`r15`?
Thankfully, no! Because `switch_context` is invoked via a `call` instruction, any function that calls `switch_context` is by definition **not** a leaf function, so the compiler is forbidden from storing live data in the red zone across the `call`.

### 2. Return Stack Buffer (RSB) Underflows
Modern Intel and AMD cores have a hardware structure called the **Return Stack Buffer (RSB)**—usually 16 to 32 entries deep—that predicts the target of `ret` instructions by matching them with prior `call` instructions.
When we swap `rsp` inside `switch_context` and execute `retq`, the CPU pops an address that belongs to a completely different fiber than the one that executed `call switch_context`. That triggers an RSB misprediction penalty of roughly $15\text{–}20$ clock cycles (~$4\text{ ns}$). Even with that branch predictor flush, our total switch time is still well under $15\text{ ns}$—about two orders of magnitude faster than asking the Linux kernel to do it for us.

### 3. Compiler Optimizations and TLS
If you scale this up from a single-threaded cooperative scheduler to an $M:N$ work-stealing scheduler across multiple OS threads, watch out for `thread_local` variables. GCC and Clang cache the base address of Thread Local Storage (`%fs:0` on x86-64 Linux) in a general-purpose register across function calls if `-O3` thinks the code never changes OS threads. If a fiber yields on Thread 1 and wakes up on Thread 2, it might read Thread 1's TLS pointer unless you compile with `-mno-tls-direct-seg-refs` or isolate TLS lookups behind non-inlined wrapper functions.

For a single-threaded event loop driving `io_uring` or a physics job system, though, this tiny ~150-line setup is honestly all you need. No heap fragmentation, zero syscalls, and your CPU spends its time running actual code instead of kernel bookkeeping.
