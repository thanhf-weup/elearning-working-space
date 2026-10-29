# In-app purchase across the repos

IAP spans three repos, and each one owns a different slice. This page is the cross-repo map: who
does what, which contracts join them, what works today, and what subscriptions still need. The
per-platform mobile detail (console steps, flow states, test checklists) lives in
`elearning-mobile-app/docs/implementation/iap-android.md` and `iap-ios.md`. This page links to
them and does not repeat them.

Source checked on 2026-10-29 against these refs:

| Repo | Ref |
|---|---|
| elearning-mobile-app | `feat/trial-subscription` (has `dev` merged, including `feat/apple-iap`) |
| elearning-core-service | `develop` @ `cf962da8` |
| elearning-payment | `origin/develop` @ `79e387f`. **Not the clone default `feat/payment`, which has no IAP code.** |

---

## 1. Who owns what

| Concern | Owner | Where |
|---|---|---|
| Store SDK, purchase sheet, flow states, restore, finish (iOS) | mobile | `lib/shared/stores/store_purchase_coordinator.dart`, `lib/shared/data/` |
| Binding a purchase to a user (`obfuscatedAccountId` / `appAccountToken` = userId UUID) | mobile sets it, core checks it | coordinator → core verify use cases |
| WeUp product ↔ store SKU mapping, market, ownership, order, enrollment | **core** | `src/application/iap/`, table `iap_product_mappings`, `iap_transactions` |
| Calling Apple/Google APIs: verify, acknowledge, product create/price | **payment** | `src/infrastructure/adapters/iap/` |
| Receiving store webhooks, verifying their signatures | **payment** | `payments/iap/{apple,google}/notifications` |
| Turning webhooks into grants/revokes | **core** | `handle-iap-notification` use case |

Rule of thumb: **payment-service knows stores, core knows users and products.** Payment never
decides what a purchase grants, and core never talks to Apple or Google directly.

## 2. Purchase flow (one-time product, both platforms)

```
 mobile                     core-service                     payment-service            store
   │  buy(SKU, accountId)                                                                 │
   ├─────────────────────────────────────────────────────────────────────────────────────►│
   │◄──────────────────────────────── purchase update (token / transactionId) ────────────┤
   │  POST /coreservice/iap/{google|apple}/verify   (Bearer)                              │
   ├──────────────────────────►│ POST /payment-service/payments/iap/{..}/verify           │
   │                           │   (X-Internal-Service-Key)                               │
   │                           ├─────────────────────────────►│ get + (Google) acknowledge│
   │                           │                              ├──────────────────────────►│
   │                           │◄──── verified purchase ──────┤                           │
   │                           │ check owner + market, grant, order (one DB tx)           │
   │◄──────── 200 ─────────────┤                                                          │
   │  (iOS) finish transaction                                                            │
```

Webhooks run in parallel and also heal purchases the app never verified:

```
 store ──► payment  /payments/iap/apple/notifications   (JWS V2, verified in payment)
       ──► payment  /payments/iap/google/notifications  (Pub/Sub push, OIDC guard)
 payment ──► RabbitMQ  payment.events / payment.iap.notified ──► core handle-iap-notification
```

Plus core's cron (`application/iap/cron/iap-pending-transaction-reconciliation.cron.ts`) catches
anything stuck in `pending`.

## 3. Contracts that join the repos

| Hop | Contract | Keep in sync |
|---|---|---|
| mobile → core | `POST /coreservice/iap/google/verify {productId, productType, purchaseToken}`; `.../apple/verify {productId, productType, transactionId}` | mobile `StorePurchaseRemoteDataSource` ↔ core `verify-*-iap-http.dto.ts` |
| core → payment | `POST /payment-service/payments/iap/{apple,google}/verify`, `.../products` | shared secret `IAP_INTERNAL_SERVICE_KEY` must be identical in both `.env` files |
| payment → core | RabbitMQ `payment.events`, key `payment.iap.notified`, message `IapNotificationMessage` | **file duplicated by hand** in both repos: `src/domain/iap/contracts/iap-notification-message.contract.ts` |
| mobile ↔ core | error codes `iap.*` (409 `transaction_processing`, 403 account mismatch, 400 not mapped / unsupported market) | mobile `StorePurchaseVerificationFailureCodes` |

`IapNotificationMessage.notificationType` today is only `PURCHASE | REFUND | REFUND_DECLINED |
REFUND_REVERSED | CONSUMPTION_REQUEST`.

## 4. Status: one-time products (ebook, audiobook, course)

Code is complete on all three repos for both platforms. Mobile turned on `ENABLE_PLAY_BILLING` and
`ENABLE_APP_STORE_IAP` in prod config on 29/10/2026. The remaining items are backend hardening
already listed in the mobile docs. Re-checked against payment `develop`, they **still hold**:

| Item | Repo | Evidence |
|---|---|---|
| Apple: no Production → Sandbox fallback (App Review buys with Sandbox on a prod build) | payment | `apple-iap.adapter.ts` builds one client from `APPLE_IAP_ENVIRONMENT`; a 404 from `getTransactionInfo` becomes `BadRequest` |
| Apple verify does not reject a transaction that has `revocationDate` | payment + core | no `revocation` check in payment's verify path |
| Google acknowledges **before** core checks owner/market | payment | `verifyAndAcknowledgePurchase`. If core then rejects, Google never auto-refunds |
| Full `purchaseToken` in logs | payment | `verify-google-purchase.use-case.ts`, `handle-google-notification.use-case.ts` (the adapter logs a 12-char prefix only) |
| Shared-secret compare is `!==`, not constant-time | payment | `internal-service-key.guard.ts` |

Full list with impact: `iap-android.md` §6 and `iap-ios.md` §6 (B1–B6).

## 5. Status: subscriptions (membership plans): not started on the IAP side

Today a plan is bought through `POST /coreservice/subscription/purchase`, which is a Payoo order
creating a fixed-length `user_memberships` row (`started_at`, `expires_at`). Mobile stops at
`SubscriptionPaymentConfirmSheet`, whose button is hard-disabled. Nothing in the IAP chain handles
subscriptions:

| Repo | What blocks subscriptions | Evidence |
|---|---|---|
| payment | Apple webhook mapper maps only one-time types. `SUBSCRIBED`, `DID_RENEW`, `EXPIRED`… → `null` → **not published** | `application/iap/mappers/apple-notification-type.mapper.ts` |
| payment | Google RTDN mapper ignores `subscriptionNotification`, and voided purchases of subscription type → `null` | `google-notification-type.mapper.ts` |
| payment | Google verify uses `purchases.products` (one-time only). Subscriptions need `purchases.subscriptionsv2.get` + `purchases.subscriptions.acknowledge` | `google-iap.adapter.ts` |
| payment | Notification idempotency key is `(platform, externalTransactionId, notificationType)`. Google uses `purchaseToken` as the id, which **stays the same across renewals**, so the 2nd `RENEWED` would be dropped as a duplicate | `iap.schema.ts` `uq_iap_notification_logs_platform_tx_type` |
| payment | Product sync creates one-time products only (Play `monetization.onetimeproducts`, ASC `v2/inAppPurchases`). Subscriptions are Play `monetization.subscriptions` and ASC subscription groups | `google-iap.adapter.ts`, `apple-connect-iap.adapter.ts` |
| payment ↔ core | `IapNotificationMessage` has no subscription types and no `expiresAt` / auto-renew fields | contract file (both repos) |
| core | `IAP_ELIGIBLE_PRODUCT_TYPES = ebook \| audiobook \| course`; the mapping table can't point at a plan | `domain/iap/constants/iap-product-type.constant.ts` |
| core | `user_memberships` has no source, store transaction/token, or auto-renew state | `user-membership.schema.ts` |
| core | `handle-iap-notification` handles only purchase/refund cases | `handle-iap-notification.use-case.ts` |
| mobile | Plan model has no store product ID. The payment screen shows the BE price minus a WeUp-computed upgrade credit (`computeUpgradeQuote`), which the stores cannot charge | `features/subscription/` |
| mobile | Confirm sheet never starts a purchase | `subscription_payment_confirm_sheet.dart` |

What already carries over unchanged: the coordinator and its flow states, the account binding,
persistent purchase context (`SecurePurchaseContextStore`), restore (iOS `AppStore.sync()`), the
purchase result/verifying UI, Pub/Sub + OIDC and the JWS verification setup, and the RabbitMQ
plumbing.

### Decisions (locked 2026-10-29)

Auto-renewable subscriptions, WeUp trial only (no store intro offers), store-native upgrades. The
step-by-step implementation guide is [`11-iap-subscription-guide.md`](11-iap-subscription-guide.md).
It supersedes the short build order below. It also uses product type `subscription`, not
`subscription_plan`, to match existing `order_items`.

### Build order (summary)

1. **Stores:** create the subscription group (Apple) / subscription + base plans (Play) by hand.
   Webhook URLs and RTDN topic already exist and carry subscription events too.
2. **payment:** subscription verify (`subscriptionsv2`, acknowledge; Apple transaction + status),
   subscription webhook mapping, a renewal-safe idempotency key (Apple `notificationUUID`, Pub/Sub
   `messageId`), contract fields for expiry/renewal. Close Apple Sandbox fallback (B1) first.
3. **core:** mirror the contract; product type `subscription` in the mapping; store-source columns
   on `user_memberships`; verify creates/extends membership with the store's expiry; webhook cases
   update `expires_at`/`revoked_at`; `subscription/plans` returns store SKUs; `subscription/status`
   returns source + auto-renew and blocks a second channel.
4. **mobile:** store price on paywall/payment; real purchase from the confirm sheet via the
   coordinator (`productType: subscription`, Android upgrade via `changeSubscriptionParam`);
   Apple 3.1.2 disclosure + Terms/Privacy; "Manage subscription" link; restore entry on the paywall.
5. **Test:** sandbox buy / renew / cancel / expire / grace period / refund / upgrade / account switch /
   kill mid-purchase / restore on a new device, plus a prod build with a sandbox account.

Optional later: extend payment's product sync to subscriptions so CMS can create them.
