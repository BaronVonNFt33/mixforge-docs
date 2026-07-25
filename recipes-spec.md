# MixForge — Recipes Spec v1
**2026-07-24 · for Jason's review · R1 from the product roadmap**
*Builds on the ask-architecture consult of 2026-07-24 (reproducible recipe systems, provenance, pinning) and the existing sidecar/project-state machinery in the codebase.*

---

## 1. What a Recipe is

A **Recipe** is the complete set of *inputs* that make a render — model, prompts, LoRAs, buckets and weights, pose, misc, novelty settings, and optionally the seed. It is not an image and not a project; it's the **reproducible dial-setting of the whole tool at one moment**.

Two ways to make one, both required:

- **From an image** — point at any render you like: "make a recipe from this." Its settings are recovered from the render's sidecar JSON.
- **From current state** — "save what I've got dialed in right now," before you start fiddling.

Two ways to use one:

| Load mode | What it does | When you want it |
|---|---|---|
| **Replay exactly** | Restores settings **plus** seed, bucket picks, and the exact novelty terms that were rolled | "Give me *that image* again" — reproduce, then tweak one thing |
| **Use as settings** | Restores the dials but **re-rolls** picks/novelty/seed | "Give me *more like that*" — same look, new characters |

That distinction is the whole point: today a good render is a dead end, because the thing that made it (which refs got picked, which wildcards fired) evaporates the moment you change a slider.

## 2. What's in a Recipe

Everything below already exists in the codebase — a recipe is an honest snapshot of it, not new data:

```
meta        id · name · created_at · updated_at · recipe_version
            origin: "image" | "state" · origin_url · thumb
model       checkpoint · cfg · steps · width · height · draft
prompts     project_prompt · global_prompt (resolved TEXT, not "default")
            negative override · refine_prompt_override · held_prop
loras       [{ file, kind, weight, clip, chance, block_vector }]
buckets     [{ id, label, weight, range, enabled, removed[] }]
            + picks[] (replay mode only — the exact ref image per bucket)
style       enabled · folder · weight · weight_type · combine_embeds · start_at · end_at
pose        enabled · file · strength · start_percent · end_percent · detect_hands/face · resolution
misc        enabled · strength · chances{} · default_chance  + picks{} (replay only)
novelty     amount · strength · slot_max · on  + terms[] & regions[] (replay only)
guards      anatomy_lock · pose_content_cap · scale_lock · rescue_feet_hands
seed        value + lock_seed flag
```

**Source of truth for "from image" is the render's sidecar `.json`**, not the PNG. Verified today: PNGs carry no embedded workflow, because the scale-lock pass re-saves them through PIL and strips ComfyUI's metadata block. If an image has no sidecar (imported from outside MixForge), Make Recipe is disabled with a clear reason — offer "add as reference bucket image" instead.

## 3. Storage

One directory per recipe — trivially backed up, trivially deleted, no index file to corrupt:

```
config/recipes/<slug>__<id8>/
    recipe.json     # the payload above, written atomically (tmp + replace)
    thumb.jpg       # 256px, generated at save time
```

- `recipe_version: 1` from day one. (Learned from the sidecar drift the audit found: four undocumented shapes with no version field.)
- Atomic writes, matching the pattern already used by `render_feedback.json` and `project_state`.
- Thumbnails: **origin=image** → downscale that render. **origin=state** → the newest render of the current session; if there is none, a generated card showing checkpoint + LoRA names on the house background. Thumb is replaceable later ("set thumbnail from this render").

## 4. API

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/recipes` | List: id, name, thumb url, updated_at, summary badges (checkpoint, LoRA count, pose name, bucket count) |
| `GET` | `/api/recipes/{id}` | Full payload for loading |
| `POST` | `/api/recipes` | **Save New** — body `{name, origin, ui?, image?}` |
| `PUT` | `/api/recipes/{id}` | **Save** — overwrite an existing recipe from current state (keeps id, name, created_at) |
| `PATCH` | `/api/recipes/{id}` | Rename / replace thumbnail |
| `DELETE` | `/api/recipes/{id}` | **Delete** — moves the directory to `config/recipes/_trash/<date>/`, never hard-deletes (house rule: all deletes are recoverable) |

Thumbs served from a static mount (`/recipes/<id>/thumb.jpg`), same pattern as `/output` and `/refs`.

## 5. UI

### 5.1 Recipes browser — thumbnail grid, like the LoRA loader
A **Recipes** button in the top bar (next to Save project) opens a modal:

- **Grid of cards**: thumbnail, name, and badges (checkpoint · N LoRAs · pose · N buckets). Same visual language as the LoRA rows in Options, but laid out as a grid because the thumbnail is the identifier.
- **Per-card actions** (revealed on hover *and* on keyboard focus — the audit flagged hover-only chips as unreachable): **Load**, **Save** (overwrite this one with current state), **Rename**, **Delete**.
- **Delete** uses the house two-click arm pattern, not a native `confirm()`.
- **Save New** button in the modal header → name field, saves current state.
- Sort: recently updated first; a name filter box once there are more than ~20.

### 5.2 Load dialog
Clicking Load shows the two modes from §1 as radio buttons, plus:
- **"Also load LoRA settings"** — checked by default, with the plain-language warning that LoRA weights are global app config, so loading changes them for every stage. Unchecking keeps your current LoRA setup and loads everything else.
- **Missing assets** are listed before applying: any bucket image, pose file, or LoRA the recipe names that's no longer on disk. Load proceeds with what exists; nothing is silently dropped.
- After load: a toast — "Loaded «name» · **Revert**" — where Revert restores the settings snapshot taken immediately before applying. (Cheap undo, uses the existing persist bundle.)

### 5.3 "Make Recipe from this image"
A new action in the render action row — the same cluster as **Full Run** and **Use in Refine**, in both the gallery thumb and the lightbox. One click: name prefilled from the render, thumbnail from the render, saved. This is the fast path and probably how most recipes get made.

## 6. Implementation notes

- **Extraction (`origin=image`)** maps sidecar → recipe fields. The sidecar already records `loras_fired`, `bucket_picks`, `misc_picks`, `novelty` + `novelty_regional`, `pose`, `cfg`, `steps`, `width/height`, `checkpoint`, `seed`, and the composed `positive` — i.e. nearly the whole recipe exists today; this is mostly a rename-and-normalize layer.
- **Applying (`load`)** is a frontend action: set React state from the payload, then POST the LoRA rows to `/api/lora` if that box is checked. The existing autosave persists it like any other change.
- **Replay fidelity**: same seed + same picks + same novelty terms + same settings on the same hardware should reproduce the render essentially identically. Where it can't (a ref file was deleted), say so up front rather than producing a silent near-miss.
- **Don't couple to the project file.** A recipe is portable settings; the project file stays the working session. A recipe references bucket images by name, so recipes survive project edits but can go stale — hence the missing-asset check.

## 7. Milestones

| # | Deliverable | Acceptance |
|---|---|---|
| **1** | Store + CRUD + sidecar→recipe extraction + thumbnails | Make a recipe from a render via API; list it; delete it (lands in `_trash`) |
| **2** | Recipes modal: grid, Load (both modes), Save New, Save, Delete, Rename | Replay-exactly on a known render reproduces it; settings-mode gives fresh variety with the same look |
| **3** | "Make Recipe" action in gallery + lightbox; missing-asset warnings; Revert toast | One-click from any render; deleting a ref then loading warns instead of failing |
| **4** *(v2)* | Recipe **diff** (what changed between two), **pin as canonical** for a character, export/import a single `.mfrecipe` file | — |

## 8. Decisions I need

1. **Default load mode** — my recommendation: **"Use as settings"** as the default (the common case is "more like this"), with Replay one click away. Or flip it if exact reproduction is what you reach for more.
2. **Should Load also switch the active stage** (e.g. a recipe made from a Refine render puts you in Refine)? My recommendation: no for v1 — surprising navigation. It's recorded in the recipe either way.
3. **Recipes global, or per-project?** v1 stores them globally in `config/recipes/` so they're reusable across projects; a `project` tag on each recipe lets us filter later without a migration.

---

## Appendix — laws this must respect

Hard-won in the last three days; each one already cost a night:

1. **Version the payload** (`recipe_version`) — unversioned sidecars are why extraction needs a normalize layer at all.
2. **Atomic writes** for every config/state file; several writers in the codebase still aren't, and a crash mid-write zeroes them.
3. **All deletes go to `_trash`** — and the copy must say "moved to trash," not "cannot be undone."
4. **No hover-only controls** — focus-reveal too.
5. **Two-click arm, not `window.confirm`** — the codebase currently mixes both idioms.
6. **Applying a recipe writes global LoRA config** — that side effect must be visible and opt-out, never silent.
