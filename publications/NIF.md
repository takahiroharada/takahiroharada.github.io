---
layout: default
title: ""
permalink: /nif/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 


# Neural Intersection Function  

## Overview  
The paper introduces **Neural Intersection Function (NIF)**, a novel neural‑based approach to accelerate ray casting in physically‑based rendering.  
Instead of traversing the bottom‑level Bounding Volume Hierarchy (BVH) for each shadow‑ray query, NIF replaces this step with two lightweight multilayer perceptrons (MLPs) that predict ray–object intersections directly from a compact, alias‑free encoding of the ray parameters.  
The method is implemented on AMD GPU hardware using the HIP programming model and can be seamlessly integrated into existing rasterisation‑or‑ray‑tracing pipelines.

## Main Contributions  

| Contribution | Description |
|--------------|-------------|
| **Two‑stage neural architecture** | NIF consists of an **outer network** that quickly discards rays that miss the object’s axis‑aligned bounding box (AABB) and an **inner network** that refines the decision for rays that intersect the AABB. The outer network processes a large number of rays, while the inner network handles the far fewer rays that actually intersect the geometry. |
| **Grid‑based feature encoding** | To avoid aliasing caused by discretisation of ray parameters, the authors encode each ray’s origin, direction, and distance into a **feature grid**. For a ray $\mathbf{r}(t)=\mathbf{o}+t\mathbf{d}$ intersecting an AABB $[ \mathbf{b}_{\min},\mathbf{b}_{\max}]$, the normalized entry/exit distances are $\tau_{\text{in}} = \frac{t_{\text{in}}-t_{\text{near}}}{t_{\text{far}}-t_{\text{near}}}$ and $\tau_{\text{out}} = \frac{t_{\text{out}}-t_{\text{near}}}{t_{\text{far}}-t_{\text{near}}}$. These scalars are then lifted into a high‑dimensional vector via a **trilinear interpolation** over a 3‑D grid: $\mathbf{g} = \sum_{i,j,k} w_{ijk} \mathbf{c}_{ijk}$, where $\mathbf{c}_{ijk}$ are learnable code vectors stored at grid vertices and $w_{ijk}$ are the interpolation weights. |
| **Shared‑object training** | A **single shared MLP** is trained for the whole scene rather than per‑object networks. The outer network learns a binary visibility function $V_{\text{out}}(\mathbf{g})\in[0,1]$ indicating whether a ray hits any object’s AABB. The inner network learns a refined visibility $V_{\text{in}}(\mathbf{g})$ that approximates the true BVH intersection result. |
| **Online, viewpoint‑dependent training** | During rendering, NIF is trained **on‑the‑fly** using only the rays that are actually cast from the current camera and light configuration. The loss is a binary cross‑entropy between the network’s prediction and the ground‑truth BVH result: $\mathcal{L}= -\big(y\log\hat{y}+(1-y)\log(1-\hat{y})\big)$. |
| **Integration with existing pipelines** | NIF can be toggled per‑object: complex meshes (≥ 100 k triangles) are processed by NIF, while simple meshes continue to use conventional BVH traversal. This hybrid approach yields up to **1.53× speed‑up** on highly detailed scenes without perceptible visual degradation (PSNR ≈ 30 dB). |

## Technical Details  

1. **Ray Parameterisation**  
   For a ray $\mathbf{r}(t)=\mathbf{o}+t\mathbf{d}$ intersecting an AABB, the entry/exit distances are computed analytically:
   
$$
t_{\text{in}} = \max_{i\in\{x,y,z\}} \frac{b_{\min,i} - o_i}{d_i},\qquad
t_{\text{out}} = \min_{i\in\{x,y,z\}} \frac{b_{\max,i} - o_i}{d_i}.
$$  

   Normalised distances $\tau_{\text{in}},\tau_{\text{out}}$ are then fed to the grid encoder.

3. **Grid Encoding**  
   The 3‑D grid has resolution $R$ (e.g., $R=64$). Each cell stores a learnable code vector $\mathbf{c}_{ijk}\in\mathbb{R}^C$. For a given $(\tau_{\text{in}},\tau_{\text{out}})$, the eight surrounding vertices are identified and the feature vector is obtained by trilinear interpolation:
   
$$
\mathbf{g} = \sum_{a,b,c\in\{0,1\}} w_{abc}\,\mathbf{c}_{i+a,j+b,k+c},
$$
   
   where the weights $w_{abc}$ are products of the fractional distances along each axis.

5. **Network Architecture**  
   Both outer and inner networks are shallow MLPs with ReLU activations:
   
$$
\mathbf{h}^{(1)} = \sigma\big(\mathbf{W}^{(1)}\mathbf{g} + \mathbf{b}^{(1)}\big),\quad
\mathbf{h}^{(2)} = \sigma\big(\mathbf{W}^{(2)}\mathbf{h}^{(1)} + \mathbf{b}^{(2)}\big),\quad
\hat{y}= \text{sigmoid}\big(\mathbf{w}^{\top}\mathbf{h}^{(2)} + b\big).
$$
   
   The outer network has $\approx 1.5$ M parameters, the inner network $\approx 0.5$ M parameters.  

7. **Training Procedure**  
   - **Data collection**: For each frame, shadow rays are generated and their true BVH intersection results $y\in\{0,1\}$ are recorded.  
   - **Optimization**: Adam optimizer (β₁=0.9, β₂=0.999) with learning rate $10^{-3}$ is used.  
   - **Online update**: After each pass, the network weights are updated, allowing the model to adapt to the current view and lighting configuration.  

8. **Inference**  
   During rendering, a ray first passes through the **outer grid** (feature extraction) and **outer inference** (binary decision). If the outer network predicts a miss, the ray is discarded. If it predicts a hit, the ray is forwarded to the **inner network** for a refined decision.  

## Limitations  

| Limitation | Impact |
|------------|--------|
| **View‑dependent training** | NIF is trained only on rays emanating from the current camera. Queries from significantly different viewpoints suffer reduced accuracy, requiring re‑training. |
| **Static‑scene assumption** | The current online training pipeline cannot handle dynamic geometry efficiently; any change in mesh topology forces a full re‑training. |
| **Model‑complexity dependence** | Speed‑up is proportional to the number of triangles in the processed object. For low‑complexity meshes (≤ 100 k triangles) the overhead of grid encoding and inference outweighs BVH traversal, yielding no net gain. |
| **Redundant ray evaluations** | The current implementation may query the same ray multiple times if multiple objects are processed sequentially, leading to unnecessary compute. |
| **Limited to binary visibility** | The presented NIF predicts only shadow‑ray visibility. Extending to encode shading normals, depth, or higher‑order light transport (e.g., indirect illumination) requires additional output heads and training data. |

## Conclusion  

The authors demonstrate that a **learned neural intersection function** can replace the costly bottom‑level BVH traversal for complex geometry, achieving up to **1.53×** faster ray casting while preserving visual fidelity (PSNR ≈ 30 dB). By sharing a single MLP across all objects and employing a high‑resolution grid encoding, NIF mitigates aliasing artifacts and provides a compact representation (≈ 2 MB per scene). The method integrates cleanly into existing GPU‑based ray‑tracing pipelines, offering a hybrid solution where only the most divergent workloads are off‑loaded to the neural predictor.

## Future Directions  

1. **View‑agnostic training** – Incorporate multi‑view ray samples (e.g., via importance sampling over the hemisphere) to build a viewpoint‑independent NIF, reducing the need for frequent re‑training.  
2. **Dynamic‑scene support** – Develop incremental update schemes (e.g., continual learning or meta‑learning) that adapt the latent grid codes as geometry deforms, avoiding full retraining.  
3. **Extended ray semantics** – Augment the inner network to output additional geometric attributes (surface normal, depth, material ID) enabling NIF to replace primary‑ray intersection as well.  
4. **Redundancy elimination** – Cache successful intersection results per‑pixel or per‑tile to avoid re‑evaluating the same ray across multiple objects.  
5. **Hardware‑aware optimisation** – Exploit upcoming GPU tensor cores and sparsity‑aware kernels to further accelerate the grid‑to‑MLP pipeline, potentially surpassing the current 1.5× speed‑up.  

