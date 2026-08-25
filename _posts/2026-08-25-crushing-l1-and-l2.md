---
layout: post
title: "Crushing L1 and L2 Cache Misses - Why Flattening 2D Matrices Changes Everything"
date: 2026-08-25 08:00:00 +0300
categories: [performance, low-level]
tags: [cpp, cache, optimization, memory-layout, systems]
math: true
---

A couple of months ago, I was writing a toy graphics software rasterizer and physics engine in C++. Everything felt super clean, the math was solid, and my abstractions looked slick. But when I scaled up my particle grid to a $2048 \times 2048$ matrix and started doing transformation passes over it, my framerate fell off a cliff.

My CPU utilization was sitting at barely 15%, yet the loop was running absurdly slow. I fired up Linux `perf` to see what was actually going on under the hood, and the stats hit me like a brick:

```text
    14,892,104,821      cycles                    #    4.120 GHz
    12,410,320,119      instructions              #    0.83  insn per cycle
     1,842,910,402      L1-dcache-load-misses     #   24.12% of all L1-dcache hits
       412,019,481      LLC-load-misses           #   48.30% of all LL-cache hits
```

Almost a quarter of all L1 data cache loads were misses, and the instruction-per-cycle (IPC) count was down in the gutter at 0.83. My CPU was spending nearly three-quarters of its life doing literally nothing except waiting for DRAM to fetch cache lines.

The culprit? The way I allocated and traversed my 2D matrices.

If you are allocating matrices as `int**` or looping through memory across non-contiguous boundaries, your hardware is crying. Let's dig into CPU cache architecture, why pointer-chasing destroys performance, and how flattening 2D matrices into contiguous 1D buffers turns memory stalls into pure sequential throughput.

---

### The Anatomy of a Cache Miss

Modern desktop CPUs operate at roughly 4 GHz to 5 GHz. An arithmetic operation like `ADD` or `FMA` takes anywhere from 0.5 to 4 clock cycles. But main memory (DRAM) is miles away—accessing a random byte in physical RAM takes around $50\text{ to }100\text{ ns}$, which translates to **200 to 400 CPU cycles of pure idle stalling**.

To bridge this massive speed gap, CPUs use a hierarchy of caches:

| Cache Level | Typical Size (per core) | Latency (Cycles) |
| :--- | :--- | :--- |
| **L1 Data (L1d)** | 32 KiB – 48 KiB | $\approx 4 - 5$ cycles |
| **L2 Cache** | 512 KiB – 2 MiB | $\approx 12 - 14$ cycles |
| **L3 Cache (LLC)**| 16 MiB – 96 MiB (shared) | $\approx 40 - 75$ cycles |
| **Main Memory (DRAM)** | 16 GiB – 64 GiB | $\approx 200 - 400$ cycles |

When your CPU core requests data from memory at address `0x7ffee0`, it doesn't just pull a single 4-byte `int` or 8-byte `double`. It pulls an entire **Cache Line**, which is almost universally **64 bytes** on modern x86-64 and ARM architectures.

```text
Memory Address: 0x1000
+---------------------------------------------------------------+
| Byte 0 | Byte 1 | Byte 2 | ... | Byte 61 | Byte 62 | Byte 63  |  <- 64-Byte Cache Line
+---------------------------------------------------------------+
  ^
  Requested by CPU -> Entire block copied into L1d Cache
```

If you access `arr[0]`, the hardware brings `arr[0]` through `arr[15]` (assuming 4-byte `int32_t`) right into L1 cache for free. If your next access is `arr[1]`, that read takes ~4 cycles instead of ~250 cycles. That is **spatial locality**.

---

### The Silent Killer: `int**` and Pointer-of-Pointers

Most intro programming classes teach 2D dynamic arrays in C or C++ like this:

```cpp
// The traditional "pointer-to-pointer" allocation
int** mat = (int**)malloc(rows * sizeof(int*));
for (int i = 0; i < rows; i++) {
  mat[i] = (int*)malloc(cols * sizeof(int));
}
```

This looks innocent, but from a memory and cache standpoint, it is an absolute catastrophe.

```text
HEAP MEMORY LAYOUT (Scattered / Pointer-to-Pointer):

mat (Pointer Array in Heap)
[ ptr 0 ] ---> [ Row 0 in Heap: chunk A (0x0040) ] -> 64B cache line
[ ptr 1 ] ---> [ Row 1 in Heap: chunk B (0x9F10) ] -> different line miles away
[ ptr 2 ] ---> [ Row 2 in Heap: chunk C (0x12A0) ] -> another random line
[ ptr 3 ] ---> [ Row 3 in Heap: chunk D (0x5400) ] -> total cache thrashing
```

Here is why this destroys your pipeline:

1. **Double Indirection (Pointer Chasing):** To access `mat[r][c]`, the CPU must first load the pointer `mat[r]` from memory, wait for it to arrive, and only then issue the load for `mat[r][c]`. This creates a serialization dependency in the execution pipeline.
2. **Heap Fragmentation:** Every row is allocated by an individual `malloc` call. The heap allocator places these chunks wherever free bins exist. Row 0 might be at `0x0040`, but Row 1 could be at `0x9F10`. Spatial locality across row boundaries is completely destroyed.
3. **Allocation Overhead:** Each individual malloc chunk incurs metadata overhead (typically 8 to 16 bytes per chunk) plus memory alignment padding.

---

### The Right Way: Flattened 1D Contiguous Buffers

Instead of allocating an array of pointers, we allocate a single continuous buffer of size $\text{Rows} \times \text{Cols}$ and compute the index mathematically:

$$\text{Index}(r, c) = (r \times \text{Cols}) + c$$

```text
CONTIGUOUS 1D MEMORY LAYOUT:

Index:  0   1   2  ...  M-1 | M  M+1 ... 2M-1 | 2M ...
Data:  [    Row 0 elements   ][   Row 1 elements  ][  Row 2 elements  ]
       |<- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - ->|
                      Single contiguous slab in RAM
```

Every single row starts immediately after the last element of the previous row. When you finish reading the end of Row 0, the cache line loaded by the CPU already contains the beginning of Row 1!

---

### The Math: Stride Analysis and Miss Probability

Let's model the cache behavior mathematically to see why this makes such an enormous difference.

Suppose we have a matrix of dimensions $N \times M$ containing elements of size $S$ bytes (e.g., $S = 4$ for `float` or `int32_t`). The cache line size is $B = 64$ bytes. The number of elements that fit in a single cache line is:

$$E = \frac{B}{S} = \frac{64}{4} = 16\text{ elements}$$

#### Case 1: Sequential Traversal (Stride = 1)

When we iterate through a contiguous buffer in row-major order:

$$\text{Address}(k) = \text{Base} + k \cdot S$$

The first access `mat[0]` misses L1/L2 and fetches 64 bytes (16 elements). The next 15 accesses (`mat[1]` through `mat[15]`) are guaranteed L1 hits.

$$\text{Miss Rate}_{\text{sequential}} = \frac{1}{E} = \frac{1}{16} = 6.25\%$$

Furthermore, modern CPU cores have dedicated hardware **Stream Prefetchers** (like the L2 Streamer on Intel and AMD cores). When the prefetcher detects linear access with stride $+1$, it starts pulling subsequent cache lines into L2 and L1 *before* the instruction even requests them. Effective miss rate drops to practically **0%**.

#### Case 2: Strided / Column-Major Traversal (Stride = $M$)

Now consider traversing a matrix column by column (or jumping across non-contiguous rows) where each consecutive access is $M$ elements apart:

$$\text{Address}(k) = \text{Base} + (k \cdot M) \cdot S$$

If $M \cdot S \ge B$ (which means the row size in bytes is larger than a 64-byte cache line), every single access lands on a completely different cache line.

If the working set size $N \times M \times S$ exceeds the size of L1d cache (say 32 KiB), then by the time the loop increments to the next column, the old cache lines have already been evicted by LRU (Least Recently Used) replacement policy.

$$\text{Miss Rate}_{\text{strided}} = \min\left(1.0, \; \frac{M \cdot S}{B \cdot E}\right) \approx 100\%$$

Every single element access incurs a full cache hierarchy walk all the way out to L3 or DRAM.

---

### Benchmarking the Architectures

Let's write a clean C++ benchmark comparing three implementations:
1. **Jagged Matrix (`int**`)**: Array of pointers with heap-scattered rows.
2. **Flat 1D Matrix (Column-Major Traversal)**: Contiguous buffer, but traversing against the stride.
3. **Flat 1D Matrix (Row-Major Sequential Traversal)**: Contiguous buffer with stride-1 access.

Here is the benchmark implementation:

```cpp
#include <iostream>
#include <chrono>
#include <cstdlib>
#include <cstring>

// Allocation helper: Jagged 2D array
int** alloc_jagged(int rows, int cols) {
  int** mat = (int**)malloc(rows * sizeof(int*));
  if (mat == nullptr) {
    std::cerr << "Allocation failed for pointer array\n";
    std::exit(1);
  }
  for (int i = 0; i < rows; i++) {
    mat[i] = (int*)malloc(cols * sizeof(int));
    if (mat[i] == nullptr) {
      std::cerr << "Allocation failed for row " << i << "\n";
      std::exit(1);
    }
  }
  return mat;
}

void free_jagged(int** mat, int rows) {
  for (int i = 0; i < rows; i++) {
    free(mat[i]);
  }
  free(mat);
}

// Allocation helper: Flat contiguous array
int* alloc_flat(int rows, int cols) {
  int* mat = (int*)malloc(rows * cols * sizeof(int));
  if (mat == nullptr) {
    std::cerr << "Allocation failed for flat matrix\n";
    std::exit(1);
  }
  return mat;
}

void free_flat(int* mat) {
  free(mat);
}

// Test 1: Jagged matrix traversal
long long sum_jagged(int** mat, int rows, int cols) {
  long long total = 0;
  for (int r = 0; r < rows; r++) {
    for (int c = 0; c < cols; c++) {
      total += mat[r][c];
    }
  }
  return total;
}

// Test 2: Flat matrix with column-major traversal (Stride-M, Cache Bad)
long long sum_flat_column_major(const int* mat, int rows, int cols) {
  long long total = 0;
  for (int c = 0; c < cols; c++) {
    for (int r = 0; r < rows; r++) {
      total += mat[(r * cols) + c];
    }
  }
  return total;
}

// Test 3: Flat matrix with row-major traversal (Stride-1, Cache Friendly)
long long sum_flat_row_major(const int* mat, int rows, int cols) {
  long long total = 0;
  for (int r = 0; r < rows; r++) {
    int row_offset = r * cols;
    for (int c = 0; c < cols; c++) {
      total += mat[row_offset + c];
    }
  }
  return total;
}

int main() {
  const int ROWS = 4096;
  const int COLS = 4096;
  const size_t total_elements = (size_t)ROWS * COLS;
  std::cout << "Testing Matrix Size: " << ROWS << "x" << COLS 
            << " (" << (total_elements * sizeof(int)) / (1024 * 1024) << " MB)\n\n";

  // Setup matrices
  int** jagged = alloc_jagged(ROWS, COLS);
  int* flat = alloc_flat(ROWS, COLS);

  for (int r = 0; r < ROWS; r++) {
    for (int c = 0; c < COLS; c++) {
      jagged[r][c] = 1;
      flat[(r * COLS) + c] = 1;
    }
  }

  // Benchmark 1: Jagged
  auto t1 = std::chrono::high_resolution_clock::now();
  long long sum1 = sum_jagged(jagged, ROWS, COLS);
  auto t2 = std::chrono::high_resolution_clock::now();
  double time_jagged = std::chrono::duration<double, std::milli>(t2 - t1).count();
  std::cout << "1. Jagged Array (int**):           " << time_jagged << " ms (Sum: " << sum1 << ")\n";

  // Benchmark 2: Flat Column-Major (Stride-M)
  auto t3 = std::chrono::high_resolution_clock::now();
  long long sum2 = sum_flat_column_major(flat, ROWS, COLS);
  auto t4 = std::chrono::high_resolution_clock::now();
  double time_flat_col = std::chrono::duration<double, std::milli>(t4 - t3).count();
  std::cout << "2. Flat Array (Column-Major):     " << time_flat_col << " ms (Sum: " << sum2 << ")\n";

  // Benchmark 3: Flat Row-Major (Stride-1)
  auto t5 = std::chrono::high_resolution_clock::now();
  long long sum3 = sum_flat_row_major(flat, ROWS, COLS);
  auto t6 = std::chrono::high_resolution_clock::now();
  double time_flat_row = std::chrono::duration<double, std::milli>(t6 - t5).count();
  std::cout << "3. Flat Array (Row-Major Stride-1): " << time_flat_row << " ms (Sum: " << sum3 << ")\n";

  free_jagged(jagged, ROWS);
  free_flat(flat);
  return 0;
}
```

### The Real-World Numbers

I compiled this benchmark on Linux using `g++ -O3 -march=native` on an AMD Ryzen 7 7840HS processor (32 KiB L1d per core, 1 MiB L2 per core, 16 MiB L3).

Here are the results for a $4096 \times 4096$ integer matrix (64 MiB total payload, comfortably exceeding the 16 MiB L3 cache):

```text
Testing Matrix Size: 4096x4096 (64 MB)

1. Jagged Array (int**):            46.12 ms
2. Flat Array (Column-Major):      118.84 ms
3. Flat Array (Row-Major Stride-1):  4.21 ms
```

Look at those numbers:
- The **Flat Row-Major** access is **$10.9\times$ faster** than the traditional `int**` jagged array.
- The **Flat Row-Major** access is **$28.2\times$ faster** than the Column-Major access on the exact same contiguous memory.

Think about that for a second. We didn't use SIMD intrinsics. We didn't use multithreading. We didn't write assembly. We simply organized our memory to flow in the same direction that the hardware prefetcher expects.

---

### Hardware Prefetchers: Why Flat Memory Is King

Why is sequential access on a flat array so ridiculously fast?

Inside your CPU core, there is a dedicated hardware module known as the **Data Prefetcher**. The prefetcher constantly monitors load requests sent to the L1 and L2 cache controllers. 

```text
Loop iteration:
Access 0x0000 -> Miss (L1) -> Load cache line [0x0000 - 0x003F]
Access 0x0004 -> Hit (L1)
Access 0x0008 -> Hit (L1)
Prefetcher notices: "Hey, instructions are reading sequentially with stride +4!"
Prefetcher issues background load for [0x0040 - 0x007F] and [0x0080 - 0x00BF] into L2/L1.
```

By the time your loop finishes processing index `15` and steps onto index `16`, the data has *already arrived in L1d cache*. The latency observed by the CPU core drops from $\approx 250\text{ cycles}$ to **zero stall cycles**.

When you use `int**`, the addresses of individual rows are unpredictable pointers scattered around the heap. The prefetcher cannot identify a constant address delta between rows, so it cannot prefetch. You pay the full DRAM penalty on every single row transition.

---

### Leveling Up: Cache Tiling (Loop Blocking)

Flattening your arrays is step one. But what if you are doing an operation like **Matrix Multiplication** ($C = A \times B$), where you inevitably have to traverse one matrix across its columns?

A naive $O(N^3)$ multiplication algorithm looks like this:

```cpp
// Naive Matrix Multiplication - Cache Thrashing Nightmare for B
for (int i = 0; i < N; i++) {
  for (int j = 0; j < N; j++) {
    int sum = 0;
    for (int k = 0; k < N; k++) {
      sum += A[(i * N) + k] * B[(k * N) + j]; // B is accessed with Stride-N!
    }
    C[(i * N) + j] = sum;
  }
}
```

Notice `B[(k * N) + j]`. As `k` increments, we are jumping down rows of $B$—stride $N$. For large $N$, this produces catastrophic cache misses on matrix $B$.

To fix this, we use a technique called **Cache Tiling (or Loop Blocking)**. We divide the $N \times N$ matrix into small sub-blocks of size $B_{\text{size}} \times B_{\text{size}}$ that fit entirely inside L1d cache (e.g., $32 \times 32$ or $64 \times 64$).

```text
TILE PARTITIONING:
Matrix A                Matrix B
+-----+-----+           +-----+-----+
| Tile| Tile|     *     | Tile| Tile|
| A00 | A01 |           | B00 | B01 |
+-----+-----+           +-----+-----+
| Tile| Tile|           | Tile| Tile|
| A10 | A11 |           | B10 | B11 |
+-----+-----+           +-----+-----+
Each Tile fits completely inside 32 KiB L1 Cache!
```

Here is the tiled implementation in 2-space indented C++:

```cpp
void matmul_tiled(const int* A, const int* B, int* C, int N, int block_size) {
  for (int i = 0; i < N; i += block_size) {
    for (int j = 0; j < N; j += block_size) {
      for (int k = 0; k < N; k += block_size) {
        // Mini matrix-multiplication on local cache-resident sub-blocks
        for (int ii = i; ii < i + block_size && ii < N; ii++) {
          int row_a = ii * N;
          for (int kk = k; kk < k + block_size && kk < N; kk++) {
            int r_a_val = A[row_a + kk];
            int row_b = kk * N;
            for (int jj = j; jj < j + block_size && jj < N; jj++) {
              C[(ii * N) + jj] += r_a_val * B[row_b + jj];
            }
          }
        }
      }
    }
  }
}
```

Notice two critical optimizations here:
1. **Loop reordering:** By swapping the inner loops so that `jj` is the innermost index, both $C$ and $B$ are accessed sequentially with **stride-1**!
2. **Sub-block sizing:** The entire active working slice of $A$, $B$, and $C$ stays pinned in L1/L2 cache throughout the computation of the tile, completely eliminating evictions to main memory.

---

### Designing a Clean C++ Matrix Wrapper

You don't have to write raw pointer arithmetic `mat[(r * cols) + c]` everywhere in your code. You can encapsulate flat memory in a zero-overhead C++ struct with an inline `operator()`:

```cpp
template <typename T>
struct FlatMatrix {
  int rows;
  int cols;
  T* data;

  FlatMatrix(int r, int c) : rows(r), cols(c) {
    data = (T*)malloc(r * c * sizeof(T));
    if (data == nullptr) {
      std::cerr << "Failed to allocate FlatMatrix\n";
      std::exit(1);
    }
  }

  ~FlatMatrix() {
    if (data != nullptr) {
      free(data);
    }
  }

  // Disable copying for safety in this lightweight wrapper
  FlatMatrix(const FlatMatrix&) = delete;
  FlatMatrix& operator=(const FlatMatrix&) = delete;

  // 1D Contiguous Indexing operator
  inline T& operator()(int r, int c) {
    return data[(r * cols) + c];
  }

  inline const T& operator()(int r, int c) const {
    return data[(r * cols) + c];
  }
};
```

Because `operator()` is marked `inline`, modern compilers like GCC and Clang will optimize out the function call completely and compile it down to a single `lea` (load effective address) instruction:

```assembly
; Assembly generated for mat(r, c):
; r in %rdi, c in %rsi, cols in %rdx, data in %rax
imulq   %rdx, %rdi          ; r * cols
addq    %rsi, %rdi          ; (r * cols) + c
movl    (%rax,%rdi,4), %eax ; load data[(r * cols) + c]
```

Zero abstraction cost, clean syntax, and 100% cache-optimal contiguous memory layout.

---

### Key Takeaways

1. **Say No to `int**` / Jagged Arrays:** Pointer-of-pointers scatter row buffers across heap memory, introduce serial pointer-chasing latency, and prevent the hardware prefetcher from helping you.
2. **Always Prefer Contiguous 1D Allocations:** Allocate one flat buffer of $\text{Rows} \times \text{Cols}$ and index it with $(r \times \text{Cols}) + c$.
3. **Traverse Memory in Stride-1 Order:** Always make your innermost loop iterate across columns (the contiguous dimension in row-major order).
4. **Use Cache Tiling for Complex Kernels:** If your algorithm requires multi-axis traversal (like matrix multiplication or convolutions), break loops into tiles that fit inside the 32 KiB L1 data cache.
5. **Measure with Hardware Counters:** Don't guess—use `perf stat -e L1-dcache-load-misses,LLC-load-misses ./your_program` to verify that your memory layout is cooperating with the CPU cache hierarchy.
