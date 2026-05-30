# Titan Engine — Telemetry & Hardware Profiling

Welcome to the central telemetry and profiling repository for the **Titan Engine**. This repository serves as the primary data archive for hardware-level benchmarks, rendering throughput logs, and physics solver diagnostics generated during the engine's architectural development.

To ensure clarity and organized data presentation, the deep-dive telemetry, visual proofs, and specific architectural breakdowns have been separated into dedicated showcase modules.

## Architecture Showcases & Data Logs

### [1. Engine Showcase 1: Cinematic Singularity](./Engine_Showcase_1.md)
* **Focus:** Sustained GPU throughput, zero-copy memory architecture, and massive scale particle simulation.
* **Key Highlights:** * Streaming and simulating up to 8 million particles via a custom Extended Position-Based Dynamics (XPBD) solver.
  * PCIe lane saturation bypassing standard staging buffers using Resizable BAR (ReBAR) and CPU Write-Combine (WC) memory mapping.
  * Deterministic execution via raw Win32 APIs and strict hardware-level thread pinning to P-Cores.

### [2. Engine Showcase 2: ALU Stress & Physics Telemetry](./Engine_Showcase_2.md)
* **Focus:** Frame pacing consistency, extreme ALU geometry workloads, and collision solver stabilization.
* **Key Highlights:**
  * Rendering 1,000,000 active collision bodies (cubes) utilizing `VK_EXT_mesh_shader` and bindless 64-bit Buffer Device Address (BDA) pointers.
  * Deep-dive visual dashboards tracking render latency variance (1% lows) and dispatch consistency under heavy, fixed geometric loads.
  * Mathematical telemetry proving the elimination of micro-velocity jitter in dense kinematic stacks using Temporal Gauss-Seidel (TGS) algorithms.

## Core Technology Stack
* **Language:** Rust (Bare-metal memory management, zero-cost abstractions)
* **Graphics API:** Vulkan 1.3 (Dynamic Rendering, Synchronization2, Bindless Architecture)
* **Execution:** Branchless SIMD Intrinsics, SIMT Compute Shaders, Subgroup Ballot Operations
* **Geometry Pipeline:** 100% GPU-driven via Task/Mesh Shaders, entirely eliminating legacy vertex and index buffer bottlenecks.

---
*Navigate to the specific showcase links above to view the detailed code architectures, performance graphs, and live YouTube telemetry recordings.*
