---
layout: default
title: ""
permalink: /Multi-Resolution Geometric Representation using Bounding Volume Hierarchy for Ray Tracing/
---

# Multi-Resolution Geometric Representation using Bounding Volume Hierarchy for Ray Tracing

**Paper:** *Multi‑Resolution Geometric Representation using Bounding Volume Hierarchy for Ray Tracing*  
**Authors:** Sho Ikeda, Paritosh Kulkarni, Takahiro Harada (AMD)  

---

## 1. Overview  

Monte‑Carlo (path) tracing delivers physically‑accurate images but remains too expensive for real‑time use, especially in interactive applications such as video games.  The dominant cost is the traversal of the acceleration structure (usually a Bounding Volume Hierarchy – BVH) for every ray.  Traditional level‑of‑detail (LOD) schemes require separate coarse meshes and explicit switching, which adds preprocessing, memory, and implementation complexity.

The authors propose **using the BVH itself as a multi‑resolution geometric representation**.  An interior BVH node’s axis‑aligned bounding box (AABB) is treated as a *geometric proxy* for all primitives in its subtree.  By deciding, at run‑time, whether a ray can stop at that proxy instead of descending to leaf triangles, the traversal cost can be reduced without any extra geometry, preprocessing, or memory beyond two integers per node.

Because a coarse proxy supplies only a hit point, the method also introduces **stochastic material sampling** to retrieve a plausible material (and texture coordinates) from the underlying triangles.  Together with a **dynamic ray‑cone spread angle**, the technique works for both occlusion rays and the full set of rays required by global illumination/path tracing.

A lightweight extension to the hardware ray‑tracing instruction set (AMD’s `image_bvh_intersect_ray`) is suggested to make the proxy test fully resident in the GPU pipeline.

---

## 2. Main Contributions  

| # | Contribution | What it solves / why it matters |
|---|--------------|---------------------------------|
| 1 | **BVH‑based multi‑resolution geometry** – treat each internal node’s AABB as a coarse representation of its descendant triangles. | Eliminates the need for separate LOD meshes, extra storage, and preprocessing. Provides a conservative bound (no holes) and allows per‑ray, per‑region LOD selection. |
| 2 | **LOD selection during traversal** – a ray‑cone test `abs(d) < 2abs(c)·tan(θ_r) + 2w_r` decides whether the AABB is “good enough”. If true, the traversal stops at that node and the AABB is used as the hit. | Reduces the number of BVH nodes visited, especially for rays that travel far or bounce many times, yielding up to ~40 % speed‑up for ambient‑occlusion rays and ~12 % for full path‑tracing workloads. |
| 3 | **Stochastic material sampling** – each BVH node stores only two integers (the range of descendant triangle indices). When a proxy is hit, a triangle is uniformly sampled from that range, and its material (and barycentric texture coordinates) are used for shading. | Provides a material for non‑occlusion rays without storing per‑node BSDF data. Works with arbitrary complex shading networks and incurs negligible memory overhead. |
| 4 | **Dynamic spread angle (θ_r)** – the cone angle grows with the number of diffuse bounces (`θ_r = max(H(b_d‑2)·T, H(b_d‑1)·θ_δ)`). Direct‑illumination and occlusion rays keep `θ_r = 0`. | Balances accuracy vs. speed: early bounces (high visual impact) use fine geometry; later bounces (low contribution) can safely use coarse proxies. |
| 5 | **Hardware‑ray‑tracing instruction extension** – proposes augmenting AMD’s `image_bvh_intersect_ray` to return hit distance, validity flag, and hit face for each child node in a compact 32‑bit word. | Enables the proxy test to be performed entirely inside the hardware traversal unit, avoiding extra memory traffic and keeping the implementation lightweight. |
| 6 | **Implementation & quantitative evaluation** – integrated into an OpenCL path tracer, tested on a Ryzen 9 5950X + Radeon RX 6800 XT platform. Demonstrated up to 14 % overall render‑time reduction and up to 2× speed‑up for deep‑bounce rays. | Shows that the approach is practical on current consumer hardware and can be added to existing engines with only a few lines of code. |

---

## 3. Conclusion  

The paper demonstrates that **the BVH itself can serve as a hierarchical, multi‑resolution geometric proxy** without any extra data structures. By coupling this proxy with a lightweight stochastic material sampling scheme and a dynamic ray‑cone spread angle, the authors extend the usefulness of coarse geometry from simple occlusion tests to full path‑tracing workloads.  

Experimental results on realistic scenes show **significant reductions in BVH traversal cost (up to 31 % fewer node visits) and overall render time (up to 14 %)** while keeping visual error low for most cases. The approach is simple to integrate—only a few extra integers per BVH node and a modest change to the traversal loop—making it attractive for existing real‑time ray‑tracing engines.

---

## 4. Future Directions  

1. **Support for transmissive and thin geometry** – develop criteria or additional proxy tests (e.g., using oriented bounding boxes or per‑node hole information) to better handle glass, foliage, and objects with internal cavities.  
2. **Improved material sampling** – incorporate triangle area, normal distribution, or importance sampling into the stochastic selection to reduce bias on non‑uniform meshes.  
3. **Hardware implementation** – collaborate with GPU vendors to realize the proposed `image_bvh_intersect_ray` extension, evaluate its impact on latency and power, and possibly expose a programmable “proxy‑accept” flag.  
4. **Adaptive spread‑angle policies** – learn or analytically derive optimal `θ_r` schedules per scene or per bounce, perhaps using a perceptual error metric or variance estimator.  
5. **Integration with denoising pipelines** – study how the additional approximation noise interacts with modern spatiotemporal denoisers; the coarse proxy may actually improve denoiser convergence for deep‑bounce rays.  
6. **Animation and dynamic scenes** – investigate how the proxy selection behaves when geometry deforms or moves, and whether incremental BVH updates preserve the LOD benefits without costly rebuilds.  
7. **Hybrid LOD schemes** – combine BVH‑based proxies with traditional mesh‑based LODs for objects that benefit from explicit geometry reduction (e.g., highly detailed characters).  

