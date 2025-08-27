---
layout: default
title: Rendering Vector Displacement Mapped Surfaces in a GPU Ray Tracer
permalink: /publications/
hidden: true
---

**“Rendering Vector Displacement Mapped Surfaces in a GPU Ray Tracer”** by **Takahiro Harada**, featured in *GPU Pro 6*:

---

## **Overview**

The paper presents a method for **direct ray tracing of vector displacement mapped (VDM) surfaces** on the GPU, specifically using **OpenCL** and tested on **AMD FirePro W9100 GPU**. The technique avoids pre-tessellation and instead builds geometry on-the-fly, enabling **high-detail rendering** with **low memory usage** and **competitive performance**.

---

## **Key Concepts**

### **1. Vector Displacement Mapping (VDM)**
- **Scalar Displacement Mapping** displaces vertices along the surface normal using scalar values.
- **Vector Displacement Mapping** uses vector values to displace vertices in arbitrary directions, allowing complex geometry like overhangs.
- VDM provides **greater modeling freedom** but poses **challenges for ray tracing**, especially on GPUs.

---

## **2. Ray Tracing with VDM**

### **Challenges**
- VDM lacks directional constraints, making intersection tests expensive.
- Requires **on-the-fly tessellation** and **displacement**, followed by **BVH construction** for acceleration.

### **Solution**
- Use **Bounding Volume Hierarchies (BVHs)** built dynamically per VD patch.
- **Batch rays** intersecting a VD patch to amortize tessellation and BVH build costs.

---

## **3. Quad BVH Construction**

### **Structure**
- Each VD patch is recursively subdivided into quads.
- Leaf nodes store:
  - Quantized AABBs (2-byte integers)
  - Texture coordinates and normals (4 bytes each)
- Nodes are stored in **breadth-first order**.
- **Stackless traversal** is used to reduce memory traffic and register pressure.

### **Compression**
- AABBs are quantized to reduce memory footprint.
- Texture coordinates and normals are also compressed.

---

## **4. GPU Implementation with OpenCL**

### **Parallelization Strategy**
- Rays intersecting VD patches are **grouped and sorted**.
- A **work group** processes one VD patch at a time:
  - Computes **LOD** (Level of Detail)
  - Builds **BVH**
  - Performs **ray intersection**
- BVH data is stored in **global memory** due to LDS size limitations.

### **Atomic Operations**
- Used to safely update hit information across work groups.
- 64-bit atomic min operations encode hit distance and other data.

---

## **5. Integration into Ray Tracer**

### **Three-Level BVH Hierarchy**
1. **Top-Level**: Stores meshes and transforms.
2. **Middle-Level**: Stores primitives (triangles, quads, VD patches).
3. **Bottom-Level**: Built on-the-fly for VD patches.

### **Traversal Strategy**
- Top and middle-level BVHs are traversed first.
- VD patch intersections are deferred and processed in batches.

---

## **6. Performance and Results**

### **Memory Usage**
- Dramatically reduced compared to pre-tessellation:
  - “Party” scene: 52GB → 16MB
  - “Pumpkin” scene: 380MB → 0.12MB

### **Rendering Time**
- Direct ray tracing is **faster** in most cases despite complexity.
- **Pre-tessellation** only faster for low-detail scenes.
- **Bottom-level BVH build** is the most time-consuming part.

### **Indirect Illumination**
- Performance degrades with deeper ray bounces due to repeated BVH builds.
- **Caching BVHs** is suggested as a future optimization.

---

## **Conclusion**

The proposed method enables **efficient GPU ray tracing** of **vector displacement mapped surfaces** with:
- **Low memory footprint**
- **Competitive or superior performance**
- **Scalability** for complex scenes

The paper emphasizes that **optimizing bottom-level BVH construction and traversal** is more critical than optimizing for simple primitives.

---

