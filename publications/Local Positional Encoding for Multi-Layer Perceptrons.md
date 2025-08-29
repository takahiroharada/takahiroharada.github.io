---
layout: default
title: ""
permalink: /lpe/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 

# Local Positional Encoding for Multi-Layer Perceptrons

**Paper:** *Local Positional Encoding for Multi‑Layer Perceptrons* – Fujieda, Yoshimura, Harada (Eurographics 2023)  

---

### 1. Problem & Motivation
- **Implicit neural representations** (e.g., NeRF, SDFs) need to encode high‑frequency detail.
- **Standard positional (Fourier) encoding** lets a small MLP learn high‑frequency signals, but it suffers from spectral bias and requires many sinusoidal bases → large memory.
- **Grid‑based (hash) encodings** store learned features in a spatial grid and use a tiny MLP as a decoder, but they need multiple resolution levels or huge hash tables to capture fine detail, which becomes memory‑intensive in higher dimensions.

**Goal:** Design an input encoding that lets a *single‑level, low‑resolution grid* and a *small MLP* represent high‑frequency signals with a modest memory footprint.

---

### 2. Core Idea – Local Positional Encoding (LPE)

| Component | Traditional approach | LPE |
|-----------|----------------------|-----|
| **Positional encoding** | Fixed sinusoidal amplitudes (same everywhere) | **Latent coefficients** (weights) per sinusoid stored in a grid cell |
| **Grid** | Stores raw feature vectors (high‑dimensional) → large memory | Stores **scalar coefficients** that modulate the amplitude of each Fourier basis locally |
| **Learning** | Only MLP weights are trained | Both MLP weights **and** grid‑cell coefficients are optimized jointly by SGD |

**How it works**

1. **Uniform grid** (e.g., $N^d$ cells) is overlaid on the input domain.
2. For each input coordinate $\mathbf{x}$ we:
   - Locate the containing cell and retrieve its **latent coefficients** $\{c_k\}$ (one per Fourier frequency).
   - Compute the usual Fourier features $\phi_k(\mathbf{x}) = \sin(2\pi f_k \mathbf{x})$ / $\cos(\cdot)$.
   - Multiply each feature by its local coefficient: $\tilde{\phi}_k(\mathbf{x}) = c_k \, \phi_k(\mathbf{x})$.
3. Concatenate the modulated features and feed them to a **tiny MLP** (e.g., 3 layers, 64 neurons per layer).
4. During training, gradients flow back to both the MLP parameters and the per‑cell coefficients, allowing the network to **adapt the amplitude of each frequency locally**—akin to a short‑time Fourier transform.

The scheme can be seen as a **spatially‑adaptive Fourier feature map**: high‑frequency components are amplified only where needed, while low‑frequency regions keep coefficients small, reducing spectral bias.

---

### 3. Technical Details

- **Positional encoding**: $ \phi(\mathbf{x}) = [\sin(2\pi f_i \mathbf{x}), \cos(2\pi f_i \mathbf{x})]_{i=1}^{M}$.  
- **Latent coefficients**: For each grid cell and each input dimension a vector $\mathbf{c}\in\mathbb{R}^{2M}$ is stored (one scalar per sinusoid).  
- **Training**: Adam optimizer, learning rate schedule as in Instant‑NGP; coefficients are updated jointly with MLP weights.  
- **Boundary handling**: Linear interpolation of coefficients from the eight (or $2^d$) neighboring cells to avoid discontinuities.  
- **Memory**: Roughly $N^d \times 2M \times d$ scalars; far less than storing full‑dimensional feature vectors for each cell (as in hash‑grid methods).

---

### 4. Experiments & Results

| Task | Baselines | Metric | LPE (single‑level grid) | Observations |
|------|-----------|--------|--------------------------|--------------|
| **2‑D image reconstruction** (1024×1024) | Positional (PE), Grid (GE) | PSNR, SSIM | **Higher PSNR/SSIM** (e.g., Bridge: 31.2 dB vs 30.1 dB) | Captures fine texture with a 64‑cell grid; visual artifacts are axis‑aligned (inherited from PE). |
| **3‑D Signed Distance Functions** (SDF) | PE, GE, Multi‑resolution Grid, Multi‑resolution Hash | IoU (intersection‑over‑union) | **IoU 0.9736** (single‑level) vs 0.9702 (multi‑grid) and 0.9848 (hash) | LPE reproduces high‑frequency surface details (e.g., statue’s hair) earlier in training; converges faster than GE. |
| **Training speed** | GE | – | **~2× faster early convergence** (64–1 024 iterations) | Because local coefficients quickly shape the spectrum. |

*Qualitative* figures show that LPE preserves high‑frequency texture (e.g., wood grain, statue wrinkles) that GE blurs out, while using far fewer parameters than multi‑resolution hash tables.

---

### 5. Significance & Contributions

1. **Hybrid encoding** that merges the spectral richness of Fourier features with the spatial locality of grid‑based primitives.
2. **Memory‑efficient**: a single low‑resolution grid + a tiny MLP can rival multi‑resolution hash encodings that need millions of entries.
3. **Training efficiency**: local amplitude control yields rapid early‑stage learning of fine details.
4. **General applicability**: demonstrated on both 2‑D image fitting and 3‑D SDF reconstruction; the method is agnostic to the downstream task (could be extended to radiance fields, material models, etc.).

---

### 6. Limitations & Future Work

- **Axis‑aligned artifacts** remain due to the underlying sinusoidal basis; possible remedies include rotating bases or using learned, non‑axis‑aligned frequencies.
- **Scalability to high‑dimensional inputs** (e.g., 5‑D light fields) increases memory linearly with dimension; compressing coefficient storage (e.g., low‑rank factorization) is an open direction.
- **Multi‑resolution extension**: the authors note that stacking several LPE grids (coarse‑to‑fine) should further improve quality and is a natural next step.

---

### 7. Bottom‑Line Takeaway

*Local Positional Encoding* provides a **simple yet powerful** way to endow a compact MLP with the ability to model high‑frequency signals while keeping the memory budget low. By learning per‑cell amplitude weights for Fourier features, the method adapts to spatially varying detail, achieving results comparable to state‑of‑the‑art multi‑resolution hash encodings but with a single‑level grid and a much smaller network. This makes it attractive for real‑time or resource‑constrained neural graphics applications.
