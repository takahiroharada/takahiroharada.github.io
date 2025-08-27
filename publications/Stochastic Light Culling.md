---
layout: default
title: Stochastic Light Culling 
permalink: /Stochastic Light Culling/
nav_exclude: true
---

## **Title**: Stochastic Light Culling  
### **Authors**: Yusuke Tokuyoshi (Square Enix), Takahiro Harada (AMD)

---

## **Overview**

The paper introduces a novel **unbiased light culling technique** for rendering scenes with many light sources, particularly **virtual point lights (VPLs)**. Traditional methods like **tiled lighting with clamped light ranges** introduce bias and darkening artifacts. The proposed method uses **stochastic sampling via Russian roulette** to determine light influence ranges, achieving **sublinear shading cost** and **unbiased results**.

---

## **Key Contributions**

1. **Stochastic Fall-off Function**:
   - Replaces deterministic clamping with a probabilistic approach.
   - Uses Russian roulette to decide whether a light contributes to a shading point.
   - Ensures unbiased estimation of radiance.

2. **Sublinear Shading Cost**:
   - Light evaluation cost per shading point becomes sublinear with respect to the total number of lights.
   - Controlled via a user-specified error bound.

3. **Integration with Tiled Lighting**:
   - Demonstrates how stochastic light culling can be integrated into tiled lighting frameworks (e.g., Forward+, clustered shading).
   - Achieves real-time performance with tens of thousands of VPLs.

4. **Point Cloud Culling for Imperfect Shadow Maps (ISMs)**:
   - Accelerates ISM rendering by culling invisible points based on stochastic light ranges.
   - Reduces splatting cost in ISM generation.

5. **Extension to Path Tracing**:
   - Introduces a **bounding sphere tree** for light culling in progressive path tracing.
   - Supports area lights and multi-bounce global illumination.
   - Implements GPU optimizations to reduce divergence and memory usage.

---

## **Technical Details**

### 1. **Stochastic Fall-off Function**
- Traditional fall-off: $$f(l) = \frac{1}{l^2}$$ (infinite range).
- Stochastic version:
  - Probability of evaluating a light:  
    $$p_i(l) = \min\left(\frac{f(l)}{\alpha_i}, 1\right)$$
  - Light is culled if $$p_i(l) < \xi_i$$ (random number).
  - Effective range:  
    $$r_i = \frac{1}{\sqrt{\alpha_i \xi_i}}$$

### 2. **Error Bound-Based Control**
- Variance of the estimator:  
  $$\sigma_i^2(l) = \left(\frac{1}{p_i(l)} - 1\right)f^2(l)$$
- Error bound per light:  
  $$\epsilon_i(x, \hat{\omega}) = \sigma_i(l) \cdot E \cdot I_i \cdot V(x, x_i) \cdot \rho(x, \hat{\omega}, \hat{\omega}_i) \cdot \max(\hat{\omega}_i \cdot \hat{n}, 0)$$
- Solving for α:
  $$\alpha_i = \frac{2\pi \epsilon_{\text{max}}}{E \cdot \max_{\hat{\omega}'} I_i(\hat{\omega}')}$$

### 3. **Tiled Lighting Integration**
- Algorithm consists of:
  1. Light range computation (randomized).
  2. Light culling (same as traditional tiled lighting).
  3. Shading (modified fall-off function).

### 4. **Point Cloud Culling for ISMs**
- Uses visibility and stochastic range to cull points before splatting.
- Reduces ISM generation time significantly.

### 5. **Path Tracing with Bounding Sphere Tree**
- Builds a tree of lights with stochastic ranges.
- Traverses tree to find overlapping lights.
- Uses **reservoir sampling** to limit memory usage.
- Applies **resampled importance sampling** to restrict shadow rays to one per shading point.

---

## **Experimental Results**

- **Real-time rendering** of 65,536 VPLs in ~5 ms.
- **AISMs** (with shadows) for 16,384 VPLs in ~22 ms.
- **Path tracing** with 50,000+ area lights shows significant noise reduction and performance gain.
- RMSE comparisons show stochastic method outperforms clamping and uniform sampling.

---

## **Limitations & Future Work**

- **Banding artifacts** due to shared random numbers.
- **Directional importance** and high-frequency BRDFs not fully addressed.
- Future directions include:
  - Better bounding volumes.
  - Improved ISM performance.
  - Clustering shading points.
  - Integration with bidirectional path tracing.

---

