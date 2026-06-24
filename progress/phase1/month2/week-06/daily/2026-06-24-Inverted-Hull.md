# Daily Log — 2026-06-24

**Phase:** 1 | **Month:** 2 | **Week:** 06 | **Topic:** Outline Rendering Theory — Inverted Hull / Back-Face Culling Trick

---

## 📖 Theory: The Geometry Behind Inverted Hull Outlines

Inverted Hull (also known as the Back-Face Culling outline trick) is an outline technique built on **additional geometry** rather than a screen-space post-process. Instead of detecting edges from the depth/normal buffer after the scene has been rendered, this technique adds a **duplicate mesh** that functions purely as a black "shell" sitting just behind the silhouette of the original object.

The geometry is straightforward: every vertex on the duplicate mesh is pushed outward along its own normal —

```
P' = P + N * OutlineWidth
```

— and the winding order on the duplicate mesh is flipped (or its face culling mode is inverted), so that only the **back face** of the shell survives the rasterizer. Because it is slightly larger than the original mesh, its back face peeks out exactly at the silhouette edge, producing an outline without ever sampling the depth buffer.

This makes the technique fundamentally different from the `fwidth(NormalWS)`-based edge detection studied last week (see `2026-06-22-Toon-Shader.md`) — there, the outline is a **derived screen-space signal**; here, the outline is **physical additional geometry** that genuinely exists in the scene.

---

## 🗺️ Concept Map: Inverted Hull Logic Flow

```
Original Mesh (Pass 1)
    → Rendered normally, full depth-test, full shading

Duplicate Mesh (Pass 2 — "Hull")
    → Vertex Position + (Normal * OutlineWidth)  → Expanded Hull
        → Flip Winding Order / Invert Cull Mode  → Only Back-Face survives
            → Base Color = Solid Black, Unlit     → Hull Shading
                → Rendered with correct depth ordering relative to Original Mesh
                    → Result: black line visible only at the silhouette edge
```

| Stage                   | Operation                                      | Visual Purpose                                                        |
| ------------------------ | ----------------------------------------------- | ---------------------------------------------------------------------- |
| 1. Hull Generation       | Duplicate the mesh, render it as a second pass  | Provides the additional "shell" geometry                              |
| 2. Normal-Based Offset   | `P + N * Width`                                 | Pushes vertices outward so the shell is slightly larger               |
| 3. Winding Inversion     | Flip cull mode / invert face order              | Ensures only the inner side of the shell faces the camera             |
| 4. Unlit Black Shading   | Black base color, lighting response disabled    | Outline stays unaffected by scene lighting, always solid              |
| 5. Render Order / Depth  | Hull rendered with correct depth bias/ordering  | Prevents the hull from swallowing interior detail of the original mesh |

---

## 🔬 Geometric Failure Analysis

The core of today's research was the question: **what kind of geometry causes this trick to fail or look wrong?**

### 1. Thin / flat meshes (cloth, leaves, paper)
The hull's offset width can exceed the actual thickness of the mesh. The front-side and back-side hulls then overlap, or the outline ends up covering the entire thin surface instead of just its edge.

### 2. Hard edges / non-smoothed normals
At sharp corners without normal smoothing, adjacent faces can have drastically different normal directions. Since the offset is normal-based, the hull tears apart at these corners — producing gaps, spikes, or visible seams exactly at geometric folds.

### 3. Non-manifold meshes (hair, perforated cloth, open topology)
The front-face/back-face concept assumes a closed (manifold) mesh. On open topology, back-face orientation becomes inconsistent, causing the hull to appear see-through or partially disappear unpredictably.

### 4. Overlapping / intersecting objects
When two mesh parts intersect (e.g., armor layered over a tunic), one part's hull can be clipped by the depth of the other. The outline then appears broken or discontinuous at the intersection.

### 5. Non-uniform scaling
If the offset is achieved through uniform vertex scaling instead of normal-based offset, outline thickness becomes inconsistent across areas of the mesh with different proportions (e.g., a thin arm versus a thick torso).

### 6. Extreme camera distance
Because the offset is calculated in world-space, an outline that looks proportional at a normal viewing distance will appear too thick when the camera is very close, or disappear entirely at long range.

---

## 💡 Insights & New Understanding

### Inverted Hull vs. Screen-Space Edge Detection — The Fundamental Trade-off

Inverted Hull is computationally cheap (no additional depth/normal buffer sampling required) and its outline stays consistently attached to the geometry in world-space. Its weakness lies exactly where its strength is: because it **depends on the mesh's own topology**, it inherits every topology problem — non-manifold geometry, hard edges, intersections.

Screen-space edge detection (`fwidth`, depth discontinuity), studied last week, does not care about topology at all — it reads the final render buffer. However, it fails to detect internal edges that produce no depth/normal discontinuity (e.g., a printed line pattern on a costume texture).

### Why the Industry Favors a Hybrid Approach

Research shows that modern studios (including pipelines in the style of Genshin Impact / Zelda: BOTW) never rely on a single technique alone:

| Case                            | Technique of Choice                              |
| -------------------------------- | -------------------------------------------------- |
| Body, armor, props (solid)      | Inverted Hull — cheap, consistent in world-space   |
| Hair, thin cloth, foliage        | Post-process screen-space edge detection          |
| Sharp corners on hard-surfaces  | Separate smooth-normal channel (vertex color/UV2) to prevent hull tearing |

The smooth-normal channel solution is worth noting in detail: artists **bake pre-averaged normals** into an additional channel during export from Maya/Blender, so that the hull's offset direction does not inherit the mesh's hard-shaded original normals and tear at corners.

### Distance-Based Width Scaling

To address the extreme-camera-distance failure case, the standard solution is to multiply `OutlineWidth` by the camera-to-vertex distance inside the vertex shader, so outline thickness stays consistent in **screen-space** even though the offset calculation itself still happens in **world-space**. This is the same underlying principle used when UI elements are distance-scaled in many engines.

---

## 🎯 TA Skills Practiced Today

| Skill                                          | Relevance as a Technical Artist                                                          |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Geometry-Based Outline Theory**              | Understanding the trade-offs of geometry-driven outlines versus screen-space signals       |
| **Normal-Based Vertex Offset**                 | Foundational concept behind World Position Offset, applicable far beyond outlines alone   |
| **Topology Failure Case Analysis**             | Ability to predict where a shading technique will fail before implementation               |
| **Industry Pipeline Research**                 | Researching how major studios combine multiple outline techniques in a hybrid pipeline    |

---

## 📝 Technical Artist Notes

**Inverted Hull is not a one-size-fits-all trick.** Today's research confirms there is no single outline technique that is optimal across every geometry type in a scene. A good technical artist selects the technique per asset type, not per project.

**A smooth-normal channel is a long-term investment.** Baking a separate normal channel for outline purposes sounds like additional pipeline overhead, but it is the difference between an outline that reads as professional versus one that visibly breaks at every corner of a character.

**Depth bias and render order are frequent sources of unexpected bugs.** Even with flawless geometry, the render order between the original mesh and the hull can cause z-fighting or have the hull swallow fine detail if not configured correctly.

---

## 🔧 Tools & Environment

- **Engine:** Unreal Engine 5.4 (Material Editor + Custom HLSL Node)
- **Research Focus:** Inverted Hull / Back-Face Culling outline technique
- **Key Research Question:** "What are the geometric weaknesses of the back-face outline trick? When does the technique fail or look wrong?"
