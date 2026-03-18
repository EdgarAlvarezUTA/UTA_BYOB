# PRD: Cloudinary → S3 Migration (Enterprise External-Storage)

## Executive Summary
- Vision: allow the customer to have full ownership of source data (assets currently stored in Cloudinary repository) vs exposing to 3rd party (Cloudinary dependency).
- Problem statement: Currently, the assets are stored in Cloudinary's cloud, this makes the customer vulnerable because the assets are the core of the business and they exist outside of its domain.
- Target users: ~300 internal UTA users who interact with Cloudinary indirectly through UTA-owned web applications (ClientProfiles, Submissions, UTASpeakers, Contentful, AuthData, ClientShowcases, UTAWebsite). End users never interact with Cloudinary directly.
- Goal: Provide a concise plan to migrate assets while keeping Cloudinary as a management/proxy layer and using external-storage (S3) as the originals' source-of-truth.


## Success Criteria
- 100% of the legacy assets migrated and still being served as they are now without any negative impact(still being served as they have been). New assets(meaning the type of asset may not exist today) is also being stored and can be served correctly in the new S3. The origin of the files should be in UTA S3 but still respecting the metadata in Cloudinary.

## User Journeys
This is a migration project with minimal user intervention, only for running the scripts. However, the users will continue using Cloudinary as the proxy of the files, now stored in S3, and for asset transformations and CDN.

**Asset Upload Flow (current & post-migration — unchanged from end-user perspective):**
1. A UTA internal user operates one of the UTA web applications: ClientProfiles, Submissions, UTASpeakers, Contentful, AuthData, ClientShowcases, or UTAWebsite.
2. The user uploads an asset (image, video, audio, or PDF) through the web application UI.
3. The web application backend programmatically uploads the asset to Cloudinary via the Cloudinary API/SDK — the end user has no awareness of Cloudinary.
4. Post-migration: the web application flow is identical; Cloudinary receives the upload and stores the original in UTA's S3 bucket (BYOB), returning the same Cloudinary URL as today.

**Asset Delivery Flow (current & post-migration — unchanged):**
1. A consumer (internal or external) requests a media asset via a UTA web property.
2. The web application renders a Cloudinary delivery URL (via CNAME).
3. Cloudinary's CDN serves the asset, fetching from UTA's S3 when not yet cached.
4. The end user receives the asset — the origin (S3 vs. Cloudinary native storage) is fully transparent.

**Stakeholder context:** Cloudinary usage at UTA is described as "rudimentary" — it functions primarily as an asset repository and delivery/CDN layer. No advanced Cloudinary features (e.g., AI tagging, Cloudinary DAM UI workflows, backup) are actively used.

## Functional Requirements
- The client is interested in testing with a small set of data first like the employee's photos. Move them to the new S3 bucket and test if the URL's still work.
- Pay attention to the APIM assumption below, we may need to modify UTA's Azure APIM.

## Non-Functional Requirements

### Asset Characteristics
- Asset types: **images, videos, audio (music), and PDFs**.
- Maximum individual asset size: **5 GB** (videos).
- Total estimated data volume: ~3 TB.

### Users & Traffic
- Internal user count: **~300 users** accessing assets via UTA web applications.
- Peak traffic window: **West Coast business hours, 9:00 AM – 5:00 PM PT**. Migration and any disruptive operations should be scheduled outside this window.

### Storage & Redundancy
- UTA assumes full responsibility for storage redundancy post-migration (Cloudinary's backup and disaster recovery is no longer in scope).
- S3 must have **cross-region geo-redundancy**: primary bucket in one AWS region with replication to a secondary region (S3 Cross-Region Replication or S3 Multi-Region Access Points).
- S3 versioning should be evaluated as part of the redundancy strategy.

### Security
- **Encryption at rest is mandatory.** S3 objects must be encrypted using SSE-S3 or SSE-KMS. KMS compatibility with Cloudinary's BYOB IAM access must be confirmed with Cloudinary PS.
- Least-privilege IAM roles for all migration workers and for Cloudinary's cross-account access.
- There's no GDPR, the images are public.

### Cloudinary API
- **Admin API rate limit: 5,000 requests/hour per environment** (confirmed via Q&A 2 — supersedes earlier assumption of "no hard cap"). Temporary increases can be requested through Cloudinary support for the migration period.
- Worker concurrency should still be capped as a cost guardrail on the AWS side.
- Throughput target: 2,000+ assets/hour achievable with auto-scaling worker pool; exact target to be confirmed with client.

### Migration Scheduling
- Migration time: TBD — to be defined with client based on business urgency and acceptable maintenance window.
- Bulk transfer operations should be scheduled **outside peak hours (before 9 AM PT or after 5 PM PT)** to avoid impact on live delivery traffic.

### Observability
- UTA has Datadog, we must integrate the UTA S3 bucket metrics with it.

## Assumptions
- Customer is on **Cloudinary Enterprise Plus & Digital Asset Management (DAM) Plan** with Cloudinary-managed storage; external-storage (S3) will be enabled as part of this migration.
- Admin API limit is **5,000 requests/hour per environment** (confirmed via Cloudinary Q&A 2 — the earlier assumption of "no hard cap" is superseded). Temporary increases for the migration window can be arranged via Cloudinary support.
- ~3TB of binary assets spanning images, videos (up to 5 GB each), audio files, and PDFs.
- **UTA does not use the Cloudinary backup feature.** No backup data exists to migrate or preserve; rollback relies on keeping the old Cloudinary environment live during the validation window.
- UTA users never interact with Cloudinary directly; all uploads flow through UTA-owned web applications.
- Cloudinary usage is "rudimentary" (stakeholder-confirmed): Cloudinary functions as an asset repository and CDN delivery layer only — no advanced DAM workflows, AI features, or Cloudinary-native backup are in active use.
- UTA useas an Azure APIM to mask Cloudinary URLs. So for instance, a Cloudinary URL like this https://res.cloudinary.com/uta-media-cloud/image/upload/c_fill,ar_16:9,g_auto,h_240,f_auto,q_auto/v1770481284/BAD_BUNNY_ixx5i0.png is masked via APIM in something like this https://clientshowcase.unitedtalent.com/_ipx/w_1080,q_75/https%3A%2F%2Fdca.unitedtalent.com%2Fimage%2Fupload%2Fw_501%2Ch_473%2Ct_Instagram%2520feed%2Cq_100%2Fv1697603662%2Fakon.jpg?url=https%3A%2F%2Fdca.unitedtalent.com%2Fimage%2Fupload%2Fw_501%2Ch_473%2Ct_Instagram%2520feed%2Cq_100%2Fv1697603662%2Fakon.jpg&w=1080&q=75. UTA calls it the DCA, as you can see in the domain. However, in one of our tests we saw that one UTA's application was using the unmasked URL, which is a technical debt the client will address.
- Cloudinary recommends creating a "new environment" for the migration. We are not sure what that implies but we agree we must not work in live environment.

## High-level Architecture Decisions
- Storage: Customer S3 is canonical; Cloudinary stores metadata and serves as management/proxy.
- S3 Redundancy: Cross-Region Replication (CRR) configured to replicate all objects to a secondary AWS region. Primary region TBD with client; secondary region should be geographically distinct (e.g., us-east-1 primary → us-west-2 secondary, or equivalent).
- Encryption: All S3 objects encrypted at rest (SSE-KMS preferred; SSE-S3 as fallback depending on Cloudinary IAM compatibility). KMS key policy must grant Cloudinary's cross-account IAM principal decrypt/encrypt access.
- Orchestration: SQS queue for tasks; worker fleet via AWS Batch or ECS Fargate.
- State & Idempotency: DynamoDB tracking per-asset state and attempt counts.
- Concurrency & Rate-Limit Handling: Auto-scaling worker pool; token-bucket or rate-limiter set to stay within 5,000 Admin API calls/hr (with headroom). Exponential backoff for transient errors + DLQ for dead letters. Temporary Admin API limit increase to be requested from Cloudinary prior to bulk migration runs.
- Observability: CloudWatch metrics + dashboards; alerts for DLQ, worker errors, auto-scaling cost thresholds, and Admin API rate limit consumption. Migration runs scheduled outside West Coast peak hours (9 AM–5 PM PT).

## Components
- Controller service: enqueues asset migration tasks (Python service).
- Workers: stateless workers that perform download/verify/upload/update operations.
- DynamoDB table: asset_key → {status, attempts, last_error, cloudinary_id, s3_etag}
- SQS queues: main queue + DLQ.
- Baseline test harness: small script to probe API headers and latency.

## Worker Flow
1. Claim message from SQS.
2. Mark `in-progress` in DynamoDB with lease TTL.
3. If Cloudinary origin already points to customer S3, verify object presence/ETag; mark complete if valid.
4. Else, download asset (or copy via provider API if supported), upload to S3 with proper ACLs and metadata.
5. Update Cloudinary metadata to external origin and verify transformation URLs.
6. Mark complete; emit metric.
7. On error: increment attempts, apply backoff, requeue or send to DLQ after max attempts.

## Testing & Verification
- Run baseline API probe to capture real limits and headers.
- Start with a pilot batch (e.g., 100 assets) at low concurrency.
- Validate sample transformation URLs and end-to-end delivery via CloudFront after mapping.
- The client does not have a QA or Staging environment for Cloudinary. The tests will be executed in **Production**. The plan is to create testing assets ourselves and run the migration on those without affecting live assets.
- There is a sandbox for AWS.

## Rollback Strategy
- Keep previous Cloudinary mapping until verification window closes.
- For assets that fail post-migration, restore prior metadata from DynamoDB snapshots.

## Security & Compliance
- **Encryption at rest is mandatory.** All S3 objects must be encrypted (SSE-KMS preferred). KMS key policy must permit Cloudinary's IAM principal to read/write encrypted objects. Confirm KMS support with Cloudinary PS before bucket creation.
- S3 bucket must be **private** (no public access); assets are served exclusively through Cloudinary's CDN layer.
- Maintain least-privilege IAM roles for migration workers and for Cloudinary's cross-account S3 access.
- S3 bucket policies must restrict access to: (a) migration worker role, (b) Cloudinary's IAM principal (cross-account), and (c) replication role for CRR.
- Enable S3 server access logging and CloudTrail data events on the bucket for audit trail of all object reads/writes.

## Timeline (high-level)
- Day 0–2: Run baseline probes (API latency, throughput), spin up infra skeleton. *(Rate limit confirmation complete — Enterprise Plus & DAM validated 2026-03-03)*
- Day 3–7: Pilot migration (100–1000 assets), tune worker concurrency for cost/throughput balance.
- Week 2–4: Scale batches, monitor costs and error rates, complete migration.

## Next Steps
1. ~~Confirm Admin API limits with Cloudinary rep~~ ✅ **Done — Enterprise Plus & DAM Plan confirmed (2026-03-03).**
2. Confirm target throughput and acceptable AWS cost ceiling with client (drives max worker count tuning).
3. Create baseline probe script and run against account to measure real API latency and throughput.
4. Implement controller + one worker prototype and run pilot (100 assets).
5. Review metrics, tune worker concurrency, and scale.

