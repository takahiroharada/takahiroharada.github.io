---
layout: default
title: ""
permalink: /ntbc/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 

# Neural Texture Block Compression

**Paper:** *Neural Texture Block Compression*  
**Authors:** Shin Fujieda, Takahiro Harada

---

### 1. Problem & Motivation
- **Material textures** (diffuse, normal, roughness, etc.) are often stored as **block‑compressed (BC) formats** (e.g., BC1‑BC5) to keep GPU memory low.
- Traditional BC encoders treat each texture **independently**, ignoring the strong **spatial and semantic correlations** that exist across the multiple maps of a single material.
- Existing neural‑based texture compressors either:
  - Encode each map separately (low compression gain), or  
  Random‑access neural compressors that need **large latent grids** (high memory, slow inference).

**Goal:** Learn a **compact, shared neural representation** for *all* texture maps of a material, enabling **sub‑linear storage** while preserving the ability to **random‑access decode** each texel in real time.

---

### 2. Core Idea
- **Treat a material as a set of correlated 2‑D signals** (RGB maps + single‑channel maps) and learn a **single neural field** that can reconstruct *any* map on demand.
- Use a **tiny MLP** (≈ 2 k parameters) plus a **learned hash‑grid encoding** (as in Instant‑NGP) to map a 2‑D coordinate **(u, v)** to a **latent feature vector**.
- The latent vector is then passed through **two lightweight heads**:
  1. **Endpoint head** – predicts the four palette colors (the “endpoints”) for a BC block.
  2. **Weight head** – predicts the per‑pixel index (0‑3 for BC1, 0‑7 for BC4) using a **soft‑argmax** (straight‑through estimator) so the whole pipeline stays differentiable.

- The network is **trained jointly** on all maps of a material, sharing the same hash‑grid and MLP weights.  
- At inference time, the **hash‑grid is baked into a small binary table** (≈ 10 KB) and the MLP weights are stored once per material.

---

### 3. Technical Contributions
| Contribution | What it solves | How it’s done |
|--------------|----------------|---------------|
| **Unified material‑wide neural field** | Captures inter‑map redundancy (e.g., diffuse and roughness often share edges) | Same hash‑grid + MLP for all maps; separate output heads only. |
| **Learned BC palette & indices** | Produces *valid* BC1/BC4 data that can be decoded by existing hardware pipelines | Soft‑max over distances → expected weight; STE for argmax to obtain hard indices. |
| **Compact storage format** | Reduces per‑material footprint from ~48 MB (standard BC) to **13 MB (aggressive)** or **27 MB (conservative)** while keeping random‑access. | Store: (i) 4 × 4 byte endpoints per 4×4 block, (ii) a tiny latent table (hash‑grid) and MLP weights. |
| **Real‑time random‑access decoding** | No need to decode whole texture; suitable for streaming or on‑the‑fly material swaps. | Decode a texel by querying the hash‑grid → MLP → endpoint/weight → reconstruct color. |

---

### 4. Results (Key Numbers)

| Dataset | Material | Original BC size | Aggressive NTBC | Conservative NTBC | PSNR (dB) – Diffuse |
|---------|----------|------------------|-----------------|-------------------|----------------------|
| **ambientCG** – *MetalPlates013* | 6 maps (RGB + 4 single‑channel) | 48 MB | **13.4 MB** (72 % reduction) | **26.7 MB** (45 % reduction) | 38.6 (aggr) / 38.6 (cons) vs. 42.5 (reference) |
| **Poly Haven** – *roof_09* | 5 maps | 40 MB | **13.4 MB** | **26.7 MB** | 37.4 / 37.4 vs. 40.3 (reference) |
| **Poly Haven** – *forest_sand_01* (hard case) | 5 high‑freq maps | 40 MB | **13.4 MB** | **26.7 MB** | 25.9 / 26.0 vs. 31.4 (reference) |

- **Visual quality:** Naïve BC (or a naïve NN that predicts weights directly) shows obvious block artifacts. NTBC removes most artifacts, at the cost of slight blur.
- **Performance:** Encoding (training) takes ~20 k steps per material (≈ 5 min on a RTX 3090). Decoding a texel is < 1 µs, fully compatible with real‑time shaders.

---

### 5. Significance & Impact
- **Storage‑efficient material libraries:** Game engines and asset pipelines can store *entire* material sets (diffuse, normal, roughness, AO, etc.) in a **fraction of the usual BC footprint** without sacrificing random‑access.
- **Drop‑in compatibility:** The compressed data can be fed to existing BC decoders; only the *encoding* stage is neural.
- **Scalable to higher‑quality BC formats:** The same framework can be extended to BC7 or ASTC, promising even better visual fidelity.
- **Foundation for future work:** Combining this with *texture‑specific encodings* (e.g., latent‑space for specular maps) or *learned hash probing* could push quality further while retaining the compact, random‑access nature.

---

### 6. Take‑away TL;DR
The authors present a **material‑wide neural compressor** that learns a tiny shared neural field for all texture maps of a material, outputs **valid block‑compressed data (BC1/BC4)**, and achieves **> 70 % storage reduction** with **real‑time random‑access decoding**. It bridges the gap between classic BC compression (fast, hardware‑friendly) and modern neural codecs (high compression, but usually non‑random‑access), opening a practical path for neural texture compression in real‑time rendering pipelines.

