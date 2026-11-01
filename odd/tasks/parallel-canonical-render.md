# Parallel canonical render

Objective: render canonical variants of different scenes concurrently, bounded by `execution.num_workers`.
Branch: `feat/workers`. Delivery: ask-on-risk. Tests: `apps/orchestrator/.venv/Scripts/python.exe -m pytest` (from `apps/orchestrator`).

## Specs

- S1: With `num_workers > 1`, up to `num_workers` scenes have their canonical MATLAB attempt running at the same time. User: "corrí @configs/experiments/static_example.yml para 2 3 y 4 workers" showed peak MATLAB concurrency 1 for every worker count.
- S2: Workers only run `_run_single_render_task`; the main thread remains the single owner of the variant index, retry chains, batch circuit, archives, scene manifests and static plan revisions.
- S3: Per-scene behavior is unchanged: canonical reuse on resume, MATLAB process retries, RAVEN T30 rejection → archive + deterministic resample + manifest/plan revision, sampling failure recording.
- S4: Results do not depend on worker count: accepted manifest order, other-variant task order and RAVEN sampling failures are ordered by scene index (`failed_scene_indices` is already sorted). Exceptions (accepted, audit logs written durably as they happen): `scene_revision_history` and the `scene_sampling_failures.jsonl` journal keep completion order.
- S5: On the first error or an open circuit, no new scene, retry or resample is dispatched; in-flight attempts finish and are resolved; then `RenderStaticRunError` is raised as today.
- S6: `num_workers: 1` keeps the existing sequential path.
- S7: Other-variants phase (`_run_render_tasks`) is unchanged.
- S8: Real run of `static_example.yml` with 1 vs 4 workers: identical outputs (modulo run paths), peak MATLAB concurrency 4, render wall time reported.

## Tasks

- [ ] T1 (S1-S7) inline — extract per-scene round/after-canonical steps, add bounded canonical dispatcher, tests RED→GREEN, docs. Commit: pending.
- [ ] T2 (S1-S7) delegated verifier — Risk item 4 (concurrency). Pending.
- [ ] T3 (S8) user-run — real MATLAB benchmark needs the user's MATLAB/RAVEN licenses. Pending.

## Log

- L1 (2026-11-01, user): "Se necesita mejorar o refactorizasr la implementacion de workers en la planificacion y renderizado de las escenas acústicas" (planning part delivered in commit da44858).
- L2 (2026-11-01, user): "si prepara el plan para la paralelización del rendeer" → plan presented in chat; user: "si, implementalo".
- L3: Route: T1 inline (one behavior, no Writer trigger reason). Engram mirror pending: Engram tools unavailable in this session.
