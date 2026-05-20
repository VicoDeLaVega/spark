# Gaussian Splatting Optimization Ideas

A collection of research directions for improving 3DGS rendering performance, memory efficiency, and visual quality. Each idea is analyzed with pros, cons, and related work.

---

## 1. Precomputed Sort Order via Indirection Texture

### Idea

Instead of sorting Gaussians every frame (a major CPU/GPU bottleneck), precompute the render order for a discrete set of camera angles and store each ordering in a 1024×1024 indirection texture. Each texel stores a GS index — texel 0 = GS ID 123, texel 1 = GS ID 234, etc. At runtime, the closest precomputed angle is selected and the corresponding texture is used to drive the render order.

For N precomputed angles (e.g. 14), you store N such textures, covering the full sphere of camera directions.

### Pros
- Eliminates per-frame sort entirely — massive win on mobile/embedded
- Indirection textures are GPU-friendly and cache-coherent
- Predictable, fixed memory footprint
- Valid use case: pure rotational viewer (360 video, VR skybox) where parallax is zero and the approach is exact

### Cons
- Sort order depends on camera **position**, not just orientation — parallax breaks the approach for any translating camera
- 14 bins covers ~26 steradians each — flickering will be severe, not subtle
- Gather access pattern on 1M entries is cache-unfriendly on GPU
- N textures × 4MB each = 56MB for 14 angles, scales poorly

### Related Work
- [Compact3D](https://arxiv.org/abs/2311.13681) — GS compression and sorting optimizations
- [EAGLES](https://arxiv.org/abs/2312.04564) — efficient acceleration of GS
- Precomputed Radiance Transfer (PRT) — general concept of precomputing view-dependent data

---

## 2. Full-Resolution GS as Synthetic Oracle for Order Optimization

### Idea

The flickering problem from Idea 1 can be mitigated by treating the precomputed sort orders themselves as optimizable quantities. A full-resolution, high-quality GS (the "oracle") renders ground-truth images from densely sampled camera angles. These supervise a minimax optimization: find the sort order per bin that minimizes `max(visual_error)` across the entire angular range of the bin, not just at its center.

Additionally, the physical layout of GS data in memory can be co-optimized to improve cache coherency.

### Pros
- Reduces flickering at bin boundaries without increasing bin count
- Oracle provides unlimited synthetic supervision
- Storage layout optimization is a one-time offline cost

### Cons
- Minimax over sort orderings = optimization over N! permutations — intractable for large N
- No gradient through a sort operation — soft-sort approaches converge to local minima
- Pushing GS toward sort-robustness degrades peak reconstruction quality
- Cache coherency via Morton curves is a 40-year-old textbook technique, no oracle needed

### Related Work
- Knowledge distillation literature (teacher-student framework)
- [4D Gaussian Splatting](https://arxiv.org/abs/2310.08528) — time-varying GS
- [Gaussian Opacity Fields](https://arxiv.org/abs/2404.10772) — view-dependent GS properties

---

## 3. View-Dependent Covariance

### Idea

In standard 3DGS, color is view-dependent (via Spherical Harmonics) but the 3D covariance is fixed. The proposal: also parameterize covariance as view-dependent using SH-like basis functions. A single large Gaussian could morph its shape — tight for specular angles, wide for diffuse — replacing several smaller Gaussians.

Parameterization: work in scale + quaternion space to preserve positive semi-definiteness. Apply SH to 3 log-scale values and 4 quaternion components (renormalized after SH evaluation).

### Pros
- Fewer total Gaussians needed in theory
- Particularly powerful for specular/reflective surfaces
- Orthogonal to color SH — both can coexist

### Cons
- Breaks the mathematical foundation: a view-dependent covariance is no longer a valid 3D probability distribution — the alpha-compositing math is no longer principled
- The 2D projected covariance is already view-dependent via the projection Jacobian — adding view-dependence on top is redundant and non-principled
- Degree-3 SH on 7 parameters = 112 floats vs. 7 — triples per-GS footprint
- SH on quaternions is numerically unstable (quaternions live on S³, not ℝ⁴ — linear combinations are not valid rotations)
- Net GS count reduction is speculative, not demonstrated

### Related Work
- [3D Gaussian Splatting](https://arxiv.org/abs/2308.04079) — baseline
- [Deformable 3D Gaussians](https://arxiv.org/abs/2309.13101) — time-dependent covariance
- [4D Gaussian Splatting](https://arxiv.org/abs/2310.08528) — view/time-varying Gaussians

---

## 4. Coplanar Gaussian Detection → Textured Large Splat

### Idea

A second-pass analysis identifies clusters of coplanar Gaussians (walls, floors, roads). These clusters are replaced by a single large flat Gaussian with a baked texture encoding spatial color variation — something SH cannot express within a single primitive.

**Detection pipeline:** extract per-GS normals from covariance eigendecomposition, cluster by (normal direction, plane distance) with spatial proximity filter.

**Texture baking:** orthographically project all small GS onto the plane, rasterize their SH color into a texture atlas. The replacement GS carries UV coordinates and the baked texture; SH on the large GS still encodes global specular variation.

### Pros
- Massive primitive count reduction on architectural/indoor scenes
- Hardware texture compression (BC7) reduces memory further
- Composable with Idea 5: the textured flat splat doubles as a depth blocker

### Cons
- GS on flat surfaces aren't only there for color — they also capture micro-occlusions, contact shadows, residual non-planarity. Replacing them loses all this
- Baking at "mean view direction" gives wrong colors from other angles — baking multiple views requires a full PBR representation
- Boundary artifacts: the large splat Z-fights with adjacent GS at its perimeter; feathering doesn't fix order-dependent compositing errors
- GS on flat regions are often not thin — many have isotropic covariance, making plane detection noisy
- This is reinventing mesh+texture for flat surfaces — meshes would do it more rigorously (see Idea 13)

### Related Work
- [Scaffold-GS](https://arxiv.org/abs/2312.00109) — anchors GS to voxel scaffolding
- [2D Gaussian Splatting](https://arxiv.org/abs/2403.17888) — flat GS as surface elements (validates the planar primitive concept)
- [SuGaR](https://arxiv.org/abs/2311.12775) — surface-aligned GS for mesh reconstruction

---

## 5. Depth-Only Opaque Splats (Z-Prepass for Gaussian Splatting)

### Idea

Inject a set of large, fully opaque "depth blocker" splats derived from coplanar opaque regions. These are rendered first in a depth-only pass, populating the depth buffer. Subsequent GS behind the blockers are discarded by early-Z hardware.

### Pros
- Reduces overdraw for scenes with high depth complexity
- Depth-only pass has zero color/SH cost
- Composable with Idea 4: same primitive serves as textured surface and depth blocker

### Cons
- The entire premise of GS rendering is that everything is semi-transparent — alpha compositing is order-dependent and assumes no hard depth cutoff. Depth blockers assume cumulative alpha = 1.0, which is never exactly true
- Which depth to write for a volumetric GS? Center, front face, back face — each choice over-culls or under-culls differently
- On desktop GPUs (Ampere, Ada), the bottleneck is the sort step, not pixel rate — overdraw reduction doesn't address the real bottleneck
- On mobile (where overdraw matters), scenes are already so aggressively pruned that the behind-blocker GS count is low

### Related Work
- Z-prepass — ubiquitous in real-time rendering (Unreal, Unity)
- [GSDF](https://arxiv.org/abs/2403.10981) — combining GS with signed distance fields

---

## 6. Tile-Based OIT for Gaussian Splatting

### Idea

Treat GS rendering as an Order-Independent Transparency (OIT) problem. Instead of a global sort, use a k-buffer per tile: each pixel collects its top-k GS contributions in a MRT (Multiple Render Targets) pass, then a resolve pass sorts and composites them correctly.

The depth per fragment can be packed efficiently using a bit trick: encode depth in the upper 30 bits of a uint32 and the layer index in the lower 2 bits. This makes (depth, layer) jointly sortable as a single integer comparison:

```glsl
uint key = (floatBitsToUint(depth) >> 2) | (layer_index & 0x3);
```

The resolve pass reads all k MRT targets, sorts the k samples per pixel, and composites back-to-front. This is the approach used by Snap's Lens Studio OIT (4 layers, 8 layers, 4+1 front modes).

### Pros
- Eliminates the global O(N log N) sort entirely
- Handles GS interpenetration correctly — per-pixel sort is exact where global sort is approximate
- Scales as O(pixels × k) rather than O(N log N) in GS count
- k-buffer with k=8-16 sufficient for many scenes when combined with a depth prepass

### Cons
- k-buffer only captures k layers correctly — dense GS scenes with 50-100 GS per pixel need large k or produce artifacts beyond k
- MRT bandwidth: k render targets × full resolution = significant memory bandwidth
- The insertion sort in the fragment shader during collection is serial per pixel — limits parallelism
- Must be combined with depth blockers (Idea 5) to reduce per-pixel GS count to tractable k

### Related Work
- Intel Adaptive Transparency (Salvi et al., I3D 2011) — k-buffer with fixed layers
- Multi-Layer Alpha Blending (DirectX 11.3) — fixed per-pixel transparency layers
- [Snap Lens Studio OIT](https://developers.snap.com/lens-studio/features/graphics/advanced/order-independent-transparency) — production 4/8 layer OIT implementation
- k-buffer / A-buffer literature (Bavoil et al.)

---

## 7. Wave Compaction for Sparse Tile Buffers

### Idea

In a tile-based GS renderer, each tile (16×16 pixels) allocates a fixed number of depth slots per pixel. With typical scene density (20-50 GS per pixel), 80-90% of slots are empty. HLSL6 wave intrinsics can compact these sparse buffers in O(1) before sorting:

```hlsl
bool is_occupied = (depth[lane_id] < INFINITY);
uint4 ballot = WaveActiveBallot(is_occupied);
uint compact_idx = WavePrefixCountBits(is_occupied);
if (is_occupied) compacted[compact_idx] = fragment[lane_id];
uint valid_count = WaveActiveCountBits(is_occupied);
```

This reduces the sort input from 256 to ~20-50 elements, dramatically cutting sort cost.

### Pros
- Wave compaction is O(1) — pure hardware ballot operations
- Reduces radix sort input size by 5-10x — sort cost scales super-linearly with N
- No memory allocation — operates in-place in shared memory

### Cons
- A tile 16×16 = 256 pixels. A wave = 32 lanes (NVIDIA) or 64 lanes (AMD). Compaction is intra-wave, not intra-tile — you must compact 8 waves separately and then merge the results
- 256 depth slots per pixel on a 16×16 tile = 65,536 entries — does not fit in shared memory (96-164KB per SM). Buffer must be reduced to 64 slots max, undermining the "almost sorted" assumption
- "Almost sorted" in a tile is screen-space intuition that doesn't hold: two pixels in the same tile can have GS from completely different depth ranges

### Related Work
- HLSL6 Wave Intrinsics documentation (Microsoft)
- GPU stream compaction literature (Harris et al., CUB library)

---

## 8. 3D Voxel Binning for Per-Cell Sorting

### Idea

Extend 2D tile rendering to 3D world space: bin Gaussians into voxel cells, sort within each cell, and render cell by cell in compute. Within a small voxel, GS are spatially local so their depth order is more stable across camera positions, potentially reducing flickering. All binning, sorting, and rendering are done in compute shaders without the traditional rasterizer.

### Pros
- Per-cell sort is cheaper than global sort when cells contain few GS
- Depth order within a voxel is more stable across camera positions than globally
- Natural integration with wave compaction (Idea 7)

### Cons
- **Depth order is NOT stable within a voxel**: two GS in the same voxel swap depth order whenever the camera moves beyond a very narrow cone. The voxel size that makes this claim true is so small each voxel contains 1-2 GS — defeating the purpose
- **Multi-voxel overlap is catastrophic**: a flat GS on a 10m wall overlaps 100 voxels; an atmospheric GS overlaps thousands. Binning cost explodes exactly where GS are largest
- **Memory access is incoherent**: adjacent threads traverse different voxels with variable-length lists at arbitrary addresses — maximum cache miss rate
- **Full compute rasterization = reinventing NeRF**: abandoning hardware triangle rasterization gives up the core speed advantage of GS over NeRF (10-100x)
- You haven't eliminated the sort — you've made it local and added binning overhead on top

### Related Work
- Clustered Shading (Olsson et al. 2012) — same 3D binning concept for lights
- [Scaffold-GS](https://arxiv.org/abs/2312.00109) — voxel grid for GS anchoring (storage, not rendering)

---

## 9. Hierarchical GS LOD (Cascaded Representation)

### Idea

Build a hierarchical octree of GS where each internal node is a single GS that approximates its children — position, covariance, and color merged by opacity-weighted average. At render time, switch from leaf GS to parent GS based on projected screen-size threshold. Distant regions use coarse parent GS; nearby regions use fine leaf GS. This mirrors the clipmap pattern (fine detail near camera, coarse far away) but within the GS representation itself.

```
Parent GS merge:
  position   = Σ(opacity_i × pos_i) / Σ(opacity_i)
  covariance = Σ(opacity_i × (cov_i + outer(d_i, d_i))) / Σ(opacity_i)
  opacity    = 1 - Π(1 - opacity_i)   // correct alpha compositing
  SH         = Σ(opacity_i × sh_i) / Σ(opacity_i)
```

The LOD switch criterion: when projected screen-size of a GS subtends less than 1 pixel, switch to its parent node.

**Original SH Clipmap variant:** Store separate SH volumes per cascade (fine/medium/coarse) with different cell sizes. Distant GS are baked into the coarse SH cascade offline.

### Pros
- Same GS representation throughout — no separate data structure or rendering path
- Correct alpha compositing at every LOD level
- Streaming-friendly: load leaf GS only within radius R, parent GS beyond
- Clipmap-style ring update: only refresh cells entering/exiting the fine cascade

### Cons (SH cascade variant specifically)
- SH is the wrong representation for transparency: it captures directional radiance, not depth-sorted alpha compositing. You need (opacity, SH) per voxel — which is a NeRF
- Memory: SH L2 per cell × 128³ grid × 3 cascades ≈ 675MB before rendering anything
- Dynamic scenes invalidate baked SH cells on every GS movement → expensive rebaking
- Transition between GS rendering and SH sampling creates a discontinuity in the compositing model

### Related Work
- [LightGaussian](https://arxiv.org/abs/2311.17245) — pruning by contribution score (opacity × projected area) achieves 10x compression simply
- Light Propagation Volumes (Kaplanyan & Dachsbacher, 2010) — cascaded SH volumes for GI
- DDGI (Dynamic Diffuse Global Illumination) — cascaded irradiance probes, same cascade update pattern
- "A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets" (2024)

---

## 10. Plane-Based Analytical Depth and BSP-Inspired Sort

### Idea

Many GS are nearly planar (one eigenvalue of covariance much smaller than the others). For these, the depth of a ray-GS intersection can be computed analytically from the plane equation — no depth buffer write needed:

```glsl
// Plane equation from GS normal n and center c: d = -dot(n, c)
// Ray-plane intersection depth:
float t = -(dot(n, ray_origin) + d) / dot(n, ray_dir);
```

For sorting a small set of planar GS within a cell, a BSP tree built offline gives exact back-to-front traversal order at runtime in O(N) — no sort, no OIT. The BSP encodes spatial relationships between planes once; at runtime, traversal direction is determined by `sign(dot(n, cam_pos) + d)` per node.

This is the 3D generalization of Carmack's BSP rendering in Doom/Quake: sort by structure, not by distance computation.

### Pros
- Analytical depth reconstruction: 4 floats per GS vs. a full depth buffer write
- BSP traversal: O(N) exact ordering, no sort, no OIT buffer
- Naturally handles non-intersecting planar GS with zero runtime overhead
- Validated by [2D Gaussian Splatting](https://arxiv.org/abs/2403.17888) which uses ray-splat intersection for correct perspective depth

### Cons
- **BSP traversal is inherently serial**: tree traversal requires knowing which side of the plane you're on before choosing the next node — cannot be parallelized across GPU threads. This is the fundamental GPU killer: 9,999 threads waiting while 1 traverses the tree
- **BSP construction is O(N² log N) worst case**: intersecting planes require splits, potentially doubling node count per intersection. 1M GS → infeasible even offline
- **Most GS are not planar**: 3DGS optimization produces GS to minimize photometric loss, not to be flat. Many are spherical or elongated. Planarity must be enforced via regularization (as in 2DGS), which degrades quality on volumetric objects
- **Gaussian tails overflow plane boundaries**: even with correct BSP ordering of plane centers, Gaussian contributions to neighboring pixels violate the hard-plane compositing assumption
- The only valid use case is within a small cell (Idea 8) with few, non-intersecting flat GS — very narrow applicability

### Related Work
- [2D Gaussian Splatting (2DGS)](https://arxiv.org/abs/2403.17888) — uses ray-splat intersection for analytical depth (validates the depth reconstruction part); still uses global sort
- BSP tree rendering (Fuchs et al. 1980) — original BSP for polygon ordering
- Doom/Quake BSP renderer — practical application of BSP back-to-front traversal

---

## 11. 2DGS Mesh Extraction → Textured Mesh Replacement for Opaque Regions

### Idea

2D Gaussian Splatting (2DGS, Huang et al. SIGGRAPH 2024) produces geometrically accurate meshes via TSDF fusion from multi-view depth maps rendered by the 2D Gaussian primitives. The key regularization terms — **depth distortion** (concentrates splats along ray) and **normal consistency** (aligns splat normals with depth gradient) — force the 2D GS to lie tightly on surfaces, enabling noise-free mesh extraction.

The proposed pipeline:
1. Train a 2DGS representation on the scene
2. Extract a mesh via TSDF fusion (voxel size 0.004, truncation 0.02)
3. Bake the GS appearance (color, view-dependent SH) into PBR textures on the mesh (albedo, roughness, normal map)
4. Replace all opaque-region GS with the textured mesh
5. Retain GS only for semi-transparent, thin, or poorly-reconstructed regions (foliage, hair, glass)

The textured mesh is rendered via standard hardware rasterization with a Z-prepass, enabling full early-Z rejection for all GS behind opaque surfaces — realizing Idea 5 correctly and completely.

### Pros
- Hardware triangle rasterization for opaque surfaces: 10-100x faster than GS for the same visual quality
- Depth buffer populated correctly from opaque mesh → early-Z culls all GS behind it (Idea 5, implemented properly)
- Texture compression (BC7), mip-mapping, anisotropic filtering — all GPU hardware features unavailable to GS
- Mesh supports shadow casting, reflections, screen-space effects (SSAO, SSR) natively
- 2DGS mesh quality is state-of-the-art among explicit methods: outperforms 3DGS and SuGaR on DTU (Chamfer distance 0.80 vs 1.96 for 3DGS), competitive with NeuS at 100x the speed
- Dramatically reduces GS count → sort cost drops proportionally
- Model size: 2DGS at 30k iterations = 52MB vs 113MB for 3DGS (already 2x smaller before mesh conversion)

### Cons
- **View-dependent appearance baking is unsolved**: 2DGS uses SH for view-dependent color, which cannot be baked into a static albedo texture without quality loss. Specular/glossy surfaces require full PBR material estimation (separate research problem: GaussianShader, GS-IR, etc.)
- **Semi-transparent surfaces explicitly excluded**: the 2DGS paper acknowledges glass, foliage, and thin structures as failure cases — exactly the hardest GS use cases remain un-meshed
- **Boundary between mesh and GS**: where the mesh ends and GS begins, compositing artifacts appear unless carefully blended
- **Dynamic scenes**: mesh is static — any moving object cannot use the mesh path
- **Mesh tessellation**: TSDF at voxel size 0.004 generates very dense meshes. Mesh simplification (quadric decimation) is needed before runtime use, which introduces geometric error
- **The densification strategy of 2DGS favors texture-rich over geometry-rich areas** (acknowledged limitation), occasionally producing less accurate meshes in geometrically complex but texturally uniform regions

### Related Work
- [2D Gaussian Splatting](https://arxiv.org/abs/2403.17888) (Huang et al., SIGGRAPH 2024) — the foundation: ray-splat intersection, TSDF mesh extraction, depth distortion and normal consistency regularization
- [SuGaR](https://arxiv.org/abs/2311.12775) — surface-aligned 3DGS for mesh reconstruction (less accurate, slower than 2DGS)
- [GaussianShader](https://arxiv.org/abs/2311.17977) / [GS-IR](https://arxiv.org/abs/2311.16473) — inverse rendering for PBR material extraction from GS
- [MobileNeRF](https://arxiv.org/abs/2208.00277) — baking NeRF into textured meshes for mobile rendering (same hybrid philosophy)

---

## Summary Table

| # | Idea | Primary Gain | Main Cost | Best Scene |
|---|---|---|---|---|
| 1 | Precomputed Sort Order | Eliminates per-frame sort | Only valid for rotational viewer | 360 / VR |
| 2 | Oracle-supervised Order Opt. | Reduces bin-boundary flickering | Intractable optimization space | All |
| 3 | View-Dependent Covariance | Fewer GS in theory | Breaks GS math, unstable training | Specular |
| 4 | Coplanar → Textured Splat | Primitive count reduction | Baking quality, boundary artifacts | Architectural |
| 5 | Depth-Only Opaque Splats | Reduced overdraw | Not the real bottleneck on desktop | Dense indoor |
| 6 | Tile-Based OIT (k-buffer) | Eliminates global sort | MRT bandwidth, k-layer limit | All |
| 7 | Wave Compaction | Faster per-tile sort | SMEM limits, intra-wave only | Sparse scenes |
| 8 | 3D Voxel Binning | Local sort < global sort | Multi-voxel overlap explosion | — |
| 9 | Hierarchical GS LOD | Fewer primitives at distance | Wrong repr. for SH variant | Large scenes |
| 10 | Plane-Based BSP Sort | O(N) exact ordering | Serial traversal kills GPU | Flat regions |
| 11 | 2DGS Mesh Replacement | Hardware raster + early-Z | View-dep. baking, boundary seams | Opaque indoor |

**Most composable combination:** Ideas 6 + 9 + 11 — hierarchical LOD reduces primitive count, mesh replacement handles opaque surfaces with hardware raster, k-buffer OIT handles remaining semi-transparent GS without a global sort.
