# TỔNG HỢP CÂU HỎI CẦN LÀM VIỆC VỚI OWNER — ĐỐI CHIẾU THEO CHUẨN THIẾT KẾ ERP

*Tổng hợp từ `Notes.md` (câu hỏi #1-30, tính đến 2026-09-13) và `LATEST_SESSION.md`
(GAP-001 → GAP-014). Sắp xếp lại theo 8 khối chuẩn của 1 hệ ERP sản xuất
gia công (Sales Order → Goods Receipt → Work Order → QC → Inventory/
Traceability → Shipping), để thấy rõ **khối nào đã đủ thông tin, khối nào
còn thiếu** — không sắp theo thứ tự thời gian hỏi như 2 file nguồn.*

---

## Cách đọc tài liệu này

Mỗi khối có 3 phần: **Đã chốt** (không cần hỏi lại) — **Còn thiếu/mơ hồ**
(cần Owner trả lời) — **Vì sao ERP chuẩn cần cái này** (để Owner hiểu tại
sao câu hỏi này quan trọng, không phải AI hỏi cho có).

---

## Bảng quan hệ dữ liệu (Cardinality Map)

*Tất cả đều đo trực tiếp trên dữ liệu thật (không suy đoán) — xem cột
"Bằng chứng" để biết đo trên bao nhiêu dòng.*

| Quan hệ | Cardinality | Bằng chứng (dữ liệu thật) | Ghi chú / ngoại lệ |
|---|---|---|---|
| SO/PO → Traveler# | **1 : N** (trung bình ~22, tối đa 42/PO) | 1.853 dòng WORK ORDER (`PACKING SLIPS.xlsm`) | Chiều ngược gần như 1:1 — **6/1.847 traveler (0,3%)** có 2 PO khác nhau, nghi lỗi nhập liệu (1 case còn lẫn kiểu dữ liệu số/chữ trong cùng cột PO) |
| Traveler# → Part# | **1 : 1 tuyệt đối** | 1.811 dòng CHECKING SUMMARY, 1.598/1.598 khớp | Sau khi chuẩn hóa khác biệt hoa/thường |
| Traveler# → Pot# | **1:1 tại 1 THỜI ĐIỂM**, KHÔNG phải vĩnh viễn | 1.853 dòng WORK ORDER — **469/1.141 Pot# (41,1%) bị tái sử dụng** cho ≥2 traveler khác nhau theo thời gian | ⚠️ Pot# là thùng chứa vật lý quay vòng (nhận→rỗng→trả→nhận lại) — chỉ 1:1 trong nhóm traveler CÒN MỞ (chưa Shipped) cùng lúc; tra cứu toàn lịch sử (search/`/trace`) có thể hiện nhiều traveler không liên quan cho cùng 1 Pot# — KHÔNG phải lỗi |
| Traveler# → LOT NO. | **1 : 1** (chiều thuận, với mã lot THẬT) | (bộ 1.811 dòng CHECKING SUMMARY) | Chiều ngược (LOT→Traveler) có 3/1.595 ngoại lệ thật → khóa hệ thống dùng cặp **(Traveler#, LOT NO.)**. Xem quy ước LOT NO. bên dưới — 59% giá trị LOT NO. thật ra là chữ trạng thái, không phải mã lot |
| Traveler# → SKID# | **N : M** (KHÔNG phải khóa) | 77,4% skid chứa ≥2 traveler khác nhau | Chỉ là vị trí lưu kho vật lý dùng chung — không dùng để đối chiếu/chống trùng |

**Quy ước LOT NO. (mới xác nhận 2026-09-13, đo trên 1.750 dòng có LOT NO.):**

| Loại giá trị | Tỷ lệ | Ý nghĩa |
|---|---|---|
| Mã lot thật, format `6-DDD-SS-L` (vd `6-247-03-b`) | 41,0% | Lot đã đóng gói — gần như không bao giờ trùng |
| `TRAVELERRECEIVED` | **53,8%** | Traveler đã nhận, **CHƯA đóng gói/chưa có lot** — trạng thái phổ biến nhất, KHÔNG phải lỗi |
| `SORT&RETURN` | 1,7% | Hàng trả lại, không ra lot |
| `SPLIT FROM TR#{traveler}` | ~3,6% | Ghi chú tách từ traveler khác |

✅ **Đã sửa code** (`finish-good/confirm/route.ts`, hàm `isLotPlaceholder`) — loại 3 giá trị placeholder trên khỏi so sánh "LOT có đổi khác thường không", tránh báo động giả khi 1 traveler chuyển từ `TRAVELERRECEIVED` sang mã lot thật (chuyện xảy ra ở hơn nửa số dòng, hoàn toàn bình thường).
| Part# → Quantity/box (PartControl) | **1 : 1 THẬT, mỗi hậu tố = 1 sản phẩm riêng** | ✅ **XÁC NHẬN TỪ OWNER 2026-09-13** | Đã sửa `partControl.ts` — bỏ hẳn fallback "bỏ hậu tố", chỉ giữ chuẩn hóa dấu gạch ngang thuần túy (lỗi chính tả). Hệ quả: 36/39 Part# đang hoạt động (92%) cần bổ sung Quantity/box riêng — danh sách đủ ở `BANG_MA_CAN_OWNER_DUYET.md` mục 1 |
| Machine/MC# + Ngày + Ca → Traveler | Lẽ ra 1:1 (1 máy/1 lệnh/1 ca), thực tế **N** | 5,66% dữ liệu thật dùng chung 1 MC#+ngày+ca cho ≥2 traveler | ❌ **CÒN TREO (GAP-012)** — chưa rõ là máy chạy nối tiếp bình thường hay lỗi đọc nhầm số máy |
| (Traveler#, LOT NO.) → PS# | **N : 1** (nhiều traveler/lot gộp vào 1 chuyến xe) | PS 30098 gộp 9 PO khác nhau, PS 30101 gộp 11 PO | PS = 1 chuyến xe xuất hàng, KHÔNG gắn với 1 PO cụ thể |
| Traveler# → LOT NO. → "Work Order" | Traveler# = **1 pot nguyên liệu**; **LOT NO. mới là 1 Work Order thật** | Sheet "WORK ORDER" thật không có cột mã WO riêng — Notes #8 cũ: "mỗi LOT ~ 1 Work Order riêng" | ⚠️ Sửa lại 2026-09-13 — 1 traveler có thể sinh NHIỀU LOT/WO hợp lệ (không phải luôn 1:1 như ghi trước đó) |
| Traveler# → Serial# (SSCC/box) | **1 : N** (1 label đóng gói → dải serial liên tục, 1 serial/carton) | Nhãn thật: 36 carton ↔ dải serial đúng 36 số liên tiếp (kiểm chứng bằng số học 2 lần) | ❌ **MỚI PHÁT HIỆN, CHƯA đưa vào hệ thống** — đây là mã chuẩn GS1 SSCC, đọc được trên giấy nhưng extraction hiện bỏ qua |

---

## A. Master Data (Dữ liệu nền)

**Đã chốt:**
- Khóa 1:1 tuyệt đối Traveler↔LOT#↔Part#↔Pot# (trong phạm vi hợp lệ — xem Bảng quan hệ ở trên cho các giới hạn theo thời gian/WO).
- ✅ **Xác nhận từ Owner 2026-09-13**: hậu tố Part# (`-L`/`-HT`/`-A`...) là **sản phẩm khác nhau thật**, không phải biến thể vô hại. Chỉ lỗi chính tả (thiếu/thừa dấu gạch ngang, vd `06512349AA`/`06512349-AA`) mới được gộp. Đã sửa `partControl.ts` theo đúng hướng này.

**Còn thiếu:**
1. **36 Part# đang hoạt động cần bổ sung Quantity/box riêng** (tăng từ con số cũ 21/160 vì trước đó tính sai nhờ fallback) — danh sách đầy đủ kèm số traveler dùng ở `BANG_MA_CAN_OWNER_DUYET.md` mục 1.
2. **Ý nghĩa TỪNG loại hậu tố** (`-L`, `-HT`, `-A`, `-T`, `-B`, `-X`, `-MY-AK`, `-CA-IN`...) — biết là sản phẩm khác nhau rồi, nhưng chưa biết mỗi ký hiệu cụ thể nghĩa là gì (`BANG_MA_CAN_OWNER_DUYET.md` mục 2).
3. **Bảng quy ước cho field dạng enum** (Shift, Machine, Operator code) — biết nguồn thật có sẵn (sheet `Status` trong `PACKING SLIPS.xlsm`, 25 mã máy hợp lệ) nhưng chưa lấy về dùng để validate OCR. Danh sách mã nhân viên quan sát được (chưa chính thức) ở `BANG_MA_CAN_OWNER_DUYET.md` mục 4.
4. **MC# là gì về bản chất** — 1 máy vật lý độc quyền, hay 1 trạm dùng chung nhiều traveler/ca? (GAP-012 — 5,66% dữ liệu thật cho thấy dùng chung, chưa rõ là bình thường hay lỗi).
5. **Mã thiết bị QC** (máy đo Hardness/Tensile/Proof Load) — Split Form không có ô mã máy cho khu FINAL INSPECTION (khác Sorting Results có "Sorting M/C#") — chưa biết phòng QC có đánh số riêng không.

**Vì sao ERP chuẩn cần cái này:** Master data (Item Master, Work Center Master, Resource Master) là nền cho MỌI phép tính khác — thiếu 1 Part# hoặc hiểu sai 1 mã máy sẽ làm sai lệch toàn bộ báo cáo hiệu suất/giá thành phía sau, không sửa được bằng logic nghiệp vụ.

---

## B. Sales Order / Đơn hàng đầu vào (khâu 1)

**Đã chốt:**
- Giữ tên field `po` trong code/Sheet, nhưng hiểu đúng bản chất là **SO** (Sales/Work Order Infasco giao AVP gia công) — không phải PO AVP đi mua hàng (Notes #16).
- 1 SO → nhiều Traveler → nhiều LOT NO. (đã đo bằng dữ liệu thật).
- Dữ liệu PO/Traveler/Part#/Pot#/Weight/Pieces từ email lưu vào tab `RawMaterial`.

**Còn thiếu:**
4. **RawMaterial hiện là DUY NHẤT nguồn "nguyên liệu vào"** dùng trong đối chiếu khâu 2 — nhưng đây chỉ là **số dự báo qua email**, chưa phải số đã cân/đếm thật. Xem mục C bên dưới — đây là gốc rễ, không phải lỗi ở khâu 1.

**Vì sao ERP chuẩn cần cái này:** Trong mọi ERP (SAP, Odoo, Oracle), **Sales Order / Purchase Order KHÔNG BAO GIỜ được dùng trực tiếp làm số tồn kho** — nó chỉ là cam kết/dự báo. Tồn kho chỉ tăng khi có **Goods Receipt** (mục C).

---

## C. Goods Receipt / Nhận kho nguyên liệu thật — **GAP LỚN NHẤT, MỚI PHÁT HIỆN HÔM NAY**

**Đã chốt:**
- Split Form = thẻ theo dõi thật của 1 pot, khác hẳn email dự báo (Notes #11).
- Model 4 khâu (Email → Kho NVL → Finish Good → Packing List) đã chốt từ trước (Notes #12).

**Còn thiếu — đây là khối THIẾU NHIỀU NHẤT:**
5. **Bước "nhận nguyên liệu vật lý" hiện KHÔNG được ghi nhận ở đâu cả.** Bản redesign khâu 2 trước đây đã bỏ hẳn form nhập tay số thực nhận, thay bằng bảng đối chiếu tự động Forecast↔Thành phẩm — vô tình **xóa luôn** bước xác nhận "đã về kho thật", không phải chỉ bỏ việc nhập tay.
6. **Ô nào trên Split Form là số nhận kho thật** — Pieces (thường để trống trên form thật) hay Box QTY (nghi là số gốc trước khi chia lô, chưa chắc)? Owner có nhắc "SL=PLT" nhưng chưa xác định được field cụ thể trên phiếu.
7. **Ai chụp, chụp lúc nào** — nhân viên kho hay phân xưởng? Ngay lúc xe giao hàng (trước khi có bất kỳ thông tin QC/sản xuất nào)?
8. **Lưu vào đâu** — hồi sinh tab `Warehouse` (đang là dead code, sinh ra đúng để làm việc này) hay thêm field mới vào `RawMaterial`?
9. **Chênh lệch dự báo vs thực nhận xử lý thế nào** — có cần ngưỡng dung sai (tolerance) như khâu nhận hàng chuẩn ERP không, hay chỉ cảnh báo?
10. Nút chụp/quét cho bước này đặt ở đâu trên giao diện — khâu 2 (đúng vị trí nghiệp vụ) hay dùng chung nút Split Form đã có ở khâu 3?

**Vì sao ERP chuẩn cần cái này:** Đây là bước **Goods Receipt Note (GRN)** — 1 trong 3-4 giao dịch kho cơ bản nhất của MỌI hệ ERP (nhận hàng, xuất hàng, điều chuyển, điều chỉnh). Không có GRN nghĩa là hệ thống **không có cách nào phân biệt** "hàng đã thực sự về kho" với "hàng chỉ mới được báo trước qua email" — đúng là điều Owner vừa chỉ ra khi nói "chưa có số nhập kho mà, mới trên giấy".

---

## D. Work Order / Lệnh sản xuất (khâu kế hoạch)

**Đã chốt:**
- Traveler# = de facto Work Order (mỗi traveler tương ứng 1 lệnh sản xuất thật).

**Còn thiếu:**
11. Việc "kế hoạch xem như cho sản xuất" (tạo/duyệt WO) hiện là **quyết định ngoài hệ thống**, không có action nào trong app ghi nhận thời điểm này. Có cần hệ thống ghi lại mốc "đã duyệt sản xuất" không, hay giữ nguyên là việc thủ công của kế hoạch (không cần code)?

**Vì sao ERP chuẩn cần cái này:** Work Order Release là 1 trạng thái (status) chuẩn trong MES/ERP (`Draft → Released → In Progress → Done`) — giúp biết "đang chờ duyệt" khác với "đã cho chạy máy". Nếu AVP không cần theo dõi trạng thái này qua hệ thống (chỉ cần biết bằng miệng/giấy), thì không cần code — nhưng cần Owner xác nhận rõ để không làm thừa.

---

## E. Quality Management (QC) — ISO 9001 Cl.8.7

**Đã chốt:**
- Đọc được `qcStatus` (PASS/HOLD) từ khu FINAL INSPECTION trên Split Form.
- Mới thêm hôm nay: đọc bảng "Sorting Results / Defect # of PCS Found" → 2 cột `scrapQty`/`scrapDetail` trong FinishGood (ẩn mặc định).

**Còn thiếu:**
12. **CONCESSION** (nhượng bộ, cần chữ ký người có thẩm quyền độc lập theo ISO 9001 Cl.8.7) — hiện chỉ có PASS/HOLD, chưa có trạng thái thứ 3 này.
13. **Yield/hao phí chưa được tổng hợp/hiển thị** — mới lưu được số phế liệu thô (scrapQty), chưa có màn hình tính Yield% = Thành phẩm tốt / Nguyên liệu vào theo Part#/máy/ca (mà Owner nói rõ mục đích là "tính hiệu suất máy").
14. Đường nhập file `.xlsm` điện tử của Split Form (khi phân xưởng xuất file thay vì chụp ảnh) — **chưa có file mẫu thật** để biết đúng cấu trúc cột, chưa làm được.

**Vì sao ERP chuẩn cần cái này:** Non-conformance disposition chuẩn ISO 9001 có 3 nhánh (`PASS→UNRESTRICTED`, `FAIL→BLOCKED`, `CONCESSION→REWORK có ký duyệt`) — thiếu nhánh CONCESSION nghĩa là hệ thống chưa xử lý được trường hợp "lỗi nhưng vẫn chấp nhận xuất có điều kiện", vốn là tình huống thực tế phổ biến trong gia công cơ khí.

---

## F. Traceability & Lot Genealogy

**Đã chốt:**
- Khóa (Traveler#, LOT NO.) là đúng, xác nhận bằng 1.811 dòng dữ liệu thật (100% 1:1).
- SKID# **không phải** khóa định danh (77,4% skid chứa nhiều traveler khác nhau).
- Trang `/trace` đã có, tra 1 traveler xem cả 4 khâu.

**Quy trình truy vết 2 chiều (dùng đúng khóa ở Bảng quan hệ dữ liệu phía trên):**

```
TRUY VẾT XUÔI (Forward) — có nguyên liệu/PO, tìm xem đã xuất đi đâu
─────────────────────────────────────────────────────────────────
  PO / Pot#            Traveler#            LOT NO.              PS#
 (RawMaterial)  ────▶  (khóa chính)  ────▶  (FinishGood)  ────▶  (PackingList)
                                                                  = đã xuất chuyến nào

TRUY VẾT NGƯỢC (Backward) — có sản phẩm đã xuất/lỗi, tìm về nguyên liệu gốc
─────────────────────────────────────────────────────────────────
    PS#              Traveler#+LOT NO.        Pot#/MC#/QC             PO/SO gốc
(PackingList)  ────▶     (FinishGood)   ────▶  (đối chiếu)   ────▶   (RawMaterial)
```

Cả 2 chiều đều đi qua **cùng 1 khóa trung tâm: Traveler# (+LOT NO.)** — đây là lý do khóa này quan trọng nhất trong toàn hệ thống, không phải PO/PS/Pot# hay SKID#. Công cụ hiện có (ô search chung + trang `/trace`) hỗ trợ được cả 2 chiều nhưng **thủ công từng bước, chưa có nút "truy vết hàng loạt"** một lần cho cả 1 PO/Pot# — xem hạn chế ở mục 15-18 dưới.

**Vì sao phải đúng khóa — giải thích cho người hỏi "1 con ốc vít cần gì phải truy vết kỹ vậy":**

Chính AVP đã có **QC HOLD** và **bảng phế liệu (Sorting Results)** trên Split Form — nghĩa là **lô hư có thật trong thực tế**, không phải giả định. Vấn đề không phải "có cần truy vết 1 con ốc" mà là: khi phát hiện 1 lô hư, có khoanh đúng phạm vi hay không.

Ví dụ: Infasco báo lô Part# X có vài con lỗi ren. AVP cần trả lời được **"chỉ Traveler# 717XXX, LOT `6-247-04-b`, đã giao PS 30110 — các lô khác không liên quan"**.

- Dùng **SKID#** để khoanh phạm vi → **sai ngay 77,4% trường hợp** (1 skid chứa nhiều traveler khác nhau dùng chung) → có thể **kéo oan** traveler tốt nằm chung skid, hoặc **bỏ sót** traveler lỗi thật ở skid khác.
- Dùng **(Traveler#, LOT NO.)** → đúng 100% (xác nhận trên 1.811 dòng thật).

**Hậu quả nếu khoanh sai phạm vi**: hẹp hơn thực tế → bỏ sót hàng lỗi, khách phát hiện tiếp → mất uy tín nặng hơn; rộng hơn thực tế → thu hồi/kiểm tra oan hàng tốt không liên quan → tốn chi phí, ảnh hưởng giao hàng đúng hẹn của các lô vô tội khác. Đây là lý do thực tế (không phải lý thuyết ERP suông) để khóa (Traveler#, LOT NO.) — không phải SKID# — là bắt buộc.

**Còn thiếu:**
15. **Case traveler 717671** (2 dòng FinishGood, LOT# khác nhau) — Notes #7/#8 và GAP-004 vẫn treo. Dữ liệu 1.811 dòng lịch sử cho thấy Traveler→LOT# là 1:1 tuyệt đối, nên case này gần như chắc chắn là lỗi — nhưng cần **chứng từ giấy gốc** để kết luận chắc, chưa có.
16. **Vòng đời container (POT#)** — nhận, dùng, trả — hoàn toàn chưa track, dù đã biết SKID# không dùng được làm khóa thay thế.
17. **Sổ cái "còn tồn chưa xuất"** (Remaining ledger, Notes #9) — mới dừng ở ý tưởng thiết kế, chưa code (hiện chỉ có cảnh báo phát hiện trùng, chưa có mã truy vết `SCAN-{Traveler}-{n}` như đã bàn).
18. **GAP-003**: không lưu file ảnh/PDF gốc kèm mỗi dòng dữ liệu — nếu sau này cần đối chứng lại chứng từ giấy (như case 717671), sẽ không tìm lại được ảnh gốc lúc scan.

**Vì sao ERP chuẩn cần cái này:** Truy vết 2 chiều (forward/backward, "one-up-one-down") là yêu cầu bắt buộc cho ngành cơ khí/phụ tùng ô tô (IATF 16949) — thiếu vòng đời container và sổ cái tồn kho nghĩa là khi có sự cố (như 717671), không có cách tự động chứng minh đúng/sai, phải lục giấy tờ tay.

---

## G. Shipping / Xuất hàng (khâu 4)

**Đã chốt — không có gap lớn:**
- Packing Slip = 1 chuyến xe, không gắn 1 PO/SO cụ thể (xác nhận bằng 2 PS thật, mỗi PS gộp 9-11 PO khác nhau).
- Xuất lẻ (partial shipment) đã code, đã test bằng traveler thật.
- Số PS **cố ý** không tự sinh (nhập tay) — vì dãy số PS dùng chung nhiều khách hàng, tự sinh dễ trùng số thật ngoài đời.

**Còn thiếu:** không có câu hỏi mới ở khối này.

---

## H. Vận hành song song với hệ thống cũ (giai đoạn chuyển đổi)

**Phát hiện quan trọng (không phải lỗi AVP_AI, nhưng ảnh hưởng độ tin cậy dữ liệu tham chiếu):**
19. **GAP-014**: `PACKING SLIPS.xlsm` và `DAILY LOG CHECK SHEET.xlsm` — 2 file Excel đang vận hành thật — mỗi file giữ 1 bảng WORK ORDER riêng, **797/1.847 traveler bị lệch SHIPPED/PS thật** giữa 2 file. `SUMMARY IN&OUT` (báo cáo KPI ngày) sai 0/30 ngày mẫu so với WORK ORDER thật. Owner cần biết để không dùng 2 file này làm "sự thật tuyệt đối" khi đối chiếu dữ liệu AVP_AI trong giai đoạn còn chạy song song.

---

## Danh sách rút gọn — cần Owner trả lời TRƯỚC (ưu tiên cao → thấp)

| # | Câu hỏi | Khối | Vì sao ưu tiên | Ví dụ dễ hiểu |
|---|---|---|---|---|
| ~~✅~~ | ~~Hậu tố Part# là 2 sản phẩm khác nhau hay biến thể vô hại?~~ | A | **ĐÃ CHỐT 2026-09-13** — Owner xác nhận là sản phẩm khác nhau thật (lý do cụ thể mỗi hậu tố CHƯA xác nhận — không phải mạ/không mạ), đã sửa code. Việc còn lại: bổ sung Quantity/box cho 36 Part# — xem `BANG_MA_CAN_OWNER_DUYET.md` mục 1 | `11546437-MY-AK-L` giờ báo đúng "thiếu trong PartControl", không còn tự động mượn số của `11546437-MY-AK` nữa. |
| 1 | Ý nghĩa CỤ THỂ từng hậu tố Part# (`-L`, `-HT`, `-A`, `-T`, `-B`, `-X`...) là gì? | A | Biết là sản phẩm khác nhau rồi, nhưng chưa biết mỗi ký hiệu nghĩa gì — cần để mô tả đúng khi bổ sung 36 Part# thiếu | Xem danh sách đủ ở `BANG_MA_CAN_OWNER_DUYET.md` mục 2 |
| 2 | Ô nào trên Split Form là số nhận kho THẬT — Pieces hay Box QTY? ("SL=PLT" là gì?) | C | Chặn toàn bộ việc sửa lại công thức đối chiếu khâu 2 | Phiếu traveler 717365: ô "Pieces" để trống, ô "Box QTY" ghi 31.000 — không biết lấy số nào để so với số email dự báo. |
| 3 | Đồng ý hồi sinh tab `Warehouse` để ghi nhận "đã về kho thật" không? | C | Quyết định kiến trúc, ảnh hưởng nhiều code (đã hạ độ khẩn cấp vì Owner xác nhận PO/nguyên liệu về gần như cùng ngày) | Giống như trong Sheet đã có sẵn 1 quyển sổ tên "Warehouse" để ghi "hàng về ngày nào, bao nhiêu" — nhưng từ lúc redesign, không ai viết vào sổ đó nữa, mọi thứ ghi thẳng vào sổ "dự báo email" (RawMaterial) luôn. |
| 4 | Traveler 717671 — có chứng từ giấy gốc để đối chiếu không? | F | Quyết định có phải sửa/xóa dữ liệu thật trong Sheet | 1 traveler nhưng có 2 dòng thành phẩm với LOT# gần giống nhau (`6-253-13-A` và `6-251-13-A`, lệch đúng 1 số) — giống lỗi gõ nhầm, nhưng không có tờ giấy gốc để chứng minh chắc. |
| 5 | MC# là máy độc quyền hay trạm dùng chung? | A | Ảnh hưởng độ tin cậy cảnh báo "trùng máy" đang chạy | Máy MC#078, cùng ngày cùng ca, nhưng lại thấy gắn với 2 traveler khác nhau — không biết đây là 1 máy chạy nối tiếp 2 lệnh (bình thường) hay do nhân viên đọc/ghi nhầm số máy (lỗi thật). |
| 6 | finishedPartNo hay partNo (đầu form) — cái nào ưu tiên dùng thật? | A/F | Owner nói sẽ hỏi Infasco — chưa có trả lời | Cùng 1 traveler: đầu phiếu ghi Part# `11549170-L`, nhưng sticker đóng gói lại in Finished Part# khác — chưa biết Packing List cuối cùng nên lấy theo số nào. |
| 7 | Có cần chặn cứng thứ tự khâu 2→3 không (GAP-009)? | D | Ảnh hưởng UX, chưa gấp | Hiện nhân viên có thể quét lưu Finish Good (khâu 3) cho 1 traveler dù traveler đó CHƯA từng được xác nhận ở khâu 2 (Kho) — hệ thống không cảnh báo gì, cứ cho lưu bình thường. |
| 8 | Nút chụp Split Form (nhận kho) đặt ở khâu 2 hay dùng chung khâu 3? | C | Chỉ code được sau khi trả lời câu 2-3 | Hiện nút "Chụp Split Form" chỉ có trong khâu 3 (Scan/Finish Good) — nhân viên kho lúc hàng vừa về phải mở đúng trang "đóng gói thành phẩm" để chụp, dù việc này thuộc khâu 2 (nhận nguyên liệu) về mặt nghiệp vụ. |
| 9 | File `.xlsm` mẫu thật của Split Form điện tử (nếu có) | E | Để làm đường nhập file, không bắt buộc ngay | Giống như file `DAILY LOG CHECK SHEET.xlsm` đã có để xây đường nhập Excel cho khâu 3 — cần 1 file tương tự nhưng là bản điện tử của Split Form, để không phải đoán mò cấu trúc cột khi làm đường nhập. |
| 10 | Đọc thêm Serial#/SSCC (mã từng box) vào hệ thống? | F | Mới phát hiện — hoàn thiện chuỗi truy vết tới mức box, chuẩn GS1 | Nhãn đóng gói thật đã in sẵn dải serial (vd 36 số liên tiếp cho đúng 36 carton) — chỉ cần đọc thêm, không cần thiết bị mới |
| 11 | 36 Part# đang hoạt động thiếu Quantity/box — ai điền? | A | Chặn tính QUANTITY cho các Part# này (tăng từ 21 vì sửa lại cách khớp) | Xem danh sách đủ + số traveler dùng ở `BANG_MA_CAN_OWNER_DUYET.md` mục 1 |

---

## Lộ trình cải tiến theo ERP — xếp theo PHỤ THUỘC, không chỉ theo độ khó

*Nguyên tắc: việc nào cần Owner/Infasco trả lời trước thì làm trước, dù
nhỏ — vì mọi việc code phía sau đều bị chặn bởi nó. Không nên bắt đầu từ
việc "khó nhất" hay "quan trọng nhất" nếu nó chưa đủ thông tin để làm đúng
ngay từ đầu (bài học từ chính PartControl — làm rồi mới phát hiện 85% dựa
trên giả định chưa xác minh).*

### Giai đoạn 0 — Trả lời trước, KHÔNG code được nếu thiếu (việc của Owner/Infasco, không phải AI)
1. Ý nghĩa hậu tố Part# (`-L`/`-HT`/`-A`...) — ảnh hưởng 85% Part# đang chạy.
2. Ô nào trên Split Form là số nhận kho THẬT (Pieces hay Box QTY / "SL=PLT").
3. MC# là máy độc quyền hay trạm dùng chung (GAP-012) — quản đốc xưởng trả lời ngay được, không cần thu thập số liệu.

### Giai đoạn 1 — Làm ngay được, rủi ro thấp, KHÔNG phụ thuộc Giai đoạn 0
- Tổng hợp Yield/hao phí theo Part# từ `scrapQty` đã có sẵn (Yield% = Thành phẩm tốt / Nguyên liệu vào) — dữ liệu đã nằm sẵn trong Sheet, chỉ cần 1 màn hình tính lại, chưa cần theo máy (vì Work Center Master chưa có).
- Dashboard tổng quan tức thời (số traveler đang HOLD, đang lệch >10%, đã xuất hôm nay...) — dữ liệu đã có sẵn ở khâu 2/4.
- Đặt tên chính thức CCP-1..4 trong comment code (đã đề xuất ở `ARCHITECTURE.md` mục 8.3, chỉ đặt tên, không đổi hành vi).
- Sửa PartControl theo hướng đã chọn ở Giai đoạn 0 câu 1 (nếu Owner xác nhận là 2 sản phẩm khác nhau → bắt buộc khớp đúng hậu tố; nếu xác nhận vô hại → giữ nguyên fallback, chỉ cần bổ sung 21 mã thiếu).

### Giai đoạn 2 — Cần quyết định kiến trúc, phụ thuộc trực tiếp Giai đoạn 0
- Thêm bước **Goods Receipt** (chụp Split Form lúc nhận, hồi sinh tab `Warehouse`) — phụ thuộc câu 2 (số nào là số thật).
- Sửa công thức "Nguyên liệu vào" ở khâu 2 dùng số Goods Receipt thay vì RawMaterial dự báo.
- Thêm cổng kiểm soát: chỉ cho "kế hoạch tạo Work Order/cho sản xuất" khi đã có Goods Receipt xác nhận (đúng nguyên tắc MRP đã bàn) — phụ thuộc mục ngay trên.

### Giai đoạn 3 — Cần THU THẬP dữ liệu mới ngoài đời, không chỉ code
- **Work Center Master**: chính thức hóa danh sách 25 mã máy (sheet `Status`) + trạng thái AVAILABLE/DOWN/MAINTENANCE.
- **Resource Master**: danh sách mã nhân viên hợp lệ (export từ hệ chấm công nếu có).
- **Container/Asset Master (Pot#)**: danh sách Pot# hợp lệ + quy trình xác nhận trả Pot# rỗng cho Infasco.
- **OEE (hiệu suất máy)**: khó nhất — cần bắt đầu ghi giờ máy chạy/dừng, dữ liệu này **chưa từng tồn tại kể cả ở Excel cũ**.

### Giai đoạn 4 — Nền tảng lớn, làm sau cùng, chỉ khi chắc chắn cần
- Gộp RawMaterial/FinishGood/PackingList thành 1 sổ cái kiểu `stock.move` (đã đề xuất `ARCHITECTURE.md` mục 8.1/8.4) — đổi cấu trúc dữ liệu nền tảng, rủi ro cao.
- Sổ cái "còn tồn chưa xuất" + mã truy vết `SCAN-{Traveler}-{n}` (Notes #9).
- Lưu ảnh/PDF gốc kèm mỗi dòng dữ liệu (GAP-003) — để truy vết có bằng chứng giấy tờ thật, không chỉ số liệu.
- Truy vết hàng loạt (`/trace` hiện chỉ 1 traveler/lần) — nâng lên nhận cả PO/Pot# rồi tự lặp qua toàn bộ traveler liên quan.

---

*Tài liệu này tổng hợp từ `Notes.md` (30+ câu hỏi, tính đến 2026-09-13) và
`LATEST_SESSION.md` (GAP-001 → GAP-014) — không thay thế 2 file đó, chỉ sắp
xếp lại theo khối ERP chuẩn để dễ làm việc với Owner theo từng mảng thay vì
theo trình tự thời gian hỏi. Lộ trình cải tiến ở trên đồng bộ với
`ARCHITECTURE.md` mục 8.4/8.5 (đã viết trước đó) — mở rộng thêm phần Goods
Receipt/Master Data phát hiện hôm nay. Xem thêm `BANG_MA_CAN_OWNER_DUYET.md`
— danh sách chi tiết từng mã (Part#, máy, nhân viên, LOT, Serial#) cần
Owner rà soát/bổ sung, tách riêng khỏi tài liệu này vì là việc làm dần,
không phải câu hỏi thiết kế.

**Cập nhật 2026-09-13 (sau khi Owner cung cấp thông tin trực tiếp)**: PO đi
trước qua email, nguyên liệu thật về **cùng ngày** (không có độ trễ đáng kể
như lo ngại ban đầu ở khối C/D) — hạ độ khẩn cấp phần "cổng chặn Work Order"
trong lộ trình. Quy trình nhận hàng thật: nhân viên kho scan barcode Pot#
→ tự động phân Pieces về máy → sinh Split Form riêng cho máy đó — đây là
chi tiết cần nắm khi thiết kế lại bước Goods Receipt (khối C).*
