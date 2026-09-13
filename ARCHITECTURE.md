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

*Viết/cập nhật 2026-09-12, dựa trên code thật (`webapp/src/app/api/`,
`webapp/src/lib/sheets.ts`) + 4 form giấy thật trong `Data/` + `BRS_TRS.md`.
Cập nhật lại file này mỗi khi kiến trúc đổi — đây là tài liệu sống.*
