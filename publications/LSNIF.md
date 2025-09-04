---
layout: default
title: ""
permalink: /LSNIF/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 

# LSNIF: Locally-Subdivided Neural Intersection Function

**Paper:** *Locally‑Subdivided Neural Intersection Function (LSNIF)* – SIGGRAPH 2024 (Washington, DC)  

**Authors:** Shin Fujieda, Chih‑Chen Kao, Takahiro Harada  

---

### 1. Problem & Motivation
- **Ray‑tracing bottleneck:** Bounding‑volume hierarchies (BVHs) dominate runtime cost in path tracing, especially for complex scenes with millions of triangles.
- **Goal:** Replace the geometry‑specific part of a BVH with a compact, neural representation that can be queried at render time, while still supporting typical scene edits (camera, lighting, object transforms).

### 2. Core Idea – LSNIF
- **Neural Intersection Function (NIF):** A small MLP that, given a ray and a set of sampled points along the ray, predicts the exact intersection point, surface normal, and shading normal.
- **Local subdivision:** The scene is partitioned into axis‑aligned bounding boxes (AABBs). Each AABB contains its own NIF, trained independently (“locally‑subdivided”).
- **Training data:** For each AABB, rays are cast through a dense voxel grid; the first *H* hit points (up to 64) are collected, rasterized to obtain ground‑truth geometry (normals, shading normals). These samples become the training set.
- **Encoding:** Multi‑resolution hash‑grid positional encoding (Müller et al. 2022) is used to compress the voxel‑level geometry into a low‑dimensional feature vector, which is fed to the MLP.

### 3. Methodology

| Component | Details |
|-----------|---------|
| **Voxelization** | Uniform grid of resolution *V* (e.g., 64³). Occupancy stored as a binary 3‑D texture. |
| **Feature extraction** | For each ray, up to *H* intersection points are sampled; their hash‑encoded features are concatenated → input vector (≈ H·C). |
| **Network** | 4‑layer MLP (256 hidden units) with ReLU; trained with Adam (lr = 1e‑3) for 200 k iterations. |
| **Hash table size** | *M* = 2⁹ (≈ 512 entries) – balances memory and quality. |
| **Training** | Offline, viewpoint‑independent, lighting‑independent. Takes ~2 min per object on an AMD CDNA‑2 GPU. |
| **Inference** | At render time, a ray traverses the AABB of an LSNIF; the MLP predicts intersection distance and normals. |

### 4. Key Results
- **Memory compression:** Geometry that would require > 200 MB BVH (e.g., Stanford Bunny + Dragon + Statues) is stored in < 20 MB of neural parameters.
- **Visual fidelity:**  
  - FLIP error ≈ 0.008 vs. 0.024 for the prior Neural Intersection Function (NIF).  
  - Outperforms N‑BVH (0.007 vs. 0.019) and geometric LODs at comparable memory budgets.  
- **Scene editing:** Light, camera, and rigid‑body transforms can be changed without retraining; demonstrated on animated sequences (Fig. 13).  
- **Performance:**  
  - On AMD CDNA‑2, LSNIF inference ≈ 2–3 × slower than hardware‑accelerated BVH traversal (still > 10 k rays/s).  
  - DirectX Ray‑Tracing (DXR) integration works but incurs divergence overhead; future work targets Work‑Graph API for batch inference.  

### 5. Limitations

| Issue                                 | Why it matters                                                                         | Current handling                                              |
| ------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Topological changes / deformation** | Network encodes a fixed surface.                                                       | Requires retraining.                                          |
| **Texture coordinates**               | No UV prediction → no texture mapping, normal maps.                                    | Rasterization for primary visibility only.                    |
| **Opaque‑only assumption**            | No support for transmissive or subsurface scattering.                                  | Future extension planned.                                     |
| **Input size**                        | Concatenating many point features yields large vectors → larger MLP, slower inference. | Fixed *H* = 64; explore compact encodings.                    |
| **LOD**                               | No hierarchical detail; multiple resolutions would waste memory.                       | Simple multi‑resolution voxel grids; need smarter LOD scheme. |
| **Self‑intersection epsilon**         | Fixed ε may cause artifacts on very thin geometry.                                     | Fixed small ε; adaptive scheme left for future work.          |
| **Camera types**                      | Pipeline relies on rasterization for primary visibility → perspective only.            | Extend to spherical/fisheye cameras.                          |

### 6. Future Directions
1. **Training‑free texture handling:** Learn stable UV predictions or integrate neural texture compression (e.g., Neural Texture Block Compression).  
2. **More efficient input representation:** Reduce the dimensionality of the concatenated feature vector (e.g., attention‑based pooling).  
3 Adapt **hardware acceleration**: Leverage AMD matrix cores / work‑graph API to batch MLP inference and reduce divergence.  
4. **Support for transmissive/sub‑surface materials** and **adaptive epsilon** for robust self‑intersection handling.  
5. **Level‑of‑Detail (LOD) for neural geometry**: hierarchical LSNIFs or continuous‑detail encodings.

### 7. Take‑away
LSNIF demonstrates that a **neural intersection function** can replace the geometry‑specific portion of a BVH, achieving **orders‑of‑magnitude memory compression** while still allowing **post‑training scene edits**. Although inference speed still lags behind dedicated hardware BVH traversal, the approach shifts most of the computational burden to an offline training phase, opening a path toward neural‑accelerated ray tracing as matrix‑core hardware matures.
