# TwinVQ Map Decoder — Deep Technical Analysis

Reverse engineering documentation of the TwinVQ-based tilemap reconstruction system used in the J2ME version of Rayman.

This document describes the internal structure, decoding strategy, memory layout and reconstruction logic implemented in the `decodeTwinVQ()` function of the Map Viewer.

---

# Overview

The so-called **TwinVQ map format** used in the `04d` archive is not related to the audio codec with the same name.  
In this context, it refers to a **chunk-based vector quantized tilemap compression scheme**.

The format is optimized for:

- Low memory usage
- High repetition of terrain patterns
- Fast runtime lookup
- Separation of visual and collision layers

Instead of storing a full tilemap linearly, the engine stores:

1. A set of reusable tile chunks
2. A chunk index grid
3. Optional secondary data (collision layer)

---

# Resource Structure (Per Map Entry)

After extracting offsets using `parseArch()`, each map resource begins with:

| Offset | Size | Description |
|--------|------|-------------|
| 0      | 1    | Chunk width (`cw`) |
| 1      | 1    | Chunk height (`ch`) |
| 2      | 1    | Unknown / reserved |
| 3      | 1    | Tiles per chunk (`total`) |
| 4      | 1    | Map width in chunks (`mw`) |
| 5      | 1    | Map height in chunks (`mh`) |
| 6      | 1    | Doublets size (`doublets`) |

---

# Derived Map Dimensions

The real tilemap dimensions are calculated as:

```
W = mw * cw
H = mh * ch
```

This gives the full reconstructed map resolution in tiles.

---

# Memory Layout

Immediately after the 7-byte header:

```
[ chunk data blocks ]
[ optional doublet data blocks ]
[ chunk index grid ]
```

---

# Chunk Data Blocks

Each chunk contains:

```
total = cw * ch
```

visual tile indices.

If `doublets > 0`, each chunk also contains additional per-tile data used for collision.

Chunks are tightly packed in memory.

---

# Chunk Index Grid

This is a grid of size:

```
mw × mh
```

Each cell stores a `chunkID`.

This grid defines which chunk should be used for each region of the map.

---

# Reconstruction Algorithm

Implemented in:

```js
function decodeTwinVQ(raw, off, sz)
```

The decoder reconstructs two matrices:

```
vis[y][x] → visual tile ID
col[y][x] → collision tile ID
```

---

# Detailed Decoding Steps

For each tile coordinate `(x, y)`:

---

## Step 1 — Identify Chunk Coordinates

```
chunkX = floor(x / cw)
chunkY = floor(y / ch)
```

---

## Step 2 — Read Chunk Index

```
chunkIndexOffset = off + 7 + (doublets << 1) + chunkX + chunkY * mw
chunkID = raw[chunkIndexOffset]
```

This determines which chunk contains the tile.

---

## Step 3 — Compute Local Tile Position Inside Chunk

```
localX = x % cw
localY = y % ch
```

---

## Step 4 — Compute Chunk Base Address

```
chunkBase = off + 7 + chunkID * total
```

---

## Step 5 — Read Visual Tile

```
tileIndex = chunkBase + localX + localY * cw
visualTile = raw[tileIndex]
```

---

## Step 6 — Read Collision Tile (if present)

Collision data follows visual data inside the chunk:

```
collisionBase = chunkBase + doublets
collisionTile = raw[collisionBase]
```

If the collision offset exceeds bounds, it defaults to zero.

---

# Why This Compression Model?

Instead of storing:

```
W * H bytes
```

The format stores:

```
(mw * mh) bytes           ← chunk index grid
+ (uniqueChunks * total)  ← chunk definitions
```

If chunks repeat frequently, memory usage is significantly reduced.

This is ideal for:

- Repeating terrain blocks
- Platform patterns
- Low-memory environments (CLDC / MIDP devices)

---

# Computational Complexity

## Time Complexity

```
O(W * H)
```

Each tile reconstruction is constant time.

---

## Memory Efficiency

Worst case (all chunks unique):

Memory ≈ full tilemap size

Best case (heavy chunk reuse):

Memory << full tilemap size

---

# Doublets and Collision Layer

The `doublets` parameter allows the format to store:

- Collision flags
- Physics types
- Special surface behavior
- Slopes and triggers

This separation allows:

- Rendering system → reads `vis[][]`
- Physics system → reads `col[][]`
- Shared spatial indexing

No duplication of map layout is required.

---

# Architectural Advantages

✔ Efficient memory usage  
✔ Fast tile lookup  
✔ Clean separation of concerns  
✔ Suitable for low-power devices  
✔ Reusable chunk design  

---

# Conceptual Model

The format behaves similarly to:

- A macro-tile system
- Block-based compression
- A tile atlas with chunk reuse

It is effectively a **2D vector quantization system for tilemaps**.

---

# Example

Given:

```
cw = 4
ch = 4
mw = 10
mh = 8
```

Final map size:

```
W = 40
H = 32
```

If only 6 unique chunks exist, the memory footprint becomes:

```
Chunk data = 6 * (4 * 4) = 96 bytes
Chunk index grid = 10 * 8 = 80 bytes
Total ≈ 176 bytes
```

Instead of:

```
40 * 32 = 1280 bytes
```

Significant reduction.

---

# Conclusion

The TwinVQ map system is a compact, chunk-based tilemap compression algorithm designed for constrained mobile hardware.

It reconstructs:

- A visual layer
- A collision layer
- Using chunk reuse
- With deterministic constant-time access

The decoding logic mirrors the original runtime behavior of the game engine, allowing accurate map reconstruction and editing outside the original J2ME environment.
