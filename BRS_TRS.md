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

## 6b. Nghiệp vụ mới xác nhận 2026-09-13 — đọc trực tiếp PACKING SLIPS.xlsm

- **Mô hình nghiệp vụ = "Subcontracting" (gia công thuê ngoài)** theo đúng
  thuật ngữ ERP chuẩn (xác nhận qua Odoo thật, module `mrp_subcontracting`):
  Infasco vẫn là CHỦ SỞ HỮU nguyên liệu trong lúc nó nằm ở AVP (không phải
  AVP mua nguyên liệu) — nguyên liệu giao trong **thùng POT** (tài sản tái
  sử dụng của Infasco).
- **Vòng đời 1 Pot#**: nhận (có nguyên liệu) → rỗng (đã lựa/đóng gói hết) →
  **phải trả POT rỗng lại cho Infasco** trên chính chuyến xe xuất hàng. Hiện
  app **CHƯA track bước "đã trả POT rỗng"** — có thể tính được vì Pot# đã
  1:1 với Traveler (xem mục 3), nhưng chưa làm.
- **2 số trên header Packing Slip thật (`PACKING SLIP` sheet, ô B12/B13),
  cả 2 đều GÕ TAY, không có công thức nào tính ra**:
  - `Total Pallets` = số pallet **thành phẩm** (boxes đã đóng gói) chất lên
    xe để giao cho Infasco.
  - `Total Empty` = số **POT rỗng** trả lại Infasco (không phải đổi pallet
    logistics như suy đoán ban đầu).
- **Cột NOTES (J) trên Packing Slip** = `VLOOKUP(Traveler, WORK ORDER, 10)`
  — lấy đúng cột "NOTE" của WORK ORDER, chứa ghi chú chất lượng/xử lý đặc
  biệt theo TỪNG traveler (vd thật: "SORT FOR DAMAGED LOCKING-DMF FORM",
  "0 QUANTITY-INFASCO KEEP THE TRAVELER", "WAS TR#715868-...") — khác với ô
  C12 (ghi chú tự do gõ tay kiểu "Traveler-Part#-Máy Boxes CTNS", không có
  công thức, có thể là note pallet mix, chưa xác nhận 100%).
- **AVP thực ra phục vụ hàng chục khách hàng khác ngoài Infasco** (xác nhận
  qua bảng `client` thật trong PartControl: Jobal, L&M Fasteners, Thompson
  Fasteners, Ultra Form, Dana Canada, CMI, Stelfast...) — toàn bộ app
  AVP_AI hiện tại **chỉ scope cho Infasco**. Giải thích luôn lý do dãy số
  Packing Slip không chạy liên tục +1 (khách khác dùng chung dãy số).
- **Dữ liệu lịch sử `ARCHIVE`/`ARCHIVE2` trong PACKING SLIPS.xlsm chất
  lượng kém** (đọc trực tiếp, không suy đoán): cột không có tên, trộn lẫn
  nhiều loại giao dịch (xuất hàng/trả hàng/chuyển nội bộ/giao mẫu) không có
  cột phân loại, sai kiểu dữ liệu trong cột số (vd "BIN" trong cột BOXES),
  và `ARCHIVE2` là 1 cột duy nhất nhét nhiều dòng chữ tự do không parse
  được bằng máy. App AVP_AI hiện tại đã tránh được các lỗi này (mỗi cột có
  kiểu rõ ràng, mỗi dòng atomic) nhưng vẫn thiếu field "loại giao dịch" nếu
  sau này cần track return/internal-move/sample.

## 6c. Ước tính labor-hour gõ tay WORK ORDER (2026-09-13, ước tính kỹ thuật — CHƯA đo thật)

- Dữ liệu thật: trung bình **78,5 traveler/ngày** (113 ngày mẫu thật từ
  `SUMMARY IN&OUT`, dao động 45-98/ngày). WORK ORDER xác nhận **0/1.853
  dòng** ở cả 10 cột (TRAVELER/PART#/POT/MACHINE/WEIGHT/PIECES/PO/DATE/
  LOT NO./NOTE) có công thức — **100% gõ tay**.
- **Giả định** (chuẩn ngành data-entry phổ biến, KHÔNG phải đo thật tại
  AVP — cần đo lại bằng bấm giờ thật nếu dùng cho báo cáo chính thức): 8-15
  giây/ô (tìm trên giấy + gõ + kiểm tra lại).
- **Ước tính riêng WORK ORDER**: 78,5 traveler × 10 cột = ~785 ô/ngày →
  **~1,7-3,3 giờ/ngày** (trung bình ~2,2 giờ/ngày) chỉ để gõ tay bảng gốc
  này — CHƯA tính Scanning Sheet/CHECKING SUMMARY (~20 cột/traveler), copy
  tay qua ARCHIVE2, hay gõ Packing List mỗi chuyến xe.
- **Ước tính tổng cả pipeline thủ công**: có thể lên tới **~4-6+ giờ/ngày**
  (~1.000-1.500 giờ/năm, 250 ngày làm việc) thời gian hành chính thuần túy
  — chưa kể công lựa/đóng gói vật lý.

## 6d. Ước tính labor-hour TOÀN pipeline hành chính (2026-09-13, ước tính — CHƯA đo thật)

Mở rộng mục 6c theo đúng 5 bước Andy mô tả (nhận PO → lập WORK ORDER → gửi
SX/QC dán Split Form → thu gom + gõ lại → đối chiếu kho quyết định "good" →
gõ Scanning Sheet → gõ Packing List + đếm Pallets/Empty):

| Bước | Việc | Ước tính gõ thuần/ngày |
|---|---|---|
| 1. Lập WORK ORDER (PO→Traveler) | 10 cột × 78,5 traveler/ngày (0% công thức, đã xác nhận) | ~2,2 giờ |
| 3. Thu gom Split Form giấy + cập nhật | Đi lại thu gom (không phải gõ) | ~0,5-0,75 giờ |
| 4. Đối chiếu kho + gõ Scanning Sheet | ~15 cột thực dùng × 78,5 (19/20 cột CHECKING SUMMARY xác nhận 0% công thức, chỉ "Off" có) | ~3,3 giờ |
| 5. Gõ Packing List + đếm Pallets/Empty | Chỉ 2 ô/traveler (PO/Part#/Notes tự VLOOKUP) | ~0,5 giờ |
| **Tổng gõ thuần** | | **~6,5-7,3 giờ/ngày** |

**Áp hệ số "mò tìm số liệu" Andy đề xuất (2-3 lần thời gian gõ)** — dò
VLOOKUP qua 3 sheet, đọc hiểu chữ tự do (SORT&RETURN/comma-list Reject),
kiểm tra công thức sống hay đã khóa cứng, đối chiếu kho vật lý:

```
Tổng thời gian thật = Gõ + Mò = ~7h + (2-3)×7h = ~21-28 giờ/ngày
```

→ Tổng khối lượng công việc (chia nhiều nhân viên: kế hoạch+QC+kho, không
phải 1 người), quy năm (250 ngày): **~5.250-7.000 giờ/năm** chỉ phần hành
chính/giấy tờ, chưa tính công lựa/đóng gói vật lý. Vẫn là ước tính có giả
định rõ ràng (tốc độ gõ 8-15s/ô + hệ số mò 2-3 lần) — cần đo thời gian thật
nếu dùng cho báo cáo chính thức với chủ.

## 6e. Phân tích thêm các sheet phụ (2026-09-13) — phát hiện lớn: 2 "nguồn sự thật" đã lệch nhau THẬT

- **`PACKING SLIPS.xlsm` và `DAILY LOG CHECK SHEET.xlsm` mỗi file có 1 bảng
  "WORK ORDER" RIÊNG** (1.847 vs 5.224 traveler). Trong số traveler có ở cả
  2 file, **797 traveler bị lệch SHIPPED/PS** — ví dụ thật: traveler 704978
  file 1 nói PS=29979, file 2 nói PS=30072; traveler 701128 file 1 nói CHƯA
  xuất, file 2 nói ĐÃ xuất. **Bằng chứng cụ thể cho rủi ro "2 sổ cái không
  đồng bộ"** đã cảnh báo ở các mục trước — đây không còn là giả thuyết, là
  lỗi thật đang tồn tại trong dữ liệu lịch sử.
- **`Sheet1`** (PACKING SLIPS.xlsm, 49 dòng): bảng nháp tạm cho batch mới
  (cùng PO/ngày) trước khi merge tay vào WORK ORDER chính — thêm 1 bước
  copy tay nữa ngoài ARCHIVE2.
- **`Sheet2`**: template rỗng cùng cấu trúc Table1 (Packing Slip) — khả
  năng dùng để reset Table1 mỗi ngày.
- **`Status`** — có sẵn danh sách trạng thái sản xuất chi tiết hơn đã từng
  thiết kế: `Loaded, Not started, Preloaded, Running, Shipped` — nhưng quét
  toàn bộ workbook chỉ thấy xuất hiện trong chính sheet Status, **chưa bao
  giờ được dùng thật** ở nơi khác — tính năng thiết kế rồi bỏ dở.
- **`Status` cột Machine — DANH SÁCH MÃ MÁY THẬT HỢP LỆ (nguồn dữ liệu gốc
  đã tìm được cho gap "bảng quy ước MC#" nêu ở mục 8 ARCHITECTURE.md)**:
  BF 102-112 (11 mã), MC 4/16/19/24/26/37/51/61/74/78/112 (11 mã), PCKY,
  TABLE 1, TABLE 2. Có thể dùng để validate MC#/Machine đọc từ OCR.
- **`SHIPPED`** (DAILY LOG CHECK SHEET.xlsm, 32 dòng): 1 danh sách tay
  KHÁC nữa track traveler đã xuất (TR/Part#/Pot#, không có PS/ngày) — sổ
  sách thứ 3 cho cùng 1 khái niệm "đã xuất chưa".
- **`QuantityControl`** (1.010 dòng): bảng Part#→Quantity tương tự
  PartControl nhưng có vẻ là bản cũ/khác — cũng xác nhận thêm nhiều khách
  hàng khác Infasco (S & E MANUFACTURING, JTF DAVCO, Davco...).
- **`Crosscheckk`**: chỉ 1 dòng dữ liệu, toàn bộ ô ghi literal **"Please
  enter"** — 1 tính năng đối chiếu dự định làm nhưng bị bỏ dở hoàn toàn,
  chưa từng dùng thật.

**Kết luận chung**: hệ thống Excel cũ có xu hướng **tạo bảng mới mỗi khi
cần 1 tính năng**, không bao giờ hợp nhất lại — dẫn tới nhiều "nguồn sự
thật" song song cho cùng 1 khái niệm (SHIPPED có ít nhất 3 nơi ghi khác
nhau) và đã THẬT SỰ lệch nhau (797 ca). Đây là lý do cốt lõi nhất ủng hộ
đề xuất "1 sổ cái duy nhất" ở mục 8 ARCHITECTURE.md — không phải lý thuyết
suông, mà để tránh chính xác loại lỗi đã xảy ra thật này.

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
