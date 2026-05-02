# Clustered Position Quantization Plan

This document describes a theoretical and practical experiment for reducing Spark
runtime memory by clustering nearby splats and quantizing their positions relative
to per-cluster bounds.

The intended reader is another LLM or engineer working in this repository.

## Current Runtime Layout

Spark's `PackedSplats` runtime representation stores one splat in one `RGBA32UI`
texel:

```text
1 splat = 1 uvec4 = 4 uint32 = 16 bytes
```

The packed layout is:

```text
word0:
  bits  0-7   R uint8
  bits  8-15  G uint8
  bits 16-23  B uint8
  bits 24-31  A uint8

word1:
  bits  0-15  center.x float16
  bits 16-31  center.y float16

word2:
  bits  0-15  center.z float16
  bits 16-23  quat oct x uint8
  bits 24-31  quat oct y uint8

word3:
  bits  0-7   scale.x uint8, log encoded
  bits  8-15  scale.y uint8, log encoded
  bits 16-23  scale.z uint8, log encoded
  bits 24-31  quat angle/z uint8
```

So the base 16 bytes are:

```text
RGBA:      4 bytes
Center:    6 bytes  (3 x float16)
Scale:     3 bytes  (3 x uint8 log scale)
Rotation:  3 bytes  (oct axis xy + angle)
```

The vertex shader fetch path is an exact integer texel fetch, not filtered
sampling:

```glsl
uint splatIndex = texelFetch(ordering, orderingCoord, 0)[gl_InstanceID & 3];
ivec3 texCoord = splatTexCoord(int(splatIndex));
uvec4 packed = texelFetch(extSplats, texCoord, 0);
unpackSplatEncoding(packed, center, scales, quaternion, rgba, ...);
```

`splatTexCoord(index)` maps a linear splat index into a 2048 x 2048 x D texture
array address:

```text
x = index % 2048
y = floor(index / 2048) % 2048
z = floor(index / (2048 * 2048))
```

The texture coordinate is only a memory address. It is not spatially related to
the splat unless the splats are preordered spatially.

## Goal

Test whether positions can be stored as local quantized coordinates inside
spatial clusters:

```text
current center: 3 x float16 = 6 bytes / splat
test center:    3 x uint8   = 3 bytes / splat
```

Each cluster stores `min.xyz` and `max.xyz` bounds. Splat positions are stored as
8-bit coordinates within that local bounding box.

This experiment intentionally starts with position-only compression. Scale,
rotation, color, and SH are left unchanged.

## Theoretical Memory

For cluster size 128:

```text
metadata = min.xyz + max.xyz
         = 6 float32
         = 24 bytes / cluster

metadata amortized = 24 / 128
                   = 0.1875 bytes / splat
```

Position cost:

```text
current position = 6.00 bytes / splat
clustered pos8   = 3.00 + 0.1875
                 = 3.1875 bytes / splat
```

Position-only gain:

```text
6.00 - 3.1875 = 2.8125 bytes / splat
```

If the final GPU layout is compacted accordingly, total base splat cost becomes:

```text
current PackedSplats = 16.00 bytes / splat
clustered pos8       = 13.1875 bytes / splat
theoretical gain     = 17.6%
```

For 500k splats:

```text
current packed = 500000 * 16      = 8,000,000 bytes = 7.63 MiB
clustered pos8 = 500000 * 13.1875 = 6,593,750 bytes = 6.29 MiB
gain           = 1,406,250 bytes  = 1.34 MiB
```

Important: if the GPU texture remains `RGBA32UI` with one `uvec4` per splat,
there is no real GPU memory win. Bits must be packed into a smaller texture
format or fewer texels per splat for the gain to materialize.

## Phase 1: Offline Round-Trip Test

Do not modify the renderer first.

Build an offline tool that:

1. Loads or receives a `PackedSplats`.
2. Unpacks splat centers/scales/rotations/colors.
3. Spatially reorders splats.
4. Builds clusters.
5. Quantizes positions to local `uint8 xyz`.
6. Dequantizes positions back to float.
7. Writes a normal `PackedSplats` using reconstructed positions.
8. Measures quantization error.
9. Renders the reconstructed splats through the existing renderer.

This answers the key question before touching shader layout:

```text
Is 8-bit local position quantization visually acceptable?
```

## Clustering Strategy

Start with Morton sorting and fixed-size chunks.

Algorithm:

1. Compute global bounds of all centers.
2. Normalize each center to `[0, 1]^3`.
3. Quantize normalized center to an integer grid, for example 10 or 16 bits per
   axis.
4. Compute Morton/Z-order code.
5. Sort splats by Morton code.
6. Split sorted splats into groups of 128.

This gives spatially local groups while keeping shader addressing simple:

```text
clusterId = splatIndex / 128
localId   = splatIndex % 128
```

Test cluster sizes:

```text
32, 64, 128, 256
```

Expected tradeoff:

```text
32:  lower error, more metadata overhead
64:  likely high quality
128: target compromise
256: better memory, higher artifact risk
```

## Quantization

For each cluster:

```text
minX = min(center.x)
minY = min(center.y)
minZ = min(center.z)
maxX = max(center.x)
maxY = max(center.y)
maxZ = max(center.z)
```

For each splat in the cluster:

```js
qx = round(255 * (x - minX) / (maxX - minX))
qy = round(255 * (y - minY) / (maxY - minY))
qz = round(255 * (z - minZ) / (maxZ - minZ))
```

Clamp each value to `[0, 255]`.

Handle zero range:

```js
if (maxX === minX) qx = 0
if (maxY === minY) qy = 0
if (maxZ === minZ) qz = 0
```

Reconstruction:

```js
x2 = minX + (qx / 255) * (maxX - minX)
y2 = minY + (qy / 255) * (maxY - minY)
z2 = minZ + (qz / 255) * (maxZ - minZ)
```

Expected worst-case absolute error per axis:

```text
axisError <= rangeAxis / (2 * 255)
```

3D worst-case position error:

```text
sqrt(ex^2 + ey^2 + ez^2)
```

## Error Metrics

For each splat:

```text
positionError = distance(originalCenter, reconstructedCenter)
scaleRef      = max(scale.x, scale.y, scale.z)
relativeError = positionError / max(scaleRef, epsilon)
```

Report:

```text
mean positionError
p50 positionError
p90 positionError
p99 positionError
max positionError

mean relativeError
p50 relativeError
p90 relativeError
p99 relativeError
max relativeError

count relativeError > 0.25
count relativeError > 0.50
count relativeError > 1.00
```

Also report cluster stats:

```text
cluster count
average bbox diagonal
p90 bbox diagonal
p99 bbox diagonal
max bbox diagonal
average splats per cluster
```

## Quality Gates

Initial rough acceptance targets:

```text
p90 relativeError < 0.25
p99 relativeError < 0.50
relativeError > 1.0 for less than 0.1% of splats
```

These thresholds are guesses. Visual testing matters.

If 8-bit quantization fails:

1. Try cluster size 64.
2. Try adaptive cluster splitting.
3. Try 10-bit positions.

## Adaptive Splitting

After the fixed-size Morton test, add optional splitting:

1. Build a 128-splat cluster.
2. Quantize/dequantize.
3. Compute p99 relative error inside the cluster.
4. If above threshold, split into two 64-splat clusters.
5. Repeat down to a minimum cluster size, for example 32.

This keeps large clusters for smooth/spatially compact regions and smaller
clusters where geometry is thin or high frequency.

## Phase 2: Compact GPU Layout

Only after the round-trip quality looks acceptable, introduce a new runtime
layout.

Potential new type:

```text
ClusteredSplats
```

Data:

```text
cluster metadata texture
payload texture
numSplats
clusterSize
```

Metadata texture could initially store:

```text
meta0: min.xyz + maybe count/base
meta1: max.xyz + maybe padding
```

Using `RGBA32F`:

```text
2 texels * 16 bytes = 32 bytes / cluster
```

Using packed half or normalized ints could reduce this later. Start with
`float32` metadata for simplicity.

Payload target for position-only compression:

```text
word0: rgba8
word1: qpos.xyz uint8 + maybe spare byte
word2: scale.xyz uint8 + quat part
word3: remaining quat or keep existing layout
```

However, if this remains 4 words, memory does not improve. A real compact target
should reduce from 4 words to 3 words:

```text
target payload = 3 x uint32 = 12 bytes / splat
metadata       = about 0.25 bytes / splat
total          = about 12.25 bytes / splat
gain           = about 23.4%
```

One possible 12-byte position-only-ish layout:

```text
word0:
  rgba8

word1:
  qpos.x uint8
  qpos.y uint8
  qpos.z uint8
  quat oct x uint8

word2:
  quat oct y uint8
  quat angle uint8
  scale.x uint8
  scale.y uint8

Problem:
  scale.z does not fit.
```

So a 12-byte layout requires a further compromise, for example:

1. encode scale with fewer bits,
2. derive one scale component,
3. reduce quaternion bits,
4. use bit-level packing across words,
5. use a 13-byte logical layout packed across multiple splats.

For a first shader prototype, accept a 13-byte logical layout packed across a
stream of `R32UI` words.

## Shader Decode Sketch

If splats are reordered by cluster, shader decode can use:

```glsl
uint clusterId = uint(splatIndex) / 128u;
uint localId = uint(splatIndex) & 127u;
```

Metadata fetch:

```glsl
ClusterMeta meta = readClusterMeta(clusterId);
```

Position decode:

```glsl
vec3 q = vec3(qx, qy, qz) / 255.0;
center = mix(meta.min, meta.max, q);
```

Then decode scale, rotation, and RGBA similarly to current `unpackSplatEncoding`.

## Recommended Next Implementation Step

Implement the offline round-trip first.

Suggested command/tool shape:

```text
npm run test:cluster-quant -- examples/assets/splats/foo.spz
```

Output:

```text
input splats: 500000
cluster size: 128
clusters: 3907

memory current: 7.63 MiB
memory pos8 theoretical: 6.29 MiB
memory gain: 1.34 MiB / 17.6%

position error:
  mean: ...
  p50: ...
  p90: ...
  p99: ...
  max: ...

relative error:
  mean: ...
  p50: ...
  p90: ...
  p99: ...
  max: ...
  >0.25: ...
  >0.50: ...
  >1.00: ...
```

Optionally write a reconstructed `.spz` or temporary `PackedSplats` asset for
visual comparison in the existing examples.

## Key Reminder

Cluster quantization is not enough by itself. The current GPU format is one
`RGBA32UI` texel per splat, so it always consumes 16 bytes per splat.

The experiment has two stages:

1. Prove that local 8-bit positions are visually acceptable.
2. Change the GPU payload layout so fewer bytes are actually fetched/stored.

