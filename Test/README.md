# Bộ mẫu test pipeline — PS 30098 (2026-09-09)

> Tài liệu này để Andy tự chạy tay qua UI thật (`/inbox` → `/warehouse` →
> `/scan` → `/packing-list`). Toàn bộ số liệu bên dưới lấy từ chứng từ
> thật trong `Data/Evaluation/` (PO193842.pdf, PO193853.pdf,
> `PACKING SLIP-2.pdf` = PS 30098) — **không bịa số liệu nghiệp vụ**.
> Chỗ nào không có nguồn thật (khâu 2 — số thực nhận kho) đã ghi rõ là
> "chưa có nguồn thật, tự quyết định khi test".

## Vì sao chọn PS 30098

`PACKING SLIP-2.pdf` (PS 30098, ship date 09-Sep-2026) có **32 dòng
traveler** — nhiều hơn `PACKING SLIP-1.pdf` (29 dòng). Trong số 32
traveler đó, tìm được 3 traveler có **cả PO gốc thật lẫn Packing List
thật khớp nhau 100%** (kể cả khớp đúng Quantity/box trong PartControl):

| Traveler | PO gốc | Part# (PO) | Part# (Packing List) | Box | Quantity thật | Quantity/box PartControl | Khớp? |
|---|---|---|---|---|---|---|---|
| 717844 | 193853 | `06504080-T` | `06504080-T` | 24 | 144,000 | 6,000 | ✓ 24×6000=144000 |
| 718439 | 193842 | `06104388AAHT` | `06104388AA` (bỏ hậu tố HT) | 30 | 19,800 | 660 | ✓ 30×660=19800 |
| 718440 | 193842 | `06104388AAHT` | `06104388AA` | 27 | 17,820 | 660 | ✓ 27×660=17820 |

`DESCRIPTION` thật trên Packing Slip cũng **giải mã ngược** được
Oprtr/Machine/MC#/S-P dùng thật khi xuất hàng (công thức đã xác nhận ở
`BRS_TRS.md` mục 4):

- 717844: `"190/268 TBL3 S.P"` → Oprtr=`190/268`, Machine=`Tbl`, MC#=`3`, S/P=`S/P`
- 718439: `"190/268 TBL3 S.P"` → Oprtr=`190/268`, Machine=`Tbl`, MC#=`3`, S/P=`S/P`
- 718440: `"11/252 TBL4 S.P"` → Oprtr=`11/252`, Machine=`Tbl`, MC#=`4`, S/P=`S/P`

---

## Bước 1 — Khâu 1 (`/inbox`) — dữ liệu THẬT từ PO193842.pdf + PO193853.pdf

Không có ảnh gốc từng traveler riêng — upload tạm 1 ảnh bất kỳ để mở
bảng review, rồi **gõ tay** đúng 3 dòng này (PO/Date để nguyên theo PO
gốc, không phải test giả):

| Traveler | Part# | Pot# | Weight | Pieces | Final Lot | PO | Date |
|---|---|---|---|---|---|---|---|
| 717844 | 06504080-T | 903 | 676 | 137,959 | 6-251-28-A | 193853 | 2026-09-08 |
| 718439 | 06104388AAHT | 213 | 923 | 22,349 | (trống — PO không ghi) | 193842 | 2026-09-08 |
| 718440 | 06104388AAHT | 126 | 727 | 17,603 | (trống — PO không ghi) | 193842 | 2026-09-08 |

## Bước 2 — Khâu 2 (`/warehouse`)

⚠ **Không có Split Form thật cho 3 traveler này** — không có nguồn để
lấy số thực nhận. Khi test, tự nhập số **receivedPieces** (gợi ý: dùng
đúng số Pieces dự báo ở trên nếu muốn giả định "nhận đủ như dự báo",
hoặc nhập số khác để test luôn cảnh báo lệch >5%). Ghi rõ trong Ghi chú:
`"TEST — không có Split Form thật, tự giả định"`.

## Bước 3 — Khâu 3 (`/scan`) — dữ liệu THẬT giải mã ngược từ Packing Slip

Dùng nút "+ Nhập tay 1 dòng" (không có ảnh Scanning Sheet thật cho 3
traveler này), điền:

| Traveler | Date | Oprtr | Machine | MC# | S/P | Boxes | Qty | Lot# |
|---|---|---|---|---|---|---|---|---|
| 717844 | 2026-09-09 | 190/268 | Tbl | 3 | S/P | 24 | 144000 | 6-251-28-A |
| 718439 | 2026-09-09 | 190/268 | Tbl | 3 | S/P | 30 | 19800 | (trống) |
| 718440 | 2026-09-09 | 11/252 | Tbl | 4 | S/P | 27 | 17820 | (trống) |

## Bước 4 — Khâu 4 (`/packing-list`)

Bấm Generate — kỳ vọng ra đúng DESCRIPTION + QUANTITY khớp 100% với PS
30098 thật (xem bảng đối chiếu ở trên). Vì 3 traveler khác nhau (không
trùng traveler như lần test 717365 trước), lần này **sẽ KHÔNG bị chặn**
bởi lỗi GAP-002 (chặn trùng Traveler) — dùng để so sánh hành vi bình
thường vs. case bị lỗi.

---

*Bộ mẫu này KHÔNG xóa/thay dữ liệu cũ trong Google Sheet — chỉ thêm mới,
theo đúng nguyên tắc "kho tích lũy" đã thống nhất (tham khảo `D:\Ops_Ai`).*
