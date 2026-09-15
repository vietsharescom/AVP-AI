# BẢNG DUYỆT CẬP NHẬT PARTCONTROL — 2026-09-15
## Đối chiếu file mới `Data/Partial Boxes 2026.xlsx` với PartControl đang chạy

*Mục đích: PartControl là bảng quan trọng nhất (mọi tính toán Quantity trên
Packing List đều dựa vào cột "Quantity/box" ở đây) — sai hoặc thiếu ở đây
ảnh hưởng trực tiếp số lượng ghi trên chứng từ xuất hàng thật. Bảng này chỉ
để XEM và DUYỆT — chưa sửa gì vào Sheet.*

> **Cách đọc**: cột "Đề xuất" lấy từ cột "STANDARD QUANTITY" trong 3 sheet
> tổng hợp (`INVENTORY`, `Inventory1`, `Sheet2`) của file mới — đã kiểm
> chứng trên 269 Part# thì **204 Part# (76%) khớp đúng 100%** với
> Quantity/box đang có sẵn trong PartControl, nên coi là nguồn đáng tin.
> 3 nhóm dưới đây là phần CÒN LẠI cần người duyệt.

---

## NHÓM A — 13 Part# đang THIẾU HOÀN TOÀN trong PartControl

Không có dòng nào trong PartControl — Packing List đang phải dùng tạm số
Quantity ghi tay lúc scan cho các Part# này (kém tin cậy hơn). Sắp theo số
traveler đang bị ảnh hưởng, nhiều nhất trước.

| Part# | PartControl hiện có | Ghi chú — vấn đề gì | Đề xuất thêm (Quantity/box) | Số traveler ảnh hưởng | Duyệt |
|---|---|---|---|---|---|
| `61273` | *(không có dòng nào)* | Thiếu hẳn — Packing List đang dùng số scan tay | **300** | 6 | ☐ Đồng ý ☐ Sai ☐ Cần xem giấy |
| `61525` | *(không có dòng nào)* | Thiếu hẳn | **3000** | 4 | ☐ ☐ ☐ |
| `P7241950-IN-PN` | *(không có dòng nào)* | Thiếu hẳn | **180** | 4 | ☐ ☐ ☐ |
| `11546463-TH-TC` | *(không có dòng nào)* | Thiếu hẳn — file mới cũng có 1 lần ghi lệch (480 thay vì 450, x1/3 lần) | **450** | 4 | ☐ ☐ ☐ |
| `06513514AA` | *(không có dòng nào)* | Thiếu hẳn | **360** | 3 | ☐ ☐ ☐ |
| `11546364-MY-AK` | *(không có dòng nào)* | Thiếu hẳn — số liệu rất nhất quán (10/10 lần ghi cùng 1 số) | **3500** | 3 | ☐ ☐ ☐ |
| `W20-1028-010` | *(không có dòng nào)* | Thiếu hẳn | **1450** | 3 | ☐ ☐ ☐ |
| `11546449-TH-TC` | *(không có dòng nào)* | Thiếu hẳn — chỉ có 1 lần ghi, độ tin cậy thấp hơn các dòng khác | **2500** | 3 | ☐ ☐ ☐ |
| `06512732AA` | *(không có dòng nào)* | Thiếu hẳn | **750** | 2 | ☐ ☐ ☐ |
| `1K2GZ0-00-A` | *(không có dòng nào)* | Thiếu hẳn | **4500** | 2 | ☐ ☐ ☐ |
| `H62534A` | *(không có dòng nào)* | Thiếu hẳn | **750** | 2 | ☐ ☐ ☐ |
| `11561645-MY-AK` | *(không có dòng nào)* | Thiếu hẳn — chỉ 1 traveler dùng, ít bằng chứng nhất | **4000** | 1 | ☐ ☐ ☐ |
| `24576407-B` | *(không có dòng nào)* | Thiếu hẳn — chỉ 1 traveler dùng | **1700** | 1 | ☐ ☐ ☐ |

---

## NHÓM B — 2 Part# có sẵn dòng nhưng cột Quantity/box đang TRỐNG

| Part# | PartControl hiện có | Ghi chú — vấn đề gì | Đề xuất điền (Quantity/box) | Duyệt |
|---|---|---|---|---|
| `W520111-S437` | Client: INFASCO, Label Format: 30, **Quantity/box: (trống)** | Có dòng sẵn, chỉ thiếu đúng 1 ô | **3500** (2/2 lần ghi khớp nhau) | ☐ Đồng ý ☐ Sai ☐ Cần xem giấy |
| `W520412-S442` | Client: INFASCO, Label Format: 30, **Quantity/box: (trống)** | Có dòng sẵn, chỉ thiếu đúng 1 ô | **3800** (2/2 lần ghi khớp nhau) | ☐ ☐ ☐ |

---

## NHÓM C — 6 Part# MÂU THUẪN với số đang có trong PartControl

⚠ **Nhóm rủi ro nhất — không tự sửa, bắt buộc đối chiếu giấy gốc trước khi
đổi**, vì đây là những Part# ĐANG được dùng để tính Quantity thật mỗi ngày.

| Part# | Đang có trong PartControl | File mới nói | Ghi chú — vấn đề gì | Duyệt |
|---|---|---|---|---|
| `11611960` | **8.000** | **800** | Lệch đúng 10 lần — nghi 1 trong 2 nơi bị thừa/thiếu 1 số 0 lúc gõ tay | ☐ Giữ 8.000 ☐ Đổi 800 ☐ Số khác: ____ |
| `82218` | **320** | **370** | Lệch vừa phải, không theo tỷ lệ tròn — cần xem lại | ☐ Giữ 320 ☐ Đổi 370 ☐ Số khác: ____ |
| `98055704` | **4.000** | **2.900** | Lệch lớn, không theo tỷ lệ tròn | ☐ Giữ 4.000 ☐ Đổi 2.900 ☐ Số khác: ____ |
| `11549170-MY-AK` | **400** | **280** | File mới **tự mâu thuẫn nội bộ** (280 và 400 đều xuất hiện 2/4 lần) — số 400 đang dùng trùng khớp 1 nửa số lần ghi trong file mới, có thể Part# này thật ra có 2 quy cách đóng gói khác nhau | ☐ Giữ 400 ☐ Đổi 280 ☐ Cả 2 đều đúng (2 quy cách khác nhau) |
| `P3001150` | **2.100** | **1.575** | Tỷ lệ 4:3 — nghi khác đơn vị tính (vd đếm theo lố/gross) chứ không hẳn sai | ☐ Giữ 2.100 ☐ Đổi 1.575 ☐ Số khác: ____ |
| `W722565-S900` | **1.500** | **100** | Lệch rất lớn (15 lần) — nghi Part# trong file mới thực ra là 1 quy cách đóng gói lẻ khác, không phải cùng loại | ☐ Giữ 1.500 ☐ Đổi 100 ☐ Số khác: ____ |

---

## TÓM TẮT ĐỀ XUẤT HÀNH ĐỘNG

1. **Nhóm A + B (15 Part#)**: đề xuất **duyệt và thêm luôn** — có bằng chứng
   nhất quán (khớp lặp lại nhiều lần trong file mới, và cùng phương pháp đã
   xác nhận đúng 76% trên 269 Part# khác). Rủi ro thấp nếu sai: chỉ ảnh
   hưởng Part# hiện đang PHẢI dùng tạm số scan tay — không làm tình huống
   tệ hơn hiện tại.
2. **Nhóm C (6 Part#)**: đề xuất **giữ nguyên số hiện tại trong PartControl,
   KHÔNG tự đổi** cho tới khi có người đối chiếu trực tiếp với Split
   Form/tem đóng gói thật của đúng 6 Part# này — vì đây là Part# ĐANG chạy
   thật, đổi sai sẽ làm Quantity trên Packing List sai theo.

*Sau khi có đánh dấu Duyệt ở trên, báo lại để em ghi đúng phần đã duyệt vào
Google Sheet PartControl — không ghi phần nào chưa được đánh dấu.*
