# Implementing membership subscriptions with in-app purchase (iOS + Android)

This guide covers selling WeUp membership plans through App Store and Google Play subscriptions. It
gives the steps in order across **store consoles → payment-service → core-service → mobile**, built
on the one-time IAP chain that already ships (ebook, audiobook, course). For how that chain works
today, read [`10-iap.md`](10-iap.md) first.

Checked on 2026-10-29 against mobile `feat/trial-subscription`, core `develop` `cf962da8`, payment
`origin/develop` `79e387f`, `in_app_purchase` 3.3.1 (`in_app_purchase_android` 0.5.0 = **Play Billing
Library 8.0.0**; `in_app_purchase_storekit` 0.4.13+3 = StoreKit 2), Google RTDN reference and
`subscriptionsv2` reference, and Apple's `@apple/app-store-server-library` 3.1.0 enums.

---

## 0. Decisions (locked 2026-10-29)

| Decision | Choice | Consequence |
|---|---|---|
| Store product type | **Auto-renewable subscription** | The store renews. Store notifications move `expires_at`. Cancel/refund/grace come from the store. |
| Trial | **WeUp trial only** (`subscription/trial/activate`) | Store products get **no** intro offer / free trial. Buying a plan through a store ends an active WeUp trial, exactly as a Payoo purchase does today. |
| Upgrades | **Store-native** | Apple subscription-group levels and Google replacement modes do the proration. `computeUpgradeQuote` and the credit row are **not used** on store builds. |

**Out of scope for v1:** store intro/promo/offer codes and win-back offers, price-step-up handling
beyond logging, Family Sharing (keep **off**, because Apple won't let you turn it off again), web
purchase of store-billed plans, prepaid plans.

---

## 1. Product model

**One SKU per WeUp plan, the same ID on both stores.** This follows the existing `iap_product_mappings`
convention (`iap_product_id` is shared by Apple and Google, unique, and must match
`^[a-z0-9][a-z0-9._]{0,39}$`). Suggested form: `membership.<plan code>`, e.g. `membership.premium_12m`.

| | App Store | Google Play |
|---|---|---|
| Container | **One** subscription group, e.g. "WeUp Membership". Every plan lives in it, so a user can hold only one at a time. | One **subscription** per plan (product ID = SKU). |
| Inside | One auto-renewable subscription per plan | **Exactly one auto-renewing base plan, no offers.** The plugin then picks the base plan's `offerToken` automatically; with more offers you would have to choose and pass it. |
| Upgrade/downgrade | Set by the **level** ranking inside the group (level 1 = highest) | Chosen per purchase by `ReplacementMode` (§5.3) |
| Grace period | App Store Connect → Billing Grace Period | Base plan → grace period + account hold |

**Durations must map to store periods.** Apple allows 1 week, 1/2/3/6 months and 1 year. Google allows
P1W, P1M, P3M, P6M and P1Y. Core allows `durationUnit = day`, so **a day-based plan cannot be sold
through a store**. Leave it without a mapping and the app hides it on store builds (§5.4).

**Plan order:** `subscription_plans` has no tier or level column, and `sort_order` is only for
display. Add `store_level` (§4.1) so core knows whether a change is an upgrade or a downgrade, and
make it match the Apple group levels.

### Upgrade semantics to aim for (same user experience on both stores)

| Change | Apple (automatic) | Google (we pick) | Access in WeUp |
|---|---|---|---|
| Higher level | Immediate; prorated refund of the old plan | `withTimeProration`: immediate, remaining time credited | New plan's scope **now** |
| Lower level | At next renewal | `deferred`: at next renewal | Old plan until the renewal, then new scope |
| Same level, different duration | At next renewal | `deferred` | Same as lower level |

---

## 2. End-to-end flow

```
INITIAL PURCHASE — same chain as one-time IAP
 mobile buy(SKU, accountId=userId) ─► store ─► purchase (Android token / iOS transactionId)
 mobile POST /coreservice/iap/{google|apple}/verify {productId=planId, productType=subscription, ...}
   core ─► payment /payments/iap/{..}/verify  ─► Apple getTransactionInfo  /  Google subscriptionsv2.get
   core: owner check ─► find-or-create membership chain ─► order for this period ─► 200
   (Google) core ─► payment acknowledge   ·   (iOS) mobile finishes the transaction

LIFECYCLE — no app involved
 store ─► payment webhook ─► re-read state from the store API ─► RabbitMQ payment.iap.notified
   core: RENEWED → extend expires_at + order · PLAN_CHANGED → swap plan/scope
         RENEWAL_OFF → auto_renew=false (keep access) · GRACE → expires_at = grace end
         EXPIRED → expires_at = store expiry (cron revokes) · REFUND/REVOKED → terminate now
```

### Entitlement model in core: one membership row per store subscription chain

Store a single `user_memberships` row per chain and **update it in place**: renewals, upgrades and
downgrades. Don't insert a new row each period. The reason is that the expiry cron
(`revoke-expired-membership-enrollments`) revokes **all** of a user's `membership_enrolled` enrollments
as soon as *any* of their rows is expired and unrevoked. Row-per-period would strip access at
every renewal.

**Chain key:**
- **Apple:** `originalTransactionId`. It is stable across renewals and in-group upgrades.
- **Google:** the first `purchaseToken`. A replacement creates a new token whose
  `linkedPurchaseToken` points at the old one. Follow that link to the existing row, then save the
  new token as `store_latest_token`.

Renewal webhooks carry no WeUp user, so **resolve the user through the chain row**, never through
the webhook.

**Orders:** create one `orders` row per billed period (source `iap`, one item
`{productId: plan.id, productType: 'subscription'}`) plus one `iap_transactions` row. That keeps
`subscription/purchase-history` and revenue reporting working. Period IDs:
- **Apple:** the period's `transactionId`.
- **Google:** `lineItems[].latestSuccessfulOrderId` (a new `GPA....N` each renewal). **Not** the
  `purchaseToken`, which repeats across renewals.

---

## 3. payment-service (`elearning-payment`, branch `develop`)

Ordered. P1–P2 come first because they unblock App Review.

**P1. Migrate to Apple's official library.** Replace `app-store-server-library@0.1.3` with
**`@apple/app-store-server-library` 3.x** (3.1.0 is current). The code comment calling 0.1.3 "the
latest on npm" is wrong: that's the old unscoped name. 3.1.0 has `ONE_TIME_CHARGE`,
`getAllSubscriptionStatuses`, `getTransactionHistory`, `setAppAccountToken` and the full
`NotificationTypeV2` / `Subtype` enums. Once migrated you can delete
`LenientAppleNotificationVerifier`. Re-run the existing ebook tests.

**P2. Apple Production → Sandbox fallback (B1).** If `getTransactionInfo` returns 404 in Production,
retry with a Sandbox client. Do the same when verifying notification JWS. Record `environment` on
every result. App Review buys with Sandbox accounts on production builds.

**P3. Apple subscription verify.** Reuse `POST payments/iap/apple/verify`.
- The decoded transaction already carries `type` (`Auto-Renewable Subscription`), `expiresDate`,
  `originalTransactionId`, `subscriptionGroupIdentifier`, `appAccountToken` and `revocationDate`.
- For subscription types, also call `getAllSubscriptionStatuses(originalTransactionId)` and return
  the renewal info: `autoRenewStatus`, `autoRenewProductId`, `gracePeriodExpiresDate`,
  `expirationIntent`.
- Reject when `revocationDate` is set (B2).

**P4. Google subscription verify.** Add a branch in `google-iap.adapter.ts`:
- Call `purchases.subscriptionsv2.get(packageName, token)`. **Don't use** `purchases.products.get`.
- Return:
  - `subscriptionState`: `SUBSCRIPTION_STATE_ACTIVE | IN_GRACE_PERIOD | ON_HOLD | PAUSED | CANCELED | EXPIRED | PENDING | PENDING_PURCHASE_CANCELED`
  - `lineItems[0].productId`, `.expiryTime`, `.offerDetails.basePlanId`, `.latestSuccessfulOrderId`, `.autoRenewingPlan.autoRenewEnabled`
  - `linkedPurchaseToken`, `acknowledgementState`, `externalAccountIdentifiers.obfuscatedExternalAccountId`, `regionCode`, `testPurchase`
- How core asks for subscriptions: core already sends `googleProductId`. Add `productKind:
  'subscription'` from core (it knows from the mapping) rather than guessing in payment.
- Price per period: resolve from the base plan's regional price via `monetization.subscriptions`, the
  same way `onetimeproducts` prices are resolved today.

**P5. Google acknowledge after grant, not before.** Expose
`POST payments/iap/google/subscriptions/acknowledge {token}` and call
`purchases.subscriptions.acknowledge`. Core calls it **after** it has committed the membership.
- An unacknowledged subscription is refunded and revoked by Google after 3 days, so a failed grant
  refunds itself instead of charging the user for nothing.
- This fixes the subscription version of the "acknowledge before owner check" gap in
  `iap-android.md` §6.
- Core's hourly reconcile cron retries the acknowledge.

**P6. Webhook mapping.** Both store references say a notification only signals that something
changed, so **always re-read state from the store API** and publish the verified state, not just
the event type.

| Abstract type (new) | Apple `notificationType` / subtype | Google `subscriptionNotification.notificationType` |
|---|---|---|
| `SUB_STARTED` | `SUBSCRIBED` (`INITIAL_BUY`, `RESUBSCRIBE`) | 4 `PURCHASED`, 7 `RESTARTED` |
| `SUB_RENEWED` | `DID_RENEW` (incl. `BILLING_RECOVERY`) | 2 `RENEWED`, 1 `RECOVERED` |
| `SUB_PLAN_CHANGED` | `DID_CHANGE_RENEWAL_PREF` (`UPGRADE`, `DOWNGRADE`, none = reverted) | 17 `ITEMS_CHANGED`; also any `PURCHASED` whose v2 has `linkedPurchaseToken` |
| `SUB_RENEWAL_STATUS` | `DID_CHANGE_RENEWAL_STATUS` (`AUTO_RENEW_ENABLED/DISABLED`) | 3 `CANCELED` (user turned off renew; access continues) |
| `SUB_GRACE` | `DID_FAIL_TO_RENEW` (`GRACE_PERIOD`; no subtype = billing retry without grace) | 6 `IN_GRACE_PERIOD`, 5 `ON_HOLD` |
| `SUB_EXPIRED` | `EXPIRED` (`VOLUNTARY`, `BILLING_RETRY`, `PRICE_INCREASE`, `PRODUCT_NOT_FOR_SALE`), `GRACE_PERIOD_EXPIRED` | 13 `EXPIRED` |
| `SUB_REVOKED` | `REFUND`, `REVOKE` (on a subscription transaction) | 12 `REVOKED`; `voidedPurchaseNotification.productType = 1` |
| `SUB_EXTENDED` | `RENEWAL_EXTENDED` | 9 `DEFERRED` |
| log only | `PRICE_INCREASE`, `PRICE_CHANGE`, `OFFER_REDEEMED`, `RENEWAL_EXTENSION`, `TEST`, `METADATA_UPDATE` | 10/11 pause (pause is off in our base plans), 19, 22 price, 20 `PENDING_PURCHASE_CANCELED`, 18 |

**P7. Idempotency key that survives renewals.** Today `uq_iap_notification_logs_platform_tx_type` is
`(platform, externalTransactionId, notificationType)`. On Google that id is the `purchaseToken`, so
the **2nd `RENEWED` would be dropped as a duplicate**. Add `notification_id` and make it unique:
- **Apple:** `notificationUUID`.
- **Google:** the Pub/Sub `messageId`.

**P8. Contract v2.** Extend `IapNotificationMessage` (and core's hand-copied twin, in the **same
release**) with `subscription?: { originalTransactionId | chainToken, latestToken, linkedToken?,
productId, periodTransactionId, expiresAt, autoRenew, state, gracePeriodExpiresAt?, autoRenewProductId?,
environment }`. Keep the old five types working.

**P9. Hardening that touches the same code:** stop logging full `purchaseToken`, compare
`X-Internal-Service-Key` in constant time, and add the `iap.*` error codes for revoked / not found (B4).

**Later (optional):** product sync for subscriptions (Play `monetization.subscriptions`, ASC
subscription groups API), so CMS can create store products the way it does for ebooks.

---

## 4. core-service (`elearning-core-service`, branch `develop`)

### 4.1 Schema (one migration)

| Table | Add | Why |
|---|---|---|
| `subscription_plans` | `store_level int null` | Upgrade vs downgrade direction; matches Apple group level |
| `user_memberships` | `source varchar(16) not null default 'payoo'` (`payoo`/`admin`/`apple`/`google`), `store_chain_id varchar(256) null`, `store_latest_token varchar(256) null`, `auto_renew boolean null`, `store_state varchar(32) null`, `pending_plan_id uuid null`, `updated_at` | One updatable row per chain; status/UI need source + renewal |
| | unique index `(source, store_chain_id)` where `store_chain_id is not null` | Idempotent chain lookup |
| `iap_transactions` | `expires_at timestamp null`, `membership_id uuid null` | Per-period record linked to the chain |

`iap_product_mappings` needs no columns. Map `product_id = plan.id`, `product_type = 'subscription'`.
**Use `subscription`**, the value Payoo plan orders already put in `order_items.product_type` and
that `processPaymentSuccessEvent` already skips for enrollment.

### 4.2 Code changes

1. **Product type.** Add `subscription` alongside `IAP_ELIGIBLE_PRODUCT_TYPES` as a *separate*
   constant (`IAP_SUBSCRIPTION_PRODUCT_TYPE`), so the enrollment-oriented code paths stay closed.
   Then update every place that branches on product type:
   - the admin-mapping, sync and bulk-sync DTOs' `@IsIn`
   - the verify commands
   - `verifyMappedItem` (look up the plan; `is_active`)
   - the repository adapter casts
   - the REFUND `PRODUCT_TYPE` cast
2. **Membership repository.** Add `findByChain(source, chainId)`, `update(membership)`, and
   `findActiveByUserId`. `save` is insert-only today.
3. **Extract `MembershipActivationService`** from `OrderPaymentProcessingService.processPaymentSuccessEvent`:
   activate + scope snapshot + end an active WeUp trial (`retagBulk` + `endEarly`) + activation
   email. Use it from Payoo, admin-assign and IAP, so the three can't drift apart.
4. **Grant branch.** In `IapPurchaseGrantService.grantEntitlement`, branch when
   `mapping.productType === 'subscription'`. Keep the existing account-token and market checks. Then:
   - Find the chain (Google: walk `linkedPurchaseToken`).
     - **New chain:**
       - If the user has an active **non-store** membership, don't activate. Record the
         transaction as `rejected_cross_channel` and send the Base alert for a manual refund, as
         Payoo does today.
       - Otherwise activate through the service with `expires_at = store expiresAt`.
     - **Existing chain:** update plan/scope (upgrade) or set `pending_plan_id` (deferred change), and
       update `expires_at`, `auto_renew`, `store_state`, `store_latest_token`.
   - Create the period order + `iap_transactions` row (idempotent on the period id).
   - After commit, request the Google acknowledge (P5).
5. **Verify without `productId` (B3), required for subscriptions.** iOS delivers renewal and
   upgrade transactions through `Transaction.updates`, sometimes on a device with no saved purchase
   context. Resolve the plan from the verified SKU and ignore the client's `productId` when it is
   missing. Otherwise those transactions are never finished and replay forever.
6. **Notification handler.** Add `SUB_*` cases to `handle-iap-notification` per the §3 P6 table:
   - **renewed:** extend + order
   - **plan changed:** apply now (upgrade) or at renewal (deferred → swap at the next `SUB_RENEWED`)
   - **renewal status:** set `auto_renew`
   - **grace:** `expires_at = grace end`
   - **expired:** `expires_at = store expiry`, then the cron does the revoke
   - **revoked:** `terminateNow` + revoke enrollments (reuse admin-revoke's filter that spares
     products bought outright)
   - **started (heal):** same as the grant branch when the app never called verify
7. **Expiry cron leeway.** In `findExpiredUnrevoked`, apply a **24 h leeway for `source in
   ('apple','google')` with `auto_renew = true`**, so a late renewal webhook doesn't revoke access
   for a paying user. Then add the store rows that are about to expire to the hourly IAP reconcile:
   re-read them through payment and apply the result.
8. **Reconcile cron.** `reconcile-pending-iap-transactions` uses `purchases.products.get` today. Route
   `subscription` rows to the subscription verify, and retry pending acknowledges.
9. **Endpoints.**
   - **`GET subscription/plans`:** add `iapProductId` (from the mapping, `null` when unmapped) and
     `storeLevel`. Compute `userContext.buttonAction` for store-billed members as `CURRENT` /
     `UPGRADE` / `DOWNGRADE`.
   - **`GET subscription/status`:** add `membership.source`, `autoRenew`, `iapProductId`,
     `pendingPlanId`, `storeState`. Fix `getStatus()` to treat `revoked_at` as not active; it
     ignores it today.
   - **`POST subscription/purchase` (Payoo):** keep rejecting when any membership is active, which now
     includes store ones. A store subscriber can't be charged twice through Payoo.
   - **Admin revoke of a store membership:** it can't stop store billing. Show a warning in CMS. For
     Google, optionally call `subscriptionsv2.revoke` through payment.
10. **CMS.** Allow mapping a plan (`productType: subscription`) in the admin IAP mapping screen. This
    is a separate `elearning-cms` change.

---

## 5. mobile (`elearning-mobile-app`, branch `feat/trial-subscription`)

Follow the repo's own rules (`CLAUDE.md`, `guideline-core-convention/`). This is cross-feature
`shared/` and billing work, so run `/bmad-planning` before coding.

### 5.1 Flag
Add `ENABLE_STORE_SUBSCRIPTIONS` (`config/*.json`, `Env`, and a `storeSubscriptionEnabledProvider` =
flag && `storePurchaseEnabledProvider`). It ships dark: store releases can't be rolled back
instantly, and the backend must be live first.

### 5.2 Data
- **`SubscriptionPlanModel` / mapper / entity:** add `iapProductId` (trim blank → null, as
  `ProductDetail.iapProductId` does) and `storeLevel`.
- **`SubscriptionStatus.membership`:** add `source`, `autoRenew`, `iapProductId`, `pendingPlanId`.
- **`StoreProduct`:** add `subscriptionPeriod` (ISO 8601, e.g. `P1M`) so the UI can show "₫x / month"
  from the store.
  - Android: read it from the base-plan pricing phase of `GooglePlayProductDetails`.
  - iOS: read it from `SK2Product.subscription.subscriptionPeriod`.
- **`StoreProductCatalog.query`:** on Android a subscription query returns **one `ProductDetails` per
  base plan/offer**. Keep the entry whose offer has no `offerId` (the base plan). Don't assume one
  entry per ID.

### 5.3 Billing layer
- **`StoreBillingRepository.buy`:** add an optional `replacing: StoreSubscriptionChange?`
  `{currentStoreProductId, mode: upgrade|deferred}`.
  - **Android:** build `GooglePlayPurchaseParam(productDetails, applicationUserName: accountId,
    changeSubscriptionParam: ChangeSubscriptionParam(oldPurchaseDetails: <current sub's
    GooglePlayPurchaseDetails from queryPastPurchases>, replacementMode: withTimeProration | deferred))`.
    - **Without `changeSubscriptionParam` Google starts a second, concurrent subscription and
      bills both.** If the current sub's purchase isn't on the device (bought on another Google
      account), block the change and tell the user to manage it in Play.
  - **iOS:** plain buy of the new product. StoreKit applies the group rules, so `replacing` is
    ignored.
- **`StorePurchaseCoordinator.buy`:** `productId: plan.id`, `productType: 'subscription'`.
  - `_resolveHeldByStore` must treat "already holds **this** SKU" as `owned`, but **must not**
    block buying a *different* SKU in the same group.
  - On a subscription grant, refresh `subscriptionStatusStoreProvider`. It doesn't listen to
    `entitlementRevision` today.
- **Renewals:**
  - **Android:** renewals arrive as already-acknowledged purchases, which the coordinator already
    skips. Fine.
  - **iOS:** renewal/upgrade transactions come through `Transaction.updates` and
    `Transaction.unfinished`. They get verified (idempotent) and finished. This relies on core §4.2
    step 5 when no context is saved.
- **Restore:**
  - **iOS:** the existing Profile tile runs `AppStore.sync()`; StoreKit restore reads
    `Transaction.currentEntitlements`, which includes active subscriptions. Add the same action to
    the paywall.
  - **Android:** no button needed; `restore()` already runs on resume and login.

### 5.4 UI (`lib/features/subscription/`)
- **Paywall and cards (store builds):**
  - Show only plans with `iapProductId`, with price and period from `storeProductQuoteProvider`.
    The CMS price is only a fallback while loading.
  - Hide day-based or unmapped plans.
  - Label the button from `userContext` (`CURRENT` / `UPGRADE` / `DOWNGRADE`).
- **Payment screen:** on store builds drop the subtotal/credit/total rows (`computeUpgradeQuote`),
  show the store price, and for a downgrade say "Takes effect on <renewal date>".
- **Confirm sheet:** replace the disabled button with the ebook flow.
  - Generalize `DigitalPurchaseFlowRunner` (today it takes a `ProductDetail`) to take a small
    purchase context `{storeProductId, productId, productType, productName}`.
  - Reuse `PurchaseVerifyingDialog`, `PurchaseResultSheet`, `PurchaseFlowWatcher` and the grant
    toast.
- **Required disclosure (App Store Guideline 3.1.2):** on the paywall and the sheet, show the
  subscription title, length, price per period, that it renews automatically until cancelled, how
  to cancel, and links to Terms of Use (EULA) and the Privacy Policy. Google expects the same.
- **Cross-channel guard:** if `membership.source` is `payoo`/`admin` and active, disable store
  purchase with "You already have an active membership until …". An active WeUp trial does **not**
  block; activation ends it.
- **"Manage subscription"** (info screen; replaces the "Cancel plan — coming soon" snackbar for store
  sources), via `url_launcher` (already a dependency; the StoreKit plugin has no
  `showManageSubscriptions`):
  - iOS: `https://apps.apple.com/account/subscriptions`
  - Android: `https://play.google.com/store/account/subscriptions?sku=<iapProductId>&package=com.weup.book`
  - Payoo memberships keep today's behaviour.
- **Info screen:** show "Renews on <date>" vs "Ends on <date>" from `autoRenew`, and a
  grace/billing-issue banner when `storeState` is grace/on hold.
- **l10n:** every new string in all 9 ARB files (`make check-l10n`).

### 5.5 Platform
- **iOS:** confirm the In-App Purchase capability is on for the App ID (Xcode → Signing &
  Capabilities). `Runner.entitlements` doesn't show it either way.
- **Android:** the Billing library merges `com.android.vending.BILLING`. Verify it in the merged
  manifest of a release build.

### 5.6 Tests (extend the existing fake-based suites)
- `store_billing_repository_impl_test.dart`: base-plan entry selection; `GooglePlayPurchaseParam` with
  `changeSubscriptionParam` + mode; change blocked when the old purchase isn't on the device.
- `app_store_billing_repository_impl_test.dart`: subscription period mapping; renewal from `updates`.
- `store_purchase_coordinator_test.dart`: subscription buy; held-same-SKU vs different-SKU; status
  refresh on grant; renewal verify without context.
- `subscription_screens_test.dart`: store price shown, unmapped plans hidden, cross-channel guard,
  disclosure + links, manage link per platform, flag off = today's behaviour.
- `make gen` then `make check` must be green.

---

## 6. Store console setup

Agreements, tax, banking, the App Store Server API key, Server Notifications V2 URL, RTDN topic, the
Play service account and license testers already exist from the ebook work. **Verify them; don't
recreate them.** Subscription events use the same URLs and topic.

**Console state read on 2026-10-29 (read-only check):**

| Where | Found | Action |
|---|---|---|
| ASC app | WeupBook, Apple ID `6816870449`, bundle `com.weupbook.elearning`; iOS 1.0 "Ready for Review"; 9 non-consumable ebooks, all drafts | The first subscriptions must ship with a version: 1.0, or the next one if 1.0 goes out first |
| ASC → Subscriptions | No group; **Billing Grace Period off**; **Streamlined Purchasing on** | Create the group; turn grace **on**; turn Streamlined Purchasing **off** (purchases from the App Store page carry no `appAccountToken`, so core can't bind them to a user) |
| ASC → App Information → Server Notifications | **Production URL empty**; Sandbox URL is a personal **ngrok** tunnel | Set Production to the prod payment URL and Sandbox to a stable staging URL, both `.../api/v1/payment-service/payments/iap/apple/notifications` |
| ASC → Business | 403 for the developer account | An Account Holder/Admin/Finance user confirms the Paid Apps agreement is active |
| Play app | Weup Book `com.weup.book`, in Production; no subscriptions | Create them (below) |
| Play → Monetization setup | RTDN **on**, topic `projects/weup-elearning/topics/iap-rtdn`, content "Subscriptions, voided purchases, and all one-time products"; **subscription Pause enabled** | RTDN is fine as is. Turn **Pause off**; v1 doesn't handle paused subscriptions |

**App Store Connect**
1. My Apps → WeUp → **Subscriptions** → create group "WeUp Membership" + group localization (all
   markets' languages).
2. Per plan: create an auto-renewable subscription with **Product ID = SKU**, duration, price
   (schedule per storefront for WeUp markets), localization (display name + description), review
   screenshot and review notes.
3. Order the plans by **level** (1 = highest). It must match `store_level`.
4. **Billing Grace Period:** on (e.g. 16 days, for all renewals). Core maps it via `SUB_GRACE`.
5. App Information: Privacy Policy URL; set the Terms of Use / EULA (standard Apple EULA or a custom
   one). Add the Terms link to the app description too.
6. Family Sharing: **off**.
7. Submit the first subscriptions **with the app version that sells them** (attach them on the
   version page). After that, new plans can be submitted alone.
8. Sandbox testers (in a supported region). Sandbox renews fast: 1 month ≈ 5 minutes, up to 12
   renewals a day.

**Play Console**
1. Monetize with Play → Products → **Subscriptions** → create. **Product ID = SKU** (it can't be
   changed or reused later). Fill in name and benefits.
2. Add **one base plan**: auto-renewing, billing period = plan duration, regional prices; grace
   period and account hold on; resubscribe allowed. Activate it. **No offers.**
3. Monetization setup → RTDN: confirm the topic and send a test notification.
4. License testers: test subscriptions renew on a short cycle (monthly ≈ 5 minutes).
5. Upload the build to internal testing; testers must install from the Play link.

**WeUp mapping:** in CMS, map each plan → SKU (`productType: subscription`) and set `store_level`.

---

## 7. Test matrix (sandbox on both stores; then a production build with a sandbox account for App Review)

- [ ] New purchase → one membership row, one order, correct `expires_at`, trial ended if active, Google acknowledged after the grant
- [ ] Renewal ×2+ → same row extended, one order per period (the Google 2nd-renewal dedupe bug is fixed)
- [ ] Upgrade (iOS level up / Android `withTimeProration`) → scope switches now; **no second Google subscription**
- [ ] Downgrade (deferred) → old scope until renewal, new scope after
- [ ] Turn off auto-renew → access kept, UI "Ends on…", expires → cron revokes, progress kept
- [ ] Billing failure → grace → recover; grace → expire
- [ ] Refund / revoke → immediate termination; products bought outright are kept
- [ ] Late renewal webhook → no access gap (24 h leeway + reconcile)
- [ ] Kill the app after paying, before verify → verify/finish on next launch; Google auto-refunds if the grant never happens (3-day acknowledge)
- [ ] iOS renewal on a second device with no saved context → verified via SKU, finished
- [ ] Account A buys, B logs in on the same device → B gets nothing
- [ ] Payoo member → store purchase blocked in the app; forced verify → `rejected_cross_channel` + alert
- [ ] Restore on a fresh install (iOS) → membership visible
- [ ] Unsupported region → `unsupportedRegion`
- [ ] `ENABLE_STORE_SUBSCRIPTIONS=false` → today's screens unchanged

---

## 8. Release order

1. payment P1–P2 (also unblocks the ebook App Review) → deploy.
2. payment P3–P8 + core §4 (contract v2 in **both repos in the same release**; run the migration by
   hand as usual) → deploy to staging.
3. Store products + CMS mappings on staging/sandbox.
4. Mobile behind `ENABLE_STORE_SUBSCRIPTIONS=false` → internal testing / TestFlight → flag on in
   dev/staging → full §7 matrix.
5. Production: backend live, mappings in place → submit the iOS version with its subscriptions →
   Play production rollout → flag on.

Rollback: turn the flag off in a new build. The server-side kill switch (B5) stops new purchases
without a release. **Never** remove a mapping while live subscribers exist, because renewals
resolve through it.
