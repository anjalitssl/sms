# V3 Upgrade Flow — PR Explanation (Method-by-Method)

This document lists every method implemented/modified for the **V3 Upgrade** flow, grouped by the stage of the flow they belong to. Use this for PR demo / validation walkthroughs.

---

## Stage 1: Order Creation

### 1. `OrderServiceV3Impl.createOrder()`
**File:** `service/v3/impl/OrderServiceV3Impl.java` (~line 225)
**What it does:** Entry point for order creation. Validates business context, checks idempotency (`isDuplicatedOrder`), validates the request payload, then delegates to `createPersistAndQueueOrder()`.
**Upgrade relevance:** No structural change — ensures UPGRADE order types flow through the same validated path as every other order type.

### 2. `OrderServiceV3Impl.createPersistAndQueueOrder()`
**File:** `service/v3/impl/OrderServiceV3Impl.java` (~line 355)
**What it does:** Orchestrates the whole order build — resolves customer/PIM/address/payment info, builds the Gekko payload, maps request → `SubscriptionDaoV3` list.
**What was added:** After mapping subscriptions (lines 384-390), explicitly propagates `oldRenewedOrderId` onto every subscription in an UPGRADE order.
**Why:** The mapper never set this field for V3. Without this line, there's no link back to the subscription being replaced, meaning the old subscription could never get auto-deactivated later.

### 3. `OrderServiceV3Impl.updateSubscriptionDaoListForPaaSOrder()`
**File:** `service/v3/impl/OrderServiceV3Impl.java` (~line 422)
**What it does:** PaaS-specific enrichment applied before persistence — hardware SKU mapping, related-asset linking.
**What was added:** For `OrderType.UPGRADE` (lines 448-457), calls the proration method (#4 below) when a lease-receivable line item exists, and independently derives `oldRenewedOrderId` from the `ONBOOKS_PRINTER` line item.
**Why:** Proration and linkage are split into two independent checks deliberately — an upgrade bundle might not always include a lease-receivable, but the old-subscription linkage still needs to happen regardless.

### 4. `OrderServiceV3Impl.calculateAndSetUpgradeHWFirstMonthProratedCharges()`
**File:** `service/v3/impl/OrderServiceV3Impl.java` (~line 1716)
**What it does (core new logic):** Calculates the prorated first-month lease charge for the new (upgraded) hardware subscription.

```
transitionMonthDays = days in the activation month
activeBillingDays    = transitionMonthDays - dayOfMonth(activationDate) + 1
dailyRate            = leaseAllocatedRevenue / transitionMonthDays
proratedCharge       = dailyRate * activeBillingDays   (rounded to 2 decimals)
```

Stored into `leaseSubscription.getPricing().setUpgradeHWFirstMonthProratedCharges(...)`.
**Why:** Without this, a customer upgrading mid-month would be billed a full month's lease on day one, even though they only used part of the month. If allocated revenue or the start date is missing, it logs a warning and skips — proration failure should never block order creation.

---

## Stage 2: Order Acceptance Cleanup (Gekko accepts the UPGRADE order)

**Trigger point:** `InternalQueueEventHandlerImpl.processAcceptedOrder()` (~line 260-277) — right after Gekko returns HTTP 202 accepting an UPGRADE order, it calls `subscriptionService.processUpgradeOrderAcceptance(subscriptionDaoList)` (line 276). This runs *before* activation — it's cleanup on the **old** subscription the moment the new upgrade order is accepted downstream.

### 5. `SubscriptionServiceV3Impl.processUpgradeOrderAcceptance()`
**File:** `service/v3/impl/SubscriptionServiceV3Impl.java` (~line 1534)
**What it does:** Main orchestrator for this stage. Given the newly-created subscriptions from the accepted order:
1. Extracts `oldRenewedOrderId` from the new subscriptions.
2. Looks up the old ACTIVE subscriptions tied to that order, filtered to `LEGACYRENEW`/`RENEW` order types.
3. Finds the old **overage** subscription (`PAGE_SET` product type) among them, and compares its unit price to the new subscription's overage unit price.
4. If the new overage price is higher, calls `updateOverageSubscription()` to schedule a price update on the old one; either way, calls `undoOverages()` to cancel any pending/active overage charges on the old subscription.
5. Finds the old **BASE** subscription, calls `undoChange()` on it.
6. Finally, flips **all** old renewed subscriptions to `DEACTIVATING` status (so no other operation can touch them until the system job cancels them).
**Why:** Safety-net cleanup step — the moment Gekko accepts the new upgrade order, immediately neutralize any pending overage/change requests on the old subscription and lock it into DEACTIVATING, so it can't get billed or modified again while the new subscription takes over.

### 6. `SubscriptionServiceV3Impl.undoOverages()`
**File:** `service/v3/impl/SubscriptionServiceV3Impl.java` (~line 1601)
**What it does:** Calls `stratusClientService.undoOverages(resourceId, false)` to cancel any pending overage charge on the old subscription.
**Why:** Wrapped in try/catch — failures are logged as warnings, not thrown, so one bad downstream call doesn't kill the whole upgrade-acceptance flow.

### 7. `SubscriptionServiceV3Impl.undoChange()`
**File:** `service/v3/impl/SubscriptionServiceV3Impl.java` (~line 1610)
**What it does:** Calls `stratusClientService.undoChange(resourceId, SUBSCRIPTIONID)` on the old BASE subscription — reverses any in-flight plan/quantity change request that might still be pending on it.
**Why:** Same fail-soft pattern (log + continue) so it doesn't block the rest of the cleanup.

### 8. `SubscriptionServiceV3Impl.updateOverageSubscription()`
**File:** `service/v3/impl/SubscriptionServiceV3Impl.java` (~line 1619)
**What it does:** Builds an `SccUpdateSubscriptionRequest` to push the **new, higher** overage unit price onto the old subscription for its remaining life:
- Fetches the next billing cycle start date (`fetchBillingCycleResponse`) to use as the `effectiveDate` with an `EOT` (end-of-term) policy.
- Pulls the SKU's page-count characteristic (`SUB_OVERAGE_PAGE_PLAN`) via `SubscriptionManagementUtils.getSkuCharacteristics()` so downstream billing correctly classifies it.
- Calls `stratusClientService.updateOverageSubscription(...)`.
**Why:** Mirrors V1's `createSccUpdateSubscriptionRequestItem` — if the customer's new overage rate is higher than what they were on before, we don't want to under-bill them for the transition period, so the new rate is pushed onto the old subscription's remaining cycles too.

### 9. `SubscriptionServiceV3Impl.resolveDownstreamCustomerId()`
**File:** `service/v3/impl/SubscriptionServiceV3Impl.java` (~line 1667)
**What it does:** Pulls the `downstreamCustomerId` from the subscription's customer identifiers list, falling back to `resolvePrimaryCustomerId()` if not present.
**Why:** Used to populate the `customerId` field in the SCC update request (#8) — some downstream calls require the downstream-specific ID rather than the primary customer ID.

---

## Stage 3: Activation Event (Gekko confirms the new subscription is ACTIVE)

### 10. `EventHandlerServiceV3Impl.processSubscriptionActivateEventV3()`
**File:** `service/v3/impl/EventHandlerServiceV3Impl.java` (~line 406)
**What it does:** Handles the async `SUBSCRIPTION_CONTRACT_ACTIVATE_EVENT` from Gekko — activates the new subscriptions.
**What was added:** After activation (lines 452-453), calls the two new cleanup methods below.
**Why:** Once the new upgraded subscription is confirmed active, this is the trigger point to start winding down the old one.

### 11. `EventHandlerServiceV3Impl.transitionOldRenewedSubscriptionsToDeactivatingV3()`
**File:** `service/v3/impl/EventHandlerServiceV3Impl.java` (~line 458)
**What it does:** Looks up the old subscription via `oldRenewedOrderId`, and if it's still `ACTIVE`, flips it to `DEACTIVATING` and publishes a status-change event.
**Why:** Mirrors what V1 already does — ported the same behavior to V3 so the old subscription is properly marked as being wound down, not left dangling as ACTIVE.

### 12. `EventHandlerServiceV3Impl.disableDeactivatingOldRenewedSubscriptionsV3()`
**File:** `service/v3/impl/EventHandlerServiceV3Impl.java` (~line 482)
**What it does:** Takes subscriptions already in `DEACTIVATING` and flips them to `DISABLED`, stamping `cancelledAt` (using `subscriptionContractItemCancellationDateTime` if available, else now).
**Why:** Final state transition — old subscription is fully disabled once the new one is confirmed active, closing the loop.

---

## Stage 4: Cancellation Effective Date Reconciliation

### 13. `SubscriptionServiceV3Impl.calculateCancellationEffectiveDate()`
**File:** `service/v3/impl/SubscriptionServiceV3Impl.java` (~line 1469)
**What it does:** Determines the correct cancellation effective date for the old subscription during an upgrade cancellation (`UPGRADE_CANCELLATION` type).
**Logic:** Finds the sibling `BASE` subscription (the new upgrade) for the same customer, compares the old subscription's billing start date to the new subscription's activation date — if they match, uses that date; otherwise uses `(newActivation - 1 day)`.
**Why:** Guarantees there's no billing gap or overlap — the old subscription's cancellation always lines up exactly with when the new one started billing.

### 14. `SubscriptionServiceV3Impl.getCancellationEffectiveDate()`
**File:** `service/v3/impl/SubscriptionServiceV3Impl.java` (~line 1503)
**What it does:** Helper — fetches the billing cycle for a subscription and returns the correctly-formatted effective date (handles the edge case where today is the billing cycle end date, in which case it rolls to the *next* cycle's start date).
**Why:** Gekko rejects effective dates that don't have exactly 3 millisecond digits — `Instant.toString()` drops them when nanos are zero, which happens often on midnight billing boundaries. `DateTimeUtils.getUpdatedOnDate(...)` always forces exactly 3 millisecond digits.

---

## Stage 5: Billing-cycle Proration (V1 parity, referenced for context)

### 15. `SubscriptionServiceImpl.getOutstandingLeaseBalance()` (dispatcher)
**File:** `service/impl/SubscriptionServiceImpl.java` (~line 2220)
**What it does:** Detects if the leased subscription is a migration order (uses consumption-based formula) or an UPGRADE order in its transition month, and routes to the correct outstanding-balance calculation accordingly.

### 16. `SubscriptionServiceImpl.calculateOutstandingLeaseBalanceWithProration()`
**File:** `service/impl/SubscriptionServiceImpl.java` (~line 2272)
**What it does:** For the transition month of an UPGRADE order, computes the outstanding lease balance accounting for the prorated first month:
```
totalLeaseBalance      = leaseAmount * duration
completedCyclesBalance = leaseAmount * (currentBillingCycleNumber - 1)
outstandingBalance     = totalLeaseBalance - completedCyclesBalance - upgradeHWFirstMonthProratedCharges
```
**Why:** Without this, the outstanding/payoff balance shown to the customer during cancellation would assume a full first month was charged, over-stating what's owed. Falls back to `calculateStandardOutstandingBalance()` if dates can't be parsed.
**Note:** This piece currently exists only in the V1 `SubscriptionServiceImpl` — not yet ported to `SubscriptionServiceV3Impl`. Flagged as a parity gap to call out during PR review if V3 needs it too.

---

## Suggested Demo Order

1. `createOrder` → `createPersistAndQueueOrder` (show `oldRenewedOrderId` propagation) → `updateSubscriptionDaoListForPaaSOrder` → `calculateAndSetUpgradeHWFirstMonthProratedCharges`
2. **(Gekko acceptance)** `InternalQueueEventHandlerImpl.processAcceptedOrder` → `processUpgradeOrderAcceptance` → `undoOverages` → `undoChange` → `updateOverageSubscription` → `resolveDownstreamCustomerId`
3. **(Activation event)** `processSubscriptionActivateEventV3` → `transitionOldRenewedSubscriptionsToDeactivatingV3` → `disableDeactivatingOldRenewedSubscriptionsV3`
4. **(Cancellation)** `calculateCancellationEffectiveDate` → `getCancellationEffectiveDate`
5. **(Billing, V1 parity note)** `getOutstandingLeaseBalance` → `calculateOutstandingLeaseBalanceWithProration`

