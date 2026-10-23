# OPL Refunds — V1 vs V3 Flow Analysis (Subscription Management)

> **Scope**: Refund capability exposed by `ecomm-subscriptionmgt` (SMS) and the downstream processing performed by `ecomm-billingmgt` (BMS) and Gekko.
>
> **Note on versioning terminology**: The SMS codebase does **not** contain a standalone `/v1/.../refunds` REST endpoint. The refund capability was first introduced on the **V2** API (`SubscriptionV2RestController`) and later re-exposed on the **V3** API (`SubscriptionV3RestController`). For the purpose of this document — and to match the naming requested — **"V1" refers to the current/baseline production refund flow (implemented as the V2 API contract)**, and **"V3" refers to the newer API-version refund flow**. Both are documented in full below, followed by a side-by-side comparison.

---

## 1. High-Level Picture

```
Client / CAS / Admin Tool
        │
        │ POST /v2|/v3/subscriptions/{id}/refunds
        ▼
┌───────────────────────────┐        ┌──────────────────────────┐        ┌───────────────┐
│  ecomm-subscriptionmgt     │  REST  │   ecomm-billingmgt (BMS)  │  REST  │     Gekko      │
│  (SMS) - Controller→Service│ ─────► │  Controller→Service       │ ─────► │ (Payment/Ledger│
│  →StratusClient            │        │  →Gekko Client            │        │  Engine)       │
└───────────────────────────┘        └──────────────────────────┘        └───────────────┘
        │                                      │
        │                                      ▼
        │                         MongoDB: order_refund_details (audit)
        ▼
MongoDB: subscriptionorder / subscription (lookup only)
        │
        ▼
AIC Error-Handler Workflow Service (async, fire-and-forget)
   /aic-error-handler/v2/workflows/refund-services-by-line-items/trigger
```

Two distinct refund-related call paths exist in SMS:

1. **Direct / Explicit Refund API** — `POST /{version}/subscriptions/{id}/refunds` — the "refund" that this document focuses on (CAS/Admin initiated, ad-hoc financial credit).
2. **Auto-Refund during Cancellation** — an internal side-effect of `cancelSubscriptionV2/V3()` when `autoRefund=true` is passed on an **IMM (immediate) cancellation**. This does **not** call BMS/Gekko directly; instead it calls the **Order Settlement Service (OSS)** `refund-intents` API. It is covered in §5 for completeness since it is also "refund" logic inside Subscription Management, and reuses the same orchestration/eligibility concepts.

---

## 2. V1 (Current / V2 API) Refund Flow — Detailed Walkthrough

### 2.1 Where it starts
**Entry point (Controller)**
- File: `src/main/java/com/hp/ecomm/subscriptionmgt/controller/SubscriptionV2RestController.java` (lines ~448-501)
- `POST /v2/subscriptions/{id}/refunds?idType=SUBSCRIPTIONID&primaryCustomerId=...`
- Secured by `@ValidateAction(actions = {Action.REFUND_V2_POST})`
- Request body: `List<SubscriptionRefundRequestV2>` (one entry per refund line-item/charge)
- Delegates straight through to the service layer:
  ```java
  public SubscriptionRefundResponse processRefund(... String id, String subscriptionIdType,
          String primaryCustomerId, List<SubscriptionRefundRequestV2> refundRequestList) {
      return subscriptionService.processRefund(id, subscriptionIdType, primaryCustomerId, refundRequestList);
  }
  ```

### 2.2 What happens internally (Service Layer)
**File**: `service/v2/impl/SubscriptionServiceV2Impl.java` → `processRefund()` (line ~1239)

Step-by-step:
1. Look up the subscription: `subscriptionDaoRepository.findByResourceId(subscriptionId)`.
   - If not found → `handleError(RESOURCE_NOT_FOUND, ...)` → `404`.
2. **LFP validation** — `validateLfpSubscriptions(List.of(subscriptionDao))`: blocks refund processing for subscriptions belonging to the "Lease Financing Partner" workflow that must be handled differently.
3. **Mapping** — `SubscriptionMapper.toRefundRequestV2(subscriptionDao, refundRequestList)` converts the SMS-level request into the BMS-level DTO `BillingSubscriptionRefundRequestV2`, enriching it with:
   - `downstreamResourceId` → Gekko subscription reference id (`subscriptionDao.getPlatformInfo().getGekkoReferenceId()`)
   - `customerId` → downstream (Gekko) customer id
   - `currency`, `countryCode` → pulled from the subscription record (**not** validated against each other — see Known Gaps)
   - `invoiceChargeDate` → resolved via `resolveInvoiceChargeDate()`, which falls back intelligently based on subscription status/cancel date if the caller didn't supply one
   - `sku`, `quantity`, `value`, `unitPrice`, `chargeReason`, `chargeDescription`, `refundCategory`, `invoiceInfo` — passed through from the caller
4. **Downstream call** — `stratusClientService.processRefund(subscriptionId, subscriptionIdType, billingRefundRequest)` (→ see §2.3).
5. **Post-processing / async notification** — if BMS returns a non-null response, SMS fires an **async error-handler workflow trigger** (fire-and-forget, best-effort reconciliation hook — see §4) via `triggerErrorHandlerWorkflow(subscriptionDao, refundResponse.getRequestId(), REFUND, null)`.
6. Returns `SubscriptionRefundResponse` (wraps the Gekko `requestId`) back up through the controller to the caller.

### 2.3 Call to BMS (StratusClientServiceV2Impl)
**File**: `service/v2/impl/StratusClientServiceV2Impl.java` → `processRefund()` (line ~812)

```java
RequestParams<SubscriptionRefundResponse> params = RequestParams.<SubscriptionRefundResponse>builder()
        .method(HttpMethod.POST)
        .serviceId(SERVICE_ID_BMS)
        .uriTemplate(SUBSCRIPTION_REFUND_URI)        // /v2/subscriptions/{id}/refunds
        .pathParams(pathParams)                      // {id}
        .queryParams(queryParams)                    // idType
        .body(refundRequestList)
        .responseType(SubscriptionRefundResponse.class)
        .uri(oneCloudConfigHolder.getBmsSvcUrl())
        .build();
```
- Auth: OAuth2 bearer token obtained via `getOrRefreshOneCloudToken()` (OneCloud token cache).
- Target service: **BMS** (`ecomm-billingmgt`), via its "OneCloud"-registered base URL.

### 2.4 BMS side — where the money actually moves

**Controller**: `controller/v2/SubscriptionV2Controller.java` → `requestRefund()` (line ~62)
- `POST /v2/subscriptions/{id}/refunds` secured by `@ValidateAction(actions = Action.REFUNDS_V2_POST)`.
- Accepts `List<RefundV2Request>` (sku, value, chargeReason, refundCategory, invoice info, etc. — extends `BaseChargeV2Request`).
- Converts each `RefundV2Request` → internal `RefundRequest` (`getRefundRequestList()`), then calls:
  `subscriptionService.requestRefund(casSubId, idType.name(), refundRequestList)`.

**Service**: `service/impl/SubscriptionServiceImpl.java` → `requestRefund()` (line ~801)

1. **Guard clause**: refund list must not be null/empty.
2. **ID-type validation**: `gekkoIdType` must be a valid `IdType` enum value.
3. For **each** refund line item:
   - `validateRefundRequest(id, refundRequest)` →
     - `validateCommonRequestField()`: blank/null checks on subscriptionId, gekkoSubscriptionId, customerId, sku, currency, countryCode, invoiceChargeDate, chargeDescription, invoiceInfo, chargeReason.
     - Unit price must be non-null and **≥ 0**.
   - `validateAndGetSubscriptionOrderDaoByGekkoReferenceId()` — looks up `SubscriptionOrderDao` in Mongo (`subscriptionorder` collection) by the Gekko reference id; throws `404 NotFoundResource` if missing.
   - `validateSubscriptionOrderDaoWithIdType()` — cross-checks that the `id` passed matches the resolved order record (defence against spoofed/mismatched identifiers).
4. **Build Gekko charge request** — `formGekkoTransactionRefundRequestList()` builds one `GekkoTransactionRefundRequest` per line item:
   - `chargeType = CREDIT` (refunds are modeled as a **negative/credit charge**, not a dedicated "refund" transaction type in Gekko)
   - `chargeDate` formatted from `invoiceChargeDate`
   - sku, countryCode, currency, chargeQuantity, value, unitPrice, chargeReason, chargeDescription (mapped from `refundCategory`), refundCategory, invoiceInfo
5. **Resolve identifier type to send to Gekko** (feature-flag driven):
   - Reads `REFUND_PARAM_CHANGE` feature flag.
   - If `gekkoIdType == SUBSCRIPTIONID`, decides whether to send `GEKKO` (raw Gekko subscription id) or `SUBSCRIPTIONCONTRACT` (contract id) as the identifier type, otherwise passes through the caller-supplied idType.
   - Resolves the actual identifier value accordingly (`gekkoSubscriptionId`, `subscriptionContractId`, or the original `id`).
6. **Call Gekko**:
   ```java
   response = gekkoClientService.createTransactionCharge(
       identifierToGekko, gekkoTransactionChargeRequestListObject,
       refundRequestList.getFirst().getCustomerId(), requestIdType);
   ```
   - On Gekko error (`StratusException`), the BMS error payload is translated via `getActualStatusErrorMessageFromGekko()` and re-thrown as a `GenericServiceError`.
7. **Persist audit trail** — builds an `OrderRefundDao` (`SubscriptionMapper.toNonReturnDeviceOrderRefundDao()`) and saves it to MongoDB collection **`order_refund_details`**:
   | Field | Purpose |
   |---|---|
   | `casSubscriptionId` | CAS subscription id |
   | `idType` | id type used |
   | `refundRequest` | JSON snapshot of the inbound refund request |
   | `gekkoRequest` | JSON snapshot of what was actually sent to Gekko |
   | `gekkoRequestId` | the `requestId` returned by Gekko |
   | `refundDate` | timestamp |
8. **Response** — `new RefundResponse(response.getRequestId())`. Note: the response is **charge-level** only — it contains **no refund status, no refund id, no credit-note number**; the caller only gets Gekko's async-processing `requestId`.

### 2.5 Where it ends
- The HTTP response (`SubscriptionRefundResponse { requestId }`) bubbles back: Gekko → BMS → SMS (`StratusClientServiceV2Impl`) → SMS Service → SMS Controller → caller.
- In parallel, SMS fires the **AIC Error-Handler workflow** (`refund-services-by-line-items`) — this is a best-effort, fire-and-forget call for downstream reconciliation/alerting if something fails later; it does **not** block or change the refund response.
- **There is no synchronous confirmation that money was actually credited** — Gekko processes the `CREDIT` charge asynchronously; SMS/BMS only know that the request was accepted (`requestId` returned), not that it settled.

### 2.6 Services / Systems touched (V1/V2 flow)
| # | Component | Type | Call |
|---|---|---|---|
| 1 | `SubscriptionV2RestController` (SMS) | REST controller | Entry point |
| 2 | `SubscriptionServiceV2Impl` (SMS) | Service | Orchestration, mapping, LFP validation |
| 3 | `subscriptionDaoRepository` | MongoDB | Read subscription |
| 4 | `StratusClientServiceV2Impl` (SMS) | HTTP client (WebClient) | Calls BMS |
| 5 | `SubscriptionV2Controller` (BMS) | REST controller | BMS entry point |
| 6 | `SubscriptionServiceImpl` (BMS) | Service | Validation, Gekko request building, audit persistence |
| 7 | `subscriptionOrderRepository` | MongoDB | Lookup `SubscriptionOrderDao` |
| 8 | `GekkoClientService` | HTTP client | `createTransactionCharge()` → Gekko `CREDIT` charge |
| 9 | `orderRefundDaoRepository` | MongoDB | Persist `order_refund_details` (audit) |
| 10 | AIC Error-Handler Service | Async HTTP (fire-and-forget) | `refund-services-by-line-items/trigger` |

---

## 3. V3 Refund Flow — Detailed Walkthrough

### 3.1 Where it starts
**Entry point (Controller)**
- File: `src/main/java/com/hp/ecomm/subscriptionmgt/controller/v3/SubscriptionV3RestController.java` (lines ~510-561)
- `POST /v3/subscriptions/{id}/refunds?idType=SUBSCRIPTIONID&primaryCustomerId=...`
- **Same security action** reused: `@ValidateAction(actions = {Action.REFUND_V2_POST})` — V3 did not mint a new `REFUND_V3_POST` action.
- **Same request body type** reused: `List<SubscriptionRefundRequestV2>` (no new `SubscriptionRefundRequestV3` model was introduced).
- Delegates to a **separate service bean**: `SubscriptionServiceV3` (interface `service/v3/SubscriptionServiceV3.java`), *not* `SubscriptionServiceV2`.

### 3.2 What happens internally (Service Layer)
**File**: `service/v3/impl/SubscriptionServiceV3Impl.java` → `processRefund()` (line ~2556)

Step-by-step:
1. **ID-type pre-validation** (new in V3): `validateSubscriptionIdType(subscriptionIdType)`.
   - If idType is `PROVIDERCONTRACT` or `SUBSCRIPTIONCONTRACT`, `primaryCustomerId` becomes **mandatory** — throws `PRIMARY_CUSTOMER_ID_MANDATORY` otherwise. *(This validation does not exist in the V1/V2 flow.)*
2. Look up the subscription from the **V3-specific repository/entity**: `subscriptionDaoV3Repository.findByResourceId(subscriptionId)` → `SubscriptionDaoV3` (not `SubscriptionDao`).
   - Not found → `404`.
3. **No `validateLfpSubscriptions()` call** here — this check (present in V1/V2) is **not applied** in the V3 refund path.
4. **Mapping** — `SubscriptionMapper.toRefundRequestV3(subscriptionDaoV3, refundRequestList)`:
   - Structurally identical output type (`BillingSubscriptionRefundRequestV2`) to the V2 mapper, but sourced from the V3 DAO shape:
     - `gekkoReferenceId` comes from `subscriptionDaoV3.getProviderInfo().getGekkoReferenceId()` (V2 used `platformInfo`).
     - `customerId` resolved via `subscriptionDaoV3.getCustomer()` (V2 used `customerV2`).
     - `invoiceChargeDate` resolution uses `subscriptionDaoV3.getCancelledAt()` (V2 used `getCancelledOn()`).
5. **Downstream call — reuses the V2 client!** 🔑 Key finding:
   ```java
   SubscriptionRefundResponse refundResponse =
       stratusClientServiceV2.processRefund(subscriptionId, subscriptionIdType, billingRefundRequest);
   ```
   V3 does **not** have its own `StratusClientServiceV3.processRefund()` — it calls the same `StratusClientServiceV2Impl.processRefund()` used by the V1/V2 flow, which in turn hits the **same BMS endpoint** (`/v2/subscriptions/{id}/refunds`).
6. **Post-processing / async notification** — simplified vs. V2:
   ```java
   private void triggerErrorHandlerWorkflow(SubscriptionDaoV3 subDao, String gekkoRequestId) {
       var errorHandlerRequest = new ErrorHandlerWorkFlowRequest(
           gekkoRequestId, getDownstreamCustomerId(subDao.getCustomer()),
           subDao.getDownstreamOrderReferenceId(), null, subDao.getCustomer().getPrimaryCustomerId());
       stratusClientServiceV2.triggerErrorHandlerWorkFlowV2(REFUND, errorHandlerRequest);
   }
   ```
   - No `cancellationType`/workflow-context is passed (always `null`) — V2's version supports an optional cancellation-type context, V3's doesn't need it for a non-cancellation refund.
7. Returns the same `SubscriptionRefundResponse` shape back to the caller.

### 3.3 BMS side
Identical to §2.4 — **V3 reuses the exact same BMS controller, service, Gekko client, and `order_refund_details` persistence as V1/V2**. There is no V3-specific BMS refund endpoint.

### 3.4 Where it ends
Same termination semantics as V1/V2 (§2.5): caller receives `requestId` only; actual settlement happens asynchronously inside Gekko; the AIC error-handler workflow is triggered best-effort for reconciliation.

### 3.5 Services / Systems touched (V3 flow)
| # | Component | Type | Call |
|---|---|---|---|
| 1 | `SubscriptionV3RestController` (SMS) | REST controller | Entry point |
| 2 | `SubscriptionServiceV3Impl` (SMS) | Service | Orchestration, mapping, idType rule |
| 3 | `subscriptionDaoV3Repository` | MongoDB | Read subscription (V3 schema) |
| 4 | `StratusClientServiceV2Impl` (SMS) *(shared/reused)* | HTTP client (WebClient) | Calls BMS — **same client as V1/V2** |
| 5 | `SubscriptionV2Controller` (BMS) *(shared/reused)* | REST controller | **Same BMS endpoint as V1/V2** |
| 6 | `SubscriptionServiceImpl` (BMS) *(shared/reused)* | Service | Same validation/Gekko logic |
| 7 | `GekkoClientService` *(shared/reused)* | HTTP client | Same `CREDIT` charge call |
| 8 | `orderRefundDaoRepository` *(shared/reused)* | MongoDB | Same `order_refund_details` audit collection |
| 9 | AIC Error-Handler Service | Async HTTP (fire-and-forget) | Same `refund-services-by-line-items/trigger`, simpler payload |

---

## 4. Side-by-Side Comparison: V1 (V2 API) vs V3

| Aspect | V1 (Current / V2 API) | V3 |
|---|---|---|
| **REST path** | `POST /v2/subscriptions/{id}/refunds` | `POST /v3/subscriptions/{id}/refunds` |
| **Controller** | `SubscriptionV2RestController` | `SubscriptionV3RestController` |
| **Security action** | `Action.REFUND_V2_POST` | `Action.REFUND_V2_POST` *(reused, no dedicated V3 action)* |
| **Request DTO** | `SubscriptionRefundRequestV2` | `SubscriptionRefundRequestV2` *(same DTO reused — no new V3 request model)* |
| **Service interface/impl** | `SubscriptionServiceV2` / `SubscriptionServiceV2Impl` | `SubscriptionServiceV3` / `SubscriptionServiceV3Impl` *(separate bean)* |
| **Subscription repository/entity** | `subscriptionDaoRepository` → `SubscriptionDao` | `subscriptionDaoV3Repository` → `SubscriptionDaoV3` |
| **idType pre-validation** | Not enforced beyond `@EnumIn` annotation | **New rule**: `PROVIDERCONTRACT`/`SUBSCRIPTIONCONTRACT` idType requires `primaryCustomerId`, else `400` |
| **LFP (Lease Financing Partner) guard** | ✅ `validateLfpSubscriptions()` called | ❌ Not called in refund path |
| **Mapper method** | `SubscriptionMapper.toRefundRequestV2()` | `SubscriptionMapper.toRefundRequestV3()` — functionally equivalent, reads from V3 DAO fields (`providerInfo`, `customer`, `cancelledAt`) |
| **HTTP client used to reach BMS** | `StratusClientServiceV2Impl.processRefund()` | **Same** `StratusClientServiceV2Impl.processRefund()` (no dedicated V3 client method) |
| **BMS endpoint invoked** | `/v2/subscriptions/{id}/refunds` (BMS `SubscriptionV2Controller`) | **Same** endpoint — no V3 BMS refund API exists |
| **Gekko call** | `GekkoClientService.createTransactionCharge(..., chargeType=CREDIT)` | **Identical**, shared code path |
| **Audit persistence** | `order_refund_details` Mongo collection via `OrderRefundDao` | **Identical**, shared code path |
| **Response shape** | `SubscriptionRefundResponse { requestId }` | **Identical** shape |
| **Async error-handler workflow trigger** | Includes optional `cancellationType` workflow context | Simplified — context always `null`; uses `downstreamOrderReferenceId` instead of `gekkoOrderId` |
| **Net business logic difference** | Baseline/legacy behavior | Primarily an **adapter/rebind** onto the V3 subscription data model + stricter idType validation; **downstream financial processing is unchanged** |

### Key takeaway
V3's refund capability is **not a redesigned refund engine** — it is the **same BMS→Gekko pipeline** as V1/V2, re-wired to read from the newer `SubscriptionDaoV3` domain model and to call through the V3 controller/service layer. The only genuinely new behaviors in V3 are:
1. Mandatory `primaryCustomerId` when using contract-based id types, and
2. Omission of the LFP eligibility guard that exists in V1/V2 (needs confirmation whether this is intentional or a gap).

Everything from "map to `BillingSubscriptionRefundRequestV2`" onward — BMS validation, Gekko `CREDIT` charge, Mongo audit write, and the async error-handler trigger — is **100% shared code** between the two versions.

---

## 5. Related Flow: Auto-Refund during IMM Cancellation (context, not the primary "refunds" API)

Both V2 (`SubscriptionServiceV2Impl`) and V3 (`SubscriptionServiceV3Impl`) implement an **internal, secondary refund mechanism** triggered only as part of `cancelSubscriptionV2/V3()` when the caller requests an **Immediate (IMM) cancellation** with `autoRefund=true`:

1. `validateAutoRefundInputForImm(cancellationRequest, subscriptionDao)` — enforces that:
   - `autoRefund=false/null` ⇒ normal cancellation, no refund fields needed.
   - `autoRefund=true` ⇒ either **both** `unitsRemaining` + `unitType` are supplied (prorated refund) **or both are omitted** (full refund) — mixed/partial input is rejected.
   - `FRAUDULENT_CANCELLATION` is explicitly **disallowed** from using `autoRefund=true` (V3).
2. `initiateAutoRefund(...)` builds a `RefundIntentsRequest` (reason code `PRORATED_REFUND` or `FULL_REFUND`, `source=SMS`, billing-cycle context) and calls:
   ```java
   stratusClientServiceV2.initiateAutoRefund(id, refundIntentsRequest)
   ```
3. **This does NOT call BMS or Gekko.** It calls a **different downstream system — the Order Settlement Service (OSS)**:
   ```java
   RequestParams<RefundIntentsResponse> params = RequestParams.<RefundIntentsResponse>builder()
           .method(HttpMethod.POST)
           .serviceId(SERVICE_ID_ORDER_STATE)
           .uriTemplate(OSS_REFUND_INTENTS_URI)
           .uri(oneCloudConfigHolder.getOrderStateSvcUrl())
           .build();
   ```
4. This is shared infrastructure between V2 and V3 cancellation flows (both call the same `stratusClientServiceV2.initiateAutoRefund`).

This path is mentioned here because it is part of "refund" logic owned by Subscription Management, but it is architecturally **separate** from the explicit `/subscriptions/{id}/refunds` API that is the main subject of §2–§4 (different trigger, different target system — OSS instead of BMS/Gekko).

---

## 6. Data Model Reference

### 6.1 Request — `SubscriptionRefundRequestV2` (SMS, shared by V1 & V3)
Fields: `sku`, `quantity`, `value`, `unitPrice`, `chargeReason`, `chargeDescription`, `invoiceInfo`, `invoiceChargeDate`, `refundCategory`.

### 6.2 BMS request — `RefundV2Request extends BaseChargeV2Request`
```java
private BigDecimal value;          // amount to refund, e.g. -50.01
private ChargeReason chargeReason; // enum, required
private String refundCategory;     // e.g. SPECIAL_CONSIDERATION
```

### 6.3 `ChargeReason` enum (BMS)
```
SUBSCRIPTION_REFUND, OVERAGE_REFUND, CANCELLATION_FEE_REFUND,
HARDWARE_NON_RETURN_REFUND, RETURN_SHIPMENT_FEE_REFUND,
PREPAID_REFUND, AF, DEBIT_NOTE
```

### 6.4 Response — `RefundResponse` (BMS) / `SubscriptionRefundResponse` (SMS)
```java
public class RefundResponse {
    private String requestId;   // Gekko's async charge-processing request id — NOT a refund status
}
```

### 6.5 Audit record — `OrderRefundDao` (Mongo collection `order_refund_details`)
```java
private String casSubscriptionId;
private String idType;
private String refundRequest;   // JSON snapshot of inbound request
private String gekkoRequest;    // JSON snapshot of what was sent to Gekko
private String gekkoRequestId;  // Gekko's response id
private Instant refundDate;
```

---

## 7. Known Gaps / Risks (applies to both V1 and V3 — shared downstream code)

Sourced from existing engineering analysis in this repository (`Refund_Flow_Multi_Country_Support (2).md`, `doc/refund/GEO_OPL_Full_Refund_Capability_Dev_Analysis.md`, `doc/refund/Refund_MultiCountry_Implementation_Approach.md`):

1. **Charge-level, not invoice-line-item based** — refunds are modeled as a Gekko `CREDIT` charge, not tied to a specific invoice line item or credit note.
2. **No country/currency pairing validation** — only blank/null checks exist; a mismatched currency/country combination is not rejected.
3. **No duplicate-refund protection** — nothing prevents the same refund request from being submitted/processed twice.
4. **Fire-and-forget settlement** — SMS/BMS only receive a `requestId`; there's no synchronous or webhook-based confirmation of actual settlement/failure back into subscription state.
5. **Limited `ChargeReason` set** — e.g., no explicit `ACTIVATION_FEE_REFUND` / `PAPER_ADDON_REFUND` for broader GEO expansion.
6. **V3 omits the LFP eligibility guard** present in V1/V2 — needs product/engineering confirmation on whether lease-financed subscriptions should still be blocked from ad-hoc refunds in V3.
7. **No VAT/tax-aware refund calculation** and **no currency-rounding strategy** documented for multi-country rollout.

---

## 8. File Reference Index

**ecomm-subscriptionmgt (SMS)**
- `controller/SubscriptionV2RestController.java`
- `controller/v3/SubscriptionV3RestController.java`
- `service/v2/SubscriptionServiceV2.java`, `service/v2/impl/SubscriptionServiceV2Impl.java`
- `service/v3/SubscriptionServiceV3.java`, `service/v3/impl/SubscriptionServiceV3Impl.java`
- `service/v2/impl/StratusClientServiceV2Impl.java` (shared HTTP client for both V1 & V3)
- `repository/transfer/SubscriptionMapper.java` (`toRefundRequestV2`, `toRefundRequestV3`)
- `model/subscription/v2/SubscriptionRefundRequestV2.java`
- `model/subscription/SubscriptionRefundResponse.java`

**ecomm-billingmgt (BMS)**
- `controller/v2/SubscriptionV2Controller.java`
- `service/impl/SubscriptionServiceImpl.java` (`requestRefund`, `validateRefundRequest`, `formGekkoTransactionRefundRequestList`)
- `client/gekko/mock/GekkoClientService.java` / `GekkoClientServiceImpl.java`
- `model/refund/RefundRequest.java`, `RefundV2Request.java`, `RefundResponse.java`
- `model/subscription/GekkoTransactionRefundRequest.java`, `ChargeReason.java`
- `repository/dao/OrderRefundDao.java`, `repository/OrderRefundDaoRepository.java`
- `Refund_Flow_Multi_Country_Support (2).md`, `doc/refund/GEO_OPL_Full_Refund_Capability_Dev_Analysis.md`, `doc/refund/Refund_MultiCountry_Implementation_Approach.md` (existing gap analysis)

---

*Document generated from static code analysis of `ecomm-subscriptionmgt` and `ecomm-billingmgt` as of October 2026.*

