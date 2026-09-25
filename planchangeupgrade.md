~~~~# Analysis Document: UPGRADE & PLAN CHANGE — V1 → V3 Parity Assessment

## 1. Purpose

V1 was built for a single country/program. V3 generalizes the platform (V1 + V2 capabilities merged) to support multiple countries and business lines (`businessContext`: PaaS, LFP, WXP, etc.) via a flexible payload. This document assesses UPGRADE and PLAN CHANGE functionality in V1, maps it to the new V3 payload, and identifies gaps that must be closed so V3 has **functional parity with V1**.

---

## 2. UPGRADE — Current State (V1) vs Target State (V3)

### 2.1 V1 Behavior Summary

| Capability | V1 Implementation |
|---|---|
| Trigger | `POST /orders` with `customerOrderType=UPGRADE` |
| Old→New linkage | `productInfo.additionalItemInfo[parentOrderId]` on `onbooks_printer` item |
| Mandatory item | `returnShipmentFeeSku` must be present |
| Old sub validation | Base subscription for `parentOrderId` must exist and be `ACTIVE` |
| Overage transition pricing | Lowest of old vs new `allocatedRevenue` used for `pageSet` in month 1 |
| Hardware proration | `UpgradeHWFirstMonthProratedCharges = (leaseAllocatedRevenue / daysInMonth) × activeBillingDays` |
| Month-2 correction | Scheduled plan change (SCC) triggered if new overage > old overage |
| Old subscription cleanup | Undo overages (BMS), undo pending changes (SCC), set status → `DEACTIVATING` |
| Idempotency | `magentoOrderId` |

### 2.2 V3 Current Implementation Status

| Capability | V3 Status |
|---|---|
| Trigger | ✅ `POST /v3/orders` with `orderType=UPGRADE` |
| Old→New linkage | ✅ `item.additionalItemInfo[parentOrderId]` (Map, not List like V1) |
| Mandatory item check | ✅ `returnShipmentFeeSku` validated (`OrderRequestValidatorForPaaS.validateUpgradeItems`) |
| Old sub validation | ✅ Implemented — checks `SubscriptionDaoV3` by `orderReferenceId` + `BASE` type + `ACTIVE` status |
| Overage transition pricing (lowest-of) | ❌ **Not implemented** — no equivalent of `computeAllocatedRevenue` for `pageSet` |
| Hardware proration | ✅ Implemented (`calculateAndSetUpgradeHWFirstMonthProratedCharges`) — value stored in `pricing.upgradeHWFirstMonthProratedCharges` |
| Month-2 scheduled correction | ❌ **Not implemented** |
| Old subscription cleanup (undo overages/changes, DEACTIVATING) | ❌ **Not implemented** — `processAcceptedUpgradeOrder` equivalent missing |
| Idempotency | ✅ `orderReferenceId` |

### 2.3 Field Mapping: V1 → V3 (Upgrade-relevant fields)

| V1 Field | V3 Field | Notes |
|---|---|---|
| `magentoOrderId` | `orderReferenceId` | Now alphanumeric, cross-program key |
| `customerOrderType` | `orderType` | Same enum values |
| `productInfo.additionalItemInfo` (List<AdditionalItemInfo>) | `item.additionalItemInfo` (Map<String,Object>) | Structure simplified list→map |
| `parentOrderId` (in additionalItemInfo list) | `parentOrderId` (in additionalItemInfo map, **and** duplicated at `metadata.parentOrderId`) | V3 has it at both item and order level — needs single source of truth decision |
| `productInfo.allocatedRevenue` | `item.pricing.adjustments[type=ALLOCATED_REVENUE].amount` | Moved from flat field to adjustments array |
| `chargeInfo.upgradeHWFirstMonthProratedCharges` | `pricing.upgradeHWFirstMonthProratedCharges` | Same concept, new location (item-level `PricingV3`) |
| N/A (no businessContext) | `metadata.businessContext` | **New — required.** Drives country/program-specific validator (`OrderRequestValidatorForPaaS` etc.) |
| `country` (implicit, single-market) | `country` (ISO alpha-2, explicit, required) | V3 must support any HP market, not just one |

### 2.4 Gaps to Close for V3 Upgrade Parity

1. **Overage lowest-price logic** — Port `computeAllocatedRevenue()` logic into `OrderServiceV3Impl`, keyed off `pageSet`/`PAGE_SET` product type and `parentOrderId`, reading from `SubscriptionDaoV3Repository`.
2. **Post-Gekko old-subscription handling** — Port `processAcceptedUpgradeOrder()`: find old renewed subs by `parentOrderId`, undo BMS overages, undo SCC pending changes, set status `DEACTIVATING`.
3. **Month-2 scheduled plan change** — Port SCC trigger logic (`PUT /subscriptions/{id}/update/overage`) for when new overage > old overage.
4. **Single source of truth for `parentOrderId`** — Decide item-level vs metadata-level (currently both exist in V3 payload); align validator to use one location consistently across programs.
5. **Multi-country validation** — V1 rules were hardcoded for one market; in V3 these rules must run per `businessContext`/`country` combination (matrix-driven, similar to existing `CountryFeatureMatrix` pattern already used for GEO fee injection).

---

## 3. PLAN CHANGE — Current State (V1 + V2) vs Target State (V3)

> **Important finding:** Plan change is **not a single flow** — V1 and V2 each implemented a different variant. V3 must merge both, not just port V1.

### 3.1 V1 Behavior Summary — Full Pricing Plan Change

| Capability | V1 Implementation |
|---|---|
| Trigger | `PUT /subscriptions/{id}/update?idType=SUBSCRIPTIONID` |
| Customer key | Flat `tenantId` field |
| Change scope | Full pricing change: `productId`, `price`, `fmv`, `allocatedRevenue` (plan/SKU swap) |
| Policy modes | `IMM` (immediate) / `SOT` (start-of-term, uses BMS billing cycle date) |
| Concurrency control | `ChangeRequestDao` queue: states `PENDING → PROCESSING → CONFIRMED` / `OVERRIDDEN` / `VALIDATION_ERROR`, keyed by `tenantId` |
| Duplicate detection | Same `subscriptionContractId` + `groupId` + `subscriptionType` with status `PENDING`/`PROCESSING` → `409 Conflict` |
| Multi-item rule | Multiple items only allowed if the target subscription is `BASE` type |
| Validation | Sub not `DISABLED`; productId exists in PIM; SKU characteristic match; all subs belong to same order |
| Downstream integration | **Direct Gekko call** — `subscriptionChangePlan()` → on 202, save audit (`GekkoApiEventsAuditDao`), trigger Error Handler workflow, publish event, undo overages (BMS) |
| Confirmation mechanism | Gekko `SUBSCRIPTION_PLAN_CHANGE` **event** (push, via Error Handler) → `EventHandlerService.processPlanChangeEvent()` confirms `ChangeRequestDao`, then processes next queued `PENDING` request (oldest-first, others marked `OVERRIDDEN`) |
| De-duplication | If price/fmv/allocatedRevenue/productId identical to current sub → treated as duplicate, skipped |

### 3.2 V2 Behavior Summary — Quantity-Only Plan Change (`updateSubscriptionQuantity`)

V2 introduced a **narrower, parallel** endpoint on the same path (`PUT /v2/subscriptions/{id}/update`) that only supports seat/quantity changes — pricing changes are explicitly rejected for Monetization subscriptions.

| Capability | V2 Implementation |
|---|---|
| Trigger | `PUT /v2/subscriptions/{id}/update?idType=SUBSCRIPTIONID` |
| Customer key | `primaryCustomerId` + `primaryCustomerIdType` (orgId / userId / tenantId) — **already flexible**, not tenant-only |
| Change scope | `sku` + `quantity` only. Submitting a pricing block is a validation error for Monetization subs |
| Policy modes | `EOT` (end-of-term) / others routed like V1's IMM/SOT via `delayedRequestForChangeQuantity()` |
| Concurrency control | Same `ChangeRequestDao` mechanism, but queried via `findByPrimaryCustomerIdAndChangeTypeAndStatusInAndSubscriptionContractId` — proves the DAO already supports flexible customer-id lookups, not just `tenantId` |
| Schedule conflict checks | Reuses V1-style guards: blocks if `CHANGE_PLAN`/`CANCELLATION` already pending/processing for the subscription or its items |
| Downstream integration | **Different from V1** — EOT path calls **SCC** `updateSubscription()` (not Gekko directly), with `effectiveDate` = next billing cycle start date from BMS |
| Confirmation mechanism | **Webhook**, not Gekko event — `POST /webhooks/scc/monetization-change-quantity` → `handleSccMonetizationChangePlanQuantityEvent()` updates `SubscriptionDao.quantity` and logs to `GekkoApiEventsAuditDao` |
| Gekko idType handling | Maps `SUBSCRIPTIONID` → `GEKKO` idType before calling downstream, since Gekko doesn't understand SMS-internal `SUBSCRIPTIONID` |

### 3.3 V3 Current Implementation Status

| Capability | V3 Status |
|---|---|
| Endpoint | ❌ **Not implemented** — `SubscriptionV3RestController` only exposes `GET` (list/get), `PUT /{id}/cancel`, `PUT /{id}/undoCancel`. **No plan-change endpoint exists (neither pricing nor quantity variant).** |
| Everything else (policy routing, queueing, dedup, Gekko/SCC integration, event/webhook confirmation) | ❌ **Not started** |

### 3.4 Field Mapping: V1 + V2 → V3 (Plan-Change relevant, projected)

V3 must support **both** change types (pricing + quantity) under one flexible model, following the customer-identifier pattern V2 already proved out:

| V1 Field | V2 Field | V3 Equivalent (proposed) | Notes |
|---|---|---|---|
| `policy` (IMM/SOT) | `policy` (EOT/IMM/SOT) | Keep as-is | No change needed |
| `tenantId` (flat) | `primaryCustomerId` + `primaryCustomerIdType` | `customer.primaryCustomerId` + `customer.primaryCustomerIdType` (mirrors `CustomerV2`/`SubscriptionCancelRequestV3` pattern already used in V3) | **V3 should adopt V2's pattern**, not V1's flat `tenantId` |
| `items[].productId`, `price`, `fmv`, `allocatedRevenue` | `items[].sku`, `quantity` | Both variants needed: `items[].sku` + `quantity` (V2-style) **and** `items[].pricing.adjustments[]` (V1-style, PaaS/hardware programs) — selection driven by `businessContext` | **Key decision point** — one unified item schema supporting both change types, gated by program rules |
| N/A | Monetization blocks pricing changes | Generalize as a per-`businessContext` rule matrix: which change types (`QUANTITY`, `PRICING`, `BOTH`) are permitted per program | Avoid hardcoding "Monetization" — use the same strategy pattern as `OrderRequestValidatorForPaaS` |
| Gekko direct call + event confirmation | SCC call + webhook confirmation | V3 must support **both integration paths**, selected per program/product type (hardware/lease-based → Gekko path; seat/Monetization-based → SCC+webhook path) | Do not assume single downstream system |

### 3.5 Gaps to Close for V3 Plan Change Parity

1. **Build the endpoint** — `PUT /v3/subscriptions/{id}/update` in `SubscriptionV3RestController`, backed by a new `SubscriptionServiceV3Impl.updateSubscriptionPlan()`.
2. **Unified item schema** — Support both quantity-only (V2) and full pricing (V1) changes in one V3 request/item model, using `PricingV3.adjustments[]` for the pricing case.
3. **Adopt V2's customer-identifier pattern** — Use `primaryCustomerId` + `primaryCustomerIdType`, not V1's flat `tenantId`; `ChangeRequestDao` already supports this lookup.
4. **Port concurrency queue** — Reuse `ChangeRequestDao` state machine (`PENDING → PROCESSING → CONFIRMED/OVERRIDDEN/VALIDATION_ERROR`) against `SubscriptionDaoV3`.
5. **Port IMM/SOT/EOT effective-date logic** — Reuse BMS `getBillingCycleDate()`, same as V1/V2.
6. **Port dedup logic** — `validateDeDuplicationChargeInfoItemsWithPreviousRequest()` equivalent, adjustments-array aware.
7. **Support dual downstream integration** — Gekko-direct (V1 path, for pricing/plan-swap changes) **and** SCC+webhook (V2 path, for quantity changes) — routed by change type/program.
8. **Port event + webhook confirmation flows** — Extend `EventHandlerService.processPlanChangeEvent()` for Gekko events **and** add a V3-aware `handleSccMonetizationChangePlanQuantityEvent()` equivalent, both resolving against `SubscriptionDaoV3`.
9. **Multi-country/program scoping** — Rules like "pricing changes blocked" (V2, Monetization-specific) or "multi-item requires BASE subscription" (V1) must become a `businessContext`-driven rule matrix, not hardcoded per program name.

---

## 4. Cross-Cutting Theme: V1+V2 → V3 Flexibility

Both UPGRADE and PLAN CHANGE in V1 were written assuming **one country, one program**. For V3:

- All hardcoded business rules (return shipment fee requirement, PAGE_SET overage logic, BASE-only multi-item rule, Monetization pricing block) need to be **conditionally applied based on `metadata.businessContext`** — mirrors how `OrderRequestValidatorForPaaS` is already registered as a per-program `@Service("orderRequestValidatorPaaS")` strategy. Future programs (LFP, WXP) may need their own validator or shared rules toggled by config.
- Country-specific behavior (fees, tax, currency formatting) should follow the existing `CountryFeatureMatrix` pattern rather than being hardcoded — this is already proven for GEO fee injection in `OrderServiceV3Impl.enrichGeoFeeItems()` and should extend to upgrade/plan-change logic.
- Pricing fields have moved from **flat fields** (V1: `price`, `fmv`, `allocatedRevenue`) to an **adjustments array** (V3: `pricing.adjustments[{type, amount, code}]`). Any ported logic must read/write via this new structure, not assume flat fields.
- **Customer identifier flexibility is already proven** — V2's plan-change (quantity) endpoint replaced V1's flat `tenantId` with `primaryCustomerId` + `primaryCustomerIdType` (orgId/userId/tenantId), and `ChangeRequestDao` already supports lookups by this flexible key. V3 should adopt this pattern directly rather than re-deriving it.
- **Downstream integration is not single-path** — V1 (pricing changes) integrates directly with Gekko + event confirmation; V2 (quantity changes) integrates with SCC + webhook confirmation. V3 must support **both paths concurrently**, routed by change type/program, not assume one downstream system fits all cases.

---

## 5. Summary Table — What's Missing for V3 Parity

| Flow | Feature | Status | Priority |
|---|---|---|---|
| Upgrade | Basic validation (parentOrderId, ACTIVE check, returnShipmentFeeSku) | ✅ Done | — |
| Upgrade | HW proration calculation | ✅ Done | — |
| Upgrade | Lowest overage price (Month 1) | ❌ Missing | High |
| Upgrade | Old subscription DEACTIVATING + undo overages/changes | ❌ Missing | High |
| Upgrade | Month-2 scheduled overage correction (SCC) | ❌ Missing | High |
| Upgrade | Per-country/program rule matrix | ❌ Missing | Medium |
| Plan Change | Endpoint (PUT /v3/subscriptions/{id}/update) | ❌ Missing | High |
| Plan Change | Unified item schema (pricing + quantity) | ❌ Missing | High |
| Plan Change | Flexible customer identifier (primaryCustomerId/Type) | ❌ Missing | High |
| Plan Change | ChangeRequestDao concurrency queue integration | ❌ Missing | High |
| Plan Change | IMM/SOT/EOT effective date logic | ❌ Missing | High |
| Plan Change | Dedup logic (adjustments-array aware) | ❌ Missing | Medium |
| Plan Change | Gekko-direct integration path (pricing/plan-swap) | ❌ Missing | High |
| Plan Change | SCC + webhook integration path (quantity) | ❌ Missing | High |
| Plan Change | Event + webhook confirmation flows | ❌ Missing | High |
| Plan Change | Per-country/program rule matrix (change-type permissions) | ❌ Missing | Medium |

---

*This document is analysis-ready for grooming — each gap row can convert directly into a JIRA story.*




