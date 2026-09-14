# SESSION REPORT
## SES-20260913-002 — AVP Packing Flow (mới nhất, đọc mục này trước)

> Phiên này bắt đầu SAU khi SES-20260912-001 (bên dưới) đã đóng và
> commit+push. Nội dung SES-20260912-001 vẫn giữ nguyên bên dưới làm
> lịch sử — không xóa, chỉ nối tiếp lên trên theo đúng quy ước "AI không
> có trí nhớ giữa các phiên — session report thay thế trí nhớ đó".

---

## 1. THÔNG TIN PHIÊN

| Trường | Giá trị |
|---|---|
| Session | SES-20260913-002 |
| Ngày | 2026-09-13 (cùng ngày, sau khi SES-20260912-001 đã đóng) |
| Chủ dự án | Andy Phan (Viet), Maple Leaf Group |
| Git | **Đã commit + push nhiều lần trong phiên** (Andy xác nhận "commit push" nhiều lần). Repo gốc: push đủ lên `https://github.com/vietsharescom/AVP-AI.git`, commit cuối `99575d3`. Repo `webapp`: commit local `eea0cb0` (vẫn KHÔNG có remote riêng — GAP cũ, chưa giải quyết). |
| Trạng thái server | `localhost:3000` chạy `npm run start`, đã rebuild+restart nhiều lần trong phiên, lần cuối verify 200 OK. Tunnel cloudflared cùng link cũ (`bumper-thomson-conferencing-negotiation.trycloudflare.com`) vẫn sống lúc kiểm tra trong phiên — có thể đã đổi nếu máy/tunnel restart sau đó. |

---

## 2. ĐÃ HOÀN THÀNH TRONG PHIÊN NÀY

### 2.1 Xác nhận GAP-013 (redesign UI khâu 3/4) — Andy đã test, OK
Andy xác nhận hành vi "Import Excel chỉ lưu được traveler đã có RawMaterial"
là ĐÚNG THIẾT KẾ, không phải bug — xem chi tiết trong SES-20260912-001 mục
3.7 (giữ nguyên bên dưới, không lặp lại).

### 2.2 Sửa UX thật tìm thấy khi Andy tự test trên trình duyệt
- **Bảng "toàn bộ đã có sẵn" (0 dòng mới) từng hiện 3 lớp thông báo trùng
  lặp** (banner xanh "Đã lưu 0 dòng" + banner xanh dương "tổng 0" + banner
  đỏ lặp lại) — sửa: chỉ hiện banner khi thật sự có dòng mới lưu; bỏ banner
  đỏ lặp lại, giữ đúng 1 banner vàng liệt kê traveler trùng.
- **Bug thật: F5 refresh sau khi bấm Lưu vẫn hiện nút Lưu xanh như mới,
  bảng dữ liệu y hệt cũ** — nguyên nhân: bản nháp (`draft`) lưu localStorage
  chống mất dữ liệu, nhưng kết quả "đã kiểm tra trùng" (`duplicateWarning`/
  `allExisting`) thì KHÔNG lưu bền, nên F5 xong quên mất đã biết trùng. Sửa
  (`TravelerSection.tsx`): tự động re-check với RawMaterial ngay khi bản
  nháp xuất hiện (kể cả từ localStorage); nếu TOÀN BỘ travelẻr đã có sẵn thì
  tự dọn bản nháp luôn, báo rõ lý do; nếu MỘT PHẦN thì báo ngay, không đợi
  bấm Lưu.
- **Dọn 2 ô search nhỏ trùng chức năng** — khâu 1 (`TravelerSection.tsx`)
  và khâu 3 (`ScanSection.tsx`) có sẵn "Tra cứu trước khi upload" trùng với
  ô search chung (`GlobalSearchBar.tsx`) mới thêm phiên trước — theo yêu
  cầu Andy "search tổng phải mạnh nhất, search nhỏ dọn bớt": đã **nâng cấp
  search chung khớp MỌI field** (không chỉ vài cột cố định) + **thêm tab
  PackingList còn thiếu**, đồng thời **gỡ bỏ 2 ô search nhỏ**.
- **Thêm nút "▲ Thu gọn mục này"** ở cuối mỗi khâu (khi mở) — bảng dài (vd
  75 dòng khâu 1) trước phải cuộn lên đầu mới đóng lại được.

### 2.3 Redesign khâu 2 (Kho) theo phản hồi Andy
- Bỏ tiêu đề trùng lặp (Collapsible bar + `<h2>` bên trong cùng nói "2. Kho
  Nguyên Vật Liệu") — đổi tên bao quát hơn: **"2. Kho — Nguyên liệu · Bán
  thành phẩm · Thành phẩm · QA Hold"**.
- Bảng chuyển sang **full-width, `table-fixed`** (trước để trống nhiều
  khoảng trắng bên phải) + **sort được theo mọi cột** (bấm header, ▲/▼).
- Thêm cột **"Ngày nhận NVL"** (từ RawMaterial — 1 ngày/traveler, đáng tin)
  — ngày đóng gói từng LOT (có thể nhiều ngày khác nhau/traveler) đưa vào
  tooltip rê chuột thay vì 1 cột chính.

### 2.4 Tính năng mới: theo dõi phế liệu (scrap) từ Split Form
Andy hỏi ERP hiện đại theo dõi hao phí sản xuất thế nào — phát hiện: Split
Form có sẵn bảng **"Sorting Results / Defect # of PCS Found"** (Loose
Washer Nut, Damaged Pilot, Mixed...) chưa từng được đọc. Đã thêm:
- `extract.ts`: `ExtractedSplitForm.scrapEntries` (đọc bảng Defect, chỉ lấy
  dòng có số thật, không tự bịa 0 cho ô trống).
- `ScanSection.tsx`: tính `scrapQty`/`scrapDetail` từ `scrapEntries`, gán
  vào mọi dòng FinishGood sinh từ form đó.
- **2 cột thật mới trong Google Sheet FinishGood** (`scrapQty`,
  `scrapDetail`, cột W-X) — script `scripts/add-finishgood-scrap-columns.mjs`,
  chỉ thêm cột cuối, không đụng 22 cột/dữ liệu cũ (đã xác nhận Andy trước
  khi chạy). Ẩn mặc định trên UI cùng nhóm Type/O-C/Reject.
- **Còn thiếu**: đường nhập file `.xlsm` điện tử của Split Form (khi phân
  xưởng xuất file thay vì chụp ảnh) — chưa có file mẫu thật, chưa làm được.

### 2.5 Phát hiện lớn: Pot# và LOT NO. không đáng tin như tưởng
Đo lại trên **1.853 dòng WORK ORDER thật** (`PACKING SLIPS.xlsm`) sau khi
Andy chỉ ra "thùng chứa quay về đựng lại cùng Pot#":
- **Pot# bị TÁI SỬ DỤNG 41,1%** (469/1.141 Pot# gắn với ≥2 traveler khác
  nhau theo thời gian, vd Pot#1444 → 4 traveler khác nhau, 3 ngày khác
  nhau) — quyết định cũ "Pot# là khóa tốt" chỉ đúng **trong phạm vi
  traveler còn mở (chưa Shipped) tại cùng thời điểm**, KHÔNG đúng qua toàn
  lịch sử. Code cảnh báo trùng Pot# đã lọc đúng theo `shipped`, không bị
  ảnh hưởng — nhưng công cụ search/`/trace` tra toàn lịch sử thì chưa cảnh
  báo hiện tượng này (chưa sửa UI).
- **59% giá trị LOT NO. thật ra là CHỮ TRẠNG THÁI, không phải mã lot**:
  `TRAVELERRECEIVED` (53,8% — chưa đóng gói, KHÔNG phải lỗi), `SORT&RETURN`
  (1,7%), `SPLIT FROM TR#{traveler}` (~3,6%). Mã lot thật theo format
  `6-DDD-SS-L` chỉ 41% dữ liệu, gần như không bao giờ trùng. **Đã sửa code**
  (`finish-good/confirm/route.ts`, hàm `isLotPlaceholder`) loại 3 giá trị
  placeholder khỏi cảnh báo "LOT đổi khác thường" — tránh báo động giả khi
  chuyển từ `TRAVELERRECEIVED` sang mã lot thật (53,8% số dòng, hoàn toàn
  bình thường).
- **Sửa lại mô hình phân cấp**: Traveler# = 1 POT nguyên liệu cụ thể (không
  phải WO); **LOT NO. mới là 1 Work Order thật** — 1 traveler có thể sinh
  nhiều LOT/WO hợp lệ (khớp lại đúng Notes #8 cũ đã có từ trước, bị quên áp
  dụng). Case 717671 (GAP-004) theo mô hình đúng này CÓ THỂ là 2 WO hợp lệ,
  không mặc định là lỗi — vẫn cần chứng từ giấy gốc để kết luận chắc.

### 2.6 QUYẾT ĐỊNH DỨT ĐIỂM: hậu tố Part# là sản phẩm khác nhau thật (ảnh hưởng lớn)
Owner xác nhận trực tiếp: hậu tố Part# (`-L`/`-HT`/`-A`...) là **sản phẩm
riêng thật** (KHÔNG phải "phủ mạ vs regular" như giả thuyết ban đầu — Andy
đã đính chính, lý do cụ thể mỗi hậu tố CHƯA xác nhận), KHÔNG phải biến thể
vô hại dùng chung Quantity/box. Chỉ lỗi chính tả thuần túy (thiếu/thừa dấu
gạch ngang, vd `06512349AA`/`06512349-AA`) mới được gộp.
- **Sửa `partControl.ts`** (`findPartControlEntry`): bỏ hẳn cơ chế "bỏ hậu
  tố dò gần đúng" (rủi ro, đo được ảnh hưởng 85% Part# đang hoạt động) —
  chỉ giữ chuẩn hóa dấu gạch ngang thuần túy làm fallback duy nhất.
- **Tác động thật đo được**: 36/39 Part# đang hoạt động (92%, tăng từ 3)
  giờ đúng là "thiếu trong PartControl", cần bổ sung Quantity/box riêng —
  KHÔNG BLOCK gì (chỉ cảnh báo, giống hành vi cũ), nhưng QUANTITY trên
  Packing List sẽ lùi về dùng số ghi tay lúc scan (kém tin cậy hơn) cho tới
  khi bổ sung xong.
- Danh sách đầy đủ 36 Part# (kèm số traveler dùng) → file mới
  `BANG_MA_CAN_OWNER_DUYET.md` mục 1.

### 2.7 Phát hiện mới: mã Serial#/SSCC ở mức từng box — CHƯA đưa vào hệ thống
Trên nhãn đóng gói thật (Split Form) có dòng "Serial#: X to Y" — kiểm
chứng bằng số học: dải serial ĐÚNG BẰNG số carton (36 carton ↔ 36 số liên
tiếp; 15 carton ↔ 15 số liên tiếp, 2/2 lần khớp) → mỗi carton có 1 serial
riêng, đúng chuẩn GS1 **SSCC** (Serial Shipping Container Code). Đây là
mắt xích truy vết cấp box hiện **hoàn toàn chưa đọc** (schema
`packagingEntries` không có field này). Đã bàn với Andy nhưng **CHƯA
code** — chờ quyết định có làm không.

### 2.8 Owner cung cấp trực tiếp 6 thông tin quan trọng (qua Andy)
1. PO đi trước qua email, nguyên liệu thật về **cùng ngày** (không có độ
   trễ lớn như lo ngại ban đầu) — hạ độ khẩn cấp lo ngại "vi phạm nguyên
   tắc MRP" đã nêu trước đó.
2. Pot# do **Infasco cấp số** (không phải AVP tự đánh).
3. Quy trình nhận hàng thật: nhân viên kho **scan barcode trên thùng/bin
   Pot#** → tự động phân Pieces về đúng máy → sinh Split Form riêng cho
   máy đó — đây là chi tiết quan trọng cho thiết kế bước Goods Receipt còn
   thiếu (khối C trong báo cáo ERP).
4. LOT# từ Infasco **đôi khi giao trễ** — giải thích nguyên nhân
   `TRAVELERRECEIVED` phổ biến (đang chờ Infasco cấp LOT#, không phải lỗi).
5. Hậu tố Part# là sản phẩm riêng thật — xem mục 2.6.
6. Yêu cầu thống kê toàn bộ bảng mã hiện có cho Owner duyệt → file mới
   `BANG_MA_CAN_OWNER_DUYET.md`.

### 2.9 Tài liệu mới/cập nhật trong phiên
- **`CAU_HOI_CHO_OWNER_THEO_CHUAN_ERP.md` + `.docx`** (mới) — tổng hợp toàn
  bộ câu hỏi từ `Notes.md`+`LATEST_SESSION.md` theo 8 khối chuẩn ERP (Master
  Data → Sales Order → **Goods Receipt** → Work Order → QC → Traceability
  → Shipping → Legacy), kèm **Bảng quan hệ dữ liệu (Cardinality Map)**, sơ
  đồ truy vết Xuôi/Ngược, và **Lộ trình cải tiến 5 giai đoạn (0-4) xếp theo
  PHỤ THUỘC** (không phải theo độ khó — bài học từ chính vụ PartControl).
- **`BANG_MA_CAN_OWNER_DUYET.md` + `.docx`** (mới) — danh sách làm việc 7
  mục: 36 Part# thiếu Quantity/box, hậu tố quan sát được, mã máy (25 mã từ
  sheet `Status`), mã nhân viên quan sát được (chưa chính thức), mã thiết
  bị QC (chưa tìm thấy), quy ước LOT NO., Serial#/SSCC.
- **`BRS_TRS.md`** — mục 3b mới (Pot# tái sử dụng 41,1%, quy ước LOT NO.
  chi tiết).
- **`Notes.md`** — PHẦN 4 (câu #58-77) + PHẦN 5 (câu #78-80) — toàn bộ
  Q&A phiên này, bao gồm cả những câu đã tự sửa lại khi Andy chỉnh ("không
  phải mạ", "LOT mới là WO"...).
- **`LATEST_SESSION.md`** (file này) — mục "Quyết định đã chốt" đã sửa lại
  2 câu sai phạm vi (Pot# 1:1 "vĩnh viễn", Traveler# = WO).
- Sửa file `md_to_docx.py` (script nội bộ, không thuộc repo) — bảng dùng
  viền xám nhạt thay vì viền đen mặc định của Word.

---

## 3. TRẠNG THÁI HIỆN TẠI & GAP MỚI PHÁT SINH PHIÊN NÀY

- **GAP-015 (mới)**: Thiếu hẳn bước **Goods Receipt** (xác nhận nguyên
  liệu về kho vật lý thật) — RawMaterial hiện chỉ là dự báo email, không có
  checkpoint nào xác nhận "đã về kho thật" trước khi tính vào đối chiếu
  khâu 2. Tab `Warehouse` sinh ra đúng để làm việc này nhưng đang chết.
  Owner đã làm rõ quy trình thật (mục 2.8.3) — cần quyết định: ô nào trên
  Split Form là số nhận thật (Pieces hay Box QTY — "SL=PLT" Andy nhắc
  nhưng chưa xác định được field cụ thể), lưu vào đâu (hồi sinh
  `Warehouse` hay thêm field `RawMaterial`), nút chụp đặt ở khâu 2 hay
  dùng chung khâu 3.
- **GAP-016 (mới)**: **36 Part# đang hoạt động thiếu Quantity/box** trong
  PartControl (tăng từ GAP-007 cũ 21 mã, do sửa lại cách khớp đúng theo
  xác nhận Owner) — xem `BANG_MA_CAN_OWNER_DUYET.md` mục 1. Không block
  gì, chỉ giảm độ tin cậy QUANTITY trên Packing List cho các Part# này.
  Cùng dịp phát hiện: 5/21 mã GAP-007 cũ thực ra là lỗi chính tả (thiếu/
  thừa gạch ngang hoặc tiền tố "P"), không phải mã mới — nên tự gộp lại
  thay vì thêm dòng.
- **GAP-017 (mới)**: Mã Serial#/SSCC (mức box/carton) có sẵn trên giấy
  nhưng hệ thống chưa đọc — xem mục 2.7. Chưa quyết định có làm không.
- ~~GAP-013~~: **ĐÃ TEST, OK** — Andy xác nhận hành vi Import Excel đúng
  thiết kế (xem mục 2.1).
- Các GAP cũ (GAP-001, 003, 004, 005, 007→016, 008, 009, 010, 012, 014)
  từ SES-20260912-001 **vẫn còn nguyên trạng, chưa xử lý thêm trong phiên
  này** trừ khi ghi rõ ở trên — xem đầy đủ ở phần lịch sử bên dưới.

### Việc đang mở (chưa quyết định) — MỚI phiên này
- Có làm bước Goods Receipt (GAP-015) không, và làm thế nào — đang chờ
  Owner xác nhận field nào là số thật trên Split Form.
- Có đọc Serial#/SSCC vào hệ thống không (GAP-017).
- Ý nghĩa CỤ THỂ từng hậu tố Part# (`-L`, `-HT`, `-A`, `-T`, `-B`, `-X`,
  `-MY-AK`...) — biết là sản phẩm khác nhau rồi, chưa biết mỗi ký hiệu là
  gì.
- Danh sách mã máy (25 mã, sheet `Status`) và mã nhân viên — cần Owner xác
  nhận còn đúng/đủ, và cung cấp danh sách nhân viên chính thức.
- Mã thiết bị QC (Hardness/Tensile/Proof Load) — chưa tìm thấy trên Split
  Form, chưa biết có tồn tại ở đâu khác không.

---

## 4. BẮT ĐẦU PHIÊN SAU TỪ ĐÂY

### Việc ưu tiên tiếp theo
1. Owner trả lời câu hỏi Giai đoạn 0 trong `CAU_HOI_CHO_OWNER_THEO_CHUAN_ERP.md`
   (ô nào là số nhận kho thật, ý nghĩa hậu tố từng loại, MC# máy hay trạm)
   — mọi việc code tiếp theo ở khối Goods Receipt/PartControl phụ thuộc
   trực tiếp vào đây.
2. Rà soát dần `BANG_MA_CAN_OWNER_DUYET.md` — ưu tiên mục 1 (36 Part#
   thiếu Quantity/box, đã sắp theo số traveler dùng nhiều nhất trước).
3. Quyết định có làm Goods Receipt (GAP-015) và Serial#/SSCC (GAP-017)
   không — 2 việc mới, ảnh hưởng lớn nếu làm.
4. Các việc cũ từ SES-20260912-001 vẫn treo — xem mục 5 phần lịch sử bên
   dưới (GAP-004 717671, GAP-008/010 sửa tay Sheet, GAP-012 MC#...).
5. Kiểm tra lại link tunnel — có thể đã đổi nếu máy/tunnel restart.

### Cảnh báo cho phiên sau
- ⚠ File Word gốc `CAU_HOI_CHO_OWNER_THEO_CHUAN_ERP.docx` Andy có thể đang
  mở/chỉnh tay trực tiếp trong Word — kiểm tra khóa file trước khi ghi đè,
  đọc note Andy thêm vào (nếu có) trước khi thay thế nội dung.
- ⚠ `partControl.ts` vừa đổi hành vi — nếu thấy nhiều cảnh báo "Part#
  không có trong PartControl" hơn hẳn trước đây, đó là ĐÚNG dự kiến (tăng
  từ 3 lên 36), không phải bug mới.
- ⚠ Toàn bộ cảnh báo cũ dựa trên "Traveler# = WO" hoặc "Pot# = khóa vĩnh
  viễn" cần đọc lại đúng phạm vi mới (mục 2.5) trước khi dùng để giải
  thích cho Owner.

---

*Session Report SES-20260913-002 — cập nhật lần cuối 2026-09-13 (đóng
phiên, đã commit+push). Phần dưới đây (SES-20260912-001) là lịch sử phiên
trước, giữ nguyên không sửa.*

---
---

# SESSION REPORT (LỊCH SỬ — PHIÊN TRƯỚC)
## SES-20260912-001 — AVP Packing Flow

> **ĐỌC FILE NÀY TRƯỚC** khi bắt đầu phiên làm việc tiếp theo.
> Đây là nguồn thông tin duy nhất để AI nhớ lại ngữ cảnh giữa các phiên
> (theo đúng nguyên tắc: "AI không có trí nhớ giữa các phiên — session
> report thay thế trí nhớ đó"). Đọc thêm `ARCHITECTURE.md` (pipeline
> input/processing/output chi tiết) và `BRS_TRS.md` (công thức/ánh xạ cột)
> — cả 2 đã cập nhật theo phiên này.

---

## 1. THÔNG TIN PHIÊN

| Trường | Giá trị |
|---|---|
| Session | SES-20260912-001 (phiên rất dài, kéo qua 2 ngày, nhiều hạng mục) |
| Ngày | 2026-09-11 → 2026-09-13 (đóng phiên) |
| Chủ dự án | Andy Phan (Viet), Maple Leaf Group |
| Thư mục | `D:\AVP_AI` |
| Web app | `D:\AVP_AI\webapp` (Next.js + TypeScript) |
| Database | Google Sheets, ID `1TWYKdRHv3MioEBf2DHAvUGlCvZxsr-KXQKLCR_3BkJ4` |
| Git | **Đã commit + push CẢ 2 repo lúc đóng phiên** (Andy xác nhận "commit push" 2026-09-13). Repo gốc: push lên `https://github.com/vietsharescom/AVP-AI.git`. Repo `webapp`: commit local (KHÔNG có remote GitHub riêng — xem GAP cũ mục 4, vẫn chưa giải quyết). |
| Link demo | https://bumper-thomson-conferencing-negotiation.trycloudflare.com (đã kiểm tra còn sống lúc đóng phiên 2026-09-13 20:xx — tunnel tạm, sẽ chết khi tắt máy/tunnel restart, link đổi mỗi lần) |
| Trạng thái lúc đóng phiên | Server `localhost:3000` chạy `npm run start` (production build, đã rebuild+restart rất nhiều lần trong phiên, lần cuối verify 200 OK). Cần restart (`npm run build && npm run start`) nếu máy tắt/mở lại. |

---

## 2. TRẠNG THÁI KIỂM THỬ

Không có test suite tự động. Xác nhận bằng `npm run build` + `npm run lint`
— PASS sau mỗi thay đổi trong phiên. Test thực tế bằng:
- Dữ liệu thật từ `Data/Evaluation/` (PO193842, PO193853, PO193874,
  PO193890, PO193902, PO193904, PACKING SLIP-1/2) — bulk-upload vào
  RawMaterial qua UI thật (không phải chỉ gọi API).
- Traveler test riêng (TEST900001-5, TEST-PARTIAL...) để kiểm chứng từng
  thuật toán/bug fix bằng dữ liệu tách biệt, không đụng dữ liệu thật.
- Bộ mẫu `Test/README.md` (traveler 717844/718439/718440, đối chiếu 100%
  với Packing Slip 30098 thật) — đã chạy đủ 4 khâu, khớp chính xác.

---

## 3. ĐÃ HOÀN THÀNH TRONG PHIÊN NÀY

### 3.1 Bảo mật (đầu phiên)
- Phát hiện + xử lý secret thật bị commit nhầm vào git ở repo gốc
  (`.env`, 2 file `client_secret_*.json`) — đã gỡ khỏi git history (chỉ 1
  commit, chưa từng lên GitHub nên an toàn để amend), thêm `.gitignore`.
  Khuyến nghị Andy rotate key Groq/Gemini/OAuth cho chắc (Andy tự làm,
  ngoài khả năng của AI).

### 3.2 Xây dựng ban đầu (đã có từ trước, xem lại phiên trước) + mở rộng phiên này
4 khâu cốt lõi (Email → Kho NVL → Finish Good → Packing List) đã có từ
trước; phiên này **redesign/sửa/mở rộng đáng kể**:

- **Khâu 2 (Kho NVL) — REDESIGN HOÀN TOÀN**: bỏ form nhập tay số thực
  nhận. Giờ là **bảng đối chiếu tự động**: Nguyên liệu vào (Pieces, khâu
  1) vs Thành phẩm ra (tổng tất cả LOT NO. đã scan ở khâu 3, cộng dồn
  theo Traveler#) → tự tính chênh lệch/% lệch, cờ HOLD (từ QC đọc ở form
  02). API: `GET /api/warehouse/list` viết lại hoàn toàn (trước là danh
  sách chờ xác nhận tay, giờ là bảng đối chiếu). Route `warehouse/confirm`
  cũ vẫn còn (dead code, không xóa, không dùng nữa).
- **Khóa (Traveler#, LOT NO.)** thay cho chỉ Traveler# — sửa
  `finish-good/confirm` (cảnh báo trùng), `packing-list/generate`
  (cảnh báo scan trùng), `packing-list/approve` (chặn trùng trong 1 PS,
  chỉ đánh Shipped đúng dòng khớp cả Traveler#+LOT NO.). `sheets.ts`:
  `updateRowsWhere` giờ nhận **mảng** điều kiện lọc phụ (trước chỉ 1).
  → **Đã fix GAP-002 cũ** (xem mục 4).
- **Xuất lẻ (partial shipment)**: duyệt Packing List với Box/Quantity
  NHỎ HƠN số gốc của dòng FinishGood → tách dòng gốc thành phần đã xuất
  (Shipped=TRUE, PS mới) + phần còn lại (Shipped=FALSE, giữ nguyên để
  xuất chuyến sau). Lý do nghiệp vụ: 1 chuyến xe luôn muốn đầy xe, cần
  trộn 1 phần từ nhiều lô khác nhau, không phải luôn xuất trọn 1 lô. Đã
  gặp + sửa 1 bug thật lúc test (thứ tự append/update sai khiến dòng mới
  bị ghi đè số sai) — đã fix bằng cách update dòng gốc TRƯỚC, append dòng
  mới SAU, khóa bằng `createdAt` để chỉ trúng đúng 1 dòng.
- **4 thuật toán tự động kiểm tra khi lưu** (khâu 1 `raw-material/confirm`
  + khâu 3 `finish-good/confirm`), để không phải dò tay từng dòng:
  1. **Pot# trùng container**: Pot# là mã thùng chứa vật lý dùng lại được
     — nếu Pot# đang gắn với 1 traveler KHÁC chưa Shipped, cảnh báo (không
     chặn).
  2. **Part# chưa có PartControl** — cảnh báo ngay lúc lưu khâu 3, không
     đợi tới khâu 4 mới biết.
  3. **Traveler# lệch dải số** — so với median cùng 1 lần upload/scan,
     lệch >2000 thì cảnh báo (khả năng AI đọc nhầm chữ số).
  4. **Tổng theo lô (batch totals)** — hiện tổng Weight/Pieces (khâu 1)
     hoặc Boxes/Qty (khâu 3) **NGAY KHI ĐANG XEM** (trước khi lưu, không
     phải chỉ sau khi lưu) để đối chiếu nhanh với tổng in trên giấy, thay
     vì dò từng dòng từng số.
  5. **Chặn lưu trùng y hệt** (khâu 3): cùng Traveler#+LOT NO.+Boxes+Qty,
     chưa Shipped → tự động bỏ qua, không lưu lại; cùng Traveler#+LOT NO.
     nhưng SỐ KHÁC → vẫn lưu, chỉ cảnh báo (có thể là đóng gói đợt 2 hợp
     lệ).
- **Nút "Hủy"** ở khâu 1 và khâu 3 — xóa sạch dữ liệu đang nhập nếu lỡ
  scan nhầm file/mục khác.
- **Chặn bấm Lưu lặp vô nghĩa**: nếu TOÀN BỘ dòng còn lại đều đã tồn tại
  sẵn (0 dòng mới), nút "Xác nhận & Lưu" tự khóa + đổi nhãn, tới khi sửa/
  xóa dòng mới mở khóa lại.
- **Ô tra cứu nhanh** ở khâu 1 (`/inbox`) — gõ PO/Traveler/Part#/Pieces để
  biết ngay đã nhập vào RawMaterial chưa, trước khi tốn công upload.
- **Import Excel** (khâu 3): `POST /api/scan/extract-excel` — đọc thẳng
  `DAILY LOG CHECK SHEET.xlsm` (sheet `CHECKING SUMMARY`), không qua AI,
  chỉ lấy dòng CHƯA Shipped, chặn an toàn nếu >300 dòng. Dùng thư viện
  `xlsx` (SheetJS) — có 1 CVE mức cao (ReDoS/prototype pollution) chưa vá
  trên npm, chấp nhận vì chỉ nhân viên nội bộ upload, không phải input
  công khai (đã thử `exceljs` an toàn hơn trước nhưng không đọc được file
  thật do AutoFilter date-grouping).
- **Đọc mở rộng form 02 (Split Form)**: schema `extractSplitForm` thêm
  `finishedPartNo` (Finished Part Number trên sticker, khác `partNo` đầu
  form — CHƯA xác nhận cái nào đúng để link khâu sau), `boxQty`,
  `qcStatus` (PASS/HOLD, suy từ khu FINAL INSPECTION), `qcNotes`. Đã thêm
  cột `qcStatus` thật vào Google Sheet tab `FinishGood` (cột V, chỉ thêm
  header, không đụng dữ liệu cũ).
- **Đối chiếu chéo PO Number**:
  - Ảnh: xác định barcode PO theo VỊ TRÍ (luôn ở trên cùng), không so
    khớp với giá trị AI đã đọc (cách cũ dễ bỏ sót lỗi vì tự tham chiếu
    vòng).
  - PDF gốc điện tử (không phải ảnh scan): đọc thẳng **text layer thật**
    của PDF (thư viện `pdf-parse`/`pdfjs-dist`) để lấy PO Number, không
    qua AI đoán. Bắt được lỗi thật: Gemini đọc `PO193902.pdf` (3 trang)
    thành PO "193904" — **là số PO THẬT của 1 file khác** (`PO193904.pdf`
    cũng có sẵn), không phải nhiễu ngẫu nhiên. Traveler 718865 hiện đang
    lưu sai `po=193904` (đúng phải là `193902`) — **CHƯA SỬA**, xem mục 4.
  - ⚠ Gặp + sửa xong 1 bug hạ tầng: Next.js/Turbopack không đóng gói đúng
    file "worker" mà `pdfjs-dist` cần, khiến tính năng thất bại âm thầm
    (lỗi bị catch, trả về null, không log). Sửa bằng
    `serverExternalPackages: ["pdf-parse", "pdfjs-dist"]` trong
    `next.config.ts`.
- **Packing List UI**: sort theo PO/Part#/Traveler/Pot# (bấm header), ô
  tìm nhanh (PO/Part#/Traveler/Pot#/Lot#/Description), bảng co giãn theo
  % cột cố định (`table-fixed`) thay vì phải kéo thanh cuộn ngang, cột
  LOT# hiển thị, cột **Trạng thái** (Sẵn sàng / 🛑 HOLD) — **chặn duyệt**
  nếu còn dòng HOLD chưa xử lý.
- **Nút "Thu gọn/Mở rộng"** cho khâu 2 (đỡ chiếm màn hình ở trang gộp 4
  bước `/`).

### 3.3 Xác minh nghiệp vụ quan trọng (đối chiếu dữ liệu thật, không suy đoán)
- **Packing Slip = 1 chuyến xe/lần xuất hàng thật**, KHÔNG gắn với 1 PO cụ
  thể — xác minh bằng 2 Packing Slip thật (PS 30098 gộp 9 PO khác nhau,
  PS 30101 gộp 11 PO khác nhau), bằng chứng: header có `Total Pallets`/
  `Total Empty` (sức chứa xe tải), không liên quan PO.
- **1 PO → nhiều Traveler → nhiều LOT NO.** — xác nhận với Andy, LOT NO.
  mỗi lot ~ 1 Work Order riêng theo ERP. Nhưng kiểm tra dữ liệu LỊCH SỬ
  THẬT (`WORK ORDER` 1.853 dòng, `CHECKING SUMMARY` 1.815 dòng): hiện
  tượng 1 traveler có **>1 LOT NO. thật sự khác nhau gần như KHÔNG xảy
  ra** (0/1598 và 5/1745 — 5 case đó là do field ghi "SORT&RETURN", không
  phải lot thứ 2 thật). Nghĩa là case 717671 (2 dòng, 2 lot khác nhau) có
  thể vẫn là **bất thường/lỗi thật**, không hẳn là quy luật phổ biến —
  xem GAP-004.
- **"PO" thực chất = đơn hàng gia công đầu vào** Infasco giao AVP sản
  xuất, không phải AVP đi mua hàng — giữ nguyên tên field `po` trong code/
  Sheets/UI, chỉ ghi chú lại bản chất (không đổi tên field).
- **Đối chiếu 4 form giấy thật với 4 khâu** (xem `ARCHITECTURE.md` mục 1):
  form 02 (Split/Work Order/QC) là nguồn DÙNG CHUNG cho khâu 2 (ROUTING
  "Receive Infasco Nut") và khâu 3 (2 sticker PACKAGING) — không phải 2
  tài liệu độc lập.
- **"Box QTY" trên form 02** (ví dụ 31,000) nhiều khả năng = tổng Pieces
  gốc của cả Pot trước khi bị chia lô (giống ý nghĩa cột Pieces form 01),
  KHÔNG phải số đã đóng gói — vì Weight thật luôn ~150-1400, còn Pieces
  thật trải rộng 4.700-210.000 (31.000 khớp Pieces hơn). **Chưa xác nhận
  100%** — traveler mẫu (717365) không có trong bất kỳ PO/PS thật nào để
  đối chiếu độc lập.

### 3.4 File tài liệu đã cập nhật trong phiên
- `ARCHITECTURE.md` (mới, phòng gốc) — pipeline input/processing/output
  đầy đủ, đối chiếu 4 form giấy thật, đã cập nhật theo mọi thay đổi trên.
  Cũng xuất bản artifact (link riêng, đã publish 3 lần cập nhật).
- `BRS_TRS.md` — thêm mục PO/SO, cardinality 1 PO→nhiều Traveler→nhiều
  LOT, ví dụ thật Part# `HL3W-4320-AA-L` × 3 traveler, LOT NO. không xuất
  hiện trên Packing Slip.
- `Test/README.md` (mới) — bộ mẫu test PS 30098 (traveler 717844/718439/
  718440), đối chiếu 100% với chứng từ thật, dùng để chạy pipeline test
  end-to-end.

### 3.5 Tiếp tục phiên 2026-09-12 → 13 (nghiên cứu data thật + sửa bug thật)

- **Nghiên cứu cardinality bằng dữ liệu thật** (1.811 dòng `CHECKING SUMMARY`,
  không suy đoán) — kết quả **đính chính** giả định cũ ở mục 3.2 phiên trước
  ("717671 có thể là 2 lot hợp lệ"):
  - **Traveler → LOT#, Traveler → Part#, Traveler → Pot# đều 1:1 TUYỆT ĐỐI
    (100%, 1.598/1.598)** sau khi chuẩn hóa khác biệt hoa/thường (~5,6% khác
    biệt ban đầu chỉ do hoa/thường, không phải lot thật khác nhau).
  - LOT# → Traveler (chiều ngược) có 3/1.595 ngoại lệ thật — xác nhận khóa
    (Traveler#, LOT NO.) vẫn đúng, không đổi.
  - **SKID# không phải khóa 1:1** — 77,4% skid chứa nhiều traveler khác nhau
    (kệ/pallet dùng chung) — không dùng SKID# để phát hiện trùng lặp.
  - Machine/MC#+Ca+Ngày → Traveler: 5,66% có nhiều traveler dùng chung —
    **chưa rõ** là bình thường (máy chạy nối tiếp nhiều traveler/ca) hay lỗi
    thật có sẵn — Andy chọn "chưa rõ, để sau", cảnh báo "trùng máy" vẫn giữ
    nguyên trong code, chưa xác nhận độ tin cậy.
  - Đã cập nhật `BRS_TRS.md` mục 3 với toàn bộ số liệu trên.
- **Bug thật tìm thấy khi đối chiếu Sheet sống**: traveler **717771** —
  FinishGood lưu Pot#=282 nhưng RawMaterial ghi Pot#=18 cho cùng traveler
  (dữ liệu thật, không phải giả lập). **CHƯA SỬA trong Sheet** — Andy chọn
  tự sửa tay (tab FinishGood, cột F `pot`, dòng traveler 717771: 282→18) —
  **chưa xác nhận đã sửa xong**.
- **Bug gốc rễ đã tìm ra + SỬA + VERIFY bằng Gemini thật**: file
  `Data/Evaluation/SCANNING SHEET-1.pdf` có trang PDF bị nhúng SIDEWAYS
  (khổ dọc 612×792 nhưng nội dung là bảng ngang, không có cờ `/Rotate`) —
  khiến Gemini đọc lộn 2 dòng traveler kề nhau (717770 mất hẳn, dữ liệu lẫn
  vào 717797). Đã sửa `splitPdfPages` trong `webapp/src/lib/extract.ts` tự
  phát hiện trang khổ dọc và xoay 270° trước khi gửi AI — **đã test lại
  bằng gọi Gemini thật**, cả 2 traveler đọc đúng 100% sau khi sửa.
- **Đã thêm các thuật toán cảnh báo mới** trong
  `webapp/src/app/api/finish-good/confirm/route.ts`:
  1. Cảnh báo mạnh khi cùng Traveler (chưa Shipped) có LOT#/Part#/Pot# KHÁC
     dòng đã lưu trước — dựa trên bằng chứng 1:1 100% ở trên.
  2. Cảnh báo "nghi ngờ trùng scan (double)" — cùng traveler, Boxes/Qty
     giống hệt nhau nhưng LOT# khác (đúng mẫu case 717671).
  3. Cảnh báo "trùng máy" — cùng MC#+ngày+ca nhưng khác traveler (độ tin cậy
     CHƯA XÁC NHẬN, xem trên).
  4. **Lưới an toàn Pot#**: nếu Pot# scan khác Pot# đã biết ở RawMaterial
     cho cùng traveler → cảnh báo mạnh + gợi ý thẳng traveler đúng có thể là
     ai (tra ngược qua Pot#, vì Part# một mình không đủ định danh — 1 Part#
     lặp lại ở nhiều traveler).
  - Đã thử áp dụng barcode (kỹ thuật đã dùng cho khâu 1) cho khâu 3 nhưng
    **test thật bằng zxing cho thấy Scanning Sheet KHÔNG có barcode** —
    không áp dụng được, đã loại phương án này.
- **Sửa UI khâu 3** (`webapp/src/components/ScanSection.tsx`):
  - Thêm ô "Tra cứu trước khi upload" (giống khâu 1) — tra theo
    Traveler/Part#/Pot#/LOT#, hiển thị đồng thời dữ liệu RawMaterial +
    FinishGood đã scan trước đó (route mới `GET /api/finish-good/list`).
  - Nút upload chính giờ nhận cả PDF (trước chỉ `image/*`).
  - Bảng duyệt: thêm lại cột **Date** (đưa lên đầu, khớp thứ tự tờ giấy gốc
    — trước đó Date bị ẩn hoàn toàn, AI tự điền ngày hôm nay không ai kiểm
    tra). Thêm nút ẩn/hiện cột Type/O-C/Reject (mặc định ẩn, hiện chưa chỗ
    nào trong code dùng lại 3 cột này).
- **Sửa GAP-008 (traveler 718865, po sai)**: Andy đã xác nhận đúng là
  193902 (đối chiếu ảnh PO193902.pdf trang 3/3), chọn tự sửa tay (tab
  RawMaterial, cột F `po`, dòng traveler 718865: 193904→193902) —
  **chưa xác nhận đã sửa xong**.
- **Chưa làm** (đề xuất, chưa có quyết định): bảng quy ước/reference table
  cho các field dạng enum (Shift/Machine/Operator code/Type) để đối chiếu
  OCR — **ĐÃ TÌM ĐƯỢC NGUỒN THẬT** ở mục 3.6 dưới (sheet `Status` trong
  PACKING SLIPS.xlsm có sẵn danh sách 25 mã máy hợp lệ) — chưa code, chỉ
  mới ghi nhận nguồn.

### 3.6 Tiếp tục phiên 2026-09-13 (redesign UI khâu 3/4 + nghiên cứu sâu Excel cũ + báo cáo cuối phiên)

**A. Redesign UI theo yêu cầu Andy (đã code, CHƯA test lại trên trình duyệt
thật sau lần sửa cuối — ưu tiên #1 phiên sau):**
- Khâu 3 (Scan): sửa dedupe khi gộp dòng từ nhiều nguồn (Scanning Sheet +
  Split Form trùng cùng 1 sự kiện), sắp lại thứ tự cột theo độ quan trọng,
  đổi toàn bộ độ rộng cột sang `%` (`table-fixed`) thay vì `ch`/`rem` cố
  định — co giãn thật theo màn hình, gộp Shift/Operator/Machine/... vào 1
  nhóm ẩn mặc định (toggle). Sửa bug thật: cờ khóa nút "Lưu" không được lưu
  bền (localStorage) nên tải lại trang là mất tác dụng — đã sửa.
- Khâu 4 (Packing List): thêm ô search trung tâm ở đầu trang
  (`GlobalSearchBar.tsx`) tra xuyên RawMaterial+FinishGood+PartControl, thu
  gọn khâu 1-3 mặc định bằng `Collapsible.tsx`+`HomeClient.tsx` (khâu 4 mở
  sẵn ngay dưới ô search), thêm "+ Thêm dòng theo Traveler#" (tự điền
  PO/Part#/Pot# — đúng thao tác VLOOKUP thật nhân viên đã quen, tách logic
  dùng chung ra `packingListLine.ts` + route mới `/api/packing-list/line`),
  thêm bước "Xem trước Packing Slip" (preview đúng bố cục chứng từ thật)
  trước khi bấm nút Duyệt thật, dời ô "Số Packing Slip" lên gần ô search
  (**vẫn giữ nhập tay** — xem lý do ở mục B).
- File thay đổi: `webapp/src/components/{ScanSection,PackingListSection,
  Collapsible,GlobalSearchBar,HomeClient}.tsx`, `webapp/src/lib/
  packingListLine.ts`, `webapp/src/app/api/packing-list/{generate,line}/
  route.ts`, `webapp/src/app/page.tsx`. Build+lint pass sau mỗi bước, server
  đã restart nhiều lần — nhưng **Andy chưa xác nhận đã thử lại giao diện
  mới trên trình duyệt thật sau các sửa cuối** (đặc biệt: search trung tâm,
  add-by-traveler, preview Packing Slip).

**B. Nghiên cứu sâu 2 file Excel cũ — phát hiện lớn nhất phiên này:**
- **PACKING SLIPS.xlsm không có công thức nào tự sinh số PS** (gõ tay,
  xác nhận bằng đọc trực tiếp ô — không có formula) → giữ nguyên quyết định
  không tự động sinh số PS trong AVP_AI (rủi ro trùng số PS thật ngoài đời
  vì dãy số dùng chung nhiều khách hàng — xem BRS_TRS.md mục 6c).
- **Cột SHIPPED/PS trong WORK ORDER dùng công thức VÒNG LẶP tự tham chiếu**
  (circular, cần bật "Iterative Calculation") — kiểm tra thật thấy 1 số
  dòng vẫn còn công thức sống (rủi ro tính lại sai), số khác đã bị nhân
  viên tô vàng + Paste-Values khóa cứng tay để "cứu" dữ liệu — xác nhận
  thiết kế `shipped=TRUE` ghi 1 lần tường minh của AVP_AI là đúng hướng.
- **PHÁT HIỆN LỚN NHẤT**: `PACKING SLIPS.xlsm` và `DAILY LOG CHECK
  SHEET.xlsm` mỗi file giữ 1 bảng "WORK ORDER" RIÊNG (1.847 vs 5.224
  traveler) — trong số trùng nhau, **797 traveler bị lệch SHIPPED/PS
  thật**. `SUMMARY IN&OUT` (báo cáo KPI ngày cho quản lý) **0/30 ngày mẫu
  khớp với WORK ORDER thật**, có ngày lệch tới 105 dòng — báo cáo KPI hiện
  tại của AVP **không đáng tin**. Chi tiết đầy đủ: `BRS_TRS.md` mục 6b-6e.
- Tìm thấy sheet `Status` (PACKING SLIPS.xlsm) có sẵn **danh sách 25 mã
  máy hợp lệ thật** (BF 102-112, MC 4-112, PCKY, TABLE 1/2) — nguồn dữ liệu
  gốc cho việc validate MC#/Machine sau này, và 1 danh sách trạng thái sản
  xuất 5 bậc (Loaded/Not started/Preloaded/Running/Shipped) **được thiết kế
  nhưng chưa bao giờ dùng thật**.
- Xác nhận qua Odoo 17 thật (XML-RPC, DB `xekem`): mô hình nghiệp vụ AVP
  đúng chuẩn ERP gọi là **"Subcontracting"** — vật liệu vẫn là của khách
  hàng (Infasco) trong lúc nằm ở AVP, khớp field `is_subcontractor`/
  `property_stock_subcontractor` của Odoo thật.
- Xác nhận: **AVP thực ra phục vụ hàng chục khách hàng khác ngoài Infasco**
  (PartControl thật: Jobal, Thompson Fasteners, Ultra Form, Dana Canada,
  S & E MANUFACTURING...) — giải thích tại sao dãy số PS không chạy liên
  tục +1.
- Ước tính lao động hành chính (có giả định rõ ràng, CHƯA đo thật):
  ~21-28 giờ/ngày, quy năm ~5.250-7.000 giờ/năm — chi tiết `BRS_TRS.md`
  mục 6c/6d.

**C. Tài liệu/báo cáo cuối phiên (đã tạo mới, đã publish):**
- `BAO_CAO_DANH_GIA_HE_THONG.md` + artifact "Kiểm Toán Quy Trình AVP"
  (https://claude.ai/code/artifact/e7b4f07b-12e3-4ebe-933e-5f918f6d8162) —
  báo cáo đánh giá hiện trạng, bảng 8 giai đoạn × 7 cột, đối chiếu 8 nhóm
  nguồn lực ERP, dùng để thuyết phục chủ.
- `Bao_Cao_Danh_Gia_He_Thong_AVP.docx` — bản Word cùng nội dung, đã gửi
  cho Andy qua SendUserFile.
- `THIET_KE_HE_THONG_MOI.md` + artifact "Thiết Kế Hệ Thống AVP Mới"
  (https://claude.ai/code/artifact/b6347250-5c6d-4378-8c9a-79ba735e18bb) —
  thiết kế đề xuất: bảng Input/Output 8 công đoạn theo ERP hiện đại (voice/
  camera/OCR), so sánh hiện tại vs mới, sơ đồ kiến trúc phân lớp + pipeline
  — tham khảo trực tiếp Odoo 17 thật + Ops_Ai (ISO 9001 Cl.8.7 PASS/HOLD/
  CONCESSION, equipment_status) + KhoAI (voice/camera capture).
- Andy tự tạo thêm thư mục `Report/` (copy các file trên + PDF export 2
  artifact) — không đụng vào, chỉ đã thêm vào git nếu Andy xác nhận commit.

---

## 4. TRẠNG THÁI HIỆN TẠI & LỖ HỔNG ĐÃ BIẾT

**Giai đoạn:** MVP mở rộng, nhiều thuật toán an toàn hơn phiên trước. Vẫn
CHƯA sẵn sàng dùng thật hàng ngày — còn vài case dữ liệu thật cần Andy
xác nhận trước.

### Lỗ hổng đã biết (GAP)
- **GAP-001**: Cổng email tự động (đọc hộp thư Infasco) CHƯA làm — vẫn
  upload tay.
- ~~GAP-002~~: **ĐÃ SỬA** — khóa Shipped/trùng giờ theo (Traveler#, LOT
  NO.), không còn theo Traveler# đơn thuần. Đã kiểm chứng bằng traveler
  test thật (xuất lẻ 2 lần liên tiếp, khóa đúng dòng).
- **GAP-003**: Không lưu file ảnh/PDF gốc kèm mỗi dòng dữ liệu.
- **GAP-004**: ⚠ **VẪN MỞ, BẰNG CHỨNG MẠNH HƠN NHIỀU (2026-09-12/13)** —
  traveler 717671 (2 dòng FinishGood, LOT NO. khác nhau). Nghiên cứu lại
  bằng 1.811 dòng lịch sử thật: Traveler→LOT# là **1:1 tuyệt đối 100%**
  (1.598/1.598, không 1 ngoại lệ), nên case 717671 **gần như chắc chắn là
  lỗi**, không còn là "có thể 2 lot hợp lệ" như ghi nhận trước. CHƯA XÓA
  gì. Andy vẫn cần xem chứng từ giấy gốc 717671 để quyết định cuối.
- **GAP-005**: Chưa có test tự động.
- ~~GAP-006~~: **ĐÃ XONG** — toàn bộ code webapp + root đã commit VÀ PUSH
  lúc đóng phiên 2026-09-13 (Andy xác nhận "commit push"). Xem mục 1 (git).
- **GAP-007**: PartControl 21 mã Part# thiếu Quantity/box, chưa ai xác
  nhận (xem `BRS_TRS.md` mục 6).
- **GAP-008**: Traveler **718865** đang lưu sai `po=193904` trong
  RawMaterial (đúng phải là `193902`) — Andy đã xác nhận đúng, chọn tự sửa
  tay (tab RawMaterial cột F, dòng 718865: 193904→193902) —
  **chưa xác nhận đã sửa xong trong Sheet**.
- **GAP-009 (mới, chưa xác nhận là chủ ý hay thiếu sót)**: khâu 2 và khâu
  3 không có ràng buộc code nào bắt buộc phải xác nhận khâu 2 trước khi
  lưu khâu 3 cho cùng 1 traveler — 2 trang độc lập.
- **GAP-010 (mới 2026-09-12, đã sửa code)**: traveler **717771** — FinishGood
  lưu sai Pot#=282 (đúng phải 18 theo RawMaterial), phát hiện khi đối chiếu
  chéo Sheet sống, cùng loại lỗi với nguyên nhân GAP-011. Andy chọn tự sửa
  tay (tab FinishGood cột F, dòng 717771: 282→18) — **chưa xác nhận đã sửa
  xong trong Sheet**.
- ~~GAP-011~~: **ĐÃ SỬA + VERIFY** — PDF Scanning Sheet bị nhúng sideways
  khiến AI đọc lộn dòng (nguyên nhân gốc của GAP-010 và ca 717770/717797).
  Đã sửa `splitPdfPages` tự xoay 270° khi phát hiện trang khổ dọc, verify
  lại bằng Gemini thật — đọc đúng 100%. Xem `BRS_TRS.md` mục 7.4.
- **GAP-012 (mới, chưa quyết định)**: MC# là 1 máy độc quyền hay 1 trạm
  dùng chung nhiều traveler/ca? Dữ liệu thật cho thấy 5,66% trường hợp có
  nhiều traveler chung MC#+ca+ngày — không rõ là bình thường hay lỗi có
  sẵn. Cảnh báo "trùng máy" trong code vẫn giữ nguyên nhưng độ tin cậy CHƯA
  XÁC NHẬN. Andy chọn "chưa rõ, để sau".
- **GAP-013 (mới 2026-09-13, chưa test)**: redesign UI khâu 3/4 (search
  trung tâm, thu gọn khâu 1-3, add-by-traveler, preview Packing Slip — xem
  mục 3.6.A) đã build/lint pass và server đã restart, nhưng **Andy CHƯA
  xác nhận đã thử lại trên trình duyệt thật** sau các sửa cuối cùng của
  phiên. Ưu tiên #1 phiên sau: mở lại `/` và thử từng tính năng mới.
- **GAP-014 (mới 2026-09-13, phát hiện lớn, chưa xử lý)**: 2 file Excel
  (`PACKING SLIPS.xlsm`, `DAILY LOG CHECK SHEET.xlsm`) mỗi file giữ 1 bảng
  WORK ORDER RIÊNG, đã lệch nhau THẬT ở 797 traveler (SHIPPED/PS khác
  nhau). `SUMMARY IN&OUT` sai 0/30 ngày mẫu so với WORK ORDER thật. Đây là
  lỗi ĐANG TỒN TẠI trong dữ liệu vận hành hiện tại của AVP (không phải lỗi
  của AVP_AI) — Andy cần biết để không tin tưởng mù quáng vào 2 file này
  khi ra quyết định. Xem `BRS_TRS.md` mục 6e, và báo cáo riêng
  `BAO_CAO_DANH_GIA_HE_THONG.md`.

### 3.7 Tiếp tục kiểm thử 2026-09-13 (Andy test GAP-013 — phần khâu 3, Import Excel)

- **Andy đã test phần trên, xác nhận OK** — hành vi sau đây là ĐÚNG THIẾT KẾ,
  không phải bug (đối chiếu code trực tiếp lúc kiểm tra):
  - Import Excel (`DAILY LOG CHECK SHEET.xlsm`) đọc/quét (scan) thành công 20
    dòng lên bảng nháp trình duyệt, nhưng khi bấm Lưu thì **0/20 dòng được
    ghi vào FinishGood** — vì cả 20 traveler đều CHƯA tồn tại trong
    RawMaterial (khâu 1). Đây là chặn nghiệp vụ cố ý
    (`finish-good/confirm/route.ts`): không cho ghi Finish Good nếu Traveler#
    chưa có dòng RawMaterial tương ứng.
  - "Scan" (AI đọc + hiện bảng nháp) và "Lưu" (ghi thật vào Google Sheet qua
    `/api/finish-good/confirm`) là 2 bước tách biệt có chủ đích, đúng nguyên
    tắc AI không ghi thẳng Sheet (`ISO_AI_CONTROL.md`) — "đã scan" không có
    nghĩa "đã lưu vào kho".
  - Ô search trung tâm (`GlobalSearchBar.tsx`) tra đúng dữ liệu THẬT đã lưu
    trong 3 tab (RawMaterial/FinishGood/PartControl), không tra bảng nháp
    chưa lưu — nên traveler chưa lưu thì search không thấy là đúng.
  - Kho KHÔNG gộp chung nguyên liệu/thành phẩm — 5 tab tách biệt trong 1
    Google Sheet (RawMaterial, Warehouse, FinishGood, PartControl,
    PackingList), liên kết nhau qua Traveler# (không phải 1 bảng chung có
    field phân loại).
- **Việc đang mở phát sinh từ test này** (chưa quyết định — xem mục dưới):
  tính năng Import Excel (khâu 3) hiện chỉ dùng được cho traveler MỚI đã có
  sẵn RawMaterial qua khâu 1 của app; với dữ liệu lịch sử cũ (traveler chưa
  từng qua khâu 1 app này) thì luôn thất bại 100% — Andy chưa chọn hướng xử
  lý (giữ nguyên / thêm backfill RawMaterial tự động / để sau).

### Việc đang mở (chưa quyết định)
- **"Box QTY 31.000" trên Split Form** nghĩa là gì — nghiêng về giả
  thuyết "= Pieces gốc của Pot trước khi chia lô", nhưng CHƯA xác nhận
  100% (xem mục 3.3). CHƯA có câu trả lời chốt từ Andy — Andy nói sẽ hỏi
  lại phía chủ (Infasco) rồi quyết định sau.
- **`finishedPartNo` (sticker) vs `partNo` (đầu form 02)** — cái nào đúng
  để dùng link khâu 3/4 khi 2 số khác nhau — Andy nói "cứ lưu lại" (đã
  lưu cả 2 field), nhưng CHƯA quyết định cái nào ưu tiên dùng thật.
- **GAP-009** — có cần chặn cứng thứ tự khâu 2→3 không, hay giữ độc lập
  như hiện tại.
- **GAP-012** — nghĩa thật của MC#, xem trên.
- **Bảng quy ước/reference table** cho field dạng enum (Shift/Machine/
  Operator code) để đối chiếu OCR — đề xuất nhưng chưa làm, cần Andy cho
  nguồn dữ liệu gốc trước.

---

## 5. BẮT ĐẦU PHIÊN SAU TỪ ĐÂY

### Việc ưu tiên tiếp theo
1. **Thử lại UI mới trên trình duyệt thật** (GAP-013) — search trung tâm,
   thu gọn khâu 1-3, add-by-traveler khâu 4, preview Packing Slip — CHƯA
   được xác nhận hoạt động đúng sau các sửa cuối cùng của phiên.
2. **Xác nhận Andy đã tự sửa tay xong chưa** — traveler 718865 (`po`
   193904→193902, GAP-008) và traveler 717771 (`pot` 282→18, GAP-010) —
   nếu chưa, đưa lại đúng vị trí ô cần sửa.
3. Hỏi Andy chốt lại GAP-004 (717671) — có chứng từ giấy gốc để đối
   chiếu không, hay giữ nguyên chờ (bằng chứng 100% mới đã củng cố hướng
   "là lỗi", xem mục 4).
4. Đọc lại 2 báo cáo mới (`BAO_CAO_DANH_GIA_HE_THONG.md`,
   `THIET_KE_HE_THONG_MOI.md` + 2 artifact link ở mục 3.6.C) — quyết định
   độ ưu tiên nếu muốn triển khai hướng thiết kế mới (nhỏ/vừa/lớn đã đề
   xuất trong `ARCHITECTURE.md` mục 8.4/8.5).
5. Chốt nghĩa "Box QTY 31.000" và ưu tiên `finishedPartNo` vs `partNo`
   (Andy sẽ hỏi phía Infasco).
6. Chốt nghĩa thật của MC# (GAP-012) — 1 máy độc quyền hay trạm dùng
   chung — để biết cảnh báo "trùng máy" có đáng tin không.
7. Nếu dùng thật: cần server chạy 24/7 — bàn phương án deploy (Vercel,
   VPS...). Cân nhắc luôn: webapp chưa có remote GitHub riêng (chỉ commit
   local) — hỏi Andy có muốn tạo repo riêng hay dùng chung `AVP-AI`.
8. Kiểm tra lại link tunnel — link đổi mỗi lần restart, cần xác nhận
   link mới nếu máy đã tắt/mở lại.

### Cảnh báo cho phiên sau
- ⚠ Server dev có thể đã tắt nếu máy tắt — restart bằng `npm run build &&
  npm run start` trong `D:\AVP_AI\webapp`, port 3000. Tunnel cloudflared
  cũng cần chạy lại, link SẼ ĐỔI.
- ⚠ `.env.local` trong `webapp/` chứa API key thật — không commit (đã
  trong `.gitignore`).
- ⚠ Traveler 717671 (GAP-004, nghi trùng), 718865 (GAP-008, po sai) và
  717771 (GAP-010, pot sai) vẫn còn trong Sheet — sẽ tiếp tục làm méo báo
  cáo đối chiếu cho tới khi Andy sửa/quyết định xong.
- ⚠ Nút upload khâu 3 giờ nhận cả PDF (trước chỉ ảnh) — nếu Andy dùng thử
  bằng PDF thật, để ý cảnh báo mới (Pot# lệch RawMaterial, trùng LOT#,
  trùng máy) có báo đúng không, đặc biệt file nào KHÔNG bị sideways (fix
  xoay chỉ tác động trang khổ dọc, không đụng trang đã đúng chiều).
- ⚠ **2 file Excel vận hành hiện tại của AVP có lỗi thật đang tồn tại**
  (GAP-014: 797 traveler lệch giữa 2 bản WORK ORDER, SUMMARY IN&OUT sai
  0/30 ngày) — đây KHÔNG phải lỗi của AVP_AI, nhưng Andy cần biết trước khi
  dùng 2 file đó để đối chiếu/ra quyết định.
- ⚠ Nhiều traveler TEST-* (TEST900001-5, TEST-PARTIAL...) và PO test
  (TEST-717365, TEST-LOTKEY, TEST-HOLD, TEST-CHECKS...) đang nằm trong
  Sheet thật, đánh dấu rõ "TEST" trong PO/note — không xóa (theo đúng
  nguyên tắc "kho tích lũy" Andy đã chốt, tham khảo `D:\Ops_Ai`), nhưng
  cần biết để không nhầm là dữ liệu thật khi đọc báo cáo.

---

## 6. QUYẾT ĐỊNH ĐÃ CHỐT (KHÔNG BÀN LẠI)

- DESCRIPTION = Oprtr + Machine + MC# + S/P + Special Notes. KHÔNG gồm
  Type/Nut-Bolt (xem `BRS_TRS.md` mục 4).
- QUANTITY = BOX × Quantity/box (PartControl), không copy thẳng TTL QNT.
- ~~Part# hậu tố mạ/vật liệu không ảnh hưởng Quantity/box~~ — **ĐÍNH CHÍNH
  2026-09-13 (SES-20260913-002)**: Owner xác nhận hậu tố (`-L`/`-HT`/`-A`...)
  là **sản phẩm khác nhau thật**, không dùng chung Quantity/box với mã gốc.
  Đã sửa `partControl.ts` — xem mục 2.6 phiên mới nhất ở đầu file.
- Ngưỡng cảnh báo mặc định cho đối chiếu số lượng: 10% (riêng cảnh báo
  lệch nhận kho ở khâu 2 dùng 5%).
- Khóa "1 lần sản xuất/đóng gói" = (Traveler#, LOT NO.), không phải chỉ
  Traveler# (xem mục 3.2, đã triển khai).
- Packing Slip = 1 chuyến xe, không gắn 1 PO cụ thể (xem mục 3.3).
- Không xóa dữ liệu tích lũy trong Sheet — hệ thống là kho lưu liên tục
  (tham khảo mô hình `D:\Ops_Ai`), không cần "dọn sạch" trước khi test.
- **Traveler# → LOT NO. / Part# / Pot# đều 1:1 tuyệt đối** (100%, xác nhận
  bằng 1.811 dòng lịch sử thật 2026-09-12/13) — bất kỳ dòng FinishGood nào
  lệch 1 trong 3 field này so với dòng khác CÙNG traveler chưa Shipped đều
  gần như chắc chắn là lỗi, không phải biến thể hợp lệ (xem `BRS_TRS.md`
  mục 3).
- **SKID# không phải khóa định danh** — 1 skid chứa chung nhiều traveler
  (77,4% thật) — không dùng SKID# để đối chiếu/chống trùng.
- **Pot# là "khóa" tốt để tự phát hiện/gợi ý Traveler# bị đọc sai — NHƯNG
  chỉ trong phạm vi traveler CÒN MỞ (chưa Shipped) tại cùng thời điểm**
  (⚠ sửa lại 2026-09-13: đo trên 1.853 dòng WORK ORDER thật, **469/1.141
  Pot# (41,1%) bị TÁI SỬ DỤNG** cho ≥2 traveler khác nhau theo thời gian —
  thùng chứa vật lý quay vòng nhận→rỗng→trả→nhận lại. KHÔNG phải khóa vĩnh
  viễn qua toàn lịch sử — xem `BRS_TRS.md` mục 3b).
- **LOT NO. mới là đơn vị "Work Order" thật, không phải Traveler#** (⚠ sửa
  lại 2026-09-13): Traveler# = 1 pot nguyên liệu cụ thể; mỗi LẦN đóng
  gói/sản xuất riêng (1 LOT NO.) mới là 1 Work Order — 1 traveler có thể
  sinh nhiều LOT/nhiều WO hợp lệ. Quy ước LOT NO. thật: chỉ 41% là mã lot
  thật (`6-DDD-SS-L`), **59% còn lại là chữ trạng thái** (`TRAVELERRECEIVED`
  53,8%, `SORT&RETURN` 1,7%, `SPLIT FROM TR#...` ~3,6%) — đã sửa code
  `finish-good/confirm/route.ts` (`isLotPlaceholder`) loại các giá trị này
  khỏi cảnh báo "LOT đổi khác thường" (xem `BRS_TRS.md` mục 3b).
- **PDF Scanning Sheet bị nhúng sideways phải tự xoay 270° trước khi gửi
  AI** — đã code hóa thành quy tắc cố định trong `splitPdfPages`, không
  phải xử lý tay từng file (xem `BRS_TRS.md` mục 7.4).

---

*Session Report — cập nhật lần cuối 2026-09-13 (đóng phiên, đã commit+push,
phiên dài kéo qua 2 ngày — xem mục 3 cho danh sách đầy đủ, mục 3.6 là phần
mới nhất: redesign UI khâu 3/4 + phát hiện 797 traveler lệch giữa 2 file
Excel + báo cáo đánh giá/thiết kế mới đã publish).*
*Đọc lại đầu phiên sau; cập nhật lại file này ở cuối mỗi phiên làm việc
tiếp theo.*
