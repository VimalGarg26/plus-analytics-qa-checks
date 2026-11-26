# Plus Analytics QA Checks

**Status:** Draft for analytics / eng alignment  
**Owner (PM):** Vimal Garg  
**Date:** 2026-11-26  
**Scope:** HF+ and Factor Plus (shared two-layer model). Cut every check by **brand**, **monthly vs annual**, and exclude `is_qa` / internal.  
**Related:** [plus-analytics-requirements-roadmap-2026-10-26.md](plus-analytics-requirements-roadmap-2026-10-26.md) · [Factor-Plus-Cancellation-Scenarios.md](Factor-Plus-Cancellation-Scenarios.md) · factor-plus-drive event dictionary

---

## How to use

| Column | Meaning |
|---|---|
| **ID** | Stable check id |
| **Assert** | Pass condition |
| **Fail if** | Common break pattern |
| **Primary signals** | Events / ledger fields to join |

**Event spine (canonical):**  
`ACTIVATED` → benefit issue / redeem → (`RENEWED` \| `DEACTIVATION_REQUESTED` → `DEACTIVATED`) → optional `REACTIVATED` → payment success on later `RENEWED`  
Cooling-off refund = deactivate with `DEACTIVATION_REASON_REFUNDED` (exclude from net adoption & membership revenue).

---

## 1. Pricing — right charge by plan

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| P1 | Activate price — monthly | Charge on `ACTIVATED` = current monthly offer (intro or list) × tax rules | Wrong amount, list charged during intro window, or intro forever | `amount_paid`, cadence = 28, offer/campaign id |
| P2 | Activate price — annual | Charge on `ACTIVATED` = current annual offer (intro or list) × tax rules | Monthly price on annual cadence (or reverse) | `amount_paid`, cadence = 365 |
| P3 | Cadence ↔ plan | `cadence_days` matches selected plan (28 / 365) | Cadence/plan mismatch | `renewal_context.cadence_days`, plan id |
| P4 | Intro → list on renew | First `RENEWED` (post-intro) bills list price for that cadence | Still intro after intro window; wrong list | `RENEWED.amount_paid`, offer expiry |
| P5 | No double charge at join | One successful membership charge per activation | Duplicate `ACTIVATED` / twin charges same window | subscription_id, charge ledger |
| P6 | Failed join payment | Payment fail → no ACTIVE membership, no benefits issued | Benefits live with $0 paid / pending forever | payment status, benefit-pass state |

---

## 2. Benefits — attach and redeem

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| B1 | Attach on activate | Correct benefit package issued at `ACTIVATED` (FS / surcharge credit per live campaign) | Missing vouchers, wrong package for arm/campaign | benefit-pass → wallet issue |
| B2 | Attach on renew | New benefit cycle issued on successful `RENEWED` | Stale cycle, double-issue, or none | renew event ↔ voucher issue |
| B3 | Redeem eligibility | Benefits redeem only while membership status ∈ ACTIVATED / RENEWED / REACTIVATED (not DEACTIVATED post-cutoff) | Redeem after hard stop | member status at order create / delivery |
| B4 | Redeem amount | $ saved (FS + surcharge) ≤ entitlement for that box / week | Over-redeem | order discount lines vs entitlement |
| B5 | Package fidelity | Redeemed benefit types match attached package (e.g. FS-only arm never gets premium credit) | Cross-package leakage | campaign / experiment arm × discount type |
| B6 | Member-order definition | Member box = latest benefit-pass event before delivery ∈ ACTIVATED/RENEWED/REACTIVATED | DEACTIVATED counted as member box | benefit-pass event before delivery |

---

## 3. Meal cancel + reactivation (Plus still in play)

Plus and meal are independent. These checks cover meal lifecycle while Plus exists.

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| M1 | Meal cancel does not kill Plus mid-period | Plus stays ACTIVE through current Plus billing period | Plus hard-cancelled same moment as meal cancel | meal cancel ts vs Plus status |
| M2 | Benefits ride to EOP after meal cancel | Benefits usable until Plus period end (even with no meal orders) | Benefits yanked on meal cancel day | benefit status vs Plus `next_renewal` |
| M3 | Plus renewal auto-suppress | If meal still inactive at Plus renewal → **no** Plus renew charge; membership ends after period | Plus renews with no active meal sub | meal sub state at `next_renewal_time` |
| M4 | Disclosure / state consistency | System state matches “two subscriptions” model (analytics: independent cancel flags) | Meal cancel flips Plus to refunded incorrectly | cancel reason codes |
| M5 | Meal reactivate inside Plus period | Meal returns while Plus still in period → benefits usable on new orders again | Benefits stuck off after meal return | meal reactivate ts × benefit redeem |
| M6 | Meal reactivate after Plus EOP (renew suppressed) | No automatic Plus; customer must re-join Plus to get benefits | Ghost Plus / benefits without new charge | Plus status after meal return |

---

## 4. Plus cancel ≤48h (cooling-off refund)

Customer-facing rule: full refund within 48h **if benefits unused**.

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| C1 | Window | Cancel ≤ activation + 48h | Refund path applied outside window | activate ts, cancel ts |
| C2 | Unused gate | No FS / surcharge / wallet benefit consumed before cancel | Refund with used benefits (see C8) | redeem events before cancel |
| C3 | Full refund | Membership fee fully refunded per Finance rules | Partial / missing / wrong amount | refund ledger vs `amount_paid` |
| C4 | Reason code | `DEACTIVATION_REASON_REFUNDED` (or brand equivalent) | Generic cancel reason on cooling-off | deactivation reason |
| C5 | Benefits removed immediately | Benefits not redeemable after refund cancel | Benefits live post-refund | post-cancel redeem attempt / flag |
| C6 | No renewal | No `RENEWED` / next membership charge | Charge after refund | renew / charge after deactivate |
| C7 | Reporting exclude | Excluded from net adoption & membership revenue | Refunded counted in net KPIs | adoption / revenue defs |
| C8 | Used inside 48h (control) | Any benefit used → **no full refund** (or policy partial only); membership not left ACTIVE unpaid | Full refund after use; or open membership | redeem + refund flags |

---

## 5. Plus cancel >48h (cancel-at-period-end)

No mid-period refund under current policy. Benefits live to end of cycle; no renewal.

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| D1 | Outside cooling-off | Cancel > activation + 48h **or** benefits already used | Incorrectly routed to refund path | activate / cancel / redeem |
| D2 | No mid-period refund | No membership refund on cancel (annual proration = separate policy flag if shipped) | Unexpected refund | refund ledger |
| D3 | Benefits stay live | Benefits redeemable until end of current billing period | Benefits die on cancel day | redeem between cancel and EOP |
| D4 | No renewal charge | No charge at `next_renewal_time` | Renewal still bills | charge at/after supposed renew |
| D5 | Benefits cut at EOP | After period end → not redeemable; membership `DEACTIVATED` | Benefits survive past EOP | status + redeem after EOP |
| D6 | Meal independence | Meal plan remains active unless customer cancels meals | Meal auto-cancelled with Plus | meal sub status |
| D7 | Monthly vs annual EOP | 28d cycle ends ~day 28; annual ~day 365 from period start | Wrong EOP by cadence | cadence × period end |

---

## 6. Healthy renewal (no cancel)

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| R1 | On-time renew | Eligible ACTIVE members charge successfully at supposed renewal | Silent non-renew without cancel/dunning | supposed vs actual renew |
| R2 | Renew amount | Correct post-intro / list amount for cadence | Wrong price on cycle 2+ | `RENEWED.amount_paid` |
| R3 | Benefits re-issued | New cycle benefits attached on renew | No new vouchers | renew ↔ issue |
| R4 | Multi-cycle funnel | Cycle n → n+1 renewal rate computable by plan | Broken subscription_id sequencing | cycle_number join |

---

## 7. Payment failure on Plus renewal

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| F1 | Dunning window | Retries per product design (e.g. 3 / 24h + pay-now) | Instant lapse with no retry; or infinite retry | payment fail events |
| F2 | Lapse clean | After failed retries → membership inactive | Stuck PENDING / ACTIVE unpaid | final status |
| F3 | Benefits stop | Benefits cut when membership lapses | Benefits after failed pay window | redeem vs lapse ts |
| F4 | No charge without benefits | Never: charged and benefits denied in same failed window | Charge + deny | charge + benefit flags |

---

## 8. Plus reactivation

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| X1 | In-cycle reverse cancel | `REACTIVATED` + `billing_cycle_reactivation=true` → **no new charge**; benefits continue on current period | Extra charge on reverse | reactivate flag + ledger |
| X2 | Re-subscribe after inactive | `REACTIVATED` with `billing_cycle_reactivation=false` → payment success on later `RENEWED`, not on `REACTIVATED` | Counting reactivate as paid | event order |
| X3 | Benefits on re-subscribe | After payment success, benefits re-attached for new cycle | ACTIVE paid with empty wallet | `RENEWED` ↔ issue |
| X4 | Do not treat DEACTIVATION_REQUESTED as inactive | Membership ACTIVE until `DEACTIVATED` for renewal metrics | Understated renew eligible base | status rules |

---

## 9. Plan switch (monthly ↔ annual)

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| S1 | Price after switch | Next charge matches new plan offer | Old price on new cadence | plan change + amount |
| S2 | Next renewal date | `next_renewal` updates to new cadence | Old 28d tick on annual (or reverse) | renewal_time |
| S3 | Benefit cadence | Benefit issue rhythm matches new plan | Orphan / double vouchers across switch | issue timeline |
| S4 | No double bill | One charge covering switch window per policy | Overlap double charge | ledger |

---

## 10. Skip / pause while Plus billed (monitor)

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| K1 | Plus still bills on skip streak | Expected under current policy if meal sub active | N/A as hard fail — **monitor** | skips × Plus charges |
| K2 | Utilization = 0 while paying | Flag members with N consecutive skipped weeks + Plus ACTIVE | Silent long unpaid-value streaks | skip weeks + fee |
| K3 | Nudge / CRM hygiene (if shipped) | Alert/nudge fires per product rule | Rule defined but never fires | CRM × cohort |

---

## 11. Enrollment / instrumentation hygiene

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| H1 | QA exclusion | All KPI queries exclude `is_qa` / internal | QA inflates adoption/revenue | qa flags |
| H2 | Single live subscription | ≤1 ACTIVE Plus subscription per customer (unless product explicitly allows) | Twin ACTIVE subs | subscription_id count |
| H3 | Funnel payment-fail visibility | Join funnel records payment fail as drop-off | Silent fail looks like abandon | `fp_*` / enroll events |
| H4 | Cooling-off in finance file | Refunded fee appears in weekly Finance refund / fee cash feed | Refund missing from FP&A | Finance weekly file |

---

## 12. Free trial (only if trial is live / in scope)

| ID | Check | Assert | Fail if | Primary signals |
|---|---|---|---|---|
| T1 | Mid-trial Plus cancel | No charge; benefits per trial policy (e.g. to D28 or immediate stop) | Surprise charge | trial flag + ledger |
| T2 | Meal cancel during trial | Plus trial ends; no charge (if that policy ships) | Plus converts paid after meal cancel | meal cancel × Plus |
| T3 | Trial → paid convert | Charge on convert date at disclosed price unless cancelled | Silent convert / wrong price | convert event |
| T4 | Pre-convert reminder | Reminder sent per design (if instrumented) | Missing reminder cohort | CRM event |

---

## Priority for first QA pass

Run in this order (highest break / legal risk first):

1. **C1–C8** — 48h refund path  
2. **D1–D7** — post-48h cancel + EOP cut  
3. **M1–M6** — meal cancel × Plus  
4. **P1–P6** + **B1–B6** — price & benefits  
5. **R1–R4** + **F1–F4** — renew / dunning  
6. **X1–X4** — Plus reactivate  
7. **S1–S4**, **K1–K3**, **H1–H4**  
8. **T\*** only if trial live  

---

## Open policy flags (do not soft-pass)

| Flag | Impact on checks |
|---|---|
| Annual mid-term **proration** | If shipped, D2 gains an allowed refund path; add amount = unused months rule |
| 48h **unused** definition | Confirm: any redeem vs any shipped box vs wallet open |
| Meal-cancel **auto-suppress renew** | M3 assumes business-preferred policy; confirm HF+ US vs FR (FR: Plus ends EOP, no forever) |
| Free trial mid-cancel benefit end | T1 timing (immediate vs D28) must match live T&Cs |

---

## Suggested next step

Turn IDs into a query checklist (one SQL / notebook cell per ID) using factor-plus-drive patterns, then wire a weekly fail-count monitor for C/D/M/P/B.
