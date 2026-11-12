# 7. Dọn dẹp sản phẩm test trên store

Staging và production dùng chung app trên store, nên mỗi lần **"Mở bán"** trên CMS staging là tạo một sản phẩm thật.
Danh sách đọc ngày 12/11/2026:

| Product ID | Tên | Nguồn | ASC | Play (base plan) | Đề xuất |
|---|---|---|---|---|---|
| `weup.membership.1m` | WeUp Membership 1 Month | Tạo tay (04/11) | Prepare for Submission, level 2, 175 quốc gia, **VN 20.000đ** | `monthly-1` Active | Quyết định: dùng làm gói thật hay thay bằng gói CMS prod |
| `weup.membership.3m` | WeUp Membership 3 Months | Tạo tay | Prepare for Submission, level 1 | `monthly-3` Active | như trên |
| `plan_12m_mvuhc2yo` | 12 Tháng | CMS staging | 1 month (sai), level 1 | 0 base plan active | Không gửi duyệt |
| `plan_3m_mw8z47vy` | WeupBook | CMS staging | 1 month (sai) | `p3m` Active ("Monthly") | Không gửi duyệt; Deactivate |
| `plan_3m_mwajqt5e` | 3 Premium | CMS staging | 1 month | Active | Không gửi duyệt; Deactivate |
| `plan_3m_mw93z2mj` | Auto Create | CMS staging | 1 month | Active | Không gửi duyệt; Deactivate |
| `plan_3m_mw94wt65` | Test 1 | CMS staging | 1 month | Active | Không gửi duyệt; Deactivate |
| `plan_3m_mwaky4mn` | Test 2 | CMS staging | 1 month | Active | Không gửi duyệt; Deactivate |
| `plan_3m_mw92jp30` / `plan_3m_mw93labd` | Test Create | CMS staging | 1 month | Active | Không gửi duyệt; Deactivate |
| `plan_3m_mw946j6c` | Test Test | CMS staging | 1 month | Active | Không gửi duyệt; Deactivate |

## Cách làm

**App Store Connect**
- Gói chưa từng gửi duyệt (*Prepare for Submission*) có thể **"Delete This Subscription"** (cuối trang chi tiết gói, ảnh [2-07](images/2-07-asc-review-information.jpg)).
  Product ID đã xoá thì **không dùng lại được**, và CMS sẽ báo lỗi nếu gặp lại ID đó.
- Cách an toàn: **không xoá**, chỉ **không chọn** các gói này khi gắn vào phiên bản app gửi duyệt. Apple chỉ duyệt gói được chọn.
- Sau đó sắp level để gói thật nằm đúng thứ tự. Gói test để level thấp nhất.

**Google Play Console**
- Mở subscription → **View base plan** → **Deactivate** ([ảnh 2-22](images/2-22-play-base-plan.jpg), ô số 2).
- Base plan đã deactivate không bán cho khách mới. Subscription và Product ID vẫn còn (Google không cho xoá subscription đã có base plan được kích hoạt).
- Trước khi deactivate, kiểm tra **không có khách thật** đang subscribe gói đó (Play → subscription → số người đang subscribe; hoặc SQL B1).

> [!NOTE]
> Sau khi dọn, phía CMS staging các gói vẫn hiện bình thường. Chỉ trên store chúng ngừng bán.

## Quy tắc từ nay

1. **Chỉ mở bán trên staging khi thật sự cần test luồng store.** Tên gói phải có tiền tố `[TEST]`.
   Sau khi test xong thì Deactivate trên Play và không gửi duyệt trên ASC.
2. Ghi lại Product ID test vào bảng trên.
3. Gói thật cho production **chỉ tạo từ CMS production**.
4. Đề xuất dài hạn: tách app test riêng trên store (bundle/package khác cho staging), hoặc cho payment-service staging trỏ tới một app khác. Khi đó thao tác trên staging không còn đụng tới app thật.
