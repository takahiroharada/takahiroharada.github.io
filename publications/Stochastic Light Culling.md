---
layout: default
title: ""
permalink: /Stochastic Light Culling/
---
## **Title**: Stochastic Light Culling  
### **Authors**: Yusuke Tokuyoshi (Square Enix), Takahiro Harada (AMD)
---
## **Overview**
The paper introduces a novel **unbiased light culling technique** for rendering scenes with many light sources, particularly **virtual point lights (VPLs)**. Traditional methods like **tiled lighting with clamped light ranges** introduce bias and darkening artifacts. The proposed method uses **stochastic sampling via Russian roulette** to determine light influence ranges, achieving **sublinear shading cost** and **unbiased results**.
---
## **Key Contributions**
1. **Stochastic Fall‑off Function**  
   - Replaces deterministic clamping with a probabilistic approach.  
   - Uses Russian roulette to decide whether a light contributes to a shading point.  
   - Ensures unbiased estimation of radiance.  

2. **Sublinear Shading Cost**  
   - Light evaluation cost per shading point becomes sublinear with respect to the total number of lights.  
   - Controlled via a user‑specified error bound.  

3. **Integration with Tiled Lighting**  
   - Demonstrates how stochastic light culling can be integrated into tiled lighting frameworks (e.g., Forward+, clustered shading).  
   - Achieves real‑time performance with tens of thousands of VPLs.  

4. **Point Cloud Culling for Imperfect Shadow Maps (ISMs)**  
   - Accelerates ISM rendering by culling invisible points based on stochastic light ranges.  
   - Reduces splatting cost in ISM generation.  

5. **Extension to Path Tracing**  
   - Introduces a **bounding sphere tree** for light culling in progressive path tracing.  
   - Supports area lights and multi‑bounce global illumination.  
   - Implements GPU optimizations to reduce divergence and memory usage.  

---
## **Technical Details**
### 1. **Stochastic Fall‑off Function**
- Traditional fall‑off (infinite range):  

  \[
  f(l) = \frac{1}{l^{2}}
  \]

- Stochastic version:  

  - Probability of evaluating a light  

    \[
    p_{i}(l)=\min\!\left(\frac{f(l)}{\alpha_{i}},\,1\right)
    \]

  - A light is culled if  

    \( p_{i}(l) < \xi_{i} \)  

    (where \(\xi_{i}\) is a uniform random number in \([0,1]\)).  

  - Effective range  

    \[
    r_{i}= \frac{1}{\sqrt{\alpha_{i}\,\xi_{i}}}
    \]

### 2. **Error‑Bound‑Based Control**
- Variance of the estimator  

  \[
  \sigma_{i}^{2}(l)=\Bigl(\frac{1}{p_{i}(l)}-1\Bigr)\,f^{2}(l)
  \]

- Error bound per light  

  \[
  \epsilon_{i}\bigl(x,\hat{\omega}\bigr)=
  \sigma_{i}(l)\;E\;I_{i}\;V\!\bigl(x,x_{i}\bigr)\;
  \rho\!\bigl(x,\hat{\omega},\hat{\omega}_{i}\bigr)\;
  \max\!\bigl(\hat{\omega}_{i}\!\cdot\!\hat{n},\,0\bigr)
  \]

- Solving for \(\alpha\) (the scaling factor that enforces the bound)  

  \[
  \alpha_{i}= \frac{2\pi\,\epsilon_{\text{max}}}
                   {E\;\max_{\hat{\omega}'} I_{i}(\hat{\omega}')}
  \]

### 3. **Tiled Lighting Integration**
The algorithm proceeds in three stages:

1. **Light‑range computation** (stochastic, per‑light).  
2. **Light culling** – identical to traditional tiled lighting (lights are placed into per‑tile lists).  
3. **Shading** – uses the stochastic fall‑off function instead of a hard‑clamp.

### 4. **Point‑Cloud Culling for ISMs**
- Visibility and stochastic range are combined to discard points that would never contribute to the final image before the splatting pass.  
- This reduces the amount of work required to generate an Imperfect Shadow Map dramatically.

### 5. **Path Tracing with a Bounding‑Sphere Tree**
- Lights are organized into a hierarchy of bounding spheres, each annotated with a stochastic range.  
- During traversal, only spheres intersecting the current shading point’s influence region are visited.  
- **Reservoir sampling** limits the number of lights stored per node, while **resampled importance sampling** restricts shadow‑ray generation to a single representative direction per shading point.

---
## **Experimental Results**
- **Real‑time rendering** of 65 536 VPLs in **5 ms**.  
- **AISMs** (with shadows) for 16 384 VPLs in **22 ms**.  
- **Path tracing** with > 50 000 area lights shows significant noise reduction and performance gain.  
- RMSE comparisons demonstrate that the stochastic method outperforms clamping and uniform sampling.

---
## **Limitations & Future Work**
- **Banding artifacts** caused by shared random numbers across pixels.  
- **Directional importance** and high‑frequency BRDFs are not fully addressed.  
- **Future directions**:  
  - Better bounding‑volume hierarchies.  
  - Faster ISM pipelines.  
  - Clustering shading points to share random seeds.  
  - Integration with bidirectional path tracing.

---
