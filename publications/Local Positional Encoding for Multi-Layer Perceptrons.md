---
layout: default
title: ""
permalink: /lpe/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 

# Local Positional Encoding for Multi‑Layer Perceptrons

## Overview
The paper addresses the problem of representing high‑frequency signals (e.g., detailed textures, fine geometry) with compact neural networks, specifically Multi‑Layer Perceptrons (MLPs). Conventional input encodings—**positional (Fourier) encoding** and **grid‑based encoding**—either require a large number of network parameters or a high‑resolution grid that consumes substantial memory. The authors propose **Local Positional Encoding (LPE)**, a hybrid that stores per‑cell modulation weights (latent coefficients) for each sinusoidal basis of positional encoding on a low‑resolution uniform grid. By learning these coefficients jointly with the MLP weights, a small MLP can accurately reconstruct high‑frequency signals while keeping the memory footprint modest.

## Main Contributions
1. **Local Positional Encoding (LPE) formulation**  
   - A uniform 3‑D (or 2‑D) grid of resolution $N$ stores *latent coefficients* $\{c_{k}^{(i)}\}$ for each sinusoidal component of the positional encoding.  
   - For an input coordinate $\mathbf{x}\in\mathbb{R}^{d}$ the encoding is

$$
\mathbf{z}(\mathbf{x}) = \big[,c_{1}^{(1)}\sin(2\pi f_{1}\mathbf{x}),;c_{1}^{(2)}\cos(2\pi f_{1}\mathbf{x}),;\dots,;c_{K}^{(1)}\sin(2\pi f_{K}\mathbf{x}),;c_{K}^{(2)}\cos(2\pi f_{K}\mathbf{x}),\big],
$$

   where $f_{k}$ are the frequencies of the Fourier features and $K$ is the number of frequency bands.  
   - The coefficients are indexed by the grid cell containing $\mathbf{x}$; linear interpolation of the coefficients at the eight (or $2^{d}$) neighboring vertices yields a smooth spatial modulation.

2. **Joint optimization of latent coefficients and MLP parameters**  
   - The loss $\mathcal{L}$ (e.g., pixel‑wise L2 for images, signed‑distance loss for SDFs) is minimized w.r.t. both the MLP weights $\theta$ and the latent coefficients $\{c_{k}^{(i)}\}$ using Adam:

$$
\theta^*,\{c^*\}= \arg\min_{\theta,\{c\}} \mathcal{L}\big(\mathrm{MLP}_{\theta}(\mathbf{z}(\mathbf{x})),; y\big).
$$

3. **Empirical validation on two tasks**  
   - **2‑D image reconstruction**: LPE achieves higher PSNR/SSIM than pure positional encoding and outperforms grid‑based encoding with far fewer parameters.  
   - **3‑D Signed Distance Functions (SDFs)**: LPE captures fine geometric details (e.g., surface ripples) that grid encoding misses, while converging faster in early training stages.

4. **Comparison with state‑of‑the‑art multi‑resolution encodings**  
   - When matched for total parameter count, a single‑level LPE attains visual quality comparable to multi‑resolution hash encodings and surpasses single‑level multi‑resolution grids, despite using far less memory.

## Technical Details

### Positional (Fourier) Encoding
Standard Fourier features map a low‑dimensional input $\mathbf{x}$ to

$$
\gamma(\mathbf{x}) = \big[\sin(2\pi \mathbf{B}\mathbf{x}),\; \cos(2\pi \mathbf{B}\mathbf{x})\big],
$$

where $\mathbf{B}\in\mathbb{R}^{K\times d}$ contains frequencies (often sampled from a Gaussian). This enables MLPs to learn high‑frequency functions but suffers from spectral bias and axis‑aligned artifacts.

### Grid‑Based Encoding
A uniform grid of resolution $N$ stores a feature vector $\mathbf{g}_{\mathbf{v}}$ per voxel $\mathbf{v}$. The input is encoded by trilinear interpolation of the neighboring voxel features

$$
\mathbf{z}_{\text{grid}}(\mathbf{x}) = \sum_{\mathbf{v}\in\mathcal{N}(\mathbf{x})} w_{\mathbf{v}}(\mathbf{x})\,\mathbf{g}_{\mathbf{v}},
$$

where $w_{\mathbf{v}}$ are interpolation weights. High‑frequency detail requires large $N$, leading to high memory usage.

### Local Positional Encoding (LPE)
LPE combines the two ideas:

1. **Spatial decomposition**: The domain is partitioned into grid cells $\mathcal{C}$.  
2. **Per‑cell modulation**: For each cell $c\in\mathcal{C}$ and each frequency band $k$, a pair of latent coefficients $(a_{c,k}^{\sin}, a_{c,k}^{\cos})$ modulates the sinusoidal basis:

$$
\tilde{\gamma}_{c}(\mathbf{x}) = \big[ a_{c,1}^{\sin}\sin(2\pi f_{1}\mathbf{x}),\; a_{c,1}^{\cos}\cos(2\pi f_{1}\mathbf{x}),\dots \big].
$$

3. **Interpolation of coefficients**: The coefficients for a query point $\mathbf{x}$ are obtained by trilinear interpolation of the eight surrounding cell coefficients, yielding a smooth spatially varying amplitude

$$
\mathbf{a}(\mathbf{x}) = \sum_{c\in\mathcal{N}(\mathbf{x})} w_{c}(\mathbf{x})\,\mathbf{a}_{c}.
$$

4. **Final encoding**: The modulated Fourier features are

$$
\mathbf{z}_{\text{LPE}}(\mathbf{x}) = \mathbf{a}(\mathbf{x}) \odot \gamma(\mathbf{x}),
$$

where $\odot$ denotes element‑wise multiplication.

### Network Architecture
A compact MLP with three hidden layers (64 neurons each) processes $\mathbf{z}_{\text{LPE}}(\mathbf{x})$. For image reconstruction a leaky‑ReLU activation is used; for SDFs a signed‑distance output head with a final tanh activation is employed.

### Training
Both the MLP weights $\theta$ and the latent coefficients $\{\mathbf{a}_{c}\}$ are optimized jointly using Adam with learning rates $\eta_{\theta}=10^{-3}$ and $\eta_{a}=10^{-2}$. The loss for image reconstruction is

$$
\mathcal{L}_{\text{img}} = \frac{1}{|\Omega|}\sum_{\mathbf{x}\in\Omega}\big\| \mathrm{MLP}_{\theta}(\mathbf{z}_{\text{LPE}}(\mathbf{x})) - I(\mathbf{x})\big\|_{2}^{2},
$$

and for SDFs

$$
\mathcal{L}_{\text{SDF}} = \frac{1}{|\Omega|}\sum_{\mathbf{x}\in\Omega}\big| \mathrm{MLP}_{\theta}(\mathbf{z}_{\text{LPE}}(\mathbf{x})) - \phi_{\text{gt}}(\mathbf{x})\big|,
$$

where $\phi_{\text{gt}}$ is the ground‑truth signed distance.

## Limitations
- **Axis‑aligned artifacts**: Because the underlying Fourier basis is aligned with the coordinate axes, LPE inherits the same grid‑like ringing artifacts, especially noticeable on smooth surfaces.
- **Memory scaling with dimensionality**: The number of latent coefficients grows linearly with the input dimension $d$. For high‑dimensional tasks (e.g., 5‑D radiance fields) the memory cost becomes significant.
- **Single‑resolution grid**: The current implementation uses a single low‑resolution grid; while competitive, it does not fully exploit the benefits of multi‑resolution hierarchies.
- **Hash collisions**: Although LPE avoids hash‑based collisions, extending it to multi‑resolution hash grids may re‑introduce such artifacts unless carefully managed.

## Conclusion
The authors present **Local Positional Encoding**, a novel hybrid encoding that equips a small MLP with the ability to learn high‑frequency functions while keeping memory usage low. By learning per‑cell amplitude modulation of Fourier features, LPE adapts to spatially varying signal content, leading to superior visual fidelity in both 2‑D image reconstruction and 3‑D SDF modeling. Empirical results demonstrate faster early‑stage convergence compared to pure grid encodings and comparable performance to state‑of‑the‑art multi‑resolution hash encodings, despite using a single‑level grid.

## Future Directions
1. **Mitigating axis‑aligned artifacts** – Investigate alternative basis functions (e.g., learned directional Fourier bases or spherical harmonics) to reduce ringing.
2. **Multi‑resolution extension** – Incorporate a hierarchy of grids (or hash tables) into LPE, enabling finer detail capture without excessive memory.
3. **Coefficient compression** – Explore low‑rank factorization or quantization of the latent coefficients to further reduce memory for high‑dimensional inputs.
4. **Application to neural radiance fields** – Integrate LPE into NeRF‑style view synthesis pipelines, potentially replacing or augmenting existing hash‑based encodings.
5. **Adaptive frequency selection** – Dynamically adjust the set of frequencies $\{f_{k}\}$ per cell based on local signal complexity, akin to adaptive Fourier feature selection.
