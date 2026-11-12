# Vận hành gói thành viên (Subscription) trên App Store & Google Play

Bộ hướng dẫn vận hành cho ticket **EP-2942 — [Mobile] Check list publish Subscription**. Mục tiêu: đưa gói thành viên
bán qua App Store / Google Play lên production và làm tài liệu vận hành lâu dài. Ảnh chụp màn hình có khung đỏ ở chỗ cần thao tác, số trong vòng tròn đỏ là thứ tự bước.

Trạng thái ghi nhận ngày **12/11/2026**. Màn hình CMS chụp trên staging. ASC và Play Console chụp trên app thật.

| # | Tài liệu | Ai đọc |
|---|---|---|
| 1 | [Cấu hình gói trên CMS](1-cms-cau-hinh-goi.md): khởi tạo, dừng, sửa giá, mở thị trường mới | Admin vận hành, PO |
| 2 | [Kiểm tra gói trên App Store Connect / Google Play Console và gửi duyệt](2-kiem-tra-store.md) | Admin vận hành, dev mobile |
| 3 | [Web E-learning: chuyển mua gói sang app](3-web.md) | PO, dev web, CSKH |
| 5 | [Đối soát doanh thu với Apple / Google](5-doi-soat.md) | Kế toán (phần A), dev/data (phần B) |
| 6 | [Checklist go-live production](6-go-live-checklist.md) | Tech lead, dev backend/mobile |
| 7 | [Dọn dẹp sản phẩm test trên store](7-don-dep-san-pham-test.md) | Admin vận hành, dev |

Mục 4 (ERP, Agent đăng sản phẩm) không nằm trong bộ này.

## Đọc trước khi thao tác

> [!WARNING]
> **Staging và production dùng chung một app trên store.** App Store Connect chỉ có app *WeupBook* (Apple ID `6816870449`),
> Google Play chỉ có *Weup Book* (`com.weup.book`). Bấm **"Mở bán"** trên CMS staging cũng **tạo sản phẩm thật** trên hai store.
> Product ID đã tạo thì không xoá hẳn và không dùng lại được. Hiện đã có 8 sản phẩm test nằm trên store, xem [mục 7](7-don-dep-san-pham-test.md).

> [!WARNING]
> **Lỗi đã biết, chưa sửa trong ticket này** (đề xuất tách ticket, chi tiết ở từng mục):
> 1. **Sai thời hạn trên store:** CMS tạo mọi gói trên store với thời hạn **1 tháng**, kể cả gói 3/6/12 tháng.
>    Trước khi sửa, **chỉ mở bán gói 1 tháng trên production**, hoặc sửa tay thời hạn trên ASC/Play trước khi gửi duyệt ([mục 2](2-kiem-tra-store.md#lỗi-thời-hạn)).
> 2. **Level trên Apple:** mọi gói CMS tạo đều ở **level 1** trên Apple, trong khi đúng ra gói dài hơn phải ở level cao hơn. Phải sắp xếp lại tay.
> 3. **"Ngừng bán" / "Xoá" / tắt thị trường** trên CMS **không gỡ gói khỏi store**. Khách store vẫn tự gia hạn.
> 4. **Thị trường lạ thành USA:** thị trường không nằm trong danh sách vn, us, en, br, id, in, mx, my, ph, th được đẩy lên Apple thành **USA**.

## Quyền & đường dẫn

| | Staging | Production |
|---|---|---|
| CMS | `https://staging.cms.weupbook.com` | `https://cms.weupbook.com` |
| Menu CMS | **Gói thành viên → Danh sách gói** (`/subscription-plans`); **Thị trường** (`/markets`) | như staging |
| App Store Connect | `https://appstoreconnect.apple.com/apps/6816870449` | (cùng app) |
| Play Console | Weup Book `com.weup.book` | (cùng app) |

Quyền CMS cần có:
- `subscription_plan:read` / `create` / `update` / `delete`
- quyền `market` (để thêm thị trường)

Quyền ở store:
- **ASC:** đọc báo cáo tài chính cần vai trò *Finance/Admin/Account Holder*.
- **Play:** "Download financial reports" và "Order management" cần quyền *View financial data* / *Manage orders*. Tài khoản dev hiện tại **chưa có** các quyền này, xem [mục 5](5-doi-soat.md).

## Tài liệu kỹ thuật liên quan

- [`10-iap.md`](../10-iap.md): bản đồ IAP giữa các repo.
- [`11-iap-subscription-guide.md`](../11-iap-subscription-guide.md): thiết kế subscription.
- [`12-iap-subscription-contract.md`](../12-iap-subscription-contract.md): contract v2 (bản ràng buộc).
