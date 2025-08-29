---
layout: default
title: ""
permalink: /NIF/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 


# Neural Intersection Function

**Paper:** *Neural Intersection Function (NIF) – Accelerating Ray‑Casting with Neural Networks*  
**Authors:** S. Fujieda, C. C. Kao, T. Harada (2023)

---

## 1. Problem & Motivation
- **Ray‑tracing** (especially shadow‑ray casting) spends a large fraction of its time traversing the **bottom‑level BVH** (BL‑BVH).  
- BL‑BVH traversal is highly divergent on GPUs: many threads follow different branches, leading to poor memory‑coherence and low utilization of the GPU’s massive parallelism.  
- Recent work has shown that **neural networks (NNs)** can represent geometry implicitly, but they have been used only as *stand‑alone* renderers or for a single object, not as a drop‑in replacement inside a conventional ray‑tracing pipeline.

**Goal:** Replace the BL‑BVH traversal with a learned function that can be embedded in an existing BVH‑based renderer, preserving the rest of the pipeline (top‑level BVH, shading, etc.) while gaining speed.

---

## 2. Core Idea – Neural Intersection Function (NIF)

NIF consists of **two small MLPs**:

| Network | Purpose | Input (parameterization) |
|---------|---------|--------------------------|
| **Outer** | Coarse “does this ray intersect any object at all?” (visibility) | 3‑D ray‑origin, ray‑direction, and a *grid‑encoded* feature vector (position, direction, distance) |
| **Inner** | Fine‑grained “where does the ray intersect the first object?” (distance) | Same grid‑encoded features + the output of the outer network (visibility) |

Both networks share a **grid‑encoding** of the ray parameters (position, direction, distance) that eliminates aliasing caused by discretisation of the view‑space. The encoding is a set of **learned feature grids** (similar to the “feature‑grid” used in recent implicit‑function works) that map continuous ray parameters to high‑dimensional vectors before feeding them to the MLPs.

### Training Pipeline
1. **Data collection:** While rendering a frame from a given camera, a small number of *shadow rays* (or any secondary rays) are generated. Their true intersection results (visibility, distance) are obtained by the standard BVH traversal.
2. **Online / offline training:** The collected ray‑intersection pairs are used to train the outer and inner networks on‑the‑fly (online) or offline.  
   - Losses: binary cross‑entropy for visibility (outer) and L1/L2 for distance (inner).  
   - Optimizer: Adam, Xavier/Glorot initialization.
3. **Inference:** At subsequent frames (or the same frame with more samples) the trained NIF replaces the BL‑BVH traversal for those rays.

### Integration
- NIF is **optional**: it is invoked only for objects whose triangle count exceeds a threshold (e.g., > 100 K). Simpler geometry still uses the classic BVH.
- The rest of the rendering pipeline (top‑level BVH, shading, material evaluation, primary‑ray rasterisation) remains unchanged.

---

## 3. Technical Contributions

| Contribution | Description |
|--------------|-------------|
| **Two‑stage neural architecture** (outer + inner) that splits the problem into a cheap visibility test and a more expensive distance regression only when needed. |
| **Grid‑encoding of ray parameters** (position, direction, distance) that removes aliasing and provides a smooth, high‑frequency representation for the MLPs. |
| **Single shared network for the whole scene** (instead of per‑object networks). The same outer/inner networks handle all objects, using the same feature grids. |
| **GPU‑native implementation** using AMD’s HIP (v4.5) and leveraging wave‑level matrix‑multiply instructions (RDNA‑3). |
| **Online training** that can be performed in a few hundred milliseconds per sample‑per‑pixel (spp). |

---

## 4. Experimental Results

| Scene (triangles) | BVH Ray‑cast time (µs) | NIF Ray‑cast time (µs) | Speed‑up | PSNR / Image quality |
|-------------------|------------------------|------------------------|----------|----------------------|
| DRAGON A (7.2 M)  | 625.3 | 139.5 | **1.24×** | Δ≈0.1 dB |
| CENTAUR A (2.5 M) | 592.0 | 123.4 | **1.46×** | Δ≈0.1 dB |
| STATUETTE (10 M)  | 391.0 | 86.2  | **1.53×** | Δ≈0.1 dB |
| STATUETTE LOW (0.5 M) | 260.6 | 91.8 | **1.00×** (no gain) | – |
| BISTRO (5 complex models, total ≈ 120 M) | 774.0 | 119.9 (complex part) + 260.6 (simple part) ≈ 380 µs | **~15 %** overall reduction | Δ≈0.1 dB |

- **Linear scaling:** The NIF runtime scales linearly with the *number of rays* processed, not with the geometric complexity (triangle count). This explains why low‑poly models see no speed‑up.
- **Image fidelity:** Shadows computed with NIF are visually indistinguishable from BVH‑based shadows (PSNR > 40 dB, Δ amplified for visual inspection still tiny).
- **Primary‑ray use:** NIF can also output shading normals and depth, enabling primary‑ray acceleration (though this was not the primary benchmark).

---

## 5. Limitations & Open Issues

| Limitation | Reason | Potential Remedy |
|------------|--------|------------------|
| **View‑specific training** – NIF is trained only on rays from the current camera. Queries from far‑off viewpoints suffer accuracy loss. | Training data is limited to the current view; the learned function does not generalise far away. | *Multi‑view* or *global* training (e.g., sampling a broader set of viewpoints) or a hierarchical mixture‑of‑experts that can be switched on‑the‑fly. |
| **Dynamic scenes** – No efficient online update scheme yet. | Current training is per‑frame, which would be too costly for moving geometry. | Incremental training / continual‑learning approaches, or a hybrid where only the *inner* network is updated for deformations. |
| **Redundant queries** – Same ray may be evaluated multiple times if an intersection is already known. | Simple kernel design does not cache intermediate results. | Ray‑level caching or early‑exit logic (adds kernel complexity; benefit currently marginal). |
| **Only secondary‑ray visibility** – Primary‑ray shading, deeper shadow‑ray paths, or other ray types not yet explored. | Focused on the most expensive part (BL‑BVH). | Extend the inner network to output additional geometric descriptors (e.g., SDF, curvature) and test on full path‑tracing workloads. |

---

## 6. Conclusions

- **NIF** demonstrates that a *learned* intersection function can replace the costly bottom‑level BVH traversal for complex geometry, yielding up to **1.53×** speed‑up while preserving visual quality.
- The approach is **compatible** with existing GPU ray‑tracing pipelines: you can selectively enable NIF for high‑poly objects and fall back to classic BVH for the rest.
- As GPU hardware continues to accelerate matrix‑multiply/ML workloads (tensor cores, wave‑level instructions), the relative advantage of NIF is expected to **grow**.

---

## 7. Take‑away for Practitioners

| When to use NIF | How to integrate |
|-----------------|------------------|
| **High‑poly models** (≥ 100 K triangles) where BL‑BVH traversal dominates. | Replace the BL‑BVH traversal kernel with the NIF outer/inner inference kernels; keep the TL‑BVH unchanged. |
| **Static scenes** or scenes where the camera does not move dramatically between frames. | Train NIF once (or a few times) per view; reuse the trained weights for many frames. |
| **Dynamic or highly view‑varying scenes** | Not yet ready; consider hybrid approaches (partial NIF, frequent re‑training) or wait for future work. |

---

### Quick Pseudocode (high‑level)

```cpp
// Pre‑render: collect training rays from current view
for each shadowRay r:
    hit = BVH_RayCast(r);               // ground‑truth
    store (r.origin, r.dir, hit.visible, hit.dist);

// Train outer/inner networks (online or offline)
train_outer_network(training_data);
train_inner_network(training_data);

// Render loop
for each shadowRay r:
    // 1️⃣ Outer query (visibility)
    feat = grid_encode(r.origin, r.dir, /*distance=*/0);
    visible = outer_MLP(feat);
    if (!visible) continue; // early out: no occluder

    // 2️⃣ Inner query (distance)
    feat = grid_encode(r.origin, r.dir, /*estimated distance*/);
    dist = inner_MLP(feat);
    // Use `dist` to compute shadow attenuation, etc.
```

---

**Bottom line:** NIF shows that a compact neural representation can serve as a *drop‑in* accelerator for the most divergent part of GPU ray‑tracing pipelines. It opens a new research direction: **embedding learned geometry functions directly inside production renderers**. Future work will need to address view‑generalisation, dynamic updates, and richer geometric outputs.

