---
layout: default
title: ""
permalink: /hiprt/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 


**“HIPRT: A Ray Tracing Framework in HIP”** by Daniel Meister, Paritosh Kulkarni, Aaryaman Vasishta, and Takahiro Harada (AMD, 2024):

---

## **1. Motivation & Background**

Ray tracing has become increasingly feasible on GPUs thanks to modern hardware with large memory and dedicated acceleration units. While NVIDIA (OptiX) and Intel (Embree) provide professional-grade GPU ray tracing frameworks, AMD lacked an equivalent open-source framework tailored for professional rendering pipelines.

**HIPRT** is introduced as a cross-platform, GPU-based **ray tracing framework in HIP (Heterogeneous-Compute Interface for Portability)** designed specifically for AMD GPUs, but usable in broader environments. Unlike Vulkan/DirectX ray tracing APIs, HIPRT:

* Provides **ray tracing only**, leaving shading and material evaluation to applications.
* Targets professional offline rendering (e.g., Blender Cycles, PBRT-v4, Radeon ProRender).
* Offers a lightweight, user-friendly API.

---

## **2. Core Contributions**

1. **Full-featured GPU ray tracing framework** for AMD hardware, with support for:

   * Multi-level instancing
   * Motion blur with non-uniform time intervals
   * Custom primitives & intersection filters
2. **Novel massively-parallel SBVH (spatial split BVH) construction algorithm** adapted to GPUs.
3. **Flexible and minimal API design**, avoiding complexities of shader binding tables.
4. **Open-source release** with ports of PBRT-v4 demonstrating integration.
5. **Technical innovations**:

   * Component-wise motion blur representation (avoiding matrix singularities found in OptiX).
   * Generic traversal stack supporting multiple strategies (private, global, dynamic).
   * Support for importing custom BVHs (enabling research comparisons).

---

## **3. HIPRT API**

* **Acceleration structures**:

  * **BLAS (Geometry)** – triangles or custom primitives.
  * **TLAS (Scene)** – instancing with affine transformations & motion blur.
* **Build modes**: Fast (LBVH), Balanced (PLOC), High Quality (SBVH).
* **User workflow**:

  * Create geometry/scene.
  * Build BVH.
  * Compile trace kernel.
* **Custom functions**: user-defined intersection/filter functions indexed by `(ray type × geometry type)`.

---

## **4. BVH Construction**

HIPRT provides three GPU-based BVH builders:

* **LBVH (Linear BVH):** Fastest, Morton code + radix sort.
* **PLOC (Parallel Locally Ordered Clustering):** Balanced speed/quality.
* **SBVH (Spatial Split BVH):** High-quality GPU adaptation of Stich et al.’s algorithm.

Key techniques:

* **Triangle pairing**: merges neighboring triangles into pairs → \~30% fewer leaves.
* **Conversion to wide BVH (4-way):** Optimized top-down pass for AMD hardware units.
* **Multi-level instancing**: supports massive scenes (e.g., Disney’s Moana Island with 31B instances in-core).
* **Batch construction**: efficient build for many small geometries (e.g., hair strands).

---

## **5. Ray Traversal**

* Hardware-accelerated ray-box and ray-triangle intersection (RDNA2/3).
* General traversal loop supports multiple node types (internal, leaf, instance).
* **Traversal stack implementations**:

  * **Private stack:** simple, local memory.
  * **Global stack:** shared+global memory, high efficiency.
  * **Dynamic stack:** shared+global, allocated adaptively to save memory.
* **Custom function dispatch** via compile-time generated function tables (avoids costly runtime indirections).
* Supports multi-level hierarchies and motion blur with correct interpolation.

---

## **6. Technical Implementation**

* \~17k lines of C++/HIP, leveraging templates and compile-time specialization.
* Cross-vendor portability via Orochi library (AMD/NVIDIA).
* Compiles into HIP binaries (no IR like SPIR-V/PTX, so architecture-specific builds required).
* Fully open-sourced to encourage research/industry adoption.

---

## **7. Evaluation**

* Tested on **AMD Radeon Pro W7900 (48GB VRAM)** against Vulkan (Fast & HQ) and Embree (CPU high-quality BVHs imported).
* Benchmarks on **10 complex scenes** (0.8M–8.2M triangles).
* **Results**:

  * **Build times**:

    * LBVH = 1.4–3.4× faster than Vulkan Fast.
    * PLOC = 1.3–4.1× faster than Vulkan HQ.
    * SBVH = slowest (3.7–8.9× slower than Vulkan HQ), but gives highest-quality BVHs.
  * **Trace times**:

    * HIPRT PLOC competitive with Vulkan; SBVH & Embree yield best performance.
    * Multi-level instancing shows HIPRT outperforming Vulkan (e.g., up to 1.8× faster for shadow rays).
  * **SAH (Surface Area Heuristic) cost**:

    * SBVH ≈ Embree (lowest cost, best quality).
    * LBVH/PLOC higher cost but faster build.
* Demonstrates trade-off between build speed vs. trace efficiency.

---

## **8. Conclusion & Impact**

HIPRT provides:

* A **practical, research-friendly, and industry-ready GPU ray tracing framework** for AMD GPUs.
* **Bridges a gap** similar to Intel’s Embree and NVIDIA’s OptiX but tailored for HIP.
* **Open-source reference** for ray tracing community.
* Demonstrated **integration in production renderers** (Blender, PBRT-v4, Radeon ProRender).

The framework balances **simplicity (API design)**, **performance (optimized BVH build/traversal)**, and **flexibility (custom primitives, motion blur, instancing)**, making it a strong candidate for both **academic research** and **production rendering pipelines**.

