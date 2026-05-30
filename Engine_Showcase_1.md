# Titan Engine 

[![Titan Engine Cinematic Singularity](https://img.youtube.com/vi/aA7iZdflfns/maxresdefault.jpg)](https://youtu.be/aA7iZdflfns)

This repository showcases Titan Engine, a custom 3D engine built from scratch using Rust and Vulkan 1.3. The core focus of this showcase is bare-metal performance optimization, zero-copy memory architecture, and sustained GPU throughput while simulating and rendering up to 8 million particles via a custom Extended Position-Based Dynamics (XPBD) solver.

## Core Architecture & Memory Pipeline

### PCIe Saturation via Resizable BAR (ReBAR)
The particle emission system explicitly bypasses the standard Vulkan staging buffer latency. By mapping VRAM directly into the CPU's memory space using `HOST_VISIBLE | DEVICE_LOCAL` flags, the engine executes sequential Write-Combine (WC) instructions. This approach saturates PCIe Gen 4 bandwidth, streaming millions of particle structures (positions, velocities, and inverse masses) directly to the GPU without polluting the CPU L1/L2 caches.

### P-Core Hardware Thread Pinning
To ensure deterministic frame execution and eliminate OS scheduler jitter, the engine probes hardware topology via `GetSystemCpuSetInformation`. Rendering and submission threads are aggressively locked to physical Performance Cores (P-Cores) using `SetThreadSelectedCpuSetMasks`, keeping Efficiency Cores (E-Cores) completely out of the critical rendering path.

### Tri-Queue Frame Graph & Synchronization
Execution is driven by a Vulkan 1.3 `Synchronization2` pipeline, utilizing timeline semaphores to coordinate a Tri-Queue system:
1. **Compute Queue:** Executes the asynchronous XPBD integration, constraint solving, and velocity update passes.
2. **Graphics Queue:** Consumes the computed state using `VK_EXT_mesh_shader` for high-throughput rasterization.
3. **Present Queue:** Manages the N+2 Mailbox swapchain for zero-latency frame delivery.

## Physics Solver & Compute Pipeline

### XPBD Compute Integration
The gravitational simulation is driven by an Extended Position-Based Dynamics compute pipeline. The pipeline operates in discrete sub-steps:
* **Integration:** Translates velocities into predicted positions.
* **Solve:** Applies hard constraints, including the gravitational pull of the 50,000-mass singularity, tangential orbital velocity, and downward funnel sloping.
* **Velocity Update:** Finalizes the particle state, dumping the output directly into the unified physical memory arena via 64-bit Buffer Device Addresses (BDA).

### Dual-Spawning Singularity Logic
Particles are dynamically evaluated in a custom compute shader against the singularity's Event Horizon threshold. Upon absorption, particles are immediately respawned using a deterministic dual-lane system:
* **Type 1 (Equatorial Disk):** Injected at the outer flat ring with high orbital velocity, gradually losing momentum and spiraling inward.
* **Type 2 (Cyclone Funnel):** Injected at high altitudes, forced to track a mathematically defined cone surface, feeding the singularity from a vertical axis.

## Rendering System

### Mesh Shader Amplification
Traditional vertex attributes and input assembly are completely bypassed. The engine utilizes `VK_EXT_mesh_shader` alongside `EmitMeshTasksEXT`. Particle data is read directly from VRAM using BDA pointers, allowing the GPU to dynamically amplify particle seeds into physical rasterized geometry entirely on-die.

### Bindless Architecture
Descriptor sets are eliminated from the inner render loop. The engine relies on a monolithic `EngineRegisters` buffer, pushing a single 64-bit root memory pointer via Push Constants. All nested data structures (Positions, Jacobians, Colors, Extrapolation buffers) are accessed directly through hardware-level pointer dereferencing in GLSL.

## WSI & Thermal Management
The engine initialization drops bloated windowing libraries in favor of raw Win32 HWND creation. The application loop utilizes a blocking `GetMessageW` pump rather than a continuous spin-loop. This ensures that when the engine is not actively processing frames or hardware input, the CPU enters a dormant state, minimizing thermal pressure and power consumption.
