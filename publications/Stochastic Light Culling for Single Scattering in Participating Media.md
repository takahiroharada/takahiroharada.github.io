---
layout: default
title: ""
permalink: /vslc/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 

# Stochastic Light Culling for Single Scattering in Participating Media

**Overview**  
The paper *“Stochastic Light Culling for Single Scattering in Participating Media”* (Eurographics 2022) proposes a lightweight, unbiased technique for evaluating single‑scattering illumination from many point and area light sources in homogeneous participating media. By extending the stochastic light‑culling idea of Tokuyoshi & Harada [TH16] to volumetric rendering, the authors avoid building and traversing hierarchical light structures (e.g., light trees). The method works by intersecting primary rays with two spherical bounds that delimit a light’s stochastic influence range, and by coupling this culling with equi‑angular distance sampling. A tile‑based GPU implementation further accelerates per‑pixel light selection.

---

### Main Contributions  

| # | Contribution | Key Idea / Formula |
|---|--------------|--------------------|
| 1 | **Stochastic light culling for volumes** – the influence of each light is limited by two concentric spheres (inner $r_i$ and outer $r_o$). | $$r_i = r\,\frac{\max_\omega I(\omega)}{\alpha},\qquad r_o = s\,\frac{\max_\omega I(\omega)}{\alpha\,\xi} \tag{5}$$ |
| 2 | **Combination with equi‑angular sampling** – inside the annular region $[t_1,t_2]$ the distance pdf follows the classic equi‑angular form, while the outer segments $[t_0,t_1]$ and $[t_2,t_3]$ use uniform PDFs. | $$p(t)=\begin{cases} p_0(t) & t\in[t_0,t_1] \\ p_1(t) & t\in[t_1,t_2] \\ p_2(t) & t\in[t_2,t_3]\end{cases}$$ with the segment PDFs defined below. |
| 3 | **Analytic segment weights** – the importance of each ray‑segment is obtained by integrating the corresponding PDFs, yielding closed‑form weights $w_0,w_1,w_2$. | $$w_0=\int_{t_0}^{t_1}\frac{1}{r_i^2}\,dt=\frac{1}{r_i^2}(t_1-t_0) \tag{6}$$ $$w_1=\int_{t_1}^{t_2}\frac{1}{D^2+t^2}\,dt=\frac{1}{D}\bigl(\theta_{t_2}-\theta_{t_1}\bigr) \tag{7}$$ $$w_2=\int_{t_2}^{t_3}\frac{1}{r_i^2}\,dt=\frac{1}{r_i^2}(t_3-t_2) \tag{8}$$ |
| 4 | **PDFs for each segment** – normalized by the total weight $w=w_0+w_1+w_2$. | $$p_0(t)=\frac{w_0}{w\,(t_1-t_0)} \tag{9}$$ $$p_1(t)=\frac{w_1}{w\,D\,(\theta_{t_2}-\theta_{t_1})\,(D^2+t^2)} \tag{10}$$ $$p_2(t)=\frac{w_2}{w\,(t_3-t_2)} \tag{11}$$ |
| 5 | **Extension to arbitrary area lights** – virtual point lights (VPLs) are placed on the light surface; tighter spherical bounds are derived for diffuse VPLs. | $$r_i^{\text{area}}=\Bigl(\frac{4}{27}\Bigr)^{\!1/4}\!\frac{r\,\Phi}{\pi\alpha},\qquad r_o^{\text{area}}=\Bigl(\frac{4}{27}\Bigr)^{\!1/4}\!\frac{s\,\Phi}{\pi\alpha\,\xi} \tag{12}$$ $$c_i = x + \frac13\Bigl(\frac{3}{4}\Bigr) \frac{r\,\Phi}{\pi\alpha}\,n,\qquad c_o = x + \frac13\Bigl(\frac{3}{4}\Bigr) \frac{s\,\Phi}{\pi\alpha\,\xi}\,n \tag{13}$$ |
| 6 | **Tile‑based light culling on the GPU** – lights intersecting a screen‑space tile frustum are collected once per tile (shared memory) and then evaluated for all rays inside the tile, drastically reducing per‑ray light‑list construction cost. | – (algorithmic description, no new equation) – |

The method is evaluated on scenes with thousands of point or triangle lights. Compared to a baseline that uses reservoir sampling + equi‑angular distance sampling (without culling), the proposed approach yields lower root‑mean‑square error (RMSE) for the same rendering time (e.g., 416 vs 380 samples/pixel in a 1‑second budget).

---

### Limitations  

1. **Homogeneous media & isotropic phase function** – Experiments are limited to constant‑density volumes with an isotropic scattering phase. Heterogeneous media or highly anisotropic phase functions may increase variance because the culling step does not account for transmittance or phase weighting.  
2. **Neglect of directional intensity for area lights** – The derivation for area lights ignores the cosine term $n\!\cdot\!\omega$ in the VPL intensity, which can introduce extra variance, especially for grazing illumination.  
3. **Benefit scales with light count** – For scenes with only a few lights (e.g., four triangle lights) the culling overhead outweighs the variance reduction; the method shines when the number of lights is large.  
4. **Single‑scattering only** – Indirect illumination (multiple scattering or VPL‑based indirect lighting) is not addressed; extending the technique to those cases would require additional handling of phase‑dependent bounds.  

---

### Conclusion  

The authors present a **simple, unbiased** algorithm for single‑scattering volumetric illumination that eliminates the need for expensive light‑tree construction. By intersecting rays with stochastic spherical influence bounds and blending equi‑angular distance sampling, the method achieves **higher image quality** (lower RMSE) at comparable performance to reservoir‑sampling baselines. The tile‑based GPU implementation demonstrates that the approach is practical for real‑time‑ish rendering of scenes with thousands of lights.

---

### Future Directions  

| Proposed Extension | Rationale & Expected Benefit |
|--------------------|------------------------------|
| **Multiple‑importance sampling (MIS) with transmittance‑based distance sampling** – combine the current uniform‑importance scheme with a transmittance‑biased pdf to better handle heterogeneous media. | Reduces variance where attenuation varies strongly along the ray. |
| **Phase‑function‑aware bounds** – derive spherical/ellipsoidal bounds that incorporate anisotropic phase functions (e.g., GGX, SGGX). | Improves culling accuracy for media with strong forward/backward scattering, lowering noise. |
| **Indirect illumination via VPLs in volumes** – apply stochastic culling to VPLs generated by bounce lighting, using phase‑function‑dependent intensity. | Enables full global illumination in participating media without hierarchical structures. |
| **Bounding ellipsoids for non‑diffuse VPLs** – replace the spherical bound with an ellipsoid aligned to the VPL’s directional distribution (e.g., GGX). | Tighter bounds → fewer lights survive culling → faster evaluation. |
| **Adaptive tile size / hierarchical screen‑space culling** – dynamically adjust tile granularity based on light density or view‑dependent metrics. | Balances culling overhead vs. per‑tile light list length, improving scalability on very high‑resolution displays. |
| **Integration with ray‑marching acceleration (e.g., sparse voxel octrees)** – exploit existing volume acceleration structures to reuse ray‑segment information for both transmittance and culling. | Potentially reduces the number of ray‑sphere intersection tests and improves cache coherence. |

These avenues aim to broaden the applicability of stochastic light culling beyond homogeneous, isotropic volumes, and to integrate it with more sophisticated sampling and acceleration techniques for production‑grade volumetric rendering.
