# SESSION REPORT
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
| Session | SES-20260912-001 (phiên dài, nhiều hạng mục, kéo qua 2 ngày) |
| Ngày | 2026-09-11 → 2026-09-13 |
| Chủ dự án | Andy Phan (Viet), Maple Leaf Group |
| Thư mục | `D:\AVP_AI` |
| Web app | `D:\AVP_AI\webapp` (Next.js + TypeScript) |
| Database | Google Sheets, ID `1TWYKdRHv3MioEBf2DHAvUGlCvZxsr-KXQKLCR_3BkJ4` |
| Git | Repo `webapp` — đã commit tới `be65200` (mục 3.4 cũ). **Phiên 2026-09-12/13 có thêm sửa code (mục 3.5) nhưng CHƯA COMMIT** — 3 file sửa + 1 route mới đang nằm ở working tree (`git status` trong `webapp/`). **CHƯA push GitHub**. Repo gốc `D:\AVP_AI` — đã push lên `https://github.com/vietsharescom/AVP-AI.git` (không tính code webapp mới nhất). |
| Link demo | https://bumper-thomson-conferencing-negotiation.trycloudflare.com (đã kiểm tra còn sống lúc đóng phiên 2026-09-13 — tunnel tạm, sẽ chết khi tắt máy/tunnel restart, link đổi mỗi lần) |
| Trạng thái lúc đóng phiên | Server `localhost:3000` chạy `npm run start` (production build, đã rebuild+restart nhiều lần trong phiên để áp dụng code mới, lần cuối đã verify 200 OK). Cần restart (`npm run build && npm run start`) nếu máy tắt/mở lại. |

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
  OCR — cần Andy cung cấp nguồn dữ liệu gốc (danh sách máy/operator thật)
  trước khi làm, không tự suy đoán.

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
- ~~GAP-006~~: **ĐÃ XONG** — code webapp đã commit tới `be65200`. Phiên
  2026-09-12/13 có sửa thêm code (mục 3.5) nhưng **CHƯA COMMIT** (khác với
  GAP-006 cũ đã commit) — cần Andy xác nhận trước khi commit/push.
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
1. **Xác nhận Andy đã tự sửa tay xong chưa** — traveler 718865 (`po`
   193904→193902, GAP-008) và traveler 717771 (`pot` 282→18, GAP-010) —
   nếu chưa, đưa lại đúng vị trí ô cần sửa.
2. Hỏi Andy chốt lại GAP-004 (717671) — có chứng từ giấy gốc để đối
   chiếu không, hay giữ nguyên chờ (bằng chứng 100% mới đã củng cố hướng
   "là lỗi", xem mục 4).
3. **Andy xác nhận có COMMIT code webapp phiên 2026-09-12/13 không**
   (3 file sửa + 1 route mới, chưa commit — xem mục 1) — chỉ commit khi
   Andy đồng ý, chưa push GitHub.
4. Chốt nghĩa "Box QTY 31.000" và ưu tiên `finishedPartNo` vs `partNo`
   (Andy sẽ hỏi phía Infasco).
5. Chốt nghĩa thật của MC# (GAP-012) — 1 máy độc quyền hay trạm dùng
   chung — để biết cảnh báo "trùng máy" có đáng tin không.
6. Nếu dùng thật: cần server chạy 24/7 — bàn phương án deploy (Vercel,
   VPS...).
7. Kiểm tra lại link tunnel — link đổi mỗi lần restart, cần xác nhận
   link mới nếu máy đã tắt/mở lại.

### Cảnh báo cho phiên sau
- ⚠ KHÔNG tự commit/push git khi Andy chưa xác nhận — phiên 2026-09-12/13
  có sửa code (mục 3.5) nhưng cố tình CHƯA commit, đang chờ Andy duyệt.
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
- Part# hậu tố mạ/vật liệu không ảnh hưởng Quantity/box.
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
- **Pot# là "khóa" tốt nhất để tự phát hiện/gợi ý Traveler# bị đọc sai** —
  tốt hơn Part# (Part# lặp lại ở nhiều traveler, không định danh riêng
  được).
- **PDF Scanning Sheet bị nhúng sideways phải tự xoay 270° trước khi gửi
  AI** — đã code hóa thành quy tắc cố định trong `splitPdfPages`, không
  phải xử lý tay từng file (xem `BRS_TRS.md` mục 7.4).

---

*Session Report — cập nhật lần cuối 2026-09-13 (phiên dài, kéo qua 2
ngày — xem mục 3 cho danh sách đầy đủ, mục 3.5 là phần mới nhất).*
*Đọc lại đầu phiên sau; cập nhật lại file này ở cuối mỗi phiên làm việc
tiếp theo.*
