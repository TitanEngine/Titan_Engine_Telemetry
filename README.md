# Titan Engine: Bare-Metal Physics & Rendering Architecture

Titan Engine is a custom, high-performance 3D engine built from scratch in Rust. Designed for extreme computational workloads, it operates directly at the silicon level, saturating CPU pipelines and GPU PCIe lanes. There are no high-level wrappers, no garbage collection overhead, and no operating system bottlenecks.

## Architecture Showcases & Data Logs

* **[Engine Showcase 1: Cinematic Singularity](./Engine_Showcase_1.md)** — Zero-copy ReBAR memory architecture and 8-million particle streaming.
* **[Engine Showcase 2: ALU Stress & Physics Telemetry](./Engine_Showcase_2.md)** — 1,000,000 dense geometry rigid bodies, `VK_EXT_mesh_shader` amplification, and hardware frame-pacing diagnostics.

### Physics Stability Proof (Live Telemetry)
[![Pyramid Stability Test](https://img.youtube.com/vi/fWDH-uWmtOA/maxresdefault.jpg)](https://youtu.be/fWDH-uWmtOA)
*Live capture of the engine executing a 3D pyramid stacking test. The simulation demonstrates the flawless stability of the Temporal Gauss-Seidel solver, maintaining structural integrity without micro-velocity jitter or collapse.*

---

## Core Technology & Memory Architecture

### The TitanCore Monolith
Memory ownership and data access are handled through a unified `TitanCore` architecture. Data is strictly separated into a `GenerationalArena` for entity management and `SolverData` for raw mathematical processing. Using disjoint mutable references via `TitanView`, the engine achieves lock-free, zero-mutex multithreading across all CPU cores.

### Data-Oriented SIMD Execution
Object-oriented abstractions are stripped away during the physics step. Rigid body data is packed into a strict Structure of Arrays (SoA). Velocities and transforms are segregated into `AlignedFloatVec` buffers, ensuring that AVX registers (`_mm256_load_ps`) process 8 floats per clock cycle with zero cache waste. 

### Cache Coherence & Memory Management
* **Software Prefetching:** The engine utilizes x86_64 intrinsics (`_mm_prefetch` with `_MM_HINT_T0`) to stage mass and velocity data into the L1 cache ahead of execution, mathematically hiding DRAM latency.
* **Zero Fragmentation:** Transient collision data completely bypasses the OS heap. A `LinearAllocator` reserves upfront capacity, utilizing a simple pointer bump per collision that resets at the end of the frame.
* **Hardware Alignment:** Global atomic flags are padded to 128 bytes to perfectly align with Intel/AMD L3 cache lines, eliminating MESI bus thrashing.

## Fiber-Based Job System
Titan abandons heavy OS threads in favor of a custom Fiber-based Job System. Physics updates are sliced into micro-tasks and fed into lock-free queues. Physical CPU cores continuously process these data-parallel operations directly mapped to L1/L2 caches, eliminating context-switching latency.

## Advanced Physics Pipeline

### Constraint Graph Coloring
To process massive interconnected rigid-body islands (e.g., 4,000 dense cubes) without thread starvation, the engine implements Constraint Graph Coloring. Constraints are mathematically grouped by "color" to ensure no two constraints in a batch share a body. This allows simultaneous, race-free execution across all cores.

### Dual Solver Architecture
* **Temporal Gauss-Seidel (TGS):** Rigid body constraints are resolved using a TGS solver. By chopping the 16ms frame into substeps, applying gravity per substep, and sorting constraints bottom-to-top, collision impulses propagate instantly through deep stacks. This physically eradicates micro-velocity jitter.
* **Extended Position-Based Dynamics (XPBD):** For massive particle simulations, the engine utilizes an XPBD solver to guarantee infinite stability regardless of timestep, entirely bypassing velocity-based stiffness explosions.

### Pseudo-Velocities for Penetration Recovery
Deep penetrations are resolved using Pseudo-Velocities rather than injecting artificial kinetic energy. The solver calculates the exact mathematical displacement required, moves the body, and zeros out the pseudo-velocity at the end of the step, leaving true momentum completely unaffected.

### Bare-Metal Fast Math (Newton-Raphson)
Quaternion normalization bypasses standard IEEE 754 division-by-sqrt paths (30-50 cycles). Instead, it utilizes the `_mm_rsqrt_ps` hardware approximation (5 cycles) refined by a single Newton-Raphson iteration:
$$x_1 = 0.5 x_0 (3 - v x_0^2)$$
This drops execution to ~10 cycles while maintaining $10^{-6}$ precision.

## Thermodynamics & Engine Governance
* **Entropy Grid:** A 3D thermodynamic spatial grid tracks high-speed collision "heat." If density reaches critical thresholds, micro-jitter is injected to mechanically force separation and prevent mathematical singularities.
* **Sanity Sentinels:** Background fibers continuously scan SoA buffers for `NaN` values or infinite velocity spikes. Breaches trigger the Integrity Monitor to mathematically clamp coordinates and apply atmospheric damping before array corruption occurs.
