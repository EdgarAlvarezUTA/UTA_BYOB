# Migration Path Options Analysis — v2
## Cloudinary → UTA S3 (Enterprise External-Storage)

**Project:** AwsMigration — UTA Cloudinary → S3 Migration
**Author:** Edgar (Architect)
**Date:** March 20, 2026
**Status:** Updated — post structured metadata research (March 23, 2026); three-option analysis
**Supersedes:** migration-path-options-analysis.md (v1, March 15, 2026); prior two-option v2 draft
**Distribution:** UTA Stakeholders, Cloudinary Professional Services

---

## Purpose

This document presents a structured comparison of three viable migration paths for moving ~3 TB of binary assets (images, videos, audio, PDFs) from Cloudinary-managed storage to UTA-owned S3, while retaining Cloudinary as the CDN and metadata management layer (BYOB — Bring Your Own Bucket).

**Q&A 3 resolved the single most important open question from v1:** `cld clone` is now confirmed to copy binary assets and metadata server-side. Post Q&A 3 research into the Cloudinary Admin API confirmed that structured metadata field definitions and asset-level values are fully accessible via REST endpoints (`GET /metadata_fields`, `GET /resources?metadata=true`, `PUT /resources/:asset_id`), enabling a self-directed metadata migration loop without PS involvement. This introduces a third credible option. Option D (S3 cross-account copy) remains deferred to the PS scoping session; it is not a planning basis.

**Prerequisite (applies to all options):** Cloudinary PS must configure the new BYOB-backed environment — and set the S3 backend pointer — **before** any migration work begins. This is a hard dependency confirmed in Q&A 3.

---

## What Q&A 3 Confirmed (Key Inputs to This Analysis)

| Question (prev. open) | Answer Confirmed |
|---|---|
| Does `cld clone` copy binary assets + metadata? | **Yes — server-side, no intermediary machine.** Copies binaries + `public_id`, tags, context fields. |
| Is the copy server-side or through the client? | **Server-side.** No EC2/bandwidth required for binary transfer. |
| Does `cld clone` preserve `public_id`, tags, context? | **Yes.** Fully preserved. |
| Is structured metadata (taxonomy) copied by `cld clone`? | **No.** Structured metadata taxonomy requires a separate bulk export → re-import step. |
| Are upload presets and env config copied? | **No.** Must be independently reconfigured on the new environment. |
| Does `cld clone` support incremental / phased execution? | **Yes.** Supports search expressions to clone subsets (by folder, prefix, etc.). |
| Is there an async mode for large datasets? | **Yes.** Async option available + webhook notifications on completion. |
| Does `cld sync` support environment-to-environment? | **No.** `cld sync` is local filesystem ↔ Cloudinary only. Eliminated as a migration tool. |
| When does PS configure BYOB? | **Before the clone runs.** BYOB is a prerequisite for `cld clone` to write to UTA S3. |
| CDN cache behavior at CNAME cutover | If CNAME is preserved, cached content continues to be served correctly; cache purge may not be needed. (PS to confirm.) |
| S3 key naming convention under BYOB | **Deferred to PS scoping.** UTA control over key structure is unknown. |
| Cross-account S3 copy feasibility | **Deferred to PS scoping.** Not a planning basis. |

---

## The Three Viable Options

> **Asset count confirmed:** UTA's Cloudinary environment contains approximately **33,000 assets** (~3 TB total). This figure is used throughout for rate-limit and timeline calculations.

### Option A — `cld clone` + PS-Assisted Structured Metadata Migration

PS configures the new Cloudinary environment with BYOB pointing to UTA's S3 bucket. A `cld clone` command is then executed against the source environment. Cloudinary performs a **server-side copy** of all binary assets and most metadata directly into the new environment. Because the target environment is BYOB-backed, the binaries land in UTA's S3 bucket automatically — no intermediate machine, no egress routing.

The migration is completed in two additional steps:
1. **Structured metadata taxonomy** (field definitions + asset-level values) — Danny (Q&A 3) referenced a "bulk metadata export → re-import" process that PS has established for migration engagements. **However, whether this is a native Cloudinary tool, a PS-proprietary script, or a manual process has not been confirmed.** This step is assumed to be PS-led and must be formally scoped. The delivery team cannot depend on it until PS clarifies the mechanism and timeline.
2. **Upload presets and environment configuration** are manually reconfigured on the new environment.

`cld clone` supports **search expressions** so the operation can be scoped to folders or subsets for phased execution and validation before committing to the full ~3 TB. An **async flag + webhook** notification is available for long-running operations.

**Implementation effort:** Low engineering — primarily operational. PS handles BYOB setup and structured metadata migration; delivery team handles env config replication.

**Key assumption (unconfirmed):** PS has a reliable, time-bounded process for structured metadata migration. If PS cannot commit to this, Option A degrades to Option C.

---

### Option B — Worker Fleet (Custom Uploader Loop)

A purpose-built migration engine: a Python controller enumerates all asset `public_id`s from the source environment (via Admin API, respecting the 5,000 req/hr rate limit), then enqueues them into SQS. Stateless workers pull from the queue, download each binary from the source Cloudinary environment (or its CDN), and re-upload it to the new Cloudinary environment via the **Cloudinary Upload API** — which, with BYOB active on the target environment, writes binaries to UTA S3 automatically. Per-asset metadata — tags, context, structured metadata fields — is explicitly written during the upload or via a subsequent Admin API call.

DynamoDB tracks per-asset state (status, attempts, errors, `s3_etag`). CloudWatch and Datadog provide real-time observability. A dead-letter queue captures failures for targeted remediation.

**Implementation effort:** High engineering — full controller + worker + infra build-out (ECS/Batch, SQS, DynamoDB, CloudWatch). Full programmatic control over every asset.

---

### Option C — `cld clone` + Self-Directed Admin API Metadata Loop

PS configures the new Cloudinary environment with BYOB pointing to UTA's S3 bucket. A `cld clone` command performs the same **server-side binary transfer** as Option A — binaries, `public_id`, tags, and contextual metadata are copied without routing data through an intermediary machine.

The structured metadata gap is closed by the delivery team directly, without PS involvement, using the Cloudinary Admin API:

**Step 1 — Migrate field definitions (taxonomy):**
The Admin API endpoint `GET /metadata_fields` returns all structured metadata field schemas (labels, types, datasource values, validation rules, defaults) from the source environment as JSON. A script replays each field definition against the new environment via `POST /metadata_fields`. This is a one-time operation on the order of tens to hundreds of API calls — well within rate limits.

**Step 2 — Migrate asset-level metadata values:**
A script enumerates all assets from the source environment using `GET /resources?metadata=true` (paginated, up to 500 assets/page) and records each asset's `public_id` and structured metadata values. After `cld clone` completes, a second pass calls `PUT /resources/:asset_id` on the **target environment** for each asset to re-attach its structured metadata values.

**Rate limit impact (confirmed asset count: ~33,000 assets):** At 5,000 req/hr, 33,000 write calls complete in approximately 7 hours — well within a single overnight migration window. If a temporary rate limit increase to 20,000+ req/hr is approved by Cloudinary, the metadata loop completes in under 2 hours. This is a low-risk, predictable operation at this asset count.

A lightweight tracking table (DynamoDB or a local CSV) can record per-asset metadata write state, enabling clean resumability if the loop is interrupted without reprocessing already-completed assets.

`cld clone` retains its search expression support for phased binary migration; the metadata loop mirrors the same phasing by folder or prefix.

**Implementation effort:** Medium engineering — `cld clone` is operational (same as Option A); the metadata scripts are custom but straightforward REST. No SQS/ECS/Batch build-out required. No PS dependency beyond BYOB setup.

---

## Comparison Table

| Quality Attribute | **A — `cld clone` + PS Metadata** | **B — Worker Fleet (Upload API Loop)** | **C — `cld clone` + Admin API Loop** |
|---|:---:|:---:|:---:|
| **Feasibility (confirmed)** | 🟡 PS metadata step unconfirmed | ✅ Fully understood | ✅ Admin API endpoints confirmed |
| **Binary Transfer Method** | ✅ Server-side — no intermediary | 🟡 Client-mediated — workers download + re-upload via Upload API | ✅ Server-side — no intermediary |
| **Data Egress Cost** | ✅ Zero | 🔴 Incurred — download from source + re-upload to new env | ✅ Zero |
| **Transfer Speed (~3 TB)** | ✅ Fast — server-side | 🟡 Tunable via worker concurrency; bounded by API throughput and bandwidth | ✅ Fast — server-side |
| **`public_id` Preservation** | ✅ Confirmed by `cld clone` | ✅ Guaranteed by design | ✅ Confirmed by `cld clone` |
| **Tags + Context Metadata** | ✅ Preserved automatically by `cld clone` | ✅ Explicit per-asset write | ✅ Preserved automatically by `cld clone` |
| **Structured Metadata — Field Definitions** | 🟡 Assumed PS-led; mechanism unconfirmed | ✅ Explicit write during migration | ✅ Self-directed via `GET`/`POST /metadata_fields` |
| **Structured Metadata — Asset Values** | 🟡 Assumed PS-led; mechanism unconfirmed | ✅ Explicit per-asset write | ✅ Self-directed Admin API loop (~33K assets ≈ 7 hrs at 5K req/hr) |
| **Upload Presets / Env Config** | 🟡 Manual reconfiguration required | 🟡 Same requirement | 🟡 Same requirement |
| **Admin API Rate Limit Exposure** | 🟡 `cld clone` internal rate behavior unknown | ✅ Controlled — token-bucket rate limiter | ✅ Controlled — predictable at 33K assets; fits overnight window |
| **Cloudinary PS Dependency** | 🔴 High — PS must deliver structured metadata migration | 🟡 Low — PS scoped to BYOB setup only | 🟡 Low — PS scoped to BYOB setup only |
| **Operational Visibility** | 🔴 Low — CLI black box; PS step opaque | ✅ Full — DynamoDB state, DLQ, CloudWatch, Datadog | 🟡 Medium — metadata loop observable; clone phase is a black box |
| **Per-Asset Retry / DLQ** | 🔴 Not available for clone or PS metadata step | ✅ Native — exponential backoff + SQS DLQ per asset | 🟡 Metadata loop is resumable via tracking table; clone phase not per-asset |
| **Phased / Incremental Execution** | ✅ Clone supports search expressions | ✅ Native — enqueue any subset | ✅ Clone supports search expressions; loop mirrors same phasing |
| **Async Execution** | ✅ Async flag + webhook | ✅ Native | ✅ Async flag + webhook (clone); loop is inherently async |
| **Pilot Batch (100 assets)** | ✅ Folder-scoped clone; PS metadata step harder to scope | ✅ Native — enqueue exactly 100 assets | ✅ Folder-scoped clone + scoped metadata loop |
| **Rollback Capability** | 🟡 Moderate — new env isolated; PS step state unclear | ✅ Strong — DynamoDB snapshot per asset | 🟡 Moderate — new env isolated; metadata loop is resumable |
| **Risk to Live Environment** | ✅ Isolated | ✅ Isolated | ✅ Isolated |
| **Engineering Effort** | ✅ Lowest — primarily operational | 🔴 High — full infra + application build-out | 🟡 Medium — clone is operational; metadata scripts are custom REST |
| **AWS Infra Required** | ✅ Minimal — S3 + IAM for BYOB | 🟡 Moderate — ECS/Batch, SQS, DynamoDB, CloudWatch | ✅ Minimal — S3 + IAM for BYOB (optional lightweight DynamoDB tracking) |
| **URL Continuity (CNAME)** | ✅ Confirmed viable | ✅ Confirmed viable | ✅ Confirmed viable |

**Legend:** ✅ Confirmed favorable &nbsp;|&nbsp; 🟡 Uncertain / conditional &nbsp;|&nbsp; 🔴 Risk or blocker

---

## Quality Attribute Summary (Architecture Lens)

| Quality Attribute | Recommended Option | Rationale |
|---|---|---|
| **Lowest Engineering Effort** | **A** | Server-side clone + operational steps; no infra build-out required — but PS metadata step must be confirmed |
| **Lowest Data Transfer Cost** | **A / C** | Both use server-side clone; zero egress. Option B incurs download + upload bandwidth for ~3 TB |
| **Fastest Transfer** | **A / C** | No client bandwidth constraint; Cloudinary handles binary transfer natively |
| **Reliability / Per-Asset Retry** | **B** | DLQ + backoff designed for partial failures at 3 TB scale |
| **Observability** | **B** | Only option with per-asset state, dashboards, and alerting |
| **Rollback Safety** | **B** | DynamoDB snapshots + isolated environment = safest rollback story |
| **Lowest PS Dependency** | **B / C** | PS scope limited to BYOB setup; migration engine is entirely self-directed |
| **Structured Metadata Control** | **B / C** | Explicit metadata write — no reliance on unconfirmed PS process |
| **Best Balance (effort vs. control)** | **C** | Server-side binaries (zero egress, fast) + self-directed metadata loop (confirmed APIs, predictable at 33K assets) |

---

## Recommendation

### Preferred Path: Option C (`cld clone` + Admin API Loop) — with Option A conditional on PS confirmation

Post Q&A 3 research confirmed that the Cloudinary Admin API provides full programmatic access to structured metadata field definitions (`GET /metadata_fields`, `POST /metadata_fields`) and asset-level values (`GET /resources?metadata=true`, `PUT /resources/:asset_id`). At 33,000 assets, the metadata write loop completes in approximately 7 hours at the current 5,000 req/hr rate limit — a predictable, overnight-window operation.

**Option C is preferred because:**
- Uses the same server-side binary transfer as Option A — zero egress cost, no intermediary machine.
- Eliminates the unconfirmed PS dependency for structured metadata migration. Option A assumes PS has a reliable bulk process; this has not been confirmed and introduces schedule risk.
- Metadata migration is fully self-directed, observable, and resumable by the delivery team.
- Engineering effort is moderate — `cld clone` requires no code; the metadata scripts are straightforward REST against documented endpoints.
- Predictable timeline at 33K assets: binary clone (duration TBD by PS) + 7-hour metadata loop (or under 2 hours with a rate limit increase).

**Option A remains viable if** the PS scoping session confirms that:
1. PS has a defined, time-bounded process for structured metadata migration (not just a recommendation to self-serve).
2. PS commits to executing and validating that step as part of the engagement scope.

If PS cannot confirm both, Option A degrades to Option C automatically — the binary clone path is the same; only the metadata step changes.

**Option B remains the fallback of last resort.** If PS scoping reveals that `cld clone` has unacceptable rate-limit behavior, insufficient error reporting, or cannot be reliably validated at 3 TB, the Worker Fleet provides full programmatic control at the cost of higher engineering effort and egress fees.

### Residual Risks (Options A and C) to Resolve in PS Scoping Session

1. Admin API rate consumption inside `cld clone` internally — unknown; may compete with the metadata loop's 5,000 req/hr budget.
2. S3 key naming convention under BYOB — UTA control is unknown; affects IAM scoping and bucket organization.
3. Behavior of delivery requests during the empty-bucket window of a rolling clone — fallback to old env? 404?
4. Whether async + webhook gives sufficient failure visibility at this scale, or whether a thin wrapper script is needed.

---

## Remaining Open Questions for PS Scoping Session

| # | Question | Relevant Option | Impact if Unresolved |
|---|---|---|---|
| 1 | What Admin API request rate does `cld clone` consume internally per asset at scale? Does it approach or exceed 5,000 req/hr? | A, C | May compete with or throttle the Option C metadata loop; could force Option B |
| 2 | What is the S3 key/path structure Cloudinary writes under BYOB? Is it stable and documented? | All | Required for IAM scoping, bucket organization, and any post-migration S3 tooling |
| 3 | Does UTA have any control over the S3 key naming convention under BYOB? | All | Affects long-term S3 organization strategy |
| 4 | How does `cld clone` report per-folder or per-asset failures in async mode? Is there a log or API to inspect partial failures? | A, C | Determines whether a monitoring wrapper is needed |
| 5 | How should delivery requests for assets not yet cloned be handled during a rolling transfer? (Fallback to old env? 404?) | A, C | Cutover sequencing risk |
| 6 | Can the Admin API rate limit be temporarily raised to 20,000+ req/hr for the migration window? | B, C | Option C: reduces metadata loop from ~7 hrs to under 2 hrs. Option B: determines max throughput. |
| 7 | Does PS have a defined, time-bounded process for structured metadata migration (field definitions + asset values), and will PS commit to executing it as part of the engagement scope? | A | If no, Option A degrades to Option C; affects cutover plan and PS engagement scoping |
| 8 | What is PS engagement scope, estimated hours, and timeline for BYOB configuration + environment setup? | All | Required for project planning and budgeting |

---

## Assumptions Carried Forward

- A new Cloudinary environment is mandatory before any migration activity (confirmed Q&A 3).
- PS configures BYOB backend before any clone or upload operation runs (confirmed Q&A 3).
- `public_id` values will be preserved exactly in the new environment (confirmed Q&A 2).
- CNAME cutover approach is viable for URL continuity (confirmed Q&A 2).
- `cld sync` is eliminated — local filesystem only, not usable for environment-to-environment transfer (confirmed Q&A 3).
- Old Cloudinary environment is retained post-migration as rollback safeguard (confirmed Q&A 2).
- Structured metadata taxonomy requires a separate export/re-import process regardless of option chosen (confirmed Q&A 3); Admin API endpoints for self-directed migration are confirmed (`GET /metadata_fields`, `POST /metadata_fields`, `GET /resources?metadata=true`, `PUT /resources/:asset_id`).
- Upload presets and environment configuration must be independently reconfigured on the new environment (confirmed Q&A 3).
- **Total asset count: ~33,000 assets** (~3 TB). Confirmed from UTA's Cloudinary environment. Used for rate-limit and timeline calculations.
- At 5,000 req/hr Admin API rate limit, a metadata write loop over 33,000 assets completes in approximately 7 hours.
- Encryption at rest (SSE-KMS) is mandatory on the UTA S3 bucket; KMS key policy must permit Cloudinary's IAM principal.
- Migration runs are scheduled outside West Coast peak hours (9 AM – 5 PM PT).

---

## Related Documents

- [v1: Migration Path Options Analysis](./migration-path-options-analysis.md)
- [Cloudinary Q&A 1](../../docs/Cloudinary%20Q%26A%201.md)
- [Cloudinary Q&A 2](../../docs/Cloudinary%20Q%26A%202.md)
- [Cloudinary Q&A 3](../../docs/Cloudinary%20Q%26A%203.md)

