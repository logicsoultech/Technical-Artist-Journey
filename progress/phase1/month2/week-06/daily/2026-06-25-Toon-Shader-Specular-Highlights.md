# Daily Log — 2026-06-25

**Phase:** 1 | **Month:** 2 | **Week:** 06 | **Topic:** Toon Shader Specular Highlights — Blinn & Phong Methods in Shader Graph

---

## 📖 Theory: Specular Highlights in a Toon Shader Context

Specular highlights represent the **direct reflection of a light source** off a surface. In a physically-based rendering (PBR) pipeline, the engine handles this automatically. However, because this project uses an **unlit shader** — where the engine deliberately skips all built-in lighting calculations — the entire reflection logic must be reconstructed manually from first principles inside Shader Graph.

Two classical formulas cover this:

- **Blinn-Phong** — computes the highlight by approximating the reflection using a *half-angle vector* between the camera direction and the light direction.
- **Phong** — computes the highlight using the *true reflection vector* of the light around the surface normal.

Both produce a specular lobe whose size and intensity are controlled by two exposed parameters:

| Parameter          | Effect                                              |
| ------------------ | --------------------------------------------------- |
| `Spec Focus`       | Controls sharpness / size of the highlight lobe     |
| `Spec Brightness`  | Controls peak brightness / intensity of the lobe    |

---

## 🗺️ Concept Map: Blinn-Phong Logic Flow in Shader Graph

```
Camera Vector  +  Light Direction Vector
        ↓
    [Add] → Raw Half-Angle Vector
        ↓
    [Normalize] → Unit Half-Angle Vector
        ↓
    [Dot Product] with Surface Normal
        ↓
    [Saturate] → Clamp result to [0, 1]
        ↓
    [Power] ^ Spec Focus → Sharpened Lobe
        ↓
    [Multiply] × Spec Brightness → Final Specular Value
        ↓
    (Optional) [Round] → Hard-edged anime-style highlight
```

| Node / Operation   | Purpose                                                                 |
| ------------------ | ----------------------------------------------------------------------- |
| `Add`              | Combines camera and light vectors into a raw half-angle                 |
| `Normalize`        | Ensures the half-angle vector has unit length before dot product        |
| `Dot Product`      | Measures how closely the half-angle aligns with the surface normal      |
| `Saturate`         | Clamps the dot product to [0, 1], preventing negative light values      |
| `Power`            | Raises the clamped value to the `Spec Focus` exponent — narrows lobe   |
| `Round` (optional) | Snaps the gradient to 0 or 1 — produces a hard cel-shaded highlight     |

---

## 🔬 Visual Style Variants

The specular calculation produces a continuous gradient value by default. Three visual interpretations can be derived from it without changing the underlying math:

### 1. Soft Style

Leave the result as-is after the `Power` node. The highlight fades smoothly from peak to zero, producing a painterly, watercolor-adjacent look.

### 2. Hard-Edged / Anime Style

Insert a `Round` node after `Power`. This snaps every value to either `0.0` or `1.0`, producing a perfectly circular, razor-edged highlight dot — the visual signature of classic hand-drawn anime shading.

### 3. Color Tinting vs. Additive White

Where the specular value is combined with the base color determines its visual character:

| Blending Position        | Result                                                              |
| ------------------------ | ------------------------------------------------------------------- |
| **Before** base color multiply | Highlight inherits the object's paint color — matches how traditional cel-animation handles sheen |
| **After** base color multiply  | Highlight remains pure white regardless of object color — cleaner for plastic / metallic surfaces |

---

## 💡 Insights & New Understanding

### Why Reconstruct Lighting on an Unlit Shader?

The unlit shader trade-off is fundamentally about **performance vs. control**. Dynamic lighting systems in Unreal and Unity compute full GI, shadow maps, reflection captures, and light probes — all of which are GPU-expensive. An unlit shader bypasses every one of those passes.

The cost is that *all* the visual richness must be authored manually. The benefit is that the shader runs identically on a flagship PC and a mid-tier mobile device, with zero variance from scene lighting conditions. For a stylized game targeting cross-platform release, this is often the correct engineering decision.

Reconstructing the Blinn-Phong formula node-by-node transforms a visual limitation (no auto-lighting) into a **controlled artistic parameter** — the developer decides exactly how the surface responds to light, not the engine's default approximation.

### Blinn vs. Phong: The Practical Difference

While both methods produce visually similar results for most viewing angles, the critical distinction is computational:

- **Phong** requires computing `reflect(-LightDir, Normal)` — this involves a true reflection vector, which has additional instructions.
- **Blinn** replaces that with a `normalize(CameraVec + LightDir)` half-angle — fewer instructions, and the result is more numerically stable at grazing angles.

For a shader running on potentially thousands of material instances in a mobile title, choosing Blinn over Phong for toon specular is a small but meaningful optimization decision.

### The `Round` Node as an Art Direction Decision

Adding `Round` to the specular output is a single-node change that shifts the entire visual language of the shader. This is an important pattern for a Technical Artist to internalize: **art direction decisions can map to single operations in a shader graph**. The TA's role is to expose the right nodes as parameters so an Art Director can make that call in a Material Instance without ever opening the graph.

---

## 🎯 TA Skills Practiced Today

| Skill                                      | Relevance as a Technical Artist                                                                                  |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| **Specular Math (Blinn-Phong)**            | Core shading knowledge required to build custom lighting for stylized / unlit rendering pipelines                |
| **Shader Graph Node Composition**          | Translating a mathematical formula into a clean, readable graph that other artists can maintain                  |
| **Parameter Exposure Design**              | Converting magic numbers into named parameters (`Spec Focus`, `Spec Brightness`) accessible via Material Instance |
| **Art Style Decision via Shader Logic**    | Understanding how a single node (`Round`) shifts the shader from soft-shaded to anime hard-edged                 |
| **Performance-Aware Shading Architecture**| Choosing unlit + manual lighting as a deliberate cross-platform optimization strategy                            |

---

## 📝 Technical Artist Notes

**Parameters are the interface between the shader and the artist.** A specular formula with hardcoded values is a locked tool. Exposing `Spec Focus` and `Spec Brightness` as Material Instance parameters means a 3D Artist or Character TD can iterate on the highlight size and brightness without opening the shader graph at all. That is the TA's primary deliverable — not the node graph itself, but the usable system built on top of it.

**The `Round` node is a style toggle, not a hack.** Cel-shading pipelines in production (Genshin Impact, Zelda: Tears of the Kingdom, Blue Protocol) use exactly this principle — a step or round function that converts a continuous lighting gradient into a discrete binary boundary. Understanding that this is a deliberate art direction mechanism, rather than a simplification, is important for communicating its purpose to an Art Director.

**Blinn's half-angle is the safer default.** Unless the art style explicitly requires the slightly tighter Phong lobe at specific angles, defaulting to Blinn reduces instruction count and avoids the instability that Phong exhibits at grazing incidence. This is the kind of quiet technical decision that accumulates into measurable performance headroom across a full character shader set.

---

## 🔧 Tools & Environment

- **Engine:** Unreal Engine 5 / Unity (Shader Graph — both demonstrated in source material)
- **Reference:** *Toon Shader Specular Highlights — Shader Graph Basics, Episode 40* by Ben Cloward
- **Source:** [https://www.youtube.com/watch?v=B56z6st6U8E](https://www.youtube.com/watch?v=B56z6st6U8E)

