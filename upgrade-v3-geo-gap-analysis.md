# Upgrade V1 to V3 Analysis (DE / FR / US)

## Scope and evidence baseline
- Repository analyzed: `C:\SCC\ecomm-subscription-changectrl`.
- Focus: Upgrade flow in change control, especially where an "upgrade" order is detected/routed, and readiness for geo rollout (`DE`, `FR`, `US`) in v3.
- This analysis is based on code and tests currently present in the repository.

## Direct answer: where `orderType=upgrade` is handled
`orderType` is **not present** in this codebase (no `orderType` field/usages found).

What exists instead:
1. `customerOrderType` in cancellation payload model supports `UPGRADE`:
   - `src/main/java/com/hp/ecomm/subscriptioncc/model/subscription/request/CancellationRequest.java` (field `customerOrderType`)
   - `src/main/java/com/hp/ecomm/subscriptioncc/model/subscription/CustomerOrderType.java` (`UPGRADE` enum value)
2. Upgrade execution path is routed by endpoint + event type, not `orderType`:
   - `PUT /subscriptions/{id}/update/overage` in `SCCController.updateOverage(...)`
   - Event type set to `UPGRADE_OVERAGE` in `SCCServiceImpl.createUpgradeOverageSchedule(...)`
   - Queue routing by `eventType=UPGRADE_OVERAGE` in `QueueListener.handleMessage(...)`
   - Processing in `CallbackOverageService`

So, in current code, "upgrade" is functionally represented by **overage-update flow** (`UPGRADE_OVERAGE`), not by an `orderType` field.

## Current Upgrade V1 flow (actual implementation)
1. API intake
   - `SCCController.updateOverage(...)` receives `UpdateSubscriptionRequest`.
   - Calls `SCCService.updateOverageSubscription(...)`.

2. Persist + schedule
   - `SCCServiceImpl.updateOverageSubscription(...)`:
     - Rejects duplicate pending request for same subscription.
     - Saves `UpdateOverageSubscriptionDoc` with `PENDING`.
     - Creates `ScheduledEventDoc` with `eventType=UPGRADE_OVERAGE` via `createUpgradeOverageSchedule(...)`.

3. Scheduler pickup
   - `CallbackOverageService.processCallback()` fetches due `UPGRADE_OVERAGE` schedules in `PENDING/RETRYING` and transitions status.

4. Queue dispatch
   - Base callback logic publishes `InternalQueuePayload` with `eventType=UPGRADE_OVERAGE`.

5. Queue consumption
   - `QueueListener.handleMessage(...)` routes to `CallbackOverageService.processSubscriptionOverageChange(...)`.

6. External execution
   - `CallbackOverageService.processGekkoOverageChange(...)` converts payload to `ChangePlanRequestV1` and calls `GekkoClientService.changePlan(...)`.

7. Completion
   - On `202 Accepted`: marks both update doc and schedule completed, moves schedule to history.
   - On failure: moves schedule to `RETRYING` and eventually `FAILED` after retries.

## What is geo-aware today (DE/FR/US)
- Upgrade overage flow has **no country-specific branching**.
- Upgrade payload (`UpdateSubscriptionRequest`) does not carry country code.
- No DE/FR-specific config, mapping, or webhook variation in upgrade path.

Related geo indicators elsewhere:
- `CallbackTransactionChargeService.createSubTransChargeRequest(...)` hardcodes:
  - `countryCode("US")`
  - `currency("USD")`
- This is in transaction-charge (cancellation follow-up), not overage change-plan, but it is a key geo blocker for broader regionalization.

## Gaps for V3 rollout (DE/FR/US)

### 1) Missing `orderType` contract in SCC
- Gap: SCC does not parse/use `orderType`; upgrade routing is inferred by endpoint (`/update/overage`) only.
- Impact: If upstream intends to send `orderType=upgrade`, SCC currently ignores it.
- Required for v3:
  - Define canonical field contract (`orderType` vs `customerOrderType`) for upgrade entry.
  - Add explicit validation and audit logging of that field in upgrade APIs.

### 2) No country dimension in upgrade request model
- Gap: `UpdateSubscriptionRequest` has no country/market field.
- Impact: Cannot apply DE/FR/US policy differences at SCC layer.
- Required for v3:
  - Add country context (`countryCode`, and optionally `currency`) to request model or derive from tenant/customer context.
  - Enforce allowed values (`DE`, `FR`, `US`) and defaulting strategy.

### 3) Hardcoded US in transaction-charge flow
- Gap: `countryCode` and `currency` are hardcoded to `US` and `USD`.
- Impact: Incorrect financial data for DE/FR cancellation charge posting.
- Required for v3:
  - Replace hardcoded values with derived values from request/tenant/catalog metadata.
  - Add mapping and validation for at least `US/USD`, `DE/EUR`, `FR/EUR`.

### 4) Country-specific business rules are absent
- Gap: No rule engine/feature-flag matrix by country in upgrade flow.
- Impact: Any DE/FR regulatory or product nuances cannot be represented.
- Required for v3:
  - Introduce country-aware rule abstraction (effective date offsets, webhook behavior, billing nuances, overage rounding/precision if required).
  - Externalize to config/feature flags to avoid hardcoded branching.

### 5) API documentation/contract mismatch
- Gap: `ScheduleController` OpenAPI allowable values do not include all actual event types (e.g., `UPGRADE_OVERAGE`, `CANCELLATION_SMS_UPDATE`) though enum supports them.
- Impact: Consumers may not discover or correctly query upgrade schedules.
- Required for v3:
  - Align OpenAPI annotations with real enum values.

### 6) Weak enforcement for overage `idType`
- Gap: Controller description says only `SUBSCRIPTIONID` allowed for overage, but validation accepts all `SubscriptionIdType` enum values.
- Impact: Non-canonical IDs may enter upgrade-overage path unexpectedly.
- Required for v3:
  - Enforce `SUBSCRIPTIONID` strictly in controller/service for overage endpoint.

### 7) Test coverage gap for geo rollout
- Gap: No main-flow tests for DE/FR behavior because no such behavior exists; upgrade tests are mostly US-generic.
- Required for v3:
  - Add integration/unit tests for `DE`, `FR`, `US` across:
    - API validation
    - scheduling and queue routing
    - payload transformation to Gekko/SMS
    - charge posting payload correctness (country/currency)

### 8) Retry counter bug risk in callback base logic
- Gap: `AbstractCallbackService.updateSubscriptionStatus(...)` increments retry count and stores `retryCount + 1` again in field update map during optimistic-lock fallback path.
- Impact: Potential retry-count drift in concurrent update scenarios.
- Required for v3:
  - Fix counter update logic and add concurrency-focused test.

## What is already suitable for v3 reuse
- Event-driven skeleton for scheduling and async execution is present and reusable.
- Dedicated event type for upgrade overage (`UPGRADE_OVERAGE`) already exists.
- Queue routing for upgrade overage is isolated and test-covered.
- Request->Gekko transformation for change-plan exists and can be extended with country context.

## Recommended implementation blueprint for V3 (DE/FR/US)
1. Contract
   - Decide single source for upgrade intent (`orderType=UPGRADE` or endpoint-only).
   - Version API contract and document migration path.

2. Data model
   - Add `countryCode` (required) and `currency` (derived/optional) to upgrade path docs and payloads.
   - Persist country on `UpdateOverageSubscriptionDoc` and scheduled/history records for traceability.

3. Transformation layer
   - Centralize country/currency resolver used by all outbound payload builders.
   - Remove hardcoded `US/USD`.

4. Validation and policy
   - Add enum validation for countries and fallback behavior.
   - Add country-specific rule hooks (feature-flag controlled).

5. Documentation
   - Update OpenAPI annotations for event types and overage idType restrictions.

6. Testing
   - Add matrix tests (`DE`, `FR`, `US`) for happy/negative paths and payload assertions.

## Accuracy notes
- `orderType` string does not exist in the current SCC codebase.
- Upgrade processing exists through overage endpoint + `UPGRADE_OVERAGE` event pipeline.
- Country support is effectively US-only in charge payload defaults and absent in upgrade-overage contract/rules.

