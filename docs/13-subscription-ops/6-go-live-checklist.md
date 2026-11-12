# 6. Checklist go-live production

Đánh dấu từng dòng khi xong. Cột **Hiện trạng** là giá trị đọc được ngày **12/11/2026** (chỉ đọc, không sửa gì trên store).
Thứ tự làm theo `11-iap-subscription-guide.md` §8:

**backend production → gói thật + kiểm tra store → gửi duyệt iOS → rollout Android → bật cờ mobile.**

## 6.1. Chặn go-live (phải xong trước)

| ✓ | Việc | Hiện trạng | Ở đâu |
|---|---|---|---|
| [ ] | Sửa lỗi **K1 thời hạn** + **K2 level**; hoặc chốt chỉ bán gói 1 tháng ở đợt đầu | Chưa sửa | [mục 1](1-cms-cau-hinh-goi.md#lỗi-đã-biết-của-luồng-cms-đề-xuất-tách-ticket) |
| [ ] | Xử lý lý do từ chối iOS 1.0 (*2.1.0 App Completeness*) và gửi lại kèm subscriptions | **1.0 Rejected** | ASC → iOS App ([ảnh 2-10](images/2-10-asc-version-rejected.jpg)) |
| [ ] | Gói **thật** tạo trên **CMS production** (không dùng gói staging/test), đã kiểm tra theo [mục 2](2-kiem-tra-store.md) | Chưa có | CMS prod |
| [ ] | Giá CMS = giá ASC = giá Play ở mọi thị trường. Hiện `weup.membership.1m`: ASC VN 20.000đ, CMS 59.000đ | **Lệch** | [mục 2](2-kiem-tra-store.md) |

## 6.2. App Store Connect

| ✓ | Việc | Hiện trạng |
|---|---|---|
| [x] | **Paid Apps Agreement** Active ([ảnh](images/6-04-asc-paid-apps-agreement.jpg)) | Active, 14/10/2026 – 24/09/2027 |
| [x] | **Production Server URL** = `https://api.weupbook.vn/api/v1/payment-service/payments/iap/apple/notifications` (1); Sandbox = staging (2) ([ảnh](images/6-01-asc-server-notifications.jpg)) | Đúng |
| [ ] | Xác nhận `api.weupbook.vn` (app mobile prod dùng) và `api.weupbook.com` (CMS prod dùng) **cùng trỏ tới payment-service production**. Webhook Apple đi vào `.vn` | Cần dev xác nhận |
| [ ] | **Billing Grace Period → Server Environments** chuyển sang **Production and Sandbox** ([ảnh 2-02](images/2-02-asc-grace-period.jpg), Edit) | **Only Sandbox** |
| [x] | **Streamlined Purchasing** On (theo contract v2) | On |
| [ ] | Family Sharing **tắt** ở mọi gói thật | Tắt |
| [ ] | Mỗi gói thật: localization vi + en, **review screenshot**, review notes, đúng level | Thiếu screenshot |
| [ ] | App description có link **Terms of Use (EULA)** + **Privacy Policy** (Guideline 3.1.2) | Cần kiểm tra |
| [ ] | Tài khoản sandbox cho App Review ghi trong *App Review Information* | Đã có video demo IAP, cần cập nhật cho subscription |
| [ ] | Vai trò **Finance** cho người đối soát | Cần cấp |

## 6.3. Google Play Console

| ✓ | Việc | Hiện trạng |
|---|---|---|
| [x] | **Pause = Disabled** ([ảnh](images/6-03-play-pause.jpg)) | Disabled |
| [x] | **RTDN** bật, topic `projects/weup-elearning/topics/iap-rtdn` (1), nội dung *Subscriptions, voided purchases, and all one-time products* (3) ([ảnh](images/6-02-play-rtdn.jpg)) | Đúng |
| [ ] | Trên **Google Cloud Pub/Sub**: topic `iap-rtdn` phải có **push subscription tới payment-service production** (`…/payments/iap/google/notifications`, OIDC). Sau đó bấm **Send test notification** (2) và xem log prod | Theo `12-iap-subscription-contract.md` §0: chỉ tới **staging** (chưa kiểm tra trực tiếp trên GCP) |
| [ ] | Service account của payment prod có quyền *View financial data* + *Manage orders and subscriptions* + *Manage store presence* | Cần kiểm tra |
| [ ] | Base plan gói thật: grace 7 ngày, account hold, Resubscribe Allow, **Active** | — |
| [ ] | Base plan của **gói test** đã Deactivate ([mục 7](7-don-dep-san-pham-test.md)) | 8 gói test đang Active |
| [ ] | Quyền tài chính cho người đối soát ([mục 5](5-doi-soat.md)) | Chưa có |
| [ ] | Bản Android có subscription lên track production (rollout) | — |

## 6.4. Backend production (core-service + payment-service)

| ✓ | Việc |
|---|---|
| [ ] | Deploy **payment** và **core** cùng một đợt (contract v2 `IapNotificationMessage` được copy tay ở cả hai repo) |
| [ ] | Chạy migration core: `20261029152123_create_subscription_plan_markets`, `20261105092342_iap_subscription_store_membership`, `20261109102738_add_last_synced_price_to_subscription_plan_markets`, `20261110095629_add_apple_synced_at_to_subscription_plan_markets`, `20261110143400_add_deleted_at_to_subscription_plans` (cùng các migration khác chưa chạy trên prod) |
| [ ] | Chạy migration payment: `0002_create_iap_tables`, `0003_iap_subscription_notification_id` (nếu chưa có) |
| [ ] | ENV core: `PAYMENT_SERVICE_URL`, `GOOGLE_PLAY_PACKAGE_NAME=com.weup.book`, `IAP_INTERNAL_SERVICE_KEY`, Redis/BullMQ cho queue `iap-subscription-price-apply`, `RABBITMQ_URL` |
| [ ] | ENV payment: `APPLE_IAP_*` (key App Store Server API + App Store Connect API), `GOOGLE_IAP_*` (service account, `GOOGLE_IAP_PACKAGE_NAME`), `IAP_INTERNAL_SERVICE_KEY` (giống hệt core) |
| [ ] | Gọi thử `POST …/payments/iap/apple/notifications` bằng *Request a Test Notification* (App Store Server API). Log prod nhận và verify được JWS |
| [ ] | Cron reconcile IAP chạy mỗi giờ trên prod (`IapPendingTransactionReconciliationCron`) |

## 6.5. Client

| ✓ | Việc | Hiện trạng |
|---|---|---|
| [ ] | Mobile `config/prod.json`: `ENABLE_STORE_SUBSCRIPTIONS = "true"` trong build gửi duyệt | `"false"` |
| [ ] | Mobile: paywall có disclosure 3.1.2 (giá/kỳ, tự gia hạn, cách huỷ, link Terms + Privacy) | Commit 35cfce7 bỏ link EULA/Privacy khỏi paywall, **cần kiểm tra lại** |
| [ ] | Web: merge PR `feat/subscription-download-app` ([mục 3](3-web.md)), deploy sau khi app đã có trên store | PR chờ review |

## 6.6. Sau go-live (tuần đầu)

- [ ] Một giao dịch thật mỗi store (người nội bộ mua rồi huỷ): order tạo đúng, `iap_transactions.environment = production`, Google đã acknowledge.
- [ ] Theo dõi B4 / B5 ([mục 5](5-doi-soat.md#phần-b--dev--data-sql-trên-db-core-production-chỉ-đọc)) hằng ngày.
- [ ] Đối soát tháng đầu tiên theo [mục 5](5-doi-soat.md).

**Rollback:**
- Tắt cờ mobile bằng một bản build mới.
- Ngừng bán trên CMS **và** Remove from Sale / Deactivate trên store.
- **Không xoá mapping/gói khi còn khách đang subscribe:** gia hạn của họ vẫn dựa vào gói đó.
