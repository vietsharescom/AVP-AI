# AVP Packing Flow — Kiến trúc hệ thống (Pipeline Input → Processing → Output)

> Tổng hợp từ code thật (`webapp/src/app/api/*`, `webapp/src/lib/sheets.ts`),
> `BRS_TRS.md`, và đối chiếu trực tiếp với 4 form giấy thật trong `Data/`
> (2026-09-12). Đọc file này để nắm nhanh kiến trúc, đọc `BRS_TRS.md` khi
> cần chi tiết công thức/ánh xạ cột.

## 0. Sơ đồ tổng quan

```
Infasco (email/giấy)                    Sản xuất AVP                       Xuất hàng
      |                                       |                                |
      v                                       v                                v
[1. Email/Dự báo]  ---forecast--->  [2. Kho NVL]  <--cùng 1 tờ--> [3. Finish Good]  ---> [4. Packing List] ---> Infasco
   /inbox                              /warehouse    giấy (form 02)    /scan                /packing-list
      |                                    |                                |                      |
      v                                    v                                v                      v
  RawMaterial                          Warehouse                      FinishGood            PackingList
                                    (Google Sheets — 1 spreadsheet, 5 tab, database dùng chung)

Song song (không nằm trong luồng tuần tự):
  PartControl (/part-control)  — bảng tra cứu Quantity/box, dùng ở khâu 4
  Reconciliation (/reconciliation) — đối chiếu 4 khâu, % lệch
  Trace (/trace) — truy vết ngược 1 Traveler# qua cả 4 khâu
```

Database duy nhất: **Google Sheets** (ID `1TWYKdRHv3MioEBf2DHAvUGlCvZxsr-KXQKLCR_3BkJ4`),
truy cập qua Sheets API v4, xác thực **OAuth** (không dùng Service Account vì
chính sách `iam.disableServiceAccountKeyCreation` của org AVP chặn). Thay thế
2 file `.xlsm` cũ (chỉ 1 người mở được 1 lúc) bằng 1 nguồn dữ liệu nhiều người
dùng cùng lúc.

**Nguyên tắc kiểm soát AI xuyên suốt (áp dụng cho mọi khâu có OCR):**
mỗi khâu có OCR đều tách làm 2 API riêng — `.../extract` (Gemini đọc ảnh/PDF,
chỉ trả JSON đề xuất, KHÔNG ghi Sheets) và `.../confirm` hoặc `.../approve`
(ghi thật vào Sheets, chỉ chạy sau khi người dùng xem lại trên UI và bấm xác
nhận). AI không bao giờ có quyền ghi thẳng vào database.

> **Ghi chú thuật ngữ "PO":** field `po` giữ nguyên tên trong code/Sheets/UI
> (đúng theo cách owner đang gọi), nhưng bản chất là **đơn hàng gia công đầu
> vào** Infasco giao AVP sản xuất — không phải Purchase Order AVP đi mua
> hàng. Chuỗi phân cấp đã xác nhận: **1 PO → nhiều Traveler → nhiều LOT
> NO.** (xem mục 1 và `BRS_TRS.md` mục 3). LOT NO. **không** xuất hiện trên
> Packing Slip (đã xác minh `PACKING SLIP-1.pdf` thật) — chỉ dùng làm khóa
> tracking nội bộ.

---

## 1. Bốn form giấy thật và cách chúng ánh xạ vào 4 khâu

Đối chiếu trực tiếp với file thật trong `Data/` (đã đổi tên theo thứ tự
2026-09-12):

| # | File | Vai trò thật |
|---|---|---|
| 1 | `01_Email_Purchase_Order_Traveler form.jpeg` | Infasco gửi PO — bảng nhiều dòng, mỗi dòng 1 Traveler (`Traveler, Part#, Pot#, Weight, Pieces, Label Format, Final Lot`). Nguồn **dự báo** — khâu 1. |
| 2 | `02_Work_Order_Quality_Check_Split form.jpeg` | 1 tờ = 1 Traveler. Góc trên: barcode Traveler# (khớp với dòng Traveler tương ứng trong form 01). Có "Split From: #…" + "Split Lot #" khi traveler bị **chia lô** trong sản xuất. Phần **ROUTING** (Receive Infasco Nut → Heat Treat → Plate → Final Inspection → Sort & Pack, mỗi bước có ô check/initials/date) = xác nhận QC + nhận hàng theo từng công đoạn. Góc dưới trái có **2 sticker "PACKAGING"** (Pcs/Carton, # of Cartons, Total Quantity) = **số liệu đóng gói thật đã QC đạt**. |
| 3 | `03_Finish_Goods_Scanning.jpeg` ("WRAPPING SUMMARY - INFASCO") | Sổ tổng hợp viết tay, nhiều Traveler/ngày. Cột đánh số (1)-(20) trên bản giấy khớp đúng **S1-S19** đã xác minh trong `BRS_TRS.md`. Nhân viên chép số liệu từ sticker trên form 02 vào đây. |
| 4 | `04_Delivery_Order_packing_slip.jpeg` ("PACKING SLIP") | Chứng từ xuất hàng gửi Infasco. Cột đánh số (1)-(8) khớp **P1-P8**. Tổng hợp từ 1+2+3. |

**Điểm mấu chốt**: form 02 (Split/Work Order/QC form) là nguồn dùng chung
cho **cả khâu 2 (Kho NVL — qua phần ROUTING "Receive Infasco Nut") và khâu
3 (Finish Good — qua 2 sticker PACKAGING)**, không phải 2 tài liệu độc lập.

✅ **Đã sửa (2026-09-12)**: `extractSplitForm` (extract.ts) giờ đọc thêm từ
form 02: `finishedPartNo` (sticker "Finished Part Number" — có thể khác
`partNo` đầu form, chưa xác nhận cái nào đúng để link khâu sau, tạm giữ cả
2), `boxQty`, `qcStatus` (PASS/HOLD, suy từ khu FINAL INSPECTION),
`qcNotes` (gộp các spec test thành 1 ghi chú), và `receivedChecked` (ô
"RECEIVE INFASCO NUT" trong ROUTING) **nay đã được UI khâu 2 và khâu 3 sử
dụng thật** — không còn là data chết. Khâu 2 hiện cảnh báo đỏ nếu
`qcStatus = HOLD`, và ghi chú nếu `partNo` ≠ `finishedPartNo`, vào cột
"Ghi chú" mới trên `/warehouse` (lưu vào field `note` có sẵn của tab
`Warehouse`). Test bằng đúng ảnh form 02 thật (traveler 717365) — đọc đúng
100% các trường trên.

Vẫn còn thật: khâu 2 (`/warehouse`) và khâu 3 (`/scan`) là 2 trang riêng,
mỗi trang tự gọi `/api/warehouse/extract` khi cần đọc form 02 — **không có
ràng buộc code nào bắt buộc phải xác nhận khâu 2 trước khi lưu khâu 3**
(nhân viên vẫn có thể scan Finish Good dù chưa xác nhận nhận kho cho
traveler đó). Đây là chủ ý hay cần chặn cứng — chưa hỏi lại Andy.

---

## 2. Khâu 1 — Email / Dự báo (`/inbox`)

- **Input**: ảnh/PDF form **01** (Traveler form Infasco gửi qua email).
- **Processing**:
  - `POST /api/email/extract` — Gemini OCR đọc ảnh → JSON đề xuất
    (traveler, part#, pot#, weight, pieces, PO, date...).
  - Người dùng xem lại trên UI, có đối chiếu barcode thật để bắt lỗi đọc sai.
  - `POST /api/raw-material/confirm` — ghi vào tab `RawMaterial`.
  - `GET /api/raw-material/list` — xem lại dữ liệu đã lưu.
- **Output**: tab `RawMaterial` — **CHỈ LÀ DỰ BÁO**, không phải tồn kho thật
  (Infasco báo trước hàng sẽ giao, chưa chắc đã tới).

## 3. Khâu 2 — Kho Nguyên Vật Liệu (`/warehouse`)

- **Input**: số liệu **thực nhận** tại kho (nhập tay) + ảnh form **02** để
  tra cứu/tự điền.
- **Processing**:
  - `POST /api/warehouse/extract` (`extractSplitForm`) — đọc form 02: traveler,
    partNo, finishedPartNo, pot, boxQty, pieces (thường trống), finalLot,
    splitFrom, `receivedChecked` (ô Receive Infasco Nut), `qcStatus`
    (PASS/HOLD), `qcNotes`.
  - UI: nếu `qcStatus = HOLD` → cảnh báo đỏ nổi bật, không nên xác nhận nhận
    kho khi chưa rõ tình trạng. Nếu `partNo` ≠ `finishedPartNo` → tự điền
    cảnh báo vào cột Ghi chú. Pieces chỉ tự điền khi form có ghi (hiếm).
  - Người dùng xác nhận số **thực nhận** (`receivedPieces`, `receivedWeight`),
    khác với số dự báo ở khâu 1, và có thể sửa/thêm Ghi chú tay trước khi lưu.
  - `POST /api/warehouse/confirm` — ghi vào tab `Warehouse` (kèm `note`).
  - `GET /api/warehouse/list`.
- **Output**: tab `Warehouse` — số liệu nhận kho thật, độc lập với dự báo.
- ⚠ **Split Form (02) ≠ nguồn số lượng nhận kho**: trường "Pieces" trên
  form 02 thường để trống. 2 sticker "PACKAGING" mới là số liệu đóng gói
  thật, và nó thuộc khâu 3 (Finish Good), không phải khâu này.

## 4. Khâu 3 — Finish Good / Scan (`/scan`)

- **Input**: ảnh form **03** (Scanning Sheet viết tay) HOẶC form **02**
  (2 sticker PACKAGING, PDF nhiều trang) HOẶC file Excel
  `DAILY LOG CHECK SHEET.xlsm` (sheet `CHECKING SUMMARY`) — hệ thống cũ đã
  có sẵn dữ liệu này dạng số hóa, không cần chụp lại giấy.
- **Processing**:
  - PDF nhiều trang được tự tách thành từng trang, xử lý **tuần tự** (không
    `Promise.all`) vì Gemini free tier chỉ 15 request/phút — xử lý song song
    sẽ bị lỗi 429 ngay; có retry + backoff cho 429.
  - `POST /api/scan/extract` — Gemini OCR từng trang → JSON đề xuất (đủ cột
    S1–S19: date, shift, Oprtr, traveler, part#, pot#, **lot#**, type, O/C,
    machine, MC#, S/P, special notes, boxes, TTL QNT, skid#, location...).
  - `POST /api/scan/extract-excel` **(mới, 2026-09-12)** — đọc trực tiếp
    file `.xlsx/.xlsm`, tự tìm dòng tiêu đề (chứa DATE + TRAVELER) và map
    cột theo đúng tên thật (không phải OCR, không có bước AI) → cùng JSON
    shape như trên. **Chỉ lấy các dòng CHƯA "SHIPPED"** trong file cũ (bỏ
    qua dòng đã Shipped — đó là hồ sơ đã đóng, không phải dữ liệu mới cần
    nhập). Chặn nếu đọc được >300 dòng (an toàn — tránh lỡ tay đổ nguyên
    lịch sử cũ vào Sheet đang chạy thật). Dùng thư viện `xlsx` (SheetJS) —
    có 1 CVE mức cao (ReDoS/prototype pollution) chưa có bản vá trên npm;
    chấp nhận vì chỉ nhân viên AVP tự tải file nội bộ lên, không phải input
    công khai. (Thử `exceljs` trước — an toàn hơn nhưng không đọc được file
    thật do dùng AutoFilter date-grouping mà exceljs chưa hỗ trợ.)
  - Đọc form 02 dùng lại `POST /api/warehouse/extract` (cùng `extractSplitForm`
    như khâu 2) — 2 sticker PACKAGING map thành các dòng FinishGood;
    `finishedPartNo`/`qcNotes` gộp vào `specialNotes` của mỗi dòng; cảnh báo
    đỏ nếu `qcStatus = HOLD`.
  - Có nút nhập tay khi AI không đọc được.
  - Người xem lại → `POST /api/finish-good/confirm` — ghi vào tab
    `FinishGood`. Route này cảnh báo nếu traveler đã có dòng chưa Shipped
    (nghi trùng) trước khi cho lưu tiếp.
- **Output**: tab `FinishGood` — số liệu sản xuất/đóng gói thật.
- ⚠ **Cập nhật quan trọng (2026-09-12)**: 1 Traveler → **nhiều LOT NO. là
  HỢP LỆ** khi traveler bị chia lô (split) trong sản xuất — mỗi LOT NO.
  tương đương 1 Work Order riêng theo mô hình ERP. Route
  `finish-good/confirm` hiện chỉ cảnh báo trùng theo **Traveler#**, không
  phân biệt theo LOT NO. — có thể báo nhầm 2 lot hợp lệ là "nghi trùng".
  Case thật: traveler 717671 có 2 dòng LOT NO. khác nhau
  (`6-253-13-A`/`6-251-13-A`) — từng bị coi là lỗi (GAP-004 cũ), **đang xem
  lại**, xem `LATEST_SESSION.md`.
- ⚠ Partial save: nếu 1 batch có vài dòng lỗi (traveler trùng/không tồn
  tại), chỉ skip đúng dòng lỗi, lưu phần còn lại, báo rõ dòng nào bị skip —
  không chặn cả batch. Cũng không tự clear dữ liệu form khi lưu thất bại
  (từng gây mất 171 dòng scan do code cũ xóa `rows` trước khi biết kết quả).

## 5. Khâu 4 — Packing List (`/packing-list`)

- **Input**: `FinishGood` (dòng chưa Shipped) + `RawMaterial` (tra PO/Note) +
  `PartControl` (Quantity/box).
- **Processing**:
  - `POST /api/packing-list/generate`:
    - Ghép theo `Traveler#` (khóa nối chính giữa T/S/P).
    - `DESCRIPTION = Oprtr + " " + Machine+MC# + " " + S/P + [Special Notes
      nếu có, in hoa]`. **Không** gồm Type/Nut-Bolt (đã xác nhận sai khi thử
      thêm, xem `BRS_TRS.md` mục 4).
    - `QUANTITY = BOX × Quantity/box` (tra theo Part# trong PartControl —
      thử khớp thẳng → bỏ hậu tố mạ/vật liệu (`-L`,`-HT`,`-A`...) → nếu vẫn
      không khớp, dùng tạm TTL QNT lúc scan + cảnh báo rõ).
    - Đối chiếu chéo QUANTITY tính được vs TTL QNT lúc scan, cảnh báo nếu
      lệch > ngưỡng (mặc định 10%).
    - Phát hiện traveler bị scan trùng trước khi đưa vào danh sách duyệt.
  - Người duyệt → `POST /api/packing-list/approve` — ghi vào tab
    `PackingList`, đồng thời đánh dấu `Shipped=TRUE` trên `RawMaterial` +
    `FinishGood` theo **Traveler#** (⚠ GAP-002: đánh dấu theo cả traveler,
    không theo từng LOT NO. riêng — nếu 1 traveler có nhiều lot, tất cả bị
    đánh Shipped cùng lúc dù chỉ 1 lot thật sự được duyệt xuất).
- **Output**: tab `PackingList` — chứng từ xuất hàng thật gửi Infasco.
  **Không có cột LOT NO.** trên chứng từ (đã xác minh `PACKING SLIP-1.pdf`
  thật, 2 trang) — LOT NO. chỉ là khóa tracking nội bộ.
- 💡 **Đề xuất đang chờ quyết định**: đổi khóa xác định "1 lần
  sản xuất/đóng gói" từ chỉ `Traveler#` sang `(Traveler#, LOT NO.)` — sửa
  cả `finish-good/confirm` (phát hiện trùng) và `packing-list/approve`
  (đánh dấu Shipped) — chưa làm, chờ Andy xác nhận ưu tiên.

---

## 6. Module hỗ trợ (không nằm trong luồng tuần tự 1→4)

### PartControl (`/part-control`)
- `GET /api/part-control/list`, `GET /api/part-control/missing` (21 mã
  Part# thật chưa có Quantity/box, cần chủ xác nhận — xem `BRS_TRS.md` mục
  6), `POST /api/part-control/upsert` (form thêm/sửa tay).
- Vai trò: bảng tra cứu tĩnh `Part# → Quantity/box, Client, Machine mặc
  định`, là input bắt buộc cho công thức QUANTITY ở khâu 4.

### Reconciliation (`/reconciliation`)
- `GET /api/reconciliation` — đối chiếu Forecast (RawMaterial) → Received
  (Warehouse) → Scanned (FinishGood) → Shipped (PackingList), số tuyệt đối
  + %, ngưỡng cảnh báo chỉnh được (mặc định 10%).

### Trace (`/trace`)
- `GET /api/trace?traveler=...` — nhập 1 Traveler#, đọc song song cả 4 tab
  (`RawMaterial`, `Warehouse`, `FinishGood`, `PackingList`), trả về toàn bộ
  dòng liên quan; tự cảnh báo nếu 1 khâu có >1 bản ghi. Đây là công cụ
  chính để điều tra sự cố dữ liệu — nhưng hiện cảnh báo theo Traveler# đơn
  thuần, chưa phân biệt LOT NO. (xem mục 4).

---

## 7. Lỗ hổng kiến trúc đã biết (xem chi tiết `LATEST_SESSION.md` mục 4)

- **GAP-001**: Khâu 1 vẫn phải upload tay — chưa có cổng tự đọc email
  Infasco.
- **GAP-002**: Shipped đánh dấu theo Traveler#, không theo `(Traveler#,
  LOT NO.)` — xem đề xuất ở mục 5.
- **GAP-003**: Không lưu file ảnh/PDF gốc kèm mỗi dòng dữ liệu — không truy
  ngược được "dòng này lấy từ ảnh nào" sau khi đã lưu.
- **GAP-004**: ⚠ Đang xem lại — case traveler 717671 (2 dòng FinishGood,
  2 LOT NO. khác nhau) có thể KHÔNG phải lỗi trùng như từng nghĩ, mà là 2
  lot hợp lệ. Chưa xóa gì, chờ Andy xác nhận.
- **GAP-005**: Chưa có test tự động, chỉ build/lint + test tay bằng dữ liệu
  thật.
- ~~GAP cũ: khâu 2 không đọc ROUTING form 02~~ — **đã sửa 2026-09-12**, xem
  mục 1/3.
- **GAP mới (chưa đánh số, chưa xác nhận)**: không có ràng buộc code nào
  bắt buộc xác nhận khâu 2 (Kho NVL) trước khi lưu khâu 3 (Finish Good) cho
  cùng 1 traveler — 2 trang độc lập, chưa hỏi lại Andy có nên chặn cứng hay
  không. Xem mục 1.

---

## 8. Định hướng kiến trúc hiện đại (nghiên cứu thật 2026-09-13)

Andy yêu cầu đối chiếu thiết kế AVP Packing Flow với chuẩn ERP/kho vận hiện
đại — đã nghiên cứu bằng 3 nguồn thật (không suy đoán): tài liệu kiến trúc dự
án `D:\xekem` (KhoAI — hệ thống kho AI cho SME thực phẩm gốc Việt), Odoo 17
thật đang chạy (`docker`, DB `xekem`, truy vấn trực tiếp qua XML-RPC), và mô
hình quản trị AI 4 lớp trong `D:\Ops_Ai` (ISO/IEC 42001).

### 8.1 "Kho hiện đại chỉ khác nhau ký hiệu" — xác nhận đúng bằng cấu trúc Odoo thật

Truy vấn trực tiếp Odoo (`stock.location`, `stock.quant`, `stock.move`,
`stock.lot`, `stock.picking`) xác nhận đúng trực giác của Andy:

| Khái niệm cũ (kho vật lý riêng) | Odoo hiện đại (1 hệ thống, khác NHÃN) |
|---|---|
| "Kho nguyên liệu" và "Kho thành phẩm" là 2 nơi/2 sổ sách riêng | Chỉ là 2 **`stock.location`** khác nhau trong CÙNG 1 cây vị trí (vd `WH/Raw Materials`, `WH/Finished Goods`) — đổi nhãn `usage` (internal/customer/supplier/transit), không phải 2 hệ thống |
| Tồn kho = đi đếm tay từng kho | **`stock.quant`** — 1 bảng DUY NHẤT (product, location, lot) → quantity, luôn đúng ngay lập tức, tính từ sổ cái |
| Nhập/xuất kho = nghiệp vụ riêng biệt mỗi loại | **`stock.move`** — 1 sổ cái BẤT BIẾN duy nhất ghi mọi lần di chuyển giữa 2 location (nhận NVL, tiêu thụ sản xuất, xuất hàng đều là 1 "move") |
| Truy vết lô hàng = tra nhiều bảng | **`stock.lot`** — 1 lô, xem được toàn bộ lịch sử từ nguyên liệu → thành phẩm → khách hàng |
| "Packing Slip" là tài liệu riêng của AVP | Chính là **`stock.picking`** chuẩn Odoo — 1 "transfer" từ location nội bộ AVP → location khách hàng (Infasco), gộp nhiều `stock.move` |

**Áp vào AVP Packing Flow**: `RawMaterial`, `FinishGood`, `PackingList` hiện
là 3 SHEET/BẢNG tách biệt (đúng mô hình "kho cũ"). Theo chuẩn hiện đại, cả 3
nên là **1 sổ cái duy nhất** (Traveler/Pot#/Lot# + trạng thái: forecast →
received → packed → shipped), khác nhau chỉ ở giá trị 1 cột `stage`/`state`,
không phải 3 bảng riêng — đúng tinh thần "chỉ khác ký hiệu" Andy nói. Đây là
thay đổi CẤU TRÚC DỮ LIỆU lớn, cần Andy quyết định có làm không (rủi ro: phải
viết lại toàn bộ API + kiểm tra lại mọi cảnh báo đã có).

### 8.2 Tương tác hiện đại: email/giọng nói/ảnh, biết ngay trên điện thoại

Từ tài liệu KhoAI (`D:\xekem\KhoaiTechnicalArchitect.MD`) — hệ thống thật đã
thiết kế cho đúng nhu cầu Andy mô tả ("nói vào micro để biết tồn gì, cần xuất
gì, có đủ NVL cho công nhân hôm nay không — ngay trên điện thoại"):

- 3 kênh nhập liệu chính: **chụp ảnh hóa đơn** (AI đọc), **giọng nói**
  (Whisper chuyển giọng nói → Claude hiểu ý định → cập nhật kho), **chụp ảnh
  hàng loạt để đếm** (AI đếm, server gộp nhiều vị trí lại tự động).
- Tất cả xử lý AI nằm ở **backend** (điện thoại chỉ chụp/nói, không xử lý AI
  tại chỗ) — để cập nhật model dễ, không lộ API key, máy yếu vẫn dùng được.
- Có **hàng chờ duyệt** (Review Queue) riêng cho supervisor — đúng tinh thần
  "confirm trước khi ghi Sheet" AVP Packing Flow đã làm ở cả 4 khâu.

**AVP hiện đã có** phần lõi "chụp ảnh → AI đọc → người duyệt → lưu" (khâu 1
và 3). **Chưa có**: kênh giọng nói (hỏi tồn kho bằng lời), và 1 nơi duy nhất
trả lời "hôm nay có đủ NVL không" (hiện phải tự suy ra từ bảng Reconciliation
ở khâu 2, chưa có câu trả lời trực tiếp dạng hỏi-đáp).

### 8.3 "CCP — Critical Control Point": mô hình đã có, cần đặt tên chính thức

`Ops_Ai/docs/cl08_operation/SOFTWARE_ARCHITECTURE.md` định nghĩa mô hình
quản trị AI 4 lớp (tương đương HACCP CCP áp cho phần mềm):

```
LỚP 1 — AI SINH RA (OCR/Vision/giọng nói) — AI chỉ chạy ở đây
LỚP 2 — KIỂM TRA TỰ ĐỘNG (rule engine, không LLM) — mọi cảnh báo/công thức
LỚP 3 — QUẢN TRỊ RỦI RO (truy vết lô, phân quyền)
LỚP 4 — CON NGƯỜI + KIỂM TOÁN — CCP: điểm BẮT BUỘC người duyệt
```

**Đối chiếu AVP Packing Flow hiện tại — mọi CCP đã có, nhưng chưa gọi tên**:

| CCP | Ở đâu trong AVP | Mức rủi ro |
|---|---|---|
| CCP-1 | Mọi kết quả AI đọc (khâu 1, 3) đều dừng ở màn hình xem lại, KHÔNG tự ghi Sheet | Cao — đã có |
| CCP-2 | Cảnh báo Pot#/LOT#/Part# lệch (finish-good/confirm) — người phải xem trước khi lưu | Cao — đã có |
| CCP-3 | QC HOLD chặn cứng không cho duyệt Packing List | Nghiêm trọng — đã có (chặn cứng, không chỉ cảnh báo) |
| CCP-4 (mới, đề xuất) | Số Packing Slip — vẫn cần người gõ tay vì nguồn số nằm ngoài AVP (mục "PS" ở trên) | Nghiêm trọng — đã có (không tự sinh số) |

**Đề xuất**: đặt tên chính thức "CCP-1..4" ngay trong code/comment (giống
Ops_Ai đã làm), để mỗi khi thêm tính năng AI mới, tự hỏi "cái này có phải
CCP không, đã có người duyệt chưa" — thay vì phải nhớ rải rác qua nhiều file.

### 8.4 Tóm tắt khuyến nghị (chưa làm gì, chờ Andy quyết định độ ưu tiên)

1. **Nhỏ, an toàn**: đặt tên "CCP-1..4" vào comment code hiện có (không đổi
   hành vi, chỉ đặt tên) — làm ngay được nếu Andy đồng ý.
2. **Vừa**: thêm 1 endpoint "hỏi-đáp tồn kho" (vd `/api/status` trả lời "còn
   bao nhiêu Part# X chưa Shipped, có traveler nào đang HOLD") — nền tảng
   cho voice sau này, chưa cần giọng nói thật.
3. **Lớn, rủi ro cao**: gộp RawMaterial/FinishGood/PackingList thành 1 sổ
   cái duy nhất kiểu `stock.move` — thay đổi cấu trúc dữ liệu nền tảng, nên
   làm sau cùng và chỉ khi Andy chắc chắn cần.

### 8.5 Tầm nhìn MES đầy đủ (Andy đặt ra 2026-09-13) — dashboard, yield/scrap, OEE, delivery-date

Andy mô tả tầm nhìn 1 hệ thống MES (Manufacturing Execution System) đầy đủ:
quản lý quyết định ngay, biết chính xác tồn kho/trạng thái mọi loại (NVL,
bán thành phẩm, hư hỏng/HOLD, POT rỗng, pallet), phế liệu/tỷ lệ hao hụt,
hiệu suất máy, và biết chính xác ngày giao hàng không cần chờ hỏi. Đối
chiếu từng phần với chuẩn ngành + hiện trạng:

| Tầm nhìn | Tên chuẩn ngành | Hiện trạng | Độ khó |
|---|---|---|---|
| Dashboard tổng quan tức thời | Real-time dashboard | ❌ Chưa có, phải mở từng trang | Dễ — dữ liệu đã có |
| Yield/tỷ lệ hao hụt theo lô | First Pass Yield / Scrap rate | ❌ Chưa có, nhưng TÍNH ĐƯỢC ngay (Pieces nhận − Qty tốt đã đóng gói) | Dễ-vừa — chỉ cần công thức, không cần thu thập dữ liệu mới |
| Vòng đời container (POT/pallet) | Container/Asset tracking | ❌ Chưa có (xem mục 6b BRS_TRS.md) | Vừa |
| Ngày giao hàng chính xác | Available-to-Promise / Delivery-date feasibility (đúng tên Andy dùng ở Ops_Ai) | ❌ Chưa có, nhưng ước tính được bằng FIFO: (số traveler đang chờ trước) ÷ (tốc độ xử lý TB đo được, ~78,5/ngày) — **cần Andy xác nhận có đúng xử lý theo FIFO không** trước khi code | Vừa — dữ liệu có sẵn, cần xác nhận quy tắc ưu tiên |
| Hiệu suất máy (OEE) | Overall Equipment Effectiveness | ❌ Chưa có — **chưa từng có ở cả Excel cũ**, cần THU THẬP DỮ LIỆU MỚI (giờ máy chạy/dừng), không chỉ code thêm | Khó nhất — đổi cả cách ghi nhận lúc sản xuất |

**Phát hiện củng cố mạnh nhất cho hướng "1 sổ cái duy nhất" (mục 8.1)**:
2 file Excel thật (`PACKING SLIPS.xlsm` và `DAILY LOG CHECK SHEET.xlsm`)
mỗi file tự giữ 1 bảng "WORK ORDER" riêng (1.847 vs 5.224 dòng) — trong số
travelers có ở cả 2 bên, **797 traveler bị lệch SHIPPED/PS thật** (xem
BRS_TRS.md mục 6e) — không phải rủi ro lý thuyết, là lỗi ĐÃ XẢY RA vì có
nhiều "nguồn sự thật" song song cho cùng 1 khái niệm.

**Ước tính lợi ích lao động** (xem BRS_TRS.md mục 6c/6d): ~21-28 giờ lao
động hành chính/ngày hiện tại (ước tính, cần đo thật để xác nhận) — phần
lớn là "mò tìm số liệu" giữa nhiều sổ sách lệch nhau, không phải giá trị
sản xuất thật. Đây là phần AI+cấu trúc dữ liệu tốt cắt giảm được nhiều nhất.

**Giới hạn thành thật**: tự động hóa giảm mạnh phần "gõ + mò số liệu",
nhưng KHÔNG loại bỏ hoàn toàn nhu cầu con người — CCP (mục 8.3) đòi hỏi
người có chuyên môn thật sự đối chiếu, không phải "bấm Enter" vô tri; đếm
vật lý (pallet, POT) và xử lý ngoại lệ nghiệp vụ vẫn cần người.

---

*Viết/cập nhật 2026-09-13, dựa trên code thật (`webapp/src/app/api/`,
`webapp/src/lib/sheets.ts`) + 4 form giấy thật trong `Data/` + `BRS_TRS.md` +
nghiên cứu Odoo/KhoAI/Ops_Ai thật (mục 8). Cập nhật lại file này mỗi khi kiến
trúc đổi — đây là tài liệu sống.*
