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

> **Ghi chú thuật ngữ (2026-09-12, không đổi field/code):** cột `PO` (T7/P1)
> trong Excel/Sheets/code **giữ nguyên tên** như owner đang dùng. Nhưng về
> bản chất nghiệp vụ, đây **không phải "Purchase Order" AVP đi mua hàng** —
> nó là **đơn hàng gia công đầu vào** Infasco giao cho AVP sản xuất (gần
> với khái niệm Sales/Work Order phía AVP theo mô hình ERP). Hiểu đúng bản
> chất này khi đọc code/tài liệu, nhưng KHÔNG đổi tên cột `po` ở bất cứ đâu
> (Google Sheets, API, UI) trừ khi owner yêu cầu riêng.
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

> **LOT NO. không xuất hiện trên Packing Slip** (đã xác minh lại `PACKING
> SLIP-1.pdf`, 2 trang thật — chỉ có PO NO/PART NO/TRAVELER/POT NO/
> DESCRIPTION/BOX/QUANTITY/INV DATE, không có cột LOT). LOT NO. (T9/S7)
> chỉ dùng làm khóa tracking nội bộ (Traveler#, LOT NO.) — xem mục 3 — không
> cần thêm cột LOT vào P (PackingList).

## 3. Ràng buộc cardinality (quan hệ 1-nhiều)

**Chuỗi phân cấp đã xác nhận với owner (2026-09-12):
`1 PO → nhiều Traveler → nhiều LOT NO.`**

- **1 PO → nhiều Traveler** (1 đơn hàng Infasco chia nhiều pot/traveler) —
  xác nhận lại bằng dữ liệu thật (`WORK ORDER`, vd PO 193890 có nhiều dòng
  Traveler khác nhau).
- **1 Traveler → đúng 1 Part#, đúng 1 Pot#** (traveler là khóa duy nhất định danh 1 lô nguyên liệu cụ thể).
- **1 Traveler → ĐÚNG 1 LOT NO.** (⚠ **cập nhật 2026-09-12 (lần 2), ĐÍNH CHÍNH
  giả định "hợp lệ" bên dưới** — đo lại bằng dữ liệu thật, không suy đoán
  theo lý thuyết ERP nữa): phân tích toàn bộ 1.811 dòng thật trong
  `CHECKING SUMMARY` (`DAILY LOG CHECK SHEET.xlsm`), nhóm theo Traveler# và
  so khớp LOT# (đã chuẩn hóa bỏ khác biệt hoa/thường — ~5,6% "khác nhau" ban
  đầu hóa ra chỉ là lỗi hoa/thường, không phải lot thật khác nhau) —
  **1.598/1.598 traveler (100%) chỉ có ĐÚNG 1 LOT NO., KHÔNG một ngoại lệ
  hợp lệ nào**. Tương tự, **Traveler → Part# và Traveler → Pot# cũng 100%
  1:1**. Ngược lại, LOT# → Traveler có 3/1.595 ca (0,19%) 1 LOT# dùng chung
  bởi 2 traveler khác nhau — nên khóa (Traveler#, LOT NO.) vẫn đúng, không
  đổi.
  - **Đính chính GAP-004 (traveler 717671)**: giả định trước đó ("có thể là
    2 lot hợp lệ theo lý thuyết ERP") **không còn đứng vững** — với bằng
    chứng 100% trên 1.598 traveler thật, trường hợp 717671 (LOT NO.
    `6-253-13-A` vs `6-251-13-A`, Part# cũng lệch 1 ký tự giữa 2 dòng)
    **gần như chắc chắn là lỗi đọc/scan trùng**, không phải lot hợp lệ thứ
    2. Vẫn **CHƯA XÓA gì** trong Sheet — owner cần xem chứng từ giấy gốc để
    quyết định cuối cùng (xem LATEST_SESSION.md GAP-004).
  - **Hệ quả thiết kế**: khóa đúng để định danh 1 "lần sản xuất/đóng gói"
    vẫn là **(Traveler#, LOT NO.)**, không chỉ Traveler# — ảnh hưởng logic
    đánh dấu Shipped. Đã thêm cảnh báo mạnh trong
    `finish-good/confirm/route.ts` khi 1 traveler còn mở (chưa Shipped) xuất
    hiện LOT NO./Part#/Pot# khác với dòng đã lưu trước đó cho cùng traveler.
  - **Chưa xác nhận** (cần owner làm rõ): quan hệ **Machine/MC# + Ca + Ngày →
    Traveler** — dữ liệu thật cho thấy 5,66% (97/1.714 nhóm) có NHIỀU
    traveler khác nhau dùng chung 1 MC# trong cùng ca/ngày. Không rõ đây là
    bình thường (1 máy/bàn chạy nối tiếp nhiều traveler trong ca) hay là lỗi
    thật đã có sẵn trong lịch sử — owner cần xác nhận MC# có phải 1 máy độc
    quyền hay 1 trạm dùng chung trước khi tin tưởng cảnh báo "trùng máy" đã
    thêm trong code.
  - **SKID# không phải khóa 1:1** — 77,4% skid chứa nhiều traveler khác nhau
    (kệ/pallet chứa chung nhiều lô hàng chờ xuất) — không dùng SKID# để phát
    hiện trùng lặp.
- **1 Part# → nhiều Traveler** (cùng 1 mã hàng có thể xuất hiện ở nhiều lô/nhiều PO khác nhau theo thời gian). Xác nhận lại bằng dữ liệu thật (2026-09-12, owner quan sát trực tiếp trên Traveler form): Part# `HL3W-4320-AA-L` xuất hiện ở 3 Traveler khác nhau (714250, 714254, 714255), mỗi traveler 1 Pot#/Weight/Pieces riêng, **cùng 1 ngày, cùng PO** — nhiều Traveler cùng Part# thường xuất hiện liền kề nhau trên form 01 do cùng 1 lô giao hàng bị chia nhỏ.
- **1 Part# → nhiều Pot# khác nhau** (mỗi lần giao là 1 pot riêng dù cùng Part#).
- **1 Part# → đúng 1 giá trị "Quantity per box" cố định** trong PartControl (không đổi theo traveler/pot, chỉ đổi khi đổi Part#).
- **1 Traveler đã dùng trong 1 Packing Slip (P) → không được dùng lại ở Packing Slip khác** (tránh xuất trùng 1 lô hàng 2 lần) — đây là ràng buộc SHIPPED=TRUE sau khi duyệt. Cần xem lại có nên áp dụng theo (Traveler#, LOT NO.) thay vì chỉ Traveler# hay không.

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
4. **[ĐÃ SỬA 2026-09-12] Bug thật — PDF Scanning Sheet bị đọc lộn dòng do
   trang PDF nhúng SIDEWAYS**: file `SCANNING SHEET-1.pdf` (Wrapping Summary
   in từ hệ thống, không phải ảnh chụp tay) có trang khổ dọc (612×792) nhưng
   nội dung thật là bảng khổ ngang, không có cờ `/Rotate` nào để tự sửa. Khi
   `splitPdfPages` (`src/lib/extract.ts`) tách trang gửi thẳng cho Gemini,
   model phải đọc bảng dày đặc nằm nghiêng 90° → đọc lộn 2 dòng traveler kề
   nhau (717770 ↔ 717797 — traveler 717770 bị MẤT HẲN, dữ liệu của nó lẫn
   vào dòng 717797). Đối chiếu Sheet thật cũng tìm thấy 1 ca lỗi y hệt đã có
   sẵn: **traveler 717771**, Pot# lưu sai thành 282 (đúng phải là 18 theo
   RawMaterial). **Đã sửa**: `splitPdfPages` giờ tự phát hiện trang khổ dọc
   và xoay 270° trước khi gửi AI — đã test lại bằng Gemini thật trên đúng
   trang lỗi, cả 717770 và 717797 đọc đúng 100% sau khi sửa. Đã thêm thêm
   lưới an toàn: cảnh báo mạnh trong `finish-good/confirm/route.ts` nếu
   Pot# scan được khác Pot# đã biết ở RawMaterial cho cùng traveler (dựa
   trên cardinality 1:1 tuyệt đối đã xác minh, xem mục 3).
   - **Chưa xử lý**: traveler 717771 hiện vẫn sai trong Sheet, chờ Andy xác
     nhận có sửa lại Pot# 282→18 không.
   - **Lưu ý UI**: nút upload chính ở khâu 3 (`ScanSection.tsx`, "Chụp/chọn
     ảnh tờ scanning") hiện chỉ nhận `image/*`, chưa cho chọn PDF — fix xoay
     trang này chỉ có tác dụng khi thật sự có luồng upload PDF cho bước này
     (hiện chưa bật trên UI, chỉ verify qua code trực tiếp).
