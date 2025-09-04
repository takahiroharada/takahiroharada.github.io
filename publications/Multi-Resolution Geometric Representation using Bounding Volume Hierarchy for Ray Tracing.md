---
layout: default
title: ""
permalink: /Multi-Resolution Geometric Representation using Bounding Volume Hierarchy for Ray Tracing/
---

# Multi-Resolution Geometric Representation using Bounding Volume Hierarchy for Ray Tracing  

## Overview  
The paper proposes a **multi‑resolution geometric representation** that leverages the **Bounding Volume Hierarchy (BVH)** already built for ray tracing. By treating the axis‑aligned bounding boxes (AABBs) of BVH nodes as coarse geometric proxies, the method can dynamically select a level of detail (LOD) during ray traversal without any extra pre‑processing, additional storage, or separate LOD meshes. To make this approximation usable for **global illumination** (i.e., non‑occlusion rays), the authors introduce **stochastic material sampling**, which retrieves material information from the underlying triangles of a BVH node at runtime. The approach is implemented with only two extra integer fields per BVH node and can be integrated into existing ray‑tracing pipelines by a modest change to the BVH traversal logic. Experimental results on several scenes show up to **~14 % reduction in total rendering time** and **~12 % reduction in ray‑casting time**, with visual errors that are generally acceptable for many real‑time applications.

## Main Contributions  

### 1. Multi‑Resolution Geometry via BVH AABBs  
- **Concept**: Use the AABB of any BVH node as a geometric proxy for all triangles in its subtree.  
- **LOD Selection**: During traversal, test whether the proxy is “good enough” for the current ray. If it passes, the traversal stops at that node; otherwise, the algorithm descends to children for finer detail.  
- **Ray‑Cone Test**: A ray cone with spread angle θᵣ is tracked. The proxy is accepted when the virtual sphere enclosing the AABB lies completely inside the cone, i.e.,  

$$
\| \mathbf{d} \| \;<\; 2\|\mathbf{c}\|\tan(\theta_r) \;+\; 2 w_r,
$$

where **d** is the AABB diagonal vector, **c** the vector from ray origin to the AABB centre, and *wᵣ* the cone width from the previous path segment.  

- **Dynamic Spread Angle**: The cone angle is adapted per ray bounce:  

$$
\theta_r \;=\; \max\!\bigl( H(b_d\!-\!2)\,T,\; H(b_d\!-\!1)\,\theta_\delta \bigr),
$$

with *b_d* the number of diffuse bounces, *T* a user‑specified spread, *θ_δ* a small fallback angle (3°), and *H(x)* the Heaviside step function  

$$
H(x)=\begin{cases}
1 & x\ge 0,\\
0 & x<0.
\end{cases}
$$

### 2. Stochastic Material Sampling  
- **Problem**: A coarse proxy lacks per‑triangle material data, which is required for shading in path tracing.  
- **Solution**: Store, for each BVH node, the **range of triangle indices** belonging to its subtree (two integers). When a proxy is hit, a triangle is **uniformly sampled** from this range, and its material (including complex shading networks) is used for the hit point.  
- **Texture Coordinates & Barycentrics**: After selecting a triangle, a random barycentric coordinate is sampled to interpolate texture coordinates. The geometric normal of the AABB is used as a sufficient shading normal.  

### 3. Hardware‑Ray‑Tracing Extension (Conceptual)  
- Proposes extending the AMD `image_bvh_intersect_ray` instruction to return, for each child AABB, the hit distance, a validity flag, and the hit face (encoded in 32 bits). This enables the approximation test to be performed entirely inside the hardware instruction, minimizing extra work in shader code.

## Limitations  

| Aspect | Limitation |
|--------|------------|
| **Approximation Accuracy** | The coarse AABB proxy can introduce noticeable errors for rays that interact with fine geometry (e.g., specular reflections, refractions, or when a light source is hidden by a small opening). |
| **Material Sampling Bias** | Uniform triangle sampling assumes a uniform distribution of triangles in direction space; highly non‑uniform meshes may lead to biased material selection. |
| **Area Light Handling** | Geometric approximation changes the effective area of emissive geometry. The authors mitigate this by flagging BVH nodes that contain area lights and disabling the proxy for them, but this adds a per‑scene preprocessing step. |
| **Hardware Dependency** | The proposed hardware instruction extension is not yet available on current GPUs; practical adoption would require ISA changes. |
| **Performance Bottlenecks** | In scenes where shading/shader execution dominates (e.g., heavy use of complex materials), the speed‑up from reduced traversal is limited. |

## Conclusion  
The authors demonstrate that **BVH‑based multi‑resolution geometry** combined with **stochastic material sampling** can effectively reduce ray‑tracing workload for both occlusion and global‑illumination rays. The method requires only minimal changes to existing traversal code and a negligible memory overhead (two integers per BVH node). Empirical results show up to **40 % speed‑up for ambient‑occlusion ray casting** and **~14 % overall rendering time reduction** in full path‑tracing workloads, while maintaining visual fidelity that is acceptable for many real‑time scenarios.

## Future Directions  

1. **Transmissive Surfaces** – Extend the stochastic material sampling and proxy acceptance criteria to handle refraction and volumetric scattering.  
2. **Improved Approximation Criteria** – Develop more sophisticated tests (e.g., considering hole preservation) to avoid large errors when a proxy would hide important geometric features such as torus openings.  
3. **Adaptive Sampling** – Incorporate triangle area and orientation into the stochastic material selection to reduce bias on non‑uniform meshes.  
4. **Hardware Realization** – Implement and evaluate the proposed `image_bvh_intersect_ray` extension on upcoming GPU architectures, measuring actual hardware‑level benefits.  
5. **Animation & Temporal Coherence** – Investigate how the proxy selection can be made temporally stable across frames to avoid flickering in animated scenes.  

