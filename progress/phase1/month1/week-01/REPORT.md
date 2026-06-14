# Week 1 Progress Report

**Phase:** 1 — Foundation  
**Month:** 1  
**Week:** 1 of 12  
**Focus:** Complete Bad Decision Studio + tools setup  
**Engine:** Unreal Engine 5.4 (downgraded from 5.7)

---

## Weekly Targets (from curriculum)

- [ ] GitHub repo active with daily commits
- [ ] Freya Holmér: Dot Product understood
- [ ] First HLSL custom node running
- [ ] Book of Shaders Chapter 1–5 completed
- [ ] Blender→UE5 pipeline attempted for the first time

---

## Schedule vs. Actual

### Monday–Tuesday
| Scheduled | Status |
|---|---|
| Finish Bad Decision Studio — focus on "why", not "how" | ✅ Done (pre-week) |
| AI Day 1: ask Claude about everything not understood from tutorial | ✅ Done (ongoing) |
| Alt+8 Shader Complexity in BDS scene → first baseline screenshot | ✅ Done (pre-week) |
| GitHub account + repo setup | ✅ Done (Monday 18 May) |
| Window › Statistics in UE5 → screenshot draw call count | ⏳ Carry to Wednesday |

### Wednesday
| Scheduled | Status |
|---|---|
| Create repo: ue5-ta-journey → commit first UE5 project folder | ✅ Done |
| Window › Statistics → screenshot draw call count | ⏳ Pending |

### Thursday–Friday
| Scheduled | Status |
|---|---|
| YouTube: Freya Holmér — "Why can't you multiply vectors?" (22 min) | 🔜 Upcoming |
| YouTube: Freya Holmér — "The Dot Product" (16 min) | 🔜 Upcoming |
| AI: "Explain dot product for shaders with a simple analogy" | 🔜 Upcoming |
| Practical: Material Editor → Dot Product node → Base Color → see result | 🔜 Upcoming |

### Saturday
| Scheduled | Status |
|---|---|
| Freya Holmér — "The Cross Product" (11 min) | 🔜 Upcoming |
| Create ArtStation account → upload first screenshot | 🔜 Upcoming |
| Week 1 review: what I can do, what I am still confused about | 🔜 Upcoming |

---

## Completed This Week

### Pre-week (before official Week 1 start)
- Bad Decision Studio — full scene walkthrough, understanding the "why" behind every technical decision
- Shader Complexity baseline captured (`Alt+8`) — first technical analysis performed
- 4 bugs encountered, investigated, and resolved (see `TROUBLESHOOTING.md`)

### Week 1 proper
- GitHub repo `ue5-ta-journey` initialized — public, MIT license, UnrealEngine .gitignore
- Folder structure built: `progress/week-01/daily/`, `screenshots/`, `docs/`, `config/`
- `README.md` finalized — personal story + full 5-phase roadmap + Week 1 technical findings
- `TROUBLESHOOTING.md` written — 4 bugs with root causes, hypotheses tested, fixes applied
- `docs/engine-notes.md` written — Shader Complexity, overdraw, material graph concepts
- `config/DefaultEngine.ini.patch` written — `r.Shadow.Denoiser=0` with full annotation
- Daily commit workflow established

---

## Technical Findings

**Scene:** Bad Decision Studio — 175 actors · UE 5.4

| Zone | Shader Complexity | Notes |
|---|---|---|
| Platform floor | Red | Multi-layer material, 6–7 texture samples, 3 UV scales |
| Left overhead | Magenta | Exceeds MaxShaderComplexityCount=2000 — overdraw suspected |
| Stormtroopers | Green | Efficient shaders |
| Background / sky | Green | Efficient |

**Frame data (editor viewport):** 120 FPS · 8.33ms

**Key concept understood:** Color in a material comes from the node graph, not the texture. A white texture + Multiply node + Constant3Vector = any color. This is standard material authoring practice.

---

## Bugs Resolved

| Bug | Root Cause | Fix |
|---|---|---|
| Viewport inconsistency | UE 5.7 pipeline mismatch with tutorial | Downgraded to UE 5.4 |
| Paid decal required | Color was a node parameter, not in the asset | White texture + Multiply + yellow Constant3Vector |
| CineCamera autofocus inactive | Camera not initialized as active | Pilot → Play → Stop → Exit |
| Ghost shadow on Stormtroopers | Shadow Denoiser bug in UE 5.4 | `r.Shadow.Denoiser=0` in DefaultEngine.ini |

---

## Screenshots

| File | Description |
|---|---|
| `screenshots/shader-complexity-baseline.png` | Shader Complexity heatmap — BDS baseline |
| `screenshots/ghost-shadow-glitch.png` | Ghost shadow bug before fix *(if captured)* |

---

## Honest Self-Assessment

**What went well:**
- Did not give up when UE 5.7 caused problems — made the call to downgrade and restart from scratch
- Found a creative solution to the paid decal problem that also taught a real material authoring concept
- Read output logs systematically instead of randomly changing settings
- Identified the Shadow Denoiser as root cause after 5 failed hypotheses — correct debugging methodology

**What was harder than expected:**
- GitHub setup consumed a full day — more than anticipated for someone new to version control
- The DLSS/Shadow Denoiser confusion cost significant time because the output log pointed in the wrong direction

**What I still do not fully understand:**
- Why the magenta overdraw zone appears specifically in the left section of the scene
- Whether the CineCamera pilot+play workaround has any lasting side effects on the project

---

## Next Week Preview — Week 2

**Focus:** Material Editor deep dive + first HLSL custom node

### Monday–Tuesday
- YouTube: Ben Cloward — "UE5 Materials" playlist ep 1–3 (pause every 5 min, practice immediately)
- AI: ask Claude about every node not understood, with specific context

### Wednesday
- Material Editor → Custom node → type: `return float3(1.0, 0.5, 0.0);` → connect to Base Color
- First HLSL running on GPU — experiment: change 3 numbers, watch color change
- AI: "What does float3 mean? Why 3 numbers? How does it relate to RGB?"

### Thursday–Friday
- thebookofshaders.com → read and run Chapter 1–3 in browser (type manually, do not copy-paste)
- Ben Cloward ep 4–5 → practice alongside

### Saturday
- Build first self-designed material — free design, minimum 1 math node
- Commit + upload to ArtStation

---

*Phase 1 — Foundation · Month 1 · Week 1*  
*Last updated: 2026-05-19*
