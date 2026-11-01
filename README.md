# thread-pool-lab

> **已迁移 / Moved（2026-11-01）**
>
> 当前代码、文档与后续维护已移至 [frontend-labs / labs/thread-pool](https://github.com/ieuforu/frontend-labs/tree/main/labs/thread-pool)。请在新仓库提交 Issue 和 PR。
>
> 原仓库的提交历史已完整保留在新仓库；本仓库保留历史版本。下方是迁移前的说明，其中性能与生产可用性描述不代表当前验证结论，当前行为和限制请以新仓库文档为准。


A production-grade, zero-dependency multi-core worker thread pool built with TypeScript, `SharedArrayBuffer` (SAB), and lock-free `Atomics` pipeline synchronization. Demonstrates true multi-threaded CPU parallel computation, zero-copy physical memory access, and non-blocking 120 FPS UI execution in the browser.

---

## Architecture Overview

Traditional Web Worker architectures rely on `postMessage()`, which triggers the Structured Clone algorithm. Transferring large datasets (tens to hundreds of megabytes) introduces substantial CPU serialization/deserialization latency and memory fragmentation on the main thread.

`thread-pool-lab` solves this by mapping a continuous physical memory region (`SharedArrayBuffer`) across the main thread and multiple worker threads. Threads coordinate work dynamically using low-level hardware atomic instructions (`Atomics`), eliminating communication overhead and achieving $O(0)$ memory transmission cost.

```text
       ┌────────────────────────────────────────────────────────┐
       │             SharedArrayBuffer (Shared RAM)             │
       │  [0]: Global Task Index (Atomics)  │ [1...N]: Results  │
       └────────────────────────────────────────────────────────┘
                 ▲                     ▲                     ▲
                 │ (Zero-Copy)         │ (Zero-Copy)         │ (Zero-Copy)
       ┌─────────┴─────────┐ ┌─────────┴─────────┐ ┌─────────┴─────────┐
       │   Worker Core 0   │ │   Worker Core 1   │ │   Worker Core 2   │
       │ (Atomics.add 取号) │ │ (Atomics.add 取号) │ │ (Atomics.add 取号) │
       └───────────────────┘ └───────────────────┘ └───────────────────┘
```

---

## Core Mechanics & Memory Layout

### 1. Shared Memory Mapping (`Int32Array`)

The continuous shared buffer is organized into a deterministic memory layout:

```text
[ Offset 0 ]  TASK_INDEX        -> Global atomic ticket counter (Atomics.add)
[ Offset 1 ]  TOTAL_TASKS       -> Total number of task units to process
[ Offset 2 ]  COMPLETED_COUNT   -> Atomic counter of completed task units
[ Offset 3+ ] WORK_DATA_START   -> Contiguous slots for computational output
```

### 2. Lock-Free Work-Stealing Pipeline

Instead of pre-partitioning chunks statically (which causes load imbalance when tasks vary in complexity), workers run an autonomous, lock-free work-stealing loop:

```typescript
while (true) {
  // Atomic hardware fetch-and-add (thread-safe, zero lock overhead)
  const taskIndex = Atomics.add(sharedInt32, MEM_OFFSET.TASK_INDEX, 1)
  if (taskIndex >= totalTasks) break

  // Process work directly into shared memory slot
  sharedInt32[MEM_OFFSET.WORK_DATA_START + taskIndex] = compute(taskIndex)
  Atomics.add(sharedInt32, MEM_OFFSET.COMPLETED_COUNT, 1)
}
```

- **Data Race Prevention**: `Atomics.add` operates at the CPU cache coherence bus level, ensuring no two workers receive identical task indices.
- **Dynamic Load Balancing**: Faster CPU cores naturally claim more tasks without centralized scheduling overhead.

### 3. Cross-Origin Isolation (Security Context)

Due to browser mitigations for speculative execution vulnerabilities (Spectre), `SharedArrayBuffer` requires strict cross-origin isolation headers.

Configured in `vite.config.ts`:

```typescript
server: {
  headers: {
    "Cross-Origin-Opener-Policy": "same-origin",
    "Cross-Origin-Embedder-Policy": "require-corp",
  },
}
```

---

## Performance Comparison (300,000 Computation Units)

Benchmarked on an 8-Core CPU machine under heavy bitwise math operations:

| Execution Model                  | Core Utilization                  | Total Execution Time | Main Thread Impact           | Zero-Copy                 |
| :------------------------------- | :-------------------------------- | :------------------- | :--------------------------- | :------------------------ |
| **1. Synchronous Blocking**      | Single Core (Core 0)              | ~1000 ms             | **Frozen (0 FPS, Jank)**     | N/A                       |
| **2. Cooperative Time-Slicing**  | Single Core (Core 0)              | ~1050 ms             | **Smooth (120 FPS)**         | N/A                       |
| **3. SAB + Multi-Core Parallel** | All Cores (`hardwareConcurrency`) | **~150 ms - 220 ms** | **Smooth (120 FPS, 0% CPU)** | **Yes ($O(0)$ Transfer)** |

---

## Getting Started

### Prerequisites

- Node.js (>= 18)
- pnpm (>= 8)

### Installation & Run

```bash
# Clone the repository
git clone [https://github.com/](https://github.com/)<your-username>/thread-pool-lab.git
cd thread-pool-lab

# Install dependencies
pnpm install

# Start Vite dev server with COOP/COEP headers
pnpm dev

# Typecheck and build
pnpm build
```

---

## Project Structure

```text
.
├── index.html         # Instrumentation UI, FPS probe, and benchmark panel
├── vite.config.ts     # Dev server configuration with COOP/COEP headers
├── src/
│   ├── constants.ts   # Memory layout offsets and computational workload
│   ├── worker.ts      # Web Worker pipeline execution and atomic fetching
│   ├── pool.ts        # Worker pool lifecycle and SAB allocation manager
│   └── main.ts        # Three-way benchmark runner and telemetry
├── package.json
└── tsconfig.json      # Strict TS configuration (erasableSyntaxOnly compliant)
```

---

## License

MIT
