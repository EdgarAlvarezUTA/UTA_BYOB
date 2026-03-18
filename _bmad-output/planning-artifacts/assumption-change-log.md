# Assumption Change Log

## Change #1: Cloudinary Rate Limit Re-validation

**Date Discovered:** 2026-02-23  
**Date Validated:** 2026-03-03  
**Status:** ✅ VALIDATED  
**Severity:** HIGH (affects architecture decisions)

### Original Assumption

```
Cloudinary Admin API: 500 calls/hour limit (standard tier)
→ Justifies fixed concurrency, deferred updates, three-tier retry
```

### New Finding (Validated 2026-03-03)

```
Customer is on Cloudinary Enterprise Plus & Digital Asset Management Plan
→ Admin API rate limits are significantly elevated (Enterprise Plus removes standard 500/hr cap)
→ Eliminates hard concurrency constraints; worker count becomes a cost/throughput tuning parameter
→ Simplifies architecture: no token-bucket controller needed; auto-scaling pool replaces fixed pool
```

### Change Cascade

#### 1. **Brainstorming Impact**
- Document: `_bmad-output/brainstorming/brainstorming-session-2025-02-20.md`
- Sections to update:
  - `Adapt #1 - Controlling Concurrency` → **Remove rate limit constraint**
  - `Modify #1 - Asset Update Timing` → **No longer need deferred batch**
  - `Substitute #3 - Retry Strategy` → **Simplify to 2-tier**

#### 2. **Architecture Impact**
- **Current:** Database + Queue + fixed pool + DLQ + deferred update process (4 components)
- **New:** Database + Queue + auto-scaling pool + DLQ (3 components, simpler)
- **Benefit:** Faster throughput; fewer moving parts; easier to operate

#### 3. **PRD Impact**
- **Throughput Section:** "800 assets/hour (limited by Cloudinary) →  2000+ assets/hour (auto-scaling)"
- **Architecture Diagram:** Remove "Deferred Updates Service" box
- **Risk Section:** Remove "Rate limit exhaustion" → Add "Auto-scaling cost management"

#### 4. **Implementation Impact**
- Worker concurrency is now a **tuning parameter**, not a hard constraint
- Can measure actual throughput and adjust dynamically
- Removes need for separate "deferred batch" cron job

### Decision Tree (Phase 4) Changes

**Old Decision Tree Branch:**
```
Q: "What's your rate limit tolerance?"
→ A: "Cloudinary has limits"
→ Decision: Fixed 5-10 workers, deferred updates
```

**New Decision Tree Branch:**
```
Q: "Can you scale worker count dynamically?"
→ A: "Yes, no Cloudinary limits"
→ Decision: Auto-scaling 50-500 workers, immediate updates
```

### Propagation Checklist

- [ ] Update assumption in brainstorming document
- [ ] Re-run Adapt #1 SCAMPER lens (with no-limit assumption)
- [ ] Re-run Modify #1 SCAMPER lens (simplification)
- [ ] Re-run Substitute #3 SCAMPER lens (2-tier retry suffices)
- [ ] Update Decision Tree (Phase 4) with new branches
- [x] Update PRD throughput, architecture, risk sections
- [x] Validate with client: "Is there truly no rate limit?" → **Confirmed: Enterprise Plus & DAM Plan (2026-03-03)**
- [ ] Update implementation plan if approved

### Validation Result (2026-03-03)

**VALIDATED by Edgar on 2026-03-03:**
- Client confirmed: **Cloudinary Enterprise Plus & Digital Asset Management Plan**
- Enterprise Plus removes the standard 500/hr Admin API cap; no hard rate limit applies
- Remaining open items before implementation:
  1. ~~Confirm with Cloudinary: exact rate limit for customer's plan~~ ✅ **Done — Enterprise Plus & DAM confirmed**
  2. Ask customer: "What's your target throughput?" (drives worker count tuning)
  3. Measure baseline: run a small pilot batch and observe actual API call rate
  4. Set cost guardrails: cap max workers (e.g., 500) to manage AWS Batch/Fargate costs even without Cloudinary limits

---

## Template for Future Assumption Changes

When a new assumption changes:

1. **Document the old assumption**
2. **Document the new finding**
3. **Trace impact zones** (which decisions depend on this?)
4. **Update brainstorming** (re-run SCAMPER lenses)
5. **Update Decision Tree** (re-run Phase 4)
6. **Update PRD** (cascade to all affected sections)
7. **Get client validation** before re-implementing
