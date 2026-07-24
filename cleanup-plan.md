# MixForge — Code Cleanup Plan & Product Roadmap
**2026-07-24 · full-codebase review (backend + frontend audits + architecture consult) · for Jason's review — no changes made**

The codebase is in better shape than its history suggests: the pipeline design is sound, the recent subject/style block-gating and anatomy work are architecturally clean, and the audit found no rot in the core rendering math. The debt is concentrated in four places: **one giant backend file**, **config read at the wrong time**, **a feedback system that eats its own training data**, and **a frontend that grew faster than its structure**. All of it is fixable incrementally without a rewrite.

---

# PART 1 — CODE CLEANUP PLAN

## 1.1 Critical correctness fixes (do first — these are live bugs)

**① Style-test queue bypass (P0).** `/api/style/test` skips the job queue entirely: it writes job state directly and spawns its own thread. If the dispatcher starts a queued job at the same moment, two renders fight over one GPU and one job's abort flag can cancel the other. Fix: route style tests through the same enqueue path as everything else.

**② Evolve lineage is silently destroyed (P0).** Every Evolve render writes its parent lineage (parent A/B, batch index) to the sidecar — and then the generic recipe writer immediately overwrites the same file without those fields. Every evolve render ever made has lost its family tree. This matters because lineage is the foundation of the provenance features in Part 2. Fix: single merged sidecar write.

**③ Recipes must freeze at enqueue time (P1 — this bit us tonight).** Workers re-read `lora_config.json` and re-scan LoRA folders *at render time*. Any config change while a queue is running silently changes the recipe of already-queued jobs — this is exactly how 8 renders of your 17-batch inherited my test config tonight. Fix: resolve LoRA set + CFG once when the job is enqueued and freeze them into the job. This same change also removes the biggest per-render performance hot spot (dozens of file operations per render re-scanning LoRA folders).

**④ Concurrent-write races on state files (P1).** The feedback store, LoRA config, and UI-state dict are all read-modify-written from multiple threads with no lock, and several writers are non-atomic (a crash mid-write can zero the file). Fix: one lock + one atomic-write helper shared by all config/state writers.

## 1.2 Data safety

**Feedback apply destroys the learning signal (P1).** "Apply & Clean" trashes downvoted images (recoverable) but *deletes the votes* (not recoverable) — and the votes are the fuel for the feedback-aware source picker and the tuning analysis. We rediscovered this the hard way twice this week, reconstructing votes from trash sidecars. Fix: cleared votes move to an append-only `render_feedback_archive.json`; the picker and analysis read live + archive. The thumbs you've cast become a permanent, growing taste model instead of a disposable to-do list.

**`_trash` grows forever.** Every downvote, nuke, and delete accumulates under `OUTPUT/_trash/` with no retention policy — gigabytes and counting. Fix: startup sweep deleting entries older than N days (configurable, logged).

**Sidecar schema drift.** At least four sidecar shapes coexist with no version field; every reader defensively pokes at optional keys. Fix: add `sidecar_version`, centralize the writer/reader in one module. This is also a prerequisite for the recipe/provenance system in Part 2.

**Minor:** upload cache never evicts stale entries and one code path forgets to persist it; temp-file litter (`mixforge_*` in system temp) never cleaned; votes pointing at externally-deleted files never pruned.

## 1.3 Restart debt → live config

A week of tuning history shows the same pattern: the knobs we retune most are hardcoded constants that need a server restart (tonight alone: three restarts to tune the style gate). Move the actively-retuned set into a `config/tuning.json` loaded through the same locked config store, snapshot at enqueue (per 1.1③):

- Block vectors (`SUBJECT_BLOCK_VECTOR`, `STYLE_BLOCK_VECTOR`), `LORA_CLIP_FACTOR`, `SUBJECT_CAP`
- `pose_content_cap` + `content_end_at` defaults, anatomy-lock strengths, pose head/footroom
- Novelty/regional strength caps, MISC weighting multipliers
- **The house negative prompt** — currently the only load-bearing prompt text NOT in an editable file. Move to `prompts/negative.txt` alongside the global prompt.
- Machine paths (ComfyUI URL, checkpoint/LoRA dirs, ControlNet model names) → `config/environment.json` so the code stops hardcoding `C:\Users\Jason\...`

## 1.4 Backend decomposition

`server.py` is 5,573 lines mixing routing, ComfyUI graph orchestration, ~800 lines of sheet image processing, the job queue, feedback analytics, the novelty system, and file management. Every edit risks unrelated subsystems. The audit produced a concrete module map — highlights:

- `jobs.py` (queue, dispatcher, abort), `comfy_session.py` (submit/progress/upload cache)
- `sheet.py` — the 800-line sheet compositing block, the single biggest and most self-contained extraction
- `novelty.py`, `loras.py`, `feedback.py`, `pose.py`, `regions.py`, `refs.py`
- `workers/` (one file per stage), `api/` routers (endpoints + request models)
- Import-time side effects (folder migrations, GPU poller, night-run resume thread) move into an explicit startup hook — prerequisite for any testing

Also: ~500 lines of verified dead code to delete (legacy profile compositors, the orphaned sampling package, unused novelty text-emphasis path, `MUTATION_ELEMENTS`), and the figure-extraction routine that is re-implemented **seven times** with three different thresholds collapses to one helper.

## 1.5 Frontend cleanup

- **Delete `App.jsx.pre-split-backup`** (3,261 lines of committed dead code shadowing every search).
- **Add ESLint + react-hooks plugin and fix what it flags** — one sitting; it mechanically catches the real P1 bug found (the autosave effect omits four settings from its dependency array, so toggling only novelty/gallery-filter/rescue-feet/place-object is silently never saved), plus the unused-import bloat in every component.
- **Fix the four state bugs:** the autosave deps; `carried.final` dropped on refresh (an armed Final input is lost); boot-time fetches with no error handling (server down = silent blank UI); half the API layer skipping HTTP status checks.
- **Server-side thumbnail endpoint (P1, biggest perceived-performance win).** The gallery loads full-resolution SDXL PNGs (megabytes each) for up to 400 thumbnails displayed at 128px. A `?thumb=256` endpoint plus memoized thumbnail components transforms gallery feel.
- **Split App.jsx** (1,895 lines, ~60 useState hooks): per-stage screens + extracted polling/persistence/gallery hooks. This also isolates the 500ms progress tick so it stops re-rendering the entire app during renders.
- **Consolidate six independent polling loops** into one scheduler with shared cache and backoff.
- **UX safety pass:** unify the two competing confirmation idioms (two-click arm vs `window.confirm`); "Train & clean" gets an in-app preview of what dies instead of a native confirm; Escape/focus-trap/arrow keys on all modals and the lightbox; keyboard-reachable action chips; make destructive-action copy truthful (some says "cannot be undone" for things that go to trash).

## 1.6 Performance (beyond the above)

- `/api/checkpoints` can synchronously copy multi-GB checkpoint files inside a GET (cross-volume hardlink fallback) — move to first-discovery with a marker.
- Preprocessor probing blocks `/api/project` up to ~40s when ComfyUI is down — cache negative results, cut timeout.
- Gallery listing re-parses every sidecar in OUTPUT on every poll — mtime-keyed cache or an append-only index.
- Sheet pipeline round-trips full-size PNGs through PIL↔numpy 6+ times per sheet — single in-memory pass.
- Two pure-Python per-pixel loops (pose-map heuristics) → numpy.

## 1.7 Observability & tests

- Replace ~60 bare `print()` calls with structured logging (timestamps, levels, job-id correlation); surface worker warnings ("rescue pass skipped", LoRA stack alerts) into the job record so the UI can show them.
- **Zero tests today.** The audit identified the ten highest-value seams, all pure or one-line-injectable: the weight sampler, novelty picker, prompt composition (the None/""/value contract), the path-traversal guard, resolution fitting, region geometry (site of a past batch-killing bug), the graph builders (assert block-gating/pose wiring shape), project-state save/load, and feedback analysis thresholds. Start there; the decomposition makes the rest testable.
- Give agent test scripts a versioned home in the repo (`scripts/agent/`) — the server already has first-class support (`test_run`, `test_folder`, `source: agent`) but the scripts that used it all week lived outside git.
- Fold in the process law from tonight: test scripts restore config keyed to their own job id, never on queue-drain.

## 1.8 Security note (LAN)

The server binds 0.0.0.0 by design, but that exposes unauthenticated destructive endpoints (content-nuke, delete) and — worse — Night Run's `folder_path` accepts any local path and copies its images into the served OUTPUT tree: a LAN file-disclosure vector. Minimum fix: allowlist Night Run source roots; optionally a shared-secret header on destructive endpoints.

## 1.9 Suggested sequence

| Phase | Contents | Effort |
|---|---|---|
| **0 — one evening** | Delete dead files, ESLint + mechanical fixes, the four frontend state bugs, `_trash` retention sweep | small |
| **1 — correctness** | The two P0s, config-freeze-at-enqueue, state-file locking/atomic writes, feedback vote archiving | 2–3 sessions |
| **2 — foundations** | `tuning.json` + `environment.json` + `negative.txt`, sidecar versioning, thumbnail endpoint, logging | 2–3 sessions |
| **3 — structure** | Backend decomposition (sheet.py and jobs.py first), App.jsx split, polling consolidation, first ten tests | ongoing, incremental |
| **4 — polish** | UX safety pass, perf items in 1.6, security allowlist | as touched |

Phases 1–2 are the ones that directly protect your work (renders, votes, recipes). Phase 3 is what makes every future feature cheaper.

---

# PART 2 — PRODUCT ROADMAP: MixForge as a professional creative tool

The through-line from the architecture consult: what separates a pro tool from a hobby SD frontend is not more models — it's **treating renders as assets with provenance, giving the artist control surfaces instead of re-rolls, and fitting into the rest of a working pipeline** (Photoshop, PureRef, Blender, client review). MixForge already has unusual foundations here: recipe sidecars on every render, a staged pipeline, a feedback loop, and reference-first generation. The roadmap builds on those instead of bolting on features.

Ordered by artist-value vs build-cost:

## R1. Recipe & provenance system ("go back to Tuesday's version and push the horns")
Formalize the sidecar into a first-class **Recipe**: seed, checkpoint hash, LoRA set + weights + block vectors, refs used, pose, novelty/MISC rolls, all stage settings. One-click **re-render from recipe**, **recipe diff** between any two renders ("what changed between these?"), and **pin** recipes as a character's canonical look. Depends on sidecar versioning (Phase 2) and the evolve-lineage fix. *This is the single highest-leverage feature: it converts everything you've already rendered into a reproducible, revisable archive — the thing an art director conversation actually requires.*

## R2. Character-first data model
Projects → Characters → Sessions, in SQLite. A **Character card** binds: brief, reference buckets, style pack, identity refs/LoRA, canonical palette, pinned recipes, and its whole render history with lineage. The UI's left nav becomes Projects/Characters instead of one global project folder. Everything else on this list hangs off this model.

## R3. Regional edit / inpaint on canvas
Paint a mask directly on a render → regenerate only that region (inpaint), extend the canvas (outpaint), or micro-fix with a detail brush (eyes, hands — the hand/feet detailer already exists as a masked-inpaint path; this exposes the same machinery interactively). Consult verdict: alongside provenance, this is what artists demand most — spot-fixing instead of re-rolling, keeping ownership of a 90%-right image.

## R4. Variation explorer with identity lock
Replace "GO again" with structured sweeps: pick an axis (pose, costume, palette, silhouette scale, stylization, novelty amount), hold identity constant, get a labeled grid. The bones exist — batch + seeded rolls + the new pose pack are exactly axis sweeps; this is UI + recipe parameterization, not new pipeline.

## R5. Character consistency toolkit
Extend the Sheet stage into a **shot manager**: front/¾/side/back/action, face identity locked via face-crop IPAdapter while pose ControlNet drives the body. Longer-term: one-click identity-LoRA training from a character's pinned renders, so a finished design can be redrawn in any scene. (Consult: cross-scene consistency is table stakes for pro tools in 2026.)

## R6. Palette & material control
Per-character palette (primary/secondary/accent) + material tags (worn leather, blued steel, chitin…) that auto-inject into prompts and optionally feed a color-reference branch. Consistent color identity across a character's whole history is a distinctly *concept-art* need that generic frontends ignore — and it composes perfectly with the ink-style separation you now have.

## R7. Layered export to the pipeline
- **PSD export**: character on its own layer (the figure-extraction helper from Part 1 gives the matte), separate line pass, background layer.
- **Turnaround pack**: orthographic views bundled for Blender image-plane import.
- **PureRef-style board export**: auto-built grid of a character's pinned renders with labels.
- Deterministic file naming (`PROJECT_CHAR_STAGE_SHOT_VAR`) on all exports.

## R8. Review boards (the github.io pipeline, productized)
Tonight's `renders.html` push proved the delivery loop: one-click **publish a board** (character, session, or comparison A/B) to the docs site — shareable with a client or viewable from the couch, with optional per-image notes. Later: thumbs cast on the phone flow back into the feedback store.

## R9. Feedback 2.0
With archived votes (Part 1), the taste model becomes cumulative: per-reference and per-LoRA quality scores that persist, defect tagging (anatomy / style / composition) instead of a bare thumbs-down, and periodic "the data suggests…" tuning proposals you approve — the auto-tune loop we already run, made visible and permanent.

## R10. Ops hardening for long runs
Durable job queue (survives server restart — night runs currently depend on a resume thread), a model manager panel (what's installed, hashes, which recipes depend on which checkpoint — protects reproducibility when files change), and job history with per-stage timing.

**What to skip (consult + my judgment):** multi-user collaboration/comments infrastructure, cloud rendering, training UIs beyond one-click identity LoRA, video — all high-cost, off-thesis for a local-first solo tool. The moat is *asset discipline + control surfaces on your own GPU*.

## Suggested order
**R1 → R3 → R4** (provenance, inpaint, variation explorer — the daily-driver trio), then **R2** formalizes the data model under them, then **R5–R7** (consistency, palette, export) make it studio-grade, with **R8–R10** as continuous quality-of-life. Phase 1–2 of the cleanup plan is a prerequisite for R1 (frozen recipes + versioned sidecars are literally the provenance feature's substrate).
