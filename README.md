# LemonSlice × Vexa Meeting Worker Knowledge Base

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** Audited findings override conflicting or inferred DeepWiki statements.  
> **Scope:** Curated engineering knowledge only. Raw DeepWiki answers, duplicate audits, manifests, and generated archives are intentionally not committed.

## Start here

Read the numbered documents as the source of truth. They are split by responsibility so engineers can load only the context relevant to the work they are doing.

1. `01_product_boundary_and_scope.md` — product ownership, worker responsibilities, non-goals, and reuse boundary.
2. `02_google_meet_worker_execution_path.md` — verified container-to-admission execution path.
3. `03_shared_platform_architecture.md` — shared lifecycle and Meet/Teams/Zoom Web adapter boundaries.
4. `04_incoming_meeting_audio.md` — corrected meeting-to-LemonSlice audio path.
5. `05_outgoing_agent_audio.md` — corrected LemonSlice-to-meeting audio path.
6. `06_agent_video_and_camera_injection.md` — current webpage screen-capture path and direct-track target.
7. `07_production_operations_and_security.md` — identities, callbacks, cleanup, diagnostics, ports, and security.
8. `08_repository_classification.md` — what to keep, split, replace, disable, or remove.
9. `09_target_repository_structure.md` — proposed standalone repository layout and Vexa mapping.
10. `10_recommended_lemonslice_architecture.md` — Daily-first provider architecture and LiveKit seam.
11. `11_claim_correction_register.md` — corrections to the original DeepWiki investigation and source precedence.

## Reading order

For product and architecture context, read `01`, `03`, and `10`.

For implementation work, read `02`, then the relevant media documents `04`–`06`, followed by `07`.

Before deleting or moving Vexa code, read `08` and `09`. Files named `recording` contain hidden meeting-monitoring and lifecycle behavior and must not be removed solely because recording is disabled.

## Why the combined document is not committed

The combined document duplicates all numbered files and creates two competing copies of the same knowledge. It remains available in the external source pack for single-file ingestion, but the numbered documents are the canonical GitHub version.

## Source provenance

The source investigation was audited against `tecxbro/vexa` `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`. Revalidate repository-specific claims when the implementation moves materially beyond that commit.
