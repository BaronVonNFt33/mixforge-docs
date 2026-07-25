# Spec — R3: Canvas Inpaint & Edit (v1)
**2026-07-24 · for Jason's review · sources: ask-architecture consult (2026-07-24) + graphify query of the MixForge graph**

---

## 1. What R3 is, in plain terms

Right now every fix in MixForge is a **re-roll**: you don't like a helmet, you render again and hope. R3 changes the unit of work from "generate a new image" to **"change this part of the image I already like."**

You open a render, paint over an area with a brush, type what you want there, and only that area regenerates — the rest of the pixels are untouched. Three actions cover almost everything an artist needs:

- **Add / change** — paint over the head, type "riveted brass helmet with a face shield." Or paint a mangled hand and type nothing: it just re-renders that hand properly.
- **Remove** — paint the pedestal under the feet, and it becomes clean backdrop. No hallucinated replacement object.
- **Extend** (outpaint) — grow the canvas and fill the new border, e.g. turning a tight crop into a full-body frame.

That's it. The value is that a render at 90% stops being a coin-flip re-roll and becomes a 30-second fix — which is what separates a working tool from a slot machine.

## 2. Why it's the right next build

Two payoffs from one piece of plumbing:

**(a) It closes the plinth problem.** The 4-way bisection proved pedestals are a checkpoint prior — no prompt, LoRA, reference, or style-gate setting removes them, and four attempts at pixel-surgery fills all ghosted on your gradient/halo backgrounds. Generative repaint of the below-feet band is the only remaining fix, and it's the same machinery as artist-driven removal. **Automated ground cleanup ships as milestone 1 and gets you the 98% bar.**

**(b) The backend is already 80% built.** This is the important discovery from the graph query. `build_detailer_graph()` ([sdxl_workflow.py:688](../src/mixforge/backends/sdxl_workflow.py)) — the feet/hands rescue pass you already run on every Refine — is a masked inpaint using exactly the node stack the architecture consult independently recommended as the best 2026 choice:

```
LoadImage(init) + LoadImage(mask) → ImageToMask → GrowMask(expand 6)
  → InpaintModelConditioning(noise_mask=True) → KSampler(denoise) → VAEDecode → SaveImage
```

The consult ranked `InpaintModelConditioning` (Fooocus-style inpaint conditioning) as the **best v1 choice** and explicitly noted no Impact Pack is needed. We're already there, and it's battle-tested in production. **No new custom nodes required for v1.**

## 3. What the consult says to change

Three gaps between today's detailer and professional-grade inpaint:

| Gap | Why it matters | Fix |
|---|---|---|
| **No crop-and-stitch** | Full-frame latent wastes model capacity on pixels that aren't changing; small masks come back soft | Crop an ROI around the mask (padded, snapped to /8), inpaint at native resolution, composite back. Consult called this "the highest practical reliability… best overall architecture" |
| **Whole image VAE round-trips** | Encode/decode drifts color and micro-contrast — invisible on busy art, **visible as a seam on your smooth grey/vignette backdrops.** This is likely why my pixel-fill attempts looked wrong too | Hard-composite the original outside the mask (`ImageCompositeMasked`) so untouched pixels are bit-identical |
| **No mask blur, only grow** | Grow alone leaves a hard transition | Add `MaskBlur`-equivalent feather; consult's guidance is grow generously (covers halos/edges) + modest blur (2–6px at 1K), not heavy blur |

Plus the consult's **removal-specific** recipe, which is what the plinth needs:
- Prompt describes the **destination** ("clean neutral studio backdrop, soft gradient, seamless, no objects"), never the thing being removed.
- Negative names the class being erased ("pedestal, podium, rocks, rubble, debris, platform, shadow").
- **Denoise lower than add-object**, raised only if the fill refuses to change.
- Grow the mask past the object's edge halo — ghosts come from a mask that's too tight.
- Removals fail when the prompt is object-centric or the context is too broad. Both are avoided by crop-and-stitch + destination-prompt.

## 4. Backend design

### 4.1 New endpoint
`POST /api/inpaint/run` — modelled on the existing `DetailerRequest` ([server.py:4484](../server.py)), which already carries image/denoise/checkpoint/seed/prompt/negatives.

```
InpaintRequest:
  image           str            # /output/... url of the source render
  mask            str            # base64 PNG (white = repaint) OR "auto:ground"
  mode            str            # "add" | "remove" | "extend"
  prompt          str = ""       # what goes there; empty = "just fix it in-style"
  denoise         float | None   # None = mode default (add 0.55 / remove 0.40)
  grow_px         int | None     # None = mode default (add 8 / remove 16)
  feather_px      int = 6
  extend          {top,right,bottom,left} px   # mode="extend" only
  seed, checkpoint, negatives, source, test_run …  # same as other stages
```

Returns a job like every other stage — same queue, same progress websocket, same sidecar. **Non-destructive: results save as new renders, never overwrite the source.**

### 4.2 Graph builder
`build_inpaint_graph()` in sdxl_workflow.py — extends `build_detailer_graph`, keeping its proven core and adding:

- **ROI crop-and-stitch.** Compute the mask bbox, pad by ~25% of the mask's larger dimension (min 64px context), clamp to canvas, snap to /8. Crop init+mask to ROI, inpaint at ROI resolution (upscale small ROIs toward ~1MP so SDXL is in its comfort zone), then paste back.
- **Hard composite.** `ImageCompositeMasked(destination=original, source=inpainted, mask=feathered)` so only masked pixels change.
- **Feather chain.** `GrowMask(grow_px)` → blur → composite mask.
- **House LoRAs + style stay applied** (via existing `_apply_loras`) so the patch matches the render's look. Non-negotiable — a patch in a different style is worse than the defect.
- **Optional pose ControlNet** — already supported by the detailer; keep it for anatomy fixes.
- **Extend mode**: pad the canvas first (`ImagePadForOutpaint`-equivalent), mask = the new band + a small overlap, denoise higher (new content is invented).

### 4.3 Mask sources
One code path, three producers:

1. **Painted** (artist) — PNG from the browser canvas.
2. **`auto:ground`** — the automated cleanup: rembg matte (`u2net`, installed today, 0.2s/image) → find the true feet line → mask everything below + slight overlap above. Reuses the segmentation approach already proven in [clean_plate.py](../src/mixforge/clean_plate.py) (which uses `birefnet-general` for reference stripping).
3. **Body region** (existing) — `_region_mask_path` / `_multi_region_mask_path` ([server.py:570/595](../server.py)) already build cached feathered white-on-black region PNGs. The new endpoint accepts a region name so "fix the hands" needs no painting.

All three upload through the existing cached `_upload_path()`.

### 4.4 Worker
`_inpaint_worker` — clone of `_detailer_worker`'s shape (its call path from the graph: `_resolve_output_image → _multi_region_mask_path → _upload_path → build_detailer_graph → _submit_and_wait → _stage_dest → _append_render → _render_context`). Sidecar records mask source, mode, denoise, grow, ROI box, and the parent render url — so an edit is **provenance-linked to its source**, feeding R1 later.

## 5. Frontend design

**Host: the lightbox.** `App.jsx` already holds `lightbox = {images, index}` (L135) opened from the gallery (L930), stage preview (L1581), and evolve tree (L1542) — one full-screen surface, already the place you go to inspect a render. Add an **Edit** button there that swaps the static `<img>` for `<MaskCanvas>`.

**New component `web/src/components/inpaint.jsx`** — two stacked canvases (image, mask overlay at ~50% red), pointer events for strokes.

v1 tool set (the consult's must-haves, trimmed to what earns its place immediately):

- Brush + eraser, adjustable size (mouse wheel or slider)
- **Undo/redo** — stroke-level, not pixel-level (keep a stack of stroke arrays; redraw on undo)
- Clear mask, invert mask
- Pan/zoom (reuse the lightbox's existing fit/100% zoom logic)
- Mode selector: Add / Remove / Extend
- Prompt box + denoise slider (defaulted per mode, collapsed under "advanced")
- **"Clean ground" one-click** — runs `auto:ground`, no painting
- Result compare: before/after toggle, Keep (saves as new render) or Discard

Deferred to v2: pressure sensitivity, soft/hard brush, semantic/AI-assisted selection, multi-region per-region prompts, live preview, mask history beyond undo.

**Transport: PNG with alpha, base64 in the JSON POST.** The consult's explicit v1 recommendation — exact raster shape, trivially debuggable, dump-to-disk inspectable. RLE/stroke serialization is a v2 bandwidth optimization we don't need on localhost.

## 6. Milestones

| # | Deliverable | Acceptance test |
|---|---|---|
| **1** | **Auto ground cleanup** — `auto:ground` mask + `build_inpaint_graph` + crop-stitch + hard composite. No UI. | Re-run the 15 plinth-flagged renders from the 50-batch: pedestals gone, backdrop seamless on vignette/halo backgrounds, figure untouched. Plinth-free rate ≥98%. |
| **2** | **Wire it in** — fires automatically on Discover renders whose scale-lock reports `plinth_px > 0`, behind a default-on toggle. | Fresh 20-batch: ≥98% plinth-free with no manual step. Adds ~5s to ~20% of renders. |
| **3** | **Canvas editor** — MaskCanvas in the lightbox, Add/Remove modes, prompt, Keep/Discard. | Paint a helmet onto an existing render and keep it; erase a background object; both seamless, both saved as new renders with provenance sidecars. |
| **4** | **Extend mode** + region shortcuts ("fix hands" without painting). | Extend a cropped render's canvas; run a hands fix from the region menu. |

Milestones 1–2 are the plinth fix and need no UI work. 3–4 are the artist-facing feature.

## 7. Codebase laws this must respect

Learned the hard way in the last two days — violating any of these has already cost a night:

1. **Never `set_cond_area: "mask bounds"`** — on ComfyUI 0.22 with feathered masks it produces NaN latents (the rainbow-head corruption). Use `"default"`.
2. **One sidecar write per render** — the evolve lineage bug was a second write clobbering the first.
3. **Freeze config at enqueue**, don't re-read it in the worker (mid-queue corruption).
4. **After any ComfyUI restart, smoke-test every stage** — the evolve break was silent because only Discover was checked.
5. **No silent server restarts.** Staged changes wait for Jason to bounce the server via the launcher.
6. **Test scripts restore config keyed to their own job id**, never on queue-drain.

## 8. Decisions I need from you

1. **Milestone order** — my recommendation: 1→2 first (you get 98% plinth-free within one work block, no UI risk), then 3. Alternative: jump to the canvas editor if hand-editing matters more to you right now than the automated cleanup.
2. **Inpaint checkpoint?** The consult notes a dedicated inpaint model improves removal reliability. Our detailer already works well with the base checkpoint, so v1 uses CHEYENNE and stays consistent in style. If milestone 1 struggles on the hardest backgrounds, testing an SDXL inpaint checkpoint is the first escalation — your call whether to try one preemptively.
3. **Where does Edit live?** Lightbox is my recommendation (least new surface, already the inspect view). The alternative is a dedicated stage tab like the other six.

---

*Consulted: ask-architecture (SDXL inpaint node-stack comparison, removal reliability, seam/compositing practice, mask-editor UX) · graphify query of the MixForge knowledge graph (existing detailer/mask/upload call paths, frontend lightbox hosts). Line numbers verified against current source, not the 2026-07-21 graph snapshot.*
