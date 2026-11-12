# IAP subscriptions: cross-repo contract (v2)

Binding spec for the membership-subscription work (tickets EP-2919 iOS, EP-2920 Android). Payment,
core, CMS and mobile implement against this page. It refines [`11-iap-subscription-guide.md`](11-iap-subscription-guide.md);
where the two differ, **this page wins**. Approved flowcharts:
`.archify/iap-sub-review-20261104-181247/{1-ios-purchase,2-android-purchase,3-membership-lifecycle,4-plan-change}.html`.

## 0. Store setup (done 2026-11-04/05)

| | App Store | Google Play |
|---|---|---|
| 1 month | `weup.membership.1m` (Apple ID 6830580195), **level 2** | `weup.membership.1m`, base plan `monthly-1` |
| 3 months | `weup.membership.3m` (Apple ID 6830587657), **level 1** | `weup.membership.3m`, base plan `monthly-3` |
| Group / grace | group "WeUp Membership" 22544406 · grace 16 d (Sandbox only for now) | grace 7 d + account hold 53 d · Pause disabled |
| Webhook | prod `https://api.weupbook.vn/...`, sandbox `https://staging.api.weupbook.com/...` `/api/v1/payment-service/payments/iap/apple/notifications` | RTDN topic `projects/weup-elearning/topics/iap-rtdn` → staging |
| Streamlined Purchasing | **On** (App Store page purchases arrive without `appAccountToken`) | n/a |

Staging plans: `PLAN-1M-MVUHB1PF` (59,000 VND) and `PLAN-3M-MW20MD0S` (149,000 VND), store IDs set in
`subscription_plans.app_store_product_id / google_play_product_id / google_play_base_plan_id`.

## 1. Decisions

- **Mapping source of truth = the plan's own store columns** (`app_store_product_id`, `google_play_product_id`,
  `google_play_base_plan_id`, `app_store_enabled`, `google_play_enabled`). `iap_product_mappings` is NOT used for plans.
- **Plan rank = duration.** A longer plan is a higher level (3M > 1M). Compute it from `duration_value`/`duration_unit`
  (months); **no `store_level` column, no CMS field**. Upgrade = to a longer plan (immediate), downgrade = to a shorter
  plan (at next renewal).
- Product type string for plans is **`subscription`** (matches `order_items.product_type`).
- One `user_memberships` row per store chain, updated in place. Chain key: Apple `originalTransactionId`; Google the
  first `purchaseToken` (follow `linkedPurchaseToken` on replacements).
- Period id (one `orders` + `iap_transactions` row each): Apple `transactionId`; Google `latestSuccessfulOrderId`.
- Cancel = auto-renew off; access until `expires_at`. Only refund/revoke ends access early.
- Google: acknowledge **after** core commits the grant; on rejection call revoke with full refund.
- Apple Streamlined Purchasing is on: a transaction with **no `appAccountToken`** may be bound to the verifying
  (logged-in) user **only if its chain is not yet bound to another user**; log it as `bound_by_verify`.
- Cross-channel guard: a user with an active `payoo`/`admin` membership cannot start a store subscription
  (core records `rejected_cross_channel`, Google → revoke+refund, Apple → alert for manual refund). An active WeUp
  **trial does not block**; activation ends it. Payoo purchase/upgrade/cancel-renewal endpoints reject store memberships.
- **One auto-renewing store subscription per user (2026-11-11).** The stores never check `appAccountToken` /
  `obfuscatedAccountId`, so one WeUp user could hold two subscriptions at once (two Apple IDs or Gmail accounts, or
  iPhone + Android). Both would renew and charge independently, because a subscription belongs to the store account,
  not the device. A plan cannot be upgraded across stores.
  - **Blocked:** a new store chain while the user holds another live store chain with `autoRenew = true`. The guard
    above applies (same code `iap.subscription_cross_channel`, metadata `reason: 'store_auto_renew'`).
  - **Re-read before blocking:** core re-reads that chain from the store first, because a late webhook can leave
    `autoRenew` stale. If the store says renewal is off, core syncs the row and accepts the new chain. If the store
    says the purchase is gone (410), core marks that chain `EXPIRED` and accepts. On any other re-read error core
    returns `iap.transaction_processing` and the period stays pending, for up to **24 hours** after the period row was
    created. After that core trusts the DB (`autoRenew = true`) and blocks, with `storeStateVerified: false` in the
    metadata and an alert asking for a manual check. Core cannot tell transient from permanent errors by type.
  - **Allowed:** a new chain when the other chain already has auto-renew **off**. Two rows are live at once. The app
    must warn on the confirm sheet that the current plan runs to its end date without a refund and that the remaining
    days overlap.
  - **Unaffected:** plan changes inside the same chain (Apple group, Google replacement).
  - **`GET subscription/status` with several live rows:** returns the auto-renewing store row first, then the one that
    expires last.

## 2. payment-service HTTP (internal, `X-Internal-Service-Key`, prefix `/api/v1/payment-service`)

### 2.1 `POST payments/iap/apple/verify` `{ transactionId }` (unchanged request)

Response = existing fields **plus optional `subscription`** when the transaction `type` is `Auto-Renewable Subscription`:

```jsonc
{
  "externalProductId": "weup.membership.1m",
  "originalTransactionId": "2000000111",      // chain id
  "accountToken": "uuid-or-undefined",        // appAccountToken
  "storefrontCountryCode": "VN",
  "price": 59000, "currency": "VND",
  "environment": "sandbox" | "production",
  "purchaseDate": "ISO",
  "subscription": {                            // NEW, only for auto-renewable
    "chainId": "2000000111",                  // = originalTransactionId
    "periodTransactionId": "2000000999",      // this transactionId
    "latestToken": null, "linkedToken": null,
    "productId": "weup.membership.1m",
    "basePlanId": null,
    "expiresAt": "ISO",
    "autoRenew": true,                        // from renewal info autoRenewStatus
    "autoRenewProductId": "weup.membership.1m" | null,
    "state": "ACTIVE" | "IN_GRACE" | "ON_HOLD" | "EXPIRED" | "REVOKED" | "PENDING",
    "gracePeriodExpiresAt": "ISO" | null,
    "acknowledged": true                      // always true on Apple
  }
}
```

Uses `getTransactionInfo` + `getAllSubscriptionStatuses(originalTransactionId)` (pick the entry for this
`originalTransactionId`). A transaction with `revocationDate` → HTTP 409 `{ code: "iap.transaction_revoked" }`.
Sandbox fallback already exists for verify; add it for notification JWS too.

### 2.2 `POST payments/iap/google/verify` `{ purchaseToken, googleProductId, productKind? }`

`productKind`: `"one_time"` (default, current behaviour unchanged) | `"subscription"`.
For `subscription`: call `purchases.subscriptionsv2.get`; **do not acknowledge; do not use the verification cache**
(state changes over time; still save a verification row per period id). Line item used below = the **owned** item
(the one with `latestSuccessfulOrderId`, preferring the requested SKU among those), not a deferred-replacement target;
falls back to the SKU match, then `lineItems[0]`. Response = same base fields
(`externalProductId` = `lineItem.productId`, `originalTransactionId` = **`purchaseToken`**,
`accountToken` = `externalAccountIdentifiers.obfuscatedExternalAccountId`, `storefrontCountryCode` = `regionCode`,
`price`/`currency` = `lineItem.autoRenewingPlan.recurringPrice` (fallback: base plan regional price),
`environment` = `testPurchase ? sandbox : production`, `purchaseDate` = `startTime`) plus:

> **Note `originalTransactionId` (Google subscription):** = the **`purchaseToken`**. It stays the same across renewals
> and changes only on a plan replacement (upgrade/downgrade/resubscribe → new token, old one in
> `subscription.linkedToken`). The per-period order id is `subscription.periodTransactionId` (`latestSuccessfulOrderId`),
> not this field. Core builds the chain from `subscription.latestToken` / `linkedToken`; this field is only a fallback.
> One-time Google products keep `originalTransactionId = orderId` (unchanged). The verification row per period stores
> `external_transaction_id = periodTransactionId`, `original_transaction_id = purchaseToken`.

```jsonc
"subscription": {
  "chainId": null,                       // core resolves (first token of the chain)
  "periodTransactionId": "GPA.1234-..",  // latestSuccessfulOrderId
  "latestToken": "<purchaseToken>",
  "linkedToken": "<linkedPurchaseToken>" | null,
  "productId": "weup.membership.1m",
  "basePlanId": "monthly-1",
  "expiresAt": "ISO",                    // lineItem.expiryTime
  "autoRenew": true,                     // autoRenewingPlan.autoRenewEnabled
  "autoRenewProductId": "weup.membership.1m" | null, // autoRenewEnabled ? (deferredItemReplacement?.productId ?? productId) : null
  "state": "ACTIVE" | "IN_GRACE" | "ON_HOLD" | "EXPIRED" | "REVOKED" | "PENDING",
  "gracePeriodExpiresAt": null,
  "acknowledged": false | true
}
```

State map: `SUBSCRIPTION_STATE_ACTIVE`/`CANCELED`→`ACTIVE` (CANCELED = autoRenew false), `IN_GRACE_PERIOD`→`IN_GRACE`,
`ON_HOLD`→`ON_HOLD`, `EXPIRED`→`EXPIRED`, `PENDING`→`PENDING`, `PAUSED`→`ON_HOLD`, `PENDING_PURCHASE_CANCELED`→`EXPIRED`.
Apple: status 1→ACTIVE, 2→EXPIRED, 3→ON_HOLD (billing retry), 4→IN_GRACE, 5→REVOKED.

### 2.3 NEW `POST payments/iap/google/subscriptions/acknowledge` `{ purchaseToken, googleProductId }` → `{ acknowledged: true }`
`purchases.subscriptions.acknowledge`; "already acknowledged" = success (idempotent).

### 2.4 NEW `POST payments/iap/google/subscriptions/revoke` `{ purchaseToken }` → `{ revoked: true }`
`purchases.subscriptionsv2.revoke` with `revocationContext: { fullRefund: {} }`. "Already revoked" (HTTP 410 /
"already revoked|refunded") = success (idempotent).

Google Play error classes (verify, acknowledge, revoke and RTDN re-reads): 404/410 or 400 invalid-token → **410 Gone**
(permanent); 5xx/429/network → **503** (retry); other 4xx → 400.

## 3. RabbitMQ contract v2: `payment.events` / `payment.iap.notified` / `IapNotificationMessage`

File duplicated by hand, **ship both repos in the same release**: payment
`src/domain/iap/contracts/iap-notification-message.contract.ts`, core
`src/domain/messaging/contracts/iap-notification-message.contract.ts`.

```ts
export const IAP_NOTIFICATION_TYPE = {
  PURCHASE, REFUND, REFUND_DECLINED, REFUND_REVERSED, CONSUMPTION_REQUEST,   // existing (one-time)
  SUB_STARTED, SUB_RENEWED, SUB_PLAN_CHANGED, SUB_RENEWAL_STATUS,
  SUB_GRACE, SUB_ON_HOLD, SUB_EXPIRED, SUB_REVOKED, SUB_EXTENDED,            // new
} as const;

export interface IapSubscriptionStatePayload {   // = §2 `subscription` block, dates as ISO strings
  chainId: string | null; periodTransactionId: string | null;
  latestToken: string | null; linkedToken: string | null;
  productId: string; basePlanId: string | null;
  expiresAt: string | null; autoRenew: boolean; autoRenewProductId: string | null;
  state: 'ACTIVE' | 'IN_GRACE' | 'ON_HOLD' | 'EXPIRED' | 'REVOKED' | 'PENDING';
  gracePeriodExpiresAt: string | null; acknowledged: boolean;
  /** Apple: notificationType/subtype; Google: numeric notificationType. Diagnostics only. */
  storeEvent: string;
}

export interface IapNotificationMessage {
  platform: 'apple' | 'google';
  externalTransactionId: string;
  notificationType: IapNotificationType;
  notificationId?: string;                  // NEW: Apple notificationUUID / Pub/Sub messageId
  verifiedPurchase?: IapVerifiedPurchasePayload;
  subscription?: IapSubscriptionStatePayload; // NEW: set for every SUB_* message
  reason?: string;
  timestamp: string;
}
```

Mapping (payment re-reads state from the store API before publishing; the event only signals a change):

| Type | Apple `notificationType` / subtype | Google `subscriptionNotification.notificationType` |
|---|---|---|
| SUB_STARTED | SUBSCRIBED (INITIAL_BUY, RESUBSCRIBE) | 4 PURCHASED, 7 RESTARTED |
| SUB_RENEWED | DID_RENEW (incl. BILLING_RECOVERY) | 2 RENEWED, 1 RECOVERED |
| SUB_PLAN_CHANGED | DID_CHANGE_RENEWAL_PREF (UPGRADE/DOWNGRADE/none) | 17 ITEMS_CHANGED; PURCHASED whose v2 has `linkedPurchaseToken` |
| SUB_RENEWAL_STATUS | DID_CHANGE_RENEWAL_STATUS | 3 CANCELED |
| SUB_GRACE | DID_FAIL_TO_RENEW (GRACE_PERIOD or none) | 6 IN_GRACE_PERIOD |
| SUB_ON_HOLD | — | 5 ON_HOLD, 10 PAUSED |
| SUB_EXPIRED | EXPIRED (any subtype), GRACE_PERIOD_EXPIRED | 13 EXPIRED, 20 PENDING_PURCHASE_CANCELED |
| SUB_REVOKED | REFUND / REVOKE on an auto-renewable transaction | 12 REVOKED; voidedPurchaseNotification productType=1 |
| SUB_EXTENDED | RENEWAL_EXTENDED | 9 DEFERRED |
| (log only) | PRICE_INCREASE, OFFER_REDEEMED, RENEWAL_EXTENSION, TEST, METADATA_UPDATE… | 11, 18, 19, 22 … |

Idempotency (**insert-first**): `iap_notification_logs.notification_id` (nullable) with unique index
**`(platform, notification_id)`** (Apple `notificationUUID`, Google Pub/Sub `message.messageId`); the old
`(platform, external_transaction_id, notification_type)` unique index is kept as a partial index for one-time types only
(`WHERE left(notification_type, 4) <> 'SUB_'`), so repeated SUB_RENEWED rows for the same Google token are allowed.
Handlers claim the row right after validation (`INSERT … ON CONFLICT DO NOTHING RETURNING`) — not claimed = duplicate,
200 without publishing. If the store re-read or the publish fails after claiming, the claim is deleted and the webhook
returns non-2xx so Apple / Pub/Sub redeliver. A SUB_* without a notification id is not deduped; core stays idempotent on
`subscription.periodTransactionId`.

Failure handling: Apple nested JWS (transaction/renewal) that fails to decode → 400; no subscription state can be built
→ 503 (both redelivered). Google Play 5xx/429/network → 503 (redelivered); Play 404/410 on a `voidedPurchaseNotification`
(productType=1) → SUB_REVOKED built from the voided payload (`periodTransactionId = voided.orderId`,
`latestToken = purchaseToken`, `state = REVOKED`, `productId = ""`); Play 404/410 on any other SUB_* → logged, 200, not
published. Voided events always use `voided.orderId` as `periodTransactionId`.

## 4. core-service

### 4.1 Mobile-facing verify (`/api/v1/coreservice/iap/...`)

- `POST iap/apple/verify` `{ productId?: uuid, productType: 'ebook'|'audiobook'|'course'|'subscription', transactionId }`
- `POST iap/google/verify` `{ productId?: uuid, productType: ..., purchaseToken, storeProductId? }`
  - For `subscription`: `productId` (plan id) is optional; the plan is resolved from the verified SKU
    (`findByAppStoreProductId` / `findByGooglePlayProductId`). Google needs the SKU before verifying, so the app sends
    `storeProductId` (= `weup.membership.1m`); core validates it maps to a plan with `google_play_enabled`.
  - Response unchanged: `{ orderId, alreadyProcessed }`. Errors `iap.*`: existing + `iap.subscription_cross_channel` (409;
    active Payoo/admin plan, or another store chain still auto-renewing),
    `iap.transaction_revoked` (409), `iap.subscription_not_mapped` (400).

### 4.2 `GET subscription/plans` (each plan, additive)

`appStoreProductId: string|null` (null unless `app_store_enabled`), `googlePlayProductId: string|null`,
`googlePlayBasePlanId: string|null`, `durationMonths: number`, `currency: string` (market currency),
`userContext.buttonAction: 'SELECT' | 'CURRENT' | 'UPGRADE' | 'DOWNGRADE'` (DOWNGRADE new; for store members compare
`durationMonths`).

### 4.3 `GET subscription/status` → `membership` (additive)

`source: 'payoo'|'admin'|'apple'|'google'`, `autoRenew: boolean`, `storeProductId: string|null`,
`pendingPlanId: string|null`, `storeState: 'ACTIVE'|'IN_GRACE'|'ON_HOLD'|'EXPIRED'|'REVOKED'|'PENDING'|null`.
`getStatus()` must treat `revoked_at` as not active.

## 5. mobile

Flag `ENABLE_STORE_SUBSCRIPTIONS` (default false in prod). Store purchase with `productType: 'subscription'`,
`accountId = userId`; Android plan change via `changeSubscriptionParam` (upgrade `withTimeProration`, downgrade
`deferred`); iOS renewals verified from `Transaction.updates` without saved context (send SKU only). Manage
subscription deep links: iOS `https://apps.apple.com/account/subscriptions`, Android
`https://play.google.com/store/account/subscriptions?sku=<googlePlayProductId>&package=com.weup.book`.
Terms `https://weupbook.vn/vn/policy?type=terms`, Privacy `https://weupbook.vn/vn/policy?type=privacy`.

**Store purchase gate (2026-11-11).** Before a store subscription purchase, the app reads the subscriptions held by
the store account signed in on the device (`currentSubscriptions()`). It compares them with `membership`:

| Case | App behaviour |
|---|---|
| Active `payoo`/`admin` plan | Blocked (unchanged) |
| Store account holds a subscription bound to **another** WeUp user (iOS) | Blocked, StoreKit not opened |
| Live store plan with `autoRenew = true` that this store account does not hold (other platform or other Apple ID/Gmail) | Blocked; "managed in App Store / Google Play" + Manage subscription link |
| Same, but `autoRenew = false` | Fresh purchase (no replacement). The confirm sheet shows the loss notice: current plan valid until `<date>`, no refund, `<n>` overlapping days |
| This store account holds the current plan | Plan change as before (Google replacement / Apple group) |
