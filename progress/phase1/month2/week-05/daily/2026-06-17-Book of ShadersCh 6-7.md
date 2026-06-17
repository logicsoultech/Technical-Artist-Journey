# Daily Log — 2026-06-17

**Phase:** 1 | **Month:** 2 | **Week:** 5 | **Topic:** The Book of Shaders Ch. 6–7 — Color & Shapes, mapped to UE5 Material Graph

---

## 📖 Theory: The Book of Shaders — Chapter 6: Color

Chapter 6 takes the one-dimensional shaping functions from Chapter 5 and applies them to color manipulation — mixing, gradients, and color space conversion inside GLSL shaders.

### Vector Structure & Swizzle Syntax

GLSL vectors are flexible: the same component can be addressed by different names depending on context.

- Spatial: `.x .y .z .w`
- Color: `.r .g .b .a`
- Texture coordinate: `.s .t .p .q`

**Swizzling** is the ability to reorder, duplicate, or recombine those components in any order to construct a new value — for example `magenta = yellow.rbg` reverses the channel order. This is not just syntactic sugar; it replaces what would otherwise be explicit per-component assignments.

### Color Mixing with `mix()`

`mix(a, b, t)` is the core blending function in GLSL. It interpolates between two values — scalars or color vectors — based on a `t` parameter in `[0.0, 1.0]`. The chapter connects this directly to Chapter 5: feeding a sine wave or easing function as the `t` value produces expressive, animated color transitions driven by `u_time`.

### Per-Channel Gradient Control

`mix()` also accepts a `vec3` as the blend percentage. This means each color channel can be controlled independently — for example, driving the red channel with `pow()` and the green channel with `smoothstep()` simultaneously. This is how you build gradients that shift hue and saturation at different rates, like a sunrise sky simulation.

### HSB Color Space

RGB is not intuitive for color selection — adjusting saturation or hue requires modifying all three channels simultaneously. Chapter 6 introduces **HSB (Hue, Saturation, Brightness)** as a more natural model for algorithmic color work, along with the conversion functions `rgb2hsv()` and `hsv2rgb()`.

### HSB in Polar Coordinates — The Color Wheel

HSB maps most naturally onto polar coordinates, not Cartesian:

- **Hue** → angle (`atan(y, x)`)
- **Saturation** → radius from center (`length()`)
- **Brightness** → constant `1.0`

The raw angle from `atan` returns values in `(-π, π)`. These are normalized to `[0.0, 1.0]` to produce a complete circular color spectrum.

---

## 📖 Theory: The Book of Shaders — Chapter 7: Shapes

Chapter 7 is where the math becomes visible. Instead of drawing with brushes or vectors, the GPU evaluates a mathematical condition for every pixel simultaneously — a fundamentally different mental model from traditional programming.

### Drawing a Rectangle — Why No `if` Statements

GPU shaders run in parallel across thousands of pixels. Branching with `if/else` is expensive because it breaks that parallelism. Instead, rectangles use `step()`:

- Left and bottom edges: `step(edge, st)` on normal UV coordinates
- Right and top edges: `step(edge, 1.0 - st)` on the flipped UV
- Multiply all four results together — in shader math, multiplication is a logical **AND**. A pixel is white only when all four conditions are 1.0

### Drawing Circles — Distance Fields

A circle is defined by distance from center, not by drawing a boundary. The shader calculates how far each pixel is from `vec2(0.5)` using `distance()` or `length()`. The result is a **conical gradient** — black at center, white at the edges. Applying `step()` cuts it into a sharp circle; `smoothstep()` produces an anti-aliased edge.

**Performance note:** `length()` internally calls `sqrt()`, which is expensive on GPU. The chapter offers an optimization: use `dot(v, v)` instead, which gives squared distance. For comparisons and thresholding, squared distance works identically and avoids the `sqrt` cost.

### Combining Distance Fields

Once you have multiple distance field shapes, you can combine them with basic math:

- `min(sdf_a, sdf_b)` → **union** — shapes merge together
- `max(sdf_a, sdf_b)` → **intersection** — only overlapping area survives. Applied with absolute coordinates, this produces rounded rectangles.

### Polar Shapes — Flowers, Gears, Snowflakes

Converting to polar form (`atan(y, x)` for angle, `length()` for radius) and then modulating the circle's radius using angle-dependent shaping functions distorts a circle into organic forms. The radius of a flower petal, for instance, is just a sine wave multiplied into the distance threshold.

### Regular Polygons

Combining `atan()` and a distance field formula parameterized by number of sides allows drawing any regular polygon. A triangle, pentagon, or hexagon is the same code with a different constant.

---

## 🗺️ Concept Map: GLSL Concepts → UE5 Material Graph

```
GLSL (Book of Shaders)           UE5 Material Graph
──────────────────────           ──────────────────────────────
Ch. 6 — Color

  color.rbg  (swizzle)    →      Swizzle node (type "rbg" in Details)
  Extract one channel     →      ComponentMask node (check R / G / B)
  Combine scalars → vec   →      AppendVector node

  Polar color wheel:
  TexCoord
    → Subtract 0.5               (shift pivot to center)
    → VectorToRadialValue        (runs atan2 + length internally)
       ├─ Angle output [0,1]  →  H (Hue)
       ├─ Linear Distance     →  S (Saturation)
       └─ Constant 1.0        →  V (Brightness)
    → AppendVector (H, S, V)
    → HSVToRGB
    → Base Color

Ch. 7 — Shapes

  Rectangle (step × 4)    →      GenerateBand (X) × GenerateBand (Y) → Multiply
                                  or manual: TexCoord → Subtract → Step × 4 → Multiply

  Circle (distance field):
  TexCoord
    → Subtract 0.5
    → Length               →      distance from center
    → Step / SmoothStep    →      hard or soft circle edge
    (optimization: DotProduct with itself instead of Length)

  Shape union              →      Min node
  Shape intersection       →      Max node  (→ rounded rectangle)

  Polar shapes (flowers, stars):
  VectorToRadialValue → Angle
    → Multiply by N (petal count)
    → Sine / Cosine
    → Add/Multiply to Linear Distance
    → Step                 →      final shape mask
```

---

## 🎯 TA Skills Practiced Today

| Skill | Relevance as a Technical Artist |
|---|---|
| GLSL → HLSL mental translation | UE5's Material Graph compiles to HLSL. Understanding the GLSL equivalent means you can debug node outputs and write custom HLSL expressions when nodes fall short |
| Swizzle and channel masking | Channel manipulation is constant work in material authoring — separating R/G channels from a packed texture, remapping outputs, building masks |
| SDF (Signed Distance Field) thinking | SDFs are the foundation of procedural shape generation in UE5 — used in UI materials, decals, particle systems, and custom shaders |
| Polar coordinate mapping | Used for radial gradients, vignettes, lens effects, and any circular procedural pattern in the Material Editor |
| `mix()` / `lerp()` as a design tool | Blending between values with a shaping function as the weight is one of the most reused patterns in technical art — cloth blending, wetness masks, wear gradients |

---

## 📝 Technical Artist Notes

**Why This Chapter Matters Before HLSL**
The Book of Shaders teaches you to think in the GPU's execution model — every pixel evaluated simultaneously, no loops over geometry, pure mathematical functions. UE5's Material Graph enforces the same model visually. Understanding the math first means you're not just connecting nodes; you're knowing what the nodes are computing.

**`step()` vs `smoothstep()` — A Practical Decision**
Hard edges from `step()` alias at runtime — visible stairstepping on diagonals, especially in motion. `smoothstep()` adds a one-node anti-aliasing cost in exchange for a screen-resolution-independent edge. For game materials, `smoothstep()` is almost always the right default unless you intentionally want a pixelated or stylized look.

**SDF Shapes vs. Texture Masks**
A common question: why draw shapes procedurally when you can just sample a mask texture? SDFs are resolution-independent, resolution-scalable, and trivially animatable — you can animate threshold, position, and blend parameters at runtime without resampling. Texture masks are cheaper at static use, but SDFs win for anything dynamic. Knowing when to use which is a core TA judgment call.

**`VectorToRadialValue` — Know What It's Doing**
This node is often used as a black box. Internally it runs `atan2(y, x)` for angle and `length()` for radius. If you need to offset the rotation of a polar effect, you multiply the angle output before it reaches your shaping function — not before the node. Understanding the internals lets you manipulate the output correctly.

---

## 💡 Insights & New Understanding

- Drawing shapes in a shader feels backwards the first time. You're not tracing a boundary — you're writing a mathematical condition and letting the GPU decide, per pixel, whether that pixel is inside or outside. The shape emerges from the math, not from drawing.
- `mix()` in GLSL and `Lerp` in UE5 are the same operation. The name change matters when reading documentation — GLSL uses `mix`, HLSL uses `lerp`. Knowing they're identical prevents confusion when switching between references.
- The `dot(v, v)` optimization for circle SDFs is a good example of the kind of GPU-specific thinking that separates a shader programmer from a general programmer. The math is equivalent; the performance cost is not.
- Polar coordinates unlock an entirely different category of patterns. Everything circular, radial, or rotationally symmetric — color wheels, vignettes, lens distortion, bloom shapes — is easier in polar space. This will come up constantly in VFX material work.
- UE5's **Procedural** node category (`VectorToRadialValue`, `LinearGradient`, `RadialGradient3D`, `DiamondGradient`) exists specifically so artists don't have to rebuild polar math from scratch. Knowing what these nodes do mathematically means you can extend them or replace them with custom HLSL when needed.

---

## 🔧 Tools & Environment

- **Study source:** The Book of Shaders — thebookofshaders.com (Ch. 6 & 7)
- **Engine:** Unreal Engine 5.4, Material Editor
- **Hardware:** RTX 5060 Ti 16GB
- **Session type:** Theory + node mapping (no hands-on implementation today)

---

## 📌 Plan for Next Session

- HLSL intrinsics deep dive: [learn.microsoft.com — HLSL Intrinsic Functions](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-intrinsic-functions)
  - Focus on: `lerp`, `saturate`, `smoothstep`, `frac`
- Implement the polar HSB color wheel in UE5 Material Graph (Ch. 6 hands-on)
- Build a rectangle and circle SDF manually in UE5, compare with `GenerateBand` shortcut (Ch. 7 hands-on)
