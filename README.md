# Titan_Engine_Telemetry


# **Titan Engine - Performance Telemetry & Hardware Profiling**

This document contains deep-dive telemetry data, hardware profiling, and mathematical analysis of the Titan Engine. It demonstrates the direct results of low-level optimizations including zero-copy ReBAR pipelines, Temporal Gauss-Seidel solver stabilization, and bare-metal Subgroup operations.

## **1. Physics Solver Stabilization (TGS & Newton-Raphson)**
**Objective:** Eliminate micro-velocity jitter and infinite sleep/wake loops in rigid-body kinematics.

### **The Problem: Unstable Constraint Resolution**
*In the initial implementation, massive momentum settled, but objects continually jittered (micro-velocities). The bottom graph shows the "Sleep State" sawtooth pattern—the engine entered an endless loop of waking objects up and putting them to sleep due to continuous micro-collisions.*
![Unstable Physics Telemetry](https://raw.githubusercontent.com/TitanEngine/Titan_Engine_Telemetry/main/Code_Generated_Image%20(9).png)

### **The Solution: Temporal Gauss-Seidel Application**
*After refactoring the solver using TGS and Newton-Raphson approximations, collision resolution is flawless. Past Frame 60, Active bodies (yellow) plunge smoothly to 0, while Sleeping bodies (green) surge to capacity. Both transform into perfect parallel flatlines, proving all rigid-body motion has permanently stopped, allowing the CPU to conserve power.*
![Stable Physics Telemetry](https://raw.githubusercontent.com/TitanEngine/Titan_Engine_Telemetry/main/Code_Generated_Image.png)

## **2. Frame Pacing & ALU Workload Consistency**
**Objective:** Analyze render latency against strict, constant geometric loads.

*This 3-panel visualization tracks latency (1% low spikes) against a constant load of 8M Vertices and 12M Triangles. The flatlines in the ALU Workload (bottom) prove that stuttering or variance in frame timing (top/middle) is purely due to software pacing or driver/dispatch inconsistencies, not geometric scaling.*
![Frame Pacing Visualization](https://raw.githubusercontent.com/TitanEngine/Titan_Engine_Telemetry/main/Code_Generated_Image%20(8).png)

## **3. System Throughput Under Heavy Workload**
**Objective:** Stress-test driver dispatch overhead with 1 Million procedural cubes.

*This graph captures raw frame throughput (Instant FPS) over 17.5 seconds, maintaining an average of ~706 FPS. The variance and extreme peaks (spiking toward 3000-4500 FPS) highlight erratic driver dispatch behavior when completely saturating the PCIe lanes using Write-Combine buffers.*
![System Throughput Consistency](https://raw.githubusercontent.com/TitanEngine/Titan_Engine_Telemetry/main/Code_Generated_Image%20(7).png)

## **4. Render Latency Variance Analysis**
**Objective:** Measure pipeline stability over extended frame windows.

*The dark blue baseline proves the rendering pipeline operates smoothly at ~1.5ms. The orange line (1% Lows) captures the intermittent hardware-level stalls (e.g., the 8ms spike at frame 2600) during heavy context switching or memory paging events.*
![Latency Variance Plot](https://raw.githubusercontent.com/TitanEngine/Titan_Engine_Telemetry/main/Code_Generated_Image%20(6).png)

## **5. Baseline Rendering Density Distribution**
**Objective:** Statistically plot pure hardware capability during occlusion culling.

*By plotting Instant FPS against Frame Latency, the system forms a perfect downward hyperbola. The center-of-mass statistical concentration (crosshairs) pinpoints the fundamental hardware limit for this benchmark: a solid median rendering time of **1.51ms per frame** (661 FPS density cluster) while executing Kogge-Stone Parallel Prefix Sums and Hi-Z culling.*
![Baseline Rendering Performance](https://raw.githubusercontent.com/TitanEngine/Titan_Engine_Telemetry/main/Code_Generated_Image%20(5).png)
