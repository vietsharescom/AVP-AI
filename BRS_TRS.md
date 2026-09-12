# AVP Packing Flow — Business & Technical Requirements

Nguồn: đối chiếu trực tiếp với dữ liệu thật trong `PACKING SLIPS.xlsm` (sheet
`WORK ORDER`, `PartControl`, `PACKING SLIP`) và `DAILY LOG CHECK SHEET.xlsm`
(sheet `CHECKING SUMMARY`, `WORK ORDER`). Xác nhận/sửa lại các rule nháp
trong `BRS.md` bằng dữ liệu thật — chỗ nào khác với bản nháp gốc có ghi chú
"⚠ khác bản nháp".

## 1. Bốn thực thể (bảng)

| Ký hiệu | Tên nghiệp vụ | Sheet thật | Vai trò |
|---|---|---|---|
| **T** | Traveler / Raw Material | `WORK ORDER` | Infasco giao nguyên liệu — 1 dòng = 1 traveler + 1 pot |
| **S** | Scanning / Finish Good | `CHECKING SUMMARY` | Nhân viên ghi tay khi lựa/đóng gói xong — 1 dòng = 1 lần scan 1 traveler |
| **P** | Packing Slip | `PACKING SLIP` (Table1) | Chứng từ xuất cho Infasco — ghép T + S |
| **PC** | Part Control | `PartControl` | Bảng tra cứu: mỗi Part# có "Quantity per box" cố định + Client + Machine mặc định |

## 2. Cột và ánh xạ (đã xác minh với header thật)

**T — WORK ORDER**: `T1 TRAVELER, T2 PART#, T3 POT, T4 MACHINE, T5 WEIGHT, T6 PIECES, T7 PO, T8 DATE, T9 LOT NO., T10 NOTE, T11 SHIPPED, T12 PS`

**S — CHECKING SUMMARY**: `S1 DATE, S2 SHIFT, S3 Oprtr, S4 TRAVELER, S5 PART#, S6 POT#, S7 LOT#, S8 Type, S9 O/C, S10 Machine, S11 MC#, S12 S/P, S13 Special Notes, S14 No.of boxes, S16 TTL QNT, S18 SKID#, S19 LOCATION`

**P — PACKING SLIP (Table1)**: `P1 PO NO., P2 PART NO., P3 TRAVELER, P4 POT NO, P5 DESCRIPTION, P6 BOX, P7 QUANTITY, P8 INV DATE, P9 PS, P10 NOTES`

**PC — PartControl**: `PC1 Part, PC2 Quantity per box, PC3 Client, PC4 Machine, PC5 Label Format`

### Ánh xạ khóa nối (verified — có 2 chỗ sửa so với BRS.md nháp)

| Ý nghĩa | T | S | P | Ghi chú |
|---|---|---|---|---|
| Traveler# | T1 | S4 | P3 | khớp bản nháp |
| Part# | T2 | **S5** | P2 | ⚠ bản nháp ghi "S2" — thật ra Part# là **S5**, S2 là SHIFT |
| Pot# | T3 | S6 | P4 | khớp bản nháp |
| PO# | T7 | — | P1 | P1 lấy qua VLOOKUP(Traveler, T, cột PO), S không có cột PO |
| Boxes | — | S14 | P6 | khớp bản nháp (No. of boxes = BOX) |
| Quantity | — | S16 (tham khảo) | **P7 = P6 × PC2** | ⚠ bản nháp ghi "S16=P7" (copy thẳng) — thật ra P7 là **công thức tính**, không copy trực tiếp từ S16. S16 (TTL QNT lúc scan) chỉ dùng để **đối chiếu chéo**, không phải nguồn của P7. Bằng chứng: công thức thật trong PACKING SLIP.xlsm là `=F{row}*VLOOKUP(PART NO., PartControlTable, 2, FALSE)` |
| Notes | T10 | — | P10 | VLOOKUP(Traveler, T, cột NOTE) |

## 3. Ràng buộc cardinality (quan hệ 1-nhiều)

- **1 PO → nhiều Traveler** (1 đơn hàng Infasco chia nhiều pot/traveler).
- **1 Traveler → đúng 1 Part#, đúng 1 Pot#** (traveler là khóa duy nhất định danh 1 lô nguyên liệu cụ thể).
- **1 Part# → nhiều Traveler** (cùng 1 mã hàng có thể xuất hiện ở nhiều lô/nhiều PO khác nhau theo thời gian).
- **1 Part# → nhiều Pot# khác nhau** (mỗi lần giao là 1 pot riêng dù cùng Part#).
- **1 Part# → đúng 1 giá trị "Quantity per box" cố định** trong PartControl (không đổi theo traveler/pot, chỉ đổi khi đổi Part#).
- **1 Traveler (T) → 0 hoặc nhiều dòng Scanning (S)**: về lý thuyết 1 traveler chỉ scan 1 lần, nhưng thực tế có thể bị trùng (xem mục 6 — sự cố dữ liệu đã gặp trong phiên làm việc trước, do lưu lặp khi mạng lỗi).
- **1 Traveler đã dùng trong 1 Packing Slip (P) → không được dùng lại ở Packing Slip khác** (tránh xuất trùng 1 lô hàng 2 lần) — đây là ràng buộc SHIPPED=TRUE sau khi duyệt.

## 4. Công thức DESCRIPTION (P5) — làm rõ theo dữ liệu thật

Bản nháp: `P5 = S3+S11+S12+S8+S10`. Đối chiếu với các DESCRIPTION thật đã
xuất (ARCHIVE, packing slip đã gửi Infasco), công thức đúng là ghép theo
**thứ tự** sau:

```
P5 = S3 (Oprtr) + " " + S10(Machine)+S11(MC#) + " " + S12(S/P)
     + [nếu S13(Special Notes) có giá trị thì thêm nguyên văn, in hoa]
```

Ví dụ đối chiếu thật: Oprtr=275, Machine=BF, MC#=103, S/P=S/P →
`"275 BF103 S.P"`. Traveler có ghi chú "CERTS" → thêm `"CERTS"` ở cuối.

**Đã đính chính (2026-09-11)**: bản phân tích trước có suy đoán **S8 (Type,
Nut/Blt-Scrw) được cộng vào DESCRIPTION dưới dạng chữ "BOLTS"** — dựa trên
duy nhất 1 dòng ví dụ ("157/297 TB1 S.P BOLTS CERTS"), **không có bằng
chứng chắc chắn**. Theo xác nhận trực tiếp từ AVP: Type/Nut **thường không**
được cộng vào DESCRIPTION. Nhiều khả năng "BOLTS CERTS" trong ví dụ đó chỉ
là **chữ tự do nhân viên viết tay vào Special Notes (S13)** — cùng loại với
"MIXED", "CERTS", "JCK TGT" ở các dòng khác — không phải quy tắc hệ thống
dựa trên S8. **Kết luận: S8 (Type) KHÔNG tham gia công thức DESCRIPTION.**

Đây vẫn là **best-effort draft** — nhân viên luôn phải xem lại trước khi
duyệt Packing List, vì không có cách nào suy ra 100% chính xác chỉ từ dữ
liệu số.

## 5. Công thức QUANTITY (P7)

```
P7 = P6 (BOX) × PC2 (Quantity per box, tra theo P2/PART NO. trong PartControl)
```

**Điều kiện tiên quyết**: Part# trên Packing List phải khớp được với PartControl
(xem mục 6). Nếu không khớp, không có cách tính P7 đáng tin cậy — hệ thống
hiện tại (`src/app/api/packing-list/generate/route.ts`) sẽ:
1. Thử khớp chính xác.
2. Thử bỏ dần hậu tố sau dấu `-` (ví dụ `11546367-MY-AK-L` → `11546367-MY-AK`).
3. Nếu vẫn không khớp: dùng tạm QUANTITY đã ghi lúc scan (S16) và **cảnh báo rõ** cho nhân viên.

## 6. Phân tích lệch Part# — toàn bộ dữ liệu thật (1853 dòng WORK ORDER)

| Loại khớp | Số dòng | Tỷ lệ |
|---|---|---|
| Khớp thẳng với PartControl | 1702 | 91.8% |
| Khớp sau khi bỏ hậu tố (mạ/vật liệu) | 96 | 5.2% |
| **Không khớp được (21 mã khác nhau)** | **55** | **3.0%** |

### 21 mã Part# hoàn toàn chưa có trong PartControl (cần bổ sung tay hoặc xác nhận là mã mới)

```
NU3516-00 (x9)      61273 (x6)          SL2404-00 (x5)      61525 (x4)
P7241950-IN-PN (x4) W20-1028-010 (x3)   06513514AA (x3)     06512732AA (x2)
H62534A (x2)        W20203-S440-PA (x2) W520203-S450B (x2)  06512349-AA (x2)*
1K2GZ0-00-A (x2)    06508200AA-HT (x2)  NU3034-02 (x1)      06513734AA (x1)
1K2GZ0-00 (x1)      24576407-B (x1)     1154389-PH-RO (x1)  7150-1710-03 (x1)
06104388A (x1)
```

`*06512349-AA` thực ra khớp `06512349AA` trong PartControl nếu bỏ dấu `-`
— đây là lỗi định dạng (thừa dấu gạch ngang), không phải mã mới.

**Nhận xét quan trọng**: `1K2GZ0-00` và `1K2GZ0-00-A` **cả 2 đều không có**
— khác với các case bỏ-hậu-tố khác (nơi bản GỐC luôn có sẵn), gợi ý đây là
**cả 1 dòng Part# hoàn toàn mới**, chưa từng được thêm vào PartControl, chứ
không phải chỉ thiếu 1 biến thể hậu tố.

**Khuyến nghị**: 21 mã này cần được người phụ trách PartControl (kế toán/
QC) xác nhận Quantity-per-box và thêm vào bảng — hệ thống không thể tự suy
đoán con số này.

## 7. Việc code hiện tại cần bổ sung (Technical TODO)

1. ~~Thêm S8(Type)→"BOLTS" vào DESCRIPTION~~ — **đã loại bỏ**, không có bằng
   chứng, xem mục 4.
2. **Công thức đối chiếu chéo (P7 vs S16)** — khi tạo Packing List, so sánh
   `P7` (BOX × Quantity/box tính từ PartControl) với `S16` (TTL QNT nhân
   viên ghi lúc scan traveler đó). Nếu lệch nhau quá ngưỡng (vd >10%), cảnh
   báo rõ trên dòng đó — đây là cách phát hiện sai sót ở cả 2 đầu (PartControl
   sai Quantity/box, HOẶC nhân viên ghi sai lúc scan, HOẶC AI đọc sai BOX).
   **[Đã triển khai — xem `src/app/api/packing-list/generate/route.ts`]**
3. Thêm cơ chế phát hiện **Traveler bị scan trùng** (nhiều dòng Finish Good
   cùng Traveler chưa Shipped) trước khi cho vào Packing List — đã gặp sự cố
   thật trong phiên làm việc trước (traveler 718440, 717671 mỗi cái bị trùng
   2 dòng do lưu lặp lúc lỗi mạng).
