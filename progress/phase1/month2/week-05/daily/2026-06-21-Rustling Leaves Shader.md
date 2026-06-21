# Daily Log — 2026-06-21

**Phase:** 1 | **Month:** 2 | **Week:** 5 | **Topic:** Rustling Leaves Shader & Vertex-Driven Wind Animation in UE4/UE5

---

## 📖 Theory: How Does a Rustling Leaves Shader Work?

Real foliage has near-infinite complexity — thousands of leaves moving independently across a tree. Simulating that at 30+ FPS is computationally wasteful, so engines instead bake dozens to hundreds of leaves onto a single flat polygon ("alpha card"). The risk: if that one polygon moves, every leaf baked onto it moves as a single rigid block.

A Rustling Leaves Shader solves this with a **Time-driven Sine wave** offsetting vertex position (World Position Offset), combined with a **low-resolution RGB mask texture** that splits the baked leaves into independent groups. By feeding a different time offset into each color channel, the red, green, and blue leaf groups sway with different timing and amplitude — breaking the "one solid object" look without adding real geometry.

Where the Volumetric Ice shader (yesterday) used coordinate offsetting to fake depth on a static surface, this shader uses coordinate offsetting to fake independent motion on what is, geometrically, one flat plane.

---

## 🗺️ Concept Map: Leaf Sway Logic Flow (Material Graph)

The wind-sway pipeline follows a four-stage flow:

```
Time → Multiply(per-channel speed) → Sine
→ Mask Texture (R/G/B channel split) → Multiply(Sine * Mask)
→ Add(WindStrength) → World Position Offset (Vertex)

→ [parallel branch]
Time → Sine → Multiply(small amplitude) → Vertex Normal → re-lit shading
```

| Stage | Math / Node | What It Controls |
|---|---|---|
| 1. Oscillation | `Sine( Time * Speed )` | Drives the back-and-forth sway cycle |
| 2. Group Isolation | `Mask Texture (R/G/B)` | Splits baked leaves into independently-animated groups |
| 3. Per-Group Offset | `Multiply( Sine, Mask Channel )` | Gives each group its own timing/amplitude |
| 4. Vertex Displacement | `Add → World Position Offset` | Physically moves the vertex to create sway |
| 5. Lighting Response | `Sine → Vertex Normal` | Tilts the normal slightly so lighting reacts to the sway, avoiding a flat/pasted look |

---

## 🔬 Shader Component Analysis

| Variable / Function | Visual Role in Shader | TA Analysis |
|---|---|---|
| `Time` node | Drives every oscillation in the graph | Free, continuous input — no need for a custom clock/counter |
| `Sine` wave | Converts linear time into smooth back-and-forth motion | Cheap (single ALU op) compared to procedural noise |
| RGB Mask Texture | Assigns each leaf cluster to a channel (R/G/B) | Authored once in Photoshop per foliage atlas; resolution can be very low since it's just a group ID |
| `World Position Offset` | Applies the final vertex displacement | Runs per-vertex, not per-pixel — keeps cost predictable regardless of overdraw |
| Vertex Normal manipulation | Adds a secondary, smaller sine offset to the normal | Cheap way to fake "3D-ness" without recalculating real lighting geometry |

---

## ✅ What I Did Today

### 1. Time & Sine Wave Setup

- Broke down how `Time` feeds into a `Sine` node, and how multiplying `Time` before the Sine controls oscillation speed independently from amplitude.
- Noted that amplitude is controlled *after* the Sine (multiply the output), while speed is controlled *before* it (multiply the input) — easy to mix these up.

### 2. RGB Mask for Independent Leaf Groups

- Studied how a single low-res mask texture, painted once in Photoshop, can drive completely independent motion for multiple leaf clusters baked onto one alpha card.
- Each channel (R/G/B) acts as a simple "group ID" — multiplying the Sine output by one channel isolates that group's motion from the others.

### 3. Vertex Normal Manipulation

- Reviewed why a second, smaller-amplitude Sine offset is applied to the Vertex Normal rather than just the position — without it, the lighting stays static while the geometry moves, breaking the illusion.

### 4. Performance Discipline

- Noted the video's emphasis on keeping the whole graph "cheap": few nodes, no per-pixel cost, vertex-only displacement — since this shader has to run across potentially hundreds of trees per scene.

---

## 🎯 TA Skills Practiced Today

| Skill | Relevance as a Technical Artist |
|---|---|
| **Procedural Animation (Time/Sine)** | Driving motion mathematically instead of with baked vertex animation |
| **Channel Packing for Logic** | Using R/G/B not for color, but as cheap independent "group selectors" |
| **Vertex-Stage Optimization** | Keeping animation cost in the vertex shader instead of the pixel shader |
| **Believability via Lighting** | Recognizing that motion alone isn't convincing without a matching lighting response |

---

## 💡 Insights & New Understanding

- The mask-channel trick here is the same underlying idea as yesterday's component masking (`R/G/B` isolating X/Y/Depth) — just applied to *grouping objects for independent animation* instead of *separating spatial data*. Same tool, different purpose.
- Mapping this against current-gen Unreal (5.7/5.8): the **math is unchanged** — built-in wind systems like SimpleGrassWind are still Time + Sine + Multiplier under the hood. What's changed is the *geometry* it's applied to. Engines have moved from single flat alpha cards toward **Nanite Foliage**, where individual branches are real 3D geometry assembled procedurally, animated via **Nanite Skinning** rather than a hand-painted RGB mask.
- Lighting has moved the same direction: instead of faking lighting response with a manually offset Vertex Normal, **Lumen's Two-Sided Foliage** model now handles real-time subsurface scattering through leaves, and the **Substrate** framework adds physically accurate response for wet/waxy/dry surfaces as they sway.
- None of this makes the video's method obsolete — it's the fallback (and often the *only* option) anywhere Nanite isn't available.

---

## 📝 Technical Artist Notes

**Where the GPU Cost Actually Lives**
Nanite removed raw triangle count as the main bottleneck, but it did not remove the cost of *moving* geometry. World Position Offset still has to recalculate vertex position and velocity every frame for every animated vertex — the more 3D leaves Nanite renders, the more WPO work the GPU does if they're all swaying. Flat alpha-card foliage avoids that but pays for it in overdraw from stacked transparency instead. There is no free option, only a different place to pay the cost.

**Two Real Production Tracks**
- **Hybrid / Nanite Foliage** (AAA, high-end PC/console): Procedural Vegetation Editor + Nanite Assemblies build trees from real 3D branch geometry — no overdraw, accurate shadows/Lumen, but heavier on WPO and VRAM.
- **Traditional + HISM** (mobile, Switch, low-end PC): exactly the workflow in today's video — Hierarchical Instanced Static Mesh + masked alpha cards + manual LOD chains. Still the only viable option where Shader Model 6 / Nanite isn't supported.

**Standard 2026 Cost-Control Tricks**
- `WPO Disable Distance`: automatically freezes wind animation once a tree is far enough from camera — the single biggest GPU saver for foliage-heavy scenes.
- `r.Lumen.Reflections.MaxRoughnessToTraceForFoliage 0`: skips expensive reflection tracing on foliage, since the eye won't register the difference on swaying leaves.
- Channel-packing Mask + Roughness + AO into one texture's R/G/B to cut redundant texture sampling — the same channel-packing principle from today's mask, reused for a completely different purpose.

**Why Today's Method Still Matters**
Mobile, VR, and handheld targets can't run Nanite, so mobile AAA titles still depend entirely on this exact Time + Sine + Mask approach. Distant background trees in open-world games are also frequently converted to flat billboard impostors using this same shader logic, since full Nanite geometry at that distance would burn VRAM for no visible benefit. Understanding the manual math is what makes the automated systems (and their bugs) legible later.

---

## 🔧 Tools & Environment

- **Engine:** Unreal Engine (Material Editor Graph)
- **Concept Reference:** Ben Cloward — *Rustling Leaves Shader*, UE4 Materials 101, Episode 12
