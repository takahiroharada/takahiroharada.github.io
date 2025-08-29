---
layout: default
title: ""
permalink: /forward+/
---
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax:{inlineMath:[['\$','\$'],['\\(','\\)']],processEscapes:true},CommonHTML: {matchFontHeight:false}});</script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML"></script> 


**“Forward+: Bringing Deferred Lighting to the Next Level”** by Takahiro Harada, Jay McKee, and Jason C. Yang (AMD):

---

## 🔧 Overview

**Forward+** is a hybrid rendering technique that combines the flexibility of **forward rendering** with the scalability of **deferred lighting**. It introduces a **GPU-based light culling stage** to efficiently handle scenes with **many dynamic lights**, overcoming the limitations of both traditional forward and deferred rendering.

---

## 🧱 Motivation

### Limitations of Traditional Techniques:
- **Forward Rendering**:
  - Limited number of lights due to shader permutation explosion.
  - Poor scalability with dynamic lighting.
  - Good material flexibility and transparency handling.

- **Deferred Rendering**:
  - Efficient with many lights.
  - High memory bandwidth and storage requirements (G-buffers).
  - Poor support for complex materials and transparency.
  - Limited anti-aliasing support.

### Goal:
To create a rendering pipeline that:
- Supports **many lights**.
- Maintains **material and lighting model flexibility**.
- Reduces **memory traffic** compared to deferred lighting.

---

## 🛠️ Forward+ Pipeline

Forward+ extends the forward rendering pipeline with a **light culling stage** using GPU compute shaders. The pipeline consists of:

1. **Depth Prepass**:
   - Renders scene depth to reduce overdraw.
   - Essential for minimizing cost in final shading.

2. **Light Culling**:
   - Screen is divided into **tiles** (e.g., 16×16 pixels).
   - For each tile, a list of **overlapping lights** is computed.
   - Implemented entirely on the GPU using compute shaders.

3. **Final Shading**:
   - Each pixel accesses the list of lights for its tile.
   - Full material and light data are available for accurate shading.

---

## 💡 Light Culling Techniques

Two GPU-based implementations are described:

### 1. **Gather Approach**:
- One compute shader per tile.
- Each thread checks if a light overlaps the tile’s frustum.
- Overlapping lights are stored in thread-local storage and then written to global memory.
- Simple and efficient for a moderate number of lights.

### 2. **Scatter Approach**:
- One thread per light.
- Each light determines which tiles it overlaps.
- Writes light-tile pairs to a buffer.
- Buffer is sorted by tile index (e.g., radix sort).
- More scalable for large numbers of lights.

---

## 📊 Performance & Memory Analysis

### Theoretical Memory Traffic:
- Forward+ avoids writing full-screen G-buffers.
- Writes only light indices per tile.
- Reads light indices and light data during final shading.
- Deferred lighting reads/writes more data (normals, depth, light accumulation).

### Key Formula:
To determine when Forward+ is more efficient:

$$
M < 15 \cdot \frac{1 + (1 + L)T}{T}
$$

Where:
- \( M \): average lights per tile
- \( L \): light data size (e.g., 8 floats for point lights)
- \( T \): tile size (e.g., 16×16)

### Experimental Results:
- Tested on AMD Radeon HD 6970 and A8-3510MX (integrated GPU).
- Forward+ outperformed compute-based deferred lighting in total rendering time.
- Performance gap widened under lower memory bandwidth conditions.
- Well-suited for tile-based rendering architectures (e.g., mobile GPUs).

---

## 🎯 Advantages of Forward+

- **Material Flexibility**: Supports complex BRDFs and physically-based shading.
- **Lower Memory Bandwidth**: Especially beneficial on bandwidth-constrained GPUs.
- **Scalability**: Efficiently handles thousands of dynamic lights.
- **GPU-Only Pipeline**: No CPU-side light management needed.

---

## 🔮 Future Work

- **Dynamic Tile Sizing**: Automatically optimize tile size based on scene complexity.
- **Shadowing**: Integrate per-light shadow computation (e.g., via ray casting).
- **Further Optimization**: Explore spatial subdivision for better light culling accuracy.

---

