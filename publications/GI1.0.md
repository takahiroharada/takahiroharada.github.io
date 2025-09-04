---
layout: default
title: ""
permalink: /gi-1.0/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 

# GI-1.0: A Fast Scalable Two‑Level Radiance Caching Scheme for Real‑Time Global Illumination  

## Overview  
The paper presents **GI‑1.0**, a complete ray‑traced global illumination (GI) pipeline designed for real‑time rendering of indirect diffuse lighting on modern GPUs. The method combines hardware‑accelerated ray tracing (DXR‑1.1) with a novel two‑level radiance caching hierarchy: a **screen‑space cache** (a sparse grid of probes) and a **world‑space cache** (a hash‑based spatial structure). By reusing cached radiance across frames and guiding rays with temporal variance information, the system achieves high‑quality indirect illumination at sub‑millisecond cost on contemporary graphics cards. Direct lighting from environment maps and emissive surfaces is handled at no extra cost thanks to the same guiding infrastructure.

## Main Contributions  

| # | Contribution | Description |
|---|--------------|-------------|
| 1 | **Two‑Level Radiance Caching** | A **screen cache** stores low‑frequency illumination on a regular probe grid. A **world cache** stores high‑frequency radiance in a hash‑based spatial hash (tiles → buckets). The two levels are linked via a *pre‑filter* that aggregates world‑cache samples into the screen probes. |
| 2 | **Tile‑Based Hashing with Adaptive Mip‑Levels** | World‑cache entries are placed in a 2‑D hash table indexed by tile coordinates $(x, y)$ and a mip‑level $m$ that adapts to the distance from the camera. The descriptor for a tile is $\mathbf{h} = \text{hash}(x, y, m)$. |
| 3 | **Temporal Radiance Feedback** | Radiance computed for a secondary bounce in frame $t$ is stored in the screen cache and reused in frame $t+1$ via reprojection, effectively accumulating an arbitrary number of bounces over time. |
| 4 | **World‑Space Spatiotemporal Reservoir Resampling (ReSTIR‑GI)** | The method builds on spatiotemporal reservoir sampling: each secondary ray creates a reservoir that is resampled across neighboring tiles and previous frames, reducing variance and the number of required shadow‑ray queries. |
| 5 | **Light‑Grid Sampling** | A pre‑computed **light grid** stores, for each spatial region, a short list of the most important lights (including view‑frustum lights). This list guides importance sampling of direct illumination for secondary rays, drastically reducing the number of shadow‑ray checks. |
| 6 | **Hybrid Screen‑Space / World‑Space Evaluation** | Direct illumination for secondary vertices is evaluated by tracing a single ray toward the light source; if the ray hits geometry, the world‑cache radiance is fetched, otherwise the environment map is used. This eliminates the need for a full shadow‑map cascade. |
| 7 | **Spherical‑Harmonic Projection & Temporal Denoising** | Screen probes are projected onto low‑order spherical harmonics (SH) for compact storage. A spatiotemporal variance‑guided filter (SVGF) denoises the per‑pixel interpolated irradiance, using a dis‑occlusion mask derived from reprojection. |
| 8 | **Implementation in DirectX 12 & UE5** | The authors integrate GI‑1.0 into a research DXR‑1.1 framework and as a plugin for Unreal Engine 5, enabling direct side‑by‑side visual and performance comparison with Lumen. |

### Key Equations  

1. **Rendering Equation (simplified for diffuse bounce)**  
$$
L_o(\mathbf{p},\omega_o) = L_e(\mathbf{p},\omega_o) + 
\int_{\Omega^+} f_d(\mathbf{p},\omega_i,\omega_o)\,
L_i(\mathbf{p},\omega_i)\,(\mathbf{n}\!\cdot\!\omega_i)\,d\omega_i .
$$

2. **Screen‑Probe Irradiance Accumulation** (exponential moving average across frames)  
$$
\hat{E}_t = \alpha\,E_t + (1-\alpha)\,\hat{E}_{t-1},
\qquad
\alpha = \frac{1}{1 + \sigma^2_t},
$$
where $\sigma^2_t$ is the per‑probe variance estimated from the current and previous frames.

3. **World‑Cache Radiance Prefilter (mip‑level blending)**  
$$
C_{m} = \lambda\,C_{m-1} + (1-\lambda)\,C_{m},
\qquad
\lambda \in [0,1],
$$
with $m$ the mip‑level of the tile’s hash bucket.

4. **Temporal Radiance Feedback** (reprojection of cached radiance)  
$$
L^{\text{fb}}_{t+1}(\mathbf{p}) = 
\operatorname{Reproj}\!\big(L^{\text{fb}}_{t}(\mathbf{p})\big) + 
L^{\text{new}}_{t+1}(\mathbf{p}),
$$
where $\operatorname{Reproj}$ maps the previous frame’s radiance to the current camera view.

5. **Light‑Grid Importance Metric** (biased clamped intensity)  
$$
I_g(\mathbf{r}) = \sum_{l\in\mathcal{L}_g}
\max\big(0,\mathbf{n}_l\!\cdot\!\mathbf{d}_l\big)\,
\min\big(I_l, I_{\text{max}}\big),
$$
with $\mathcal{L}_g$ the set of lights intersecting region $g$, $\mathbf{d}_l$ the direction to light $l$, and $I_{\text{max}}$ a clamping threshold.

6. **Spatiotemporal Reservoir Resampling Weight** (Generalized ReSTIR)  
$$
w_{i\rightarrow j} = \frac{p_j}{\sum_{k\in\mathcal{N}(i)} p_k},
$$
where $p_j$ is the importance of sample $j$ and $\mathcal{N}(i)$ the neighbourhood of reservoir $i$.

## Limitations  

1. **Visibility‑Dependent Multi‑Bounce Approximation** – The radiance feedback mechanism only propagates indirect light when the reflecting surface was visible in the previous frame. Occluded bounces (common in interior scenes) may be under‑represented, leading to visible “ghosting” or missing illumination.  

2. **Glossy Reflections Not Fully Integrated** – The current implementation treats low‑roughness surfaces with a single cached radiance lookup, which yields overly blurred results. The authors propose ray‑traced reflection rays for such materials, but this path is still under development.  

3. **Bias from Light‑Grid Clamping** – The light‑grid importance metric clamps light intensities to limit variance, introducing a bias that can affect color fidelity, especially for high‑dynamic‑range light sources.  

4. **Partial UE5 Material Support** – The Unreal Engine integration approximates material shading, preventing a fully fair quality comparison with Lumen.  

5. **Dependence on Hardware Ray‑Tracing** – While the algorithm can be adapted to other traversal methods, the reported performance gains rely heavily on DXR‑1.1 hardware acceleration; alternative software‑based traversal would be significantly slower.  

## Conclusion  

GI‑1.0 demonstrates that a carefully designed two‑level radiance caching hierarchy, combined with spatiotemporal reservoir resampling and a lightweight light‑grid sampler, can deliver high‑quality indirect diffuse illumination at interactive frame rates (≈2–3 ms on a Radeon RX 6900 XT at 1080p). The system achieves this with a modest ray count (¼ samples per pixel) by reusing cached radiance across frames and guiding secondary rays with variance‑based importance. The hybrid screen‑space / world‑space approach enables dynamic scenes with moving geometry and lights without rebuilding the cache each frame.

## Future Directions  

1. **Robust Multi‑Bounce Continuation in Hash Space** – Develop a method to continue light paths directly within the hash‑based world cache, removing the reliance on visibility in the previous frame and improving interior lighting fidelity.  

2. **Unbiased Light‑Grid Sampling** – Replace the clamped intensity metric with a non‑biased estimator (e.g., stochastic light culling) to eliminate bias while preserving variance reduction.  

3. **Full Glossy Reflection Pipeline** – Integrate importance‑sampled GGX reflection rays that query both the environment map and world cache, enabling accurate specular highlights for low‑roughness materials without blurring.  

4. **Hybrid Traversal Schemes** – Explore adaptive combinations of BVH ray tracing and distance‑field tracing to further reduce traversal cost when the scene geometry permits.  

5. **Direct Lighting for Primary Rays** – Leverage the existing light‑grid structure to sample direct illumination for primary path vertices, extending the method to handle thousands of dynamic lights without additional shadow‑map cascades.  

6. **Extended Engine Integration** – Complete the Unreal Engine 5 plugin to support the full material system, allowing comprehensive visual and performance benchmarking against production‑grade GI solutions such as Lumen.  

Overall, GI‑1.0 provides a compelling blueprint for scalable, real‑time global illumination that balances ray‑tracing fidelity with cache‑driven efficiency, and its modular design opens multiple avenues for further research and production adoption.

Summarized using gpt-oss:120b.#paper #gpt-oss:120b
