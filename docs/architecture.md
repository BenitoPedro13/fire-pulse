# Architecture — Fire Pulse

> **Status (2026-10-30): skeleton.** This records the decisions from the planning prompt.
> Every provider fact is `[VERIFY]` until `docs/tasks/TASK-source-inventory.md` checks it
> against a live source.

## 1. Problem statement and scope

A live, public map of fires in Brazil: satellite detections joined with wind and air quality,
so a visitor can watch a fire get detected and see what the air did nearby in the hours after.

**In scope:** `[filled in Part C]`

**Out of scope for v1:** `[filled in Part C]`. Already decided: any simulated or synthetic
data; any plume, trajectory or dispersion we compute ourselves and present as observed.

## 2. Coverage and quality tiers

`[filled in Part C]`. Already decided: "no fires detected", "fetch failed" and "no sensor
coverage" are distinct states; observed and modelled data are never confused.

## 3. Sources

The inventory is `docs/research/README.md`. `[filled in Part C]`

## 4. The canonical events

`[filled in Part C, from verified samples]`

## 5. The join

`[filled in Part C]`

## 6. System shape

`[filled in Part C]`

## 7. Hosting and cost sketch

Copies renewable-pulse / transit-pulse: one `t4g.small` Spot instance, Docker Compose, no
NAT/ALB/MSK/RDS, a Budgets alarm on day one, the shared Free-plan account (closes around
2027-03-27). `[filled in Part C]`

## 8. The ending: replay first

A short live run, then a static replay on S3 + CloudFront. `[filled in Part C]`

## 9. Retention

`[filled in Part C]`

## 10. Decisions and open questions

| # | Question | Answer | Status |
|---|---|---|---|
| 1 | Name | `fire-pulse` | **decided** (user, 2026-10-30) |
