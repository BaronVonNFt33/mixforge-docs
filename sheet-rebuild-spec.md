# MixForge — Character Sheet Rebuild Spec v1
**2026-07-24 · READY FOR REVIEW · sources: ask-architecture consult + web research (HuggingFace model cards, GitHub issues, Civitai LoRA pages) + inspection of our own sheet stage**

> **Jason's verdict that triggered this:** "these sheets are almost all unusable. Figures are the wrong size consistently, sometimes woman present, sometimes side-view is garbage… I've done MANY character sheets in ComfyUI by hand that were perfect almost every time." Plus: "inconsistent background 'squares' around the figures — they should be on clean flat background OR nice gradient."

---

## 1. The verdict

**The architecture is right. Four specific defects break it, and three of them are self-inflicted "fixes" compensating for one bad input.**

Our sheet stage already does what the professionals do — one wide canvas, one render, multi-figure OpenPose skeleton, IPAdapter identity reference. The research confirms that's the correct approach and that per-slot-render-and-composite (our `assembled` flag) is the inferior path that "still tends to drift in scale/anatomy."

So this is not a rewrite. It's removing damage.

| # | Defect | Root cause | Jason's symptom |
|---|---|---|---|
| 1 | Head keypoints deleted from the skeleton | `_build_pose_branch(..., strip_head=True)` crops the head cluster out | "side-view is garbage" |
| 2 | Turnaround map anatomically broken | `v4.jpg`: figures not at equal height, no common baseline, torsos too short / legs too long / arms too short, incomplete keypoints, head close-up sharing the canvas with full bodies | "figures are the wrong size consistently" |
| 3 | Panel squares | Post-render slot surgery: `_normalize_sheet_slots` + `_replace_front_with_ref` + `_add_head_closeup` crop, rescale and paste slots back, each carrying its own background tone | "inconsistent background squares" |
| 4 | No identity anchor at the sheet stage | `GENDER_GUARDS` is applied only in Evolve; the pinned `(male man…)` anchor is a Discover-stage regional mask and evaporates before the sheet | "sometimes woman present" |

**The through-line:** `v4.jpg` produces bad figures → we added slot-normalisation to rescale them → that created the squares. Fix the input and the compensations can be *deleted*, not tuned.

---

## 2. What the research actually settled

### 2.1 The head-keypoint trap — and why our old fix was too blunt

The reason `strip_head=True` exists is real: xinsir reproduces the white face-landmark dot cloud literally, so sheets came out with landmark-studded heads. But the cure removed *all* head information, and the consult is explicit about the consequence:

> "If you omit head keypoints, you're effectively saying 'pose unknown in the head region', so the network will improvise — often with ambiguous or front-leaning faces."

**The resolution is a distinction we never made.** Two different things were both called "the head":

- **BODY_18 head keypoints** — nose, eyes, ears, neck. Five dots and a few links. These *encode head orientation*, which is exactly what makes a side or back view render as a side or back view. **Keep these.**
- **The 68-point FACE landmark mesh** — the white dot cloud in `v4.jpg`. The xinsir maintainers state plainly: *"Do not use key map with face and hand as the union controlnet does not seem to be trained with hand/face annotation."* **Drop this.**

`strip_head` threw out both. Keeping body_18 heads while never drawing a face mesh gives us head orientation *and* no dot artifacts.

### 2.2 Construction rules for the turnaround map

From the consult plus pose-sheet workflow practice:

- **Every full-body figure identical pixel height, feet on one common baseline.** Non-negotiable — this is what makes scale consistent, and it's the rule `v4.jpg` violates.
- **Complete body_18 keypoints per figure**, including nose/eyes/ears/neck.
- **Clear horizontal separation** — each figure in its own column with padding; overlapping limbs let the model merge figures.
- **Latent must match or exceed the pose map's resolution and aspect.** Not a square latent for a wide map.
- **Head close-up is a SEPARATE render**, at its own aspect and higher detail, composited afterwards — not a fifth cluster on the body canvas. (This is also what RunComfy's and Mickmumpitz's sheet workflows do: multi-view pass, then face/detail pass, then compile.)

### 2.3 Encoding side and back views in keypoints

This is what makes views unambiguous, and it's pure geometry:

- **Front:** nose + both eyes + both ears, shoulders/hips at full width.
- **¾:** nose offset toward the turn, far eye/ear omitted or narrowed, shoulder width compressed ~15–20%.
- **Side profile:** nose present at the silhouette edge, **one ear only**, eyes collapsed to one, shoulders/hips compressed to ~35–40% width.
- **Back:** **no nose, no eyes, both ears** — that combination is what tells the model "we are behind this person." Shoulders full width, hips full width.

### 2.4 ControlNet model and strength

- xinsir's own guidance is **strength 1.0**; some report `end_percent 0.5` for OpenPose on SDXL.
- **Stick thickness matters:** xinsir models were "trained with non-default pose lines and dots that were made significantly thicker… it still works good with default lines but works better with thicker lines." `comfyui_controlnet_aux` exposes `scale_stick_for_xinsr_cn` for exactly this. Our generator must scale stick width with canvas size — a 9px half-width stick that reads fine on 768px is nearly invisible on a 3840px sheet.
- We currently force `xinsir-all-in-one` (union) at strength 1.2, held 0→100%. Union is the model whose maintainers say it lacks face annotation training. **Recommendation: use the dedicated `xinsir/controlnet-openpose-sdxl-1.0` for sheets**, strength 1.0, and A/B `end_percent` 1.0 vs 0.5.

### 2.5 Identity, costume and gender consistency

Enforced by *stacking*, all sharing one reference and prompt:

- IPAdapter full-body reference **+ face-crop as a second reference** (we already do this — keep it).
- **The prompt must carry the identity anchors at every stage.** Attribute drift late in a pipeline happens exactly when a later stage rebuilds its prompt and drops the anchors — which is our bug #4.
- Optionally a **character-sheet helper LoRA**. Candidates found: [Character Sheet XL](https://civitai.com/models/640207/character-sheet-xl), [XL Face Turn / Charturn family](https://civitai.com/models/889964/xl-face-turn-multi-view-turnaround-model-sheet-character-design), and the classic [CharTurner](https://civitai.com/models/3036/charturner-character-turnaround-helper-for-15-and-21) lineage (SD1.5 origin, XL descendants). CharTurner's documented behaviour — "keeps the outfit consistent, and can be combined with ControlNet openPose to keep the turns under control" — is precisely our need. **Treat as optional Phase 4**, tested against our own baseline, because these are style-opinionated and could fight the house ink look.

### 2.6 Background

His hand-made sheets have one continuous flat or gradient backdrop. Ours has squares because of slot surgery. **The single-render approach gives a continuous background for free** — the fix is deletion, plus one explicit prompt/negative pair: positive names a single seamless studio backdrop, negative names "panels, framed boxes, borders, grid, collage, separate panels, dividing lines."

---

## 3. The plan

### Phase 1 — Generate the turnaround map (the actual fix)
Extend the `hero2` canonical-proportion generator (already proven on Discover) to emit a **4-figure turnaround map**:

- Front · ¾ · side · back, **identical height, feet on a common baseline**, canonical 8-head proportions with correct arm length.
- Complete body_18 keypoints per figure with the per-view head encoding from §2.3.
- **No face landmark mesh, ever.**
- Stick width scaled to canvas width (xinsir wants thick).
- Bounds assertion per figure (the same guard that caught off-canvas wrists in the hero2 pack).
- Emit at the sheet's working aspect; latent matches.

**Acceptance:** overlay the generated map on a rendered sheet — every figure's joints land on its skeleton, and all four figures measure the same height ±2%.

### Phase 2 — Delete the compensations
- `strip_head=False` for sheets (nothing to strip once no face mesh exists).
- Remove `_normalize_sheet_slots`, `_replace_front_with_ref`, and the in-canvas `_add_head_closeup` paste from the classic path. **This is what kills the squares.**
- Switch sheet ControlNet to the dedicated xinsir OpenPose model at strength 1.0.

**Acceptance:** no rectangular seams; one continuous backdrop across the whole sheet.

### Phase 3 — Carry identity into the sheet
- Apply `GENDER_GUARDS` at the sheet stage (not just Evolve).
- Thread the pinned anchors + the render's ethnicity term into the sheet prompt, since regional masks don't survive the stage change.
- Add the anti-panel negatives from §2.6.

**Acceptance:** 10 consecutive sheets, zero gender flips, zero panel borders.

### Phase 4 — Optional: head close-up as its own pass, and helper LoRA
- Head close-up rendered separately at its own aspect, composited as a deliberate inset (not a normalised slot).
- A/B a character-sheet LoRA against the clean baseline.

---

## 4. Laws to respect

1. **Never delete information to suppress an artifact** — `strip_head` traded garbage side views for clean heads. Find the narrow cause (face mesh) and remove only that.
2. **Fix the input before adding post-processing.** Every slot-surgery step exists because the skeleton was wrong; each one added a new artifact.
3. **A/B against OFF, not against another setting** (the style-gate lesson — a gate-vs-gate comparison hid a 90% loss).
4. **Test anatomy at full resolution**, never from contact-sheet thumbnails.
5. **Bounds-assert every generated skeleton** — an off-canvas joint silently truncates a limb and the model invents its own.

---

## Sources

- ask-architecture consult, 2026-07-24 (turnaround architecture A vs B, keypoint completeness, view encoding, identity stacking, drift)
- [xinsir/controlnet-openpose-sdxl-1.0](https://huggingface.co/xinsir/controlnet-openpose-sdxl-1.0) — thicker sticks, strength 1.0
- [xinsir/controlnet-union-sdxl-1.0](https://huggingface.co/xinsir/controlnet-union-sdxl-1.0) — union not trained on face/hand annotation
- [comfyui_controlnet_aux issue #447](https://github.com/Fannovel16/comfyui_controlnet_aux/issues/447) — `scale_stick_for_xinsr_cn`
- [ComfyUI OpenPose Editor](https://github.com/space-nuko/ComfyUI-OpenPose-Editor) · [OpenPose Studio](https://github.com/andreszs/ComfyUI-OpenPose-Studio) · [cozymantis pose-generator node](https://github.com/cozymantis/pose-generator-comfyui-node) — pose-map authoring/merging tools
- [Character Sheet XL](https://civitai.com/models/640207/character-sheet-xl) · [XL Face Turn / Charturn](https://civitai.com/models/889964/xl-face-turn-multi-view-turnaround-model-sheet-character-design) · [IL/XL Charturn Merged](https://civitai.com/models/362559/ilxl-charturn-merged-multi-view-turnaround-model-sheet-character-design) (author: "repeat… including ControlNet (OpenPose)", 1536×1024) · [CharTurner](https://civitai.com/models/3036/charturner-character-turnaround-helper-for-15-and-21)
- Our code: `server.py` `_sheet_worker` / `_strip_head_cluster` / `_normalize_sheet_slots` / `_replace_front_with_ref`; `images/input/project_01/pose/v4.jpg`
