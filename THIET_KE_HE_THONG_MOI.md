# THUYẾT MINH THIẾT KẾ HỆ THỐNG MỚI — AVP SOLUTIONS
### Theo chuẩn ERP hiện đại (đối chiếu Odoo 17 thật + mô hình quản trị Ops_Ai)

*Ngày: 2026-09-13 | Tài liệu kế tiếp của `BAO_CAO_DANH_GIA_HE_THONG.md` (đánh giá
hiện trạng) — tài liệu này là THIẾT KẾ ĐỀ XUẤT, dựa trên nghiên cứu trực tiếp:
Odoo 17 thật (XML-RPC, DB `xekem`), dự án KhoAI (`D:\xekem`, kiến trúc AI kho
hiện đại: voice/camera/invoice-scan), và dự án Ops_Ai (`D:\Ops_Ai`, mô hình
quản trị AI 4-lớp + 18 mục thiết kế ERP chi tiết theo ISO 9001/42001).*

---

## PHẦN 1 — INPUT/OUTPUT TỪNG CÔNG ĐOẠN THEO THIẾT KẾ MỚI

Mỗi công đoạn dưới đây giữ đúng nghiệp vụ AVP đang làm — chỉ đổi CÁCH thu thập
(voice/camera/OCR thay vì gõ tay) và CÁCH lưu trữ (1 sổ cái thay vì nhiều sheet
rời). Không công đoạn nào bị bỏ, không công đoạn nào bị tự động hóa hoàn toàn
bỏ qua con người (xem cột "Điểm CCP").

| # | Công đoạn | Đầu vào (Input) | Công nghệ thu thập | Xử lý (Core Rule Engine) | Đầu ra lưu trữ (Ledger) | Phân tích dữ liệu | Báo cáo trực quan | Điểm CCP (người duyệt) |
|---|---|---|---|---|---|---|---|---|
| 1 | **Nhận PO** | Email PDF/ảnh từ Infasco | 📄 Đọc file tự động (PDF text-layer hoặc Vision OCR nếu là ảnh scan) | Đối chiếu PO trùng, tách Traveler/Part#/Pot#/Weight/Pieces, validate PartControl | 1 dòng Ledger, `stage=FORECAST` | — | Dashboard "PO mới hôm nay" | **CCP-1**: người xem lại trước khi ghi |
| 2 | **Nhận kho vật lý** | Cân/đếm thật tại bến nhận | 📷 Camera (chụp cân điện tử/tem POT) hoặc nhập tay xác nhận | So sánh forecast vs thực nhận; nếu lệch >5% → `RECEIVING_VARIANCE`; gán `qc_status=QUALITY_HOLD` chờ kiểm (theo ISO 9001 8.7, mặc định KHÔNG bắt buộc nếu Part# không cần kiểm — xem Phần 3) | `stage=RECEIVED`, `qc_status` | % lệch forecast/thực nhận theo Part# | Cảnh báo lệch trên dashboard | **CCP-2**: xác nhận số thực nhận |
| 3 | **QC lựa hàng / Split Form** | Giấy Split Form + lời QC ghi nhận | 🎙️ Voice (tay bẩn, không chạm màn hình) HOẶC 📷 Camera+Vision đọc form giấy | Trích FINAL INSPECTION → `PASS`/`QUALITY_HOLD`/`CONCESSION` (ISO 9001 8.7); nếu CONCESSION bắt buộc chữ ký người có thẩm quyền độc lập | `stage=IN_PROCESS`, `qc_status`, `dmf_id` nếu có | Tỷ lệ HOLD theo Part#/máy/ca | Heat-map lỗi theo máy × ca | **CCP-3**: QC ký xác nhận PASS/HOLD/CONCESSION |
| 4 | **Đóng gói / Scan thành phẩm** | Tem/nhãn thùng đã đóng | 📷 Camera+Vision (Scanning Sheet) hoặc nhập trực tiếp tại bàn đóng gói (tablet) | Tính **Yield/hao hụt** = Pieces nhận − Qty đóng gói tốt; validate (Traveler,Pot#,Lot#) 1:1 (đã đo 100% thật); reject có cấu trúc (mã lỗi + số lượng tách riêng, không còn chuỗi phẩy tự do) | `stage=PACKED`, `yield_pct`, `reject[]` có cấu trúc | Yield rate theo lô/Part#/máy theo thời gian | Biểu đồ Yield theo ngày, theo Part# | **CCP-1**: xem lại kết quả AI đọc trước lưu |
| 5 | **Tạo Packing List / xuất hàng** | Tìm kiếm theo Traveler# (đã có ở AVP_AI) | ⌨️ Search-to-add + 📷 quét mã vạch (tùy chọn nếu có in mã) | Build phiếu; **chặn cứng nếu còn HOLD/CONCESSION chưa xử lý**; tính POT rỗng cần trả dựa trên Ledger | `stage=SHIPPED`, 1 bản ghi chuyển động kho (giống `stock.move`) | Tỷ lệ giao đúng hẹn (on-time delivery) | Xem trước (preview) trước khi in — đã có ở AVP_AI | **CCP-3**: chặn cứng HOLD; **CCP-4**: số PS gõ tay (nguồn ngoài AVP) |
| 6 | **Theo dõi container (POT/Pallet)** | Quét mã POT lúc nhận và lúc trả | 📷 Camera/mã vạch | Cập nhật vòng đời: `nhận→rỗng→đã trả`; cảnh báo nếu POT rỗng tồn quá lâu chưa trả | Asset Ledger riêng (khóa theo Pot#) | Tồn kho POT rỗng theo thời gian | Cảnh báo "thiếu container để trả" | Không cần CCP — chỉ là đếm vật lý có ghi nhận |
| 7 | **Hỏi-đáp tồn kho tức thời** | Câu hỏi giọng nói/chat: "còn bao nhiêu Part X chưa xuất?" | 🎙️ Voice (Whisper) → LLM hiểu ý định → query Ledger | Không tính số liệu mới — chỉ TRUY VẤN Ledger đã có, LLM không được tự bịa số (nguyên tắc tuyệt đối, giống Ops_Ai) | — (chỉ đọc) | — | Trả lời trực tiếp bằng giọng nói hoặc màn hình | **CCP-1**: LLM chỉ diễn giải câu hỏi, KHÔNG tự tính số |
| 8 | **Dashboard quản lý tổng hợp** | Toàn bộ Ledger + Asset Ledger | 📊 BI/Visualization tự động | Tổng hợp KPI: Yield trung bình, % HOLD, tình trạng máy, ETA giao hàng (Available-to-Promise) | — (chỉ đọc, tổng hợp) | Xu hướng theo tuần/tháng, so sánh Part#/khách hàng | Biểu đồ, bảng màu cảnh báo (đỏ/vàng/xanh) | Không cần CCP — chỉ hiển thị, không ghi |

---

## PHẦN 2 — SO SÁNH QUY TRÌNH HIỆN TẠI vs THIẾT KẾ MỚI

| Tiêu chí | Hiện tại (Excel) — bằng chứng thật | Thiết kế mới (ERP hiện đại) |
|---|---|---|
| **Nguồn sự thật** | ≥3 nơi lưu song song (WORK ORDER ×2 file, ARCHIVE, ARCHIVE2, SUMMARY IN&OUT) — đã đo lệch nhau thật (797/1.847 traveler, 0/30 ngày KPI khớp) | **1 sổ cái duy nhất**, mọi báo cáo tính TỪ đó — không thể lệch vì chỉ có 1 nơi ghi |
| **Nhập liệu** | 100% gõ tay ở cả WORK ORDER (10 cột × 1.853 dòng) và CHECKING SUMMARY (19/20 cột × 1.811 dòng) | AI đọc ảnh/PDF (Vision) + giọng nói (Voice) khi tay bẩn — người chỉ xem lại, không gõ lại từ đầu |
| **Chất lượng/lỗi** | Cột DEFECTS có tên nhưng 0% dùng; Reject ghi chuỗi số không đơn vị; LOT NO. đôi khi bị ghi đè "SORT&RETURN" | Non-conformance có cấu trúc theo **ISO 9001 Cl.8.7**: `PASS → UNRESTRICTED`, `FAIL → BLOCKED`, `CONCESSION → REWORK` (có ký xác nhận độc lập) |
| **Container/tài sản** | Không track — Total Empty chỉ đếm tay lúc xuất, không liên kết Pot# nào | Vòng đời đầy đủ: nhận → rỗng → đã trả, cảnh báo tồn đọng |
| **Máy móc/thiết bị** | Không 1 cột nào ghi giờ chạy/dừng máy trong toàn bộ 2 file | `equipment_status`: AVAILABLE/DOWN/MAINTENANCE/DEGRADED + cờ `EQUIPMENT_DOWN`, `CAPACITY_OVERRUN_RISK` (mô hình Ops_Ai đã kiểm chứng qua sự cố thật) |
| **Báo cáo KPI** | Đếm tay, sai 100% khi đối chiếu (0/30 ngày khớp) | Tính tự động từ sổ cái — luôn khớp vì không có "đếm lại bằng tay" |
| **Ngày giao hàng** | Phải hỏi người, chờ trả lời | **Available-to-Promise**: tính tự động từ vị trí hàng đợi + tốc độ xử lý đo được |
| **Truy vết (traceability)** | Dò tay qua 4-5 sheet, không chắc đủ | 1 truy vấn genealogy: traveler → lot → thùng → chuyến hàng → khách hàng |
| **Quyết định quản lý** | Dựa trên số liệu có thể sai (đã đo: SUMMARY IN&OUT sai 100%) mà không biết | Dashboard đọc trực tiếp sổ cái — số hiển thị = số thật, không qua trung gian đếm tay |
| **Vai trò con người** | Gõ + mò số liệu phần lớn thời gian (~21-28 giờ/ngày ước tính) | Xem lại + xử lý ngoại lệ (CCP) — ít giờ hơn, giá trị cao hơn |

---

## PHẦN 3 — KIẾN TRÚC PHÂN LỚP (LAYERED ARCHITECTURE)

```
┌─────────────────────────────────────────────────────────────────────┐
│ LỚP 7 — OUT / GIAO DIỆN                                              │
│ Dashboard web · Hỏi-đáp giọng nói · In Packing Slip · Xuất báo cáo   │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │ đọc
┌───────────────────────────────▼───────────────────────────────────────┐
│ LỚP 6 — ANALYTICS / BI                                               │
│ Yield trend · Equipment status · On-time delivery · So sánh Part#    │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │ đọc
┌───────────────────────────────▼───────────────────────────────────────┐
│ LỚP 5 — CCP / HUMAN GATE (đặt tên chính thức, không rải rác)          │
│ CCP-1 Xem AI đọc · CCP-2 Xác nhận nhận kho · CCP-3 QC PASS/HOLD ·     │
│ CCP-4 Số PS gõ tay (nguồn ngoài AVP)                                  │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │ duyệt qua mới ghi
┌───────────────────────────────▼───────────────────────────────────────┐
│ LỚP 4 — GOVERNANCE / QUALITY (ISO 9001 Cl.8.7)                       │
│ Non-conformance disposition · Genealogy/truy vết · Recall            │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │
┌───────────────────────────────▼───────────────────────────────────────┐
│ LỚP 3 — LEDGER (1 SỔ CÁI DUY NHẤT — thay RawMaterial/FinishGood/      │
│ PackingList/ARCHIVE/ARCHIVE2/SUMMARY rời rạc hiện tại)                │
│ Traveler ⇄ Pot# ⇄ Lot# ⇄ stage(FORECAST→RECEIVED→IN_PROCESS→         │
│ PACKED→SHIPPED) + Asset Ledger (container)                           │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │ ghi (chỉ sau CCP)
┌───────────────────────────────▼───────────────────────────────────────┐
│ LỚP 2 — CORE / RULE ENGINE (tất định, KHÔNG LLM)                     │
│ Cardinality check (Traveler-Pot-Lot 1:1) · Yield calc ·               │
│ Available-to-Promise · Capacity/equipment feasibility ·               │
│ Safety-stock check                                                    │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │ nhận input đã trích
┌───────────────────────────────▼───────────────────────────────────────┐
│ LỚP 1 — CAPTURE (AI chỉ chạy ở đây, không chạy ở lớp nào khác)       │
│ 📄 Đọc file (PDF/email) · 📷 Camera+Vision OCR · 🎙️ Voice (Whisper)  │
│ · 🔖 Quét mã vạch/POT                                                │
└─────────────────────────────────────────────────────────────────────┘
```

**Nguyên tắc bất biến (giống Ops_Ai, đã kiểm chứng bằng thực tế vận hành)**:
- AI (Vision/Voice/LLM) **chỉ chạy ở Lớp 1** — không được tự tính số liệu nghiệp vụ ở bất kỳ lớp nào khác.
- Lớp 2 (CORE) tính TRƯỚC khi AI diễn giải — không để LLM tự "đoán" ra 1 con số nghiệp vụ.
- Ghi vào Lớp 3 (Ledger) chỉ xảy ra SAU KHI qua đúng CCP tương ứng ở Lớp 5 — không có đường ghi tắt.

### Pipeline dữ liệu (luồng chảy 1 lô hàng thật, từ trái sang phải)

```
[Email PO] ──Capture──▶ [Draft Forecast] ──CCP-1──▶ [Ledger: FORECAST]
                                                            │
[Cân/đếm thật] ──Capture──▶ [Draft Received] ──CCP-2──▶ [Ledger: RECEIVED]
                                                            │
[Voice/Split Form] ──Capture──▶ [QC Result] ──CCP-3──▶ [Ledger: IN_PROCESS + qc_status]
                                                            │
[Scanning Sheet] ──Capture──▶ [Yield tính tự động] ──CCP-1──▶ [Ledger: PACKED + yield%]
                                                            │
[Search Traveler] ──Core check HOLD──▶ [Preview Packing Slip] ──CCP-3/4──▶ [Ledger: SHIPPED]
                                                            │
                                                    [Analytics/BI đọc liên tục]
                                                            │
                                                [Dashboard/Voice trả lời tức thời]
```

---

## PHẦN 4 — GHI CHÚ TRIỂN KHAI (tham khảo trực tiếp Odoo/Ops_Ai)

- **Mô hình Ledger** tham khảo trực tiếp `stock.quant`/`stock.move`/`stock.lot`
  của Odoo (đã kiểm chứng bằng XML-RPC thật lên Odoo 17) — "kho nguyên liệu"
  và "kho thành phẩm" chỉ là 2 GIÁ TRỊ khác nhau của field `stage`, không phải
  2 bảng riêng.
- **Non-conformance PASS/HOLD/CONCESSION** tham khảo trực tiếp mục 9 và 18
  của `Ops_Ai/docs/cl08_operation/SOFTWARE_ARCHITECTURE.md` — đúng ISO 9001
  Clause 8.7, đã được Ops_Ai kiểm chứng qua tình huống thật (rượu Reid's,
  Scenario 6).
- **Equipment status/Capacity** tham khảo trực tiếp mục 13 của Ops_Ai — mô
  hình `AVAILABLE/DOWN/MAINTENANCE/DEGRADED` + cờ `EQUIPMENT_DOWN`,
  `CAPACITY_OVERRUN_RISK` đã được kiểm chứng qua sự cố thật (Scenario 3: máy
  hỏng, chạy quá tốc độ để bù, hỏng lần 2).
- **Voice/Camera capture** tham khảo trực tiếp kiến trúc KhoAI
  (`D:\xekem\KhoaiTechnicalArchitect.MD`) — Whisper cho giọng nói, Claude
  Vision cho ảnh/PDF, xử lý AI luôn ở backend (điện thoại chỉ chụp/nói).
- **Mức độ ưu tiên triển khai**: xem `ARCHITECTURE.md` mục 8.4/8.5 của
  AVP_AI — không đề xuất làm hết 1 lần, có lộ trình nhỏ→vừa→lớn.

---

*Tài liệu này là THIẾT KẾ ĐỀ XUẤT — chưa code gì. Mục đích: làm cơ sở thuyết
minh/thuyết phục trước khi quyết định đầu tư xây dựng. Xem file đi kèm
`BAO_CAO_DANH_GIA_HE_THONG.md` cho phần đánh giá hiện trạng (bằng chứng lỗi
thật) làm căn cứ so sánh.*

---

## PHẦN 5 — CẬP NHẬT 2026-09-15 (bổ sung từ 2 ngày vận hành thật sau khi viết tài liệu này)

*Nguồn: [`Report/DANH_GIA_KIEN_TRUC_HE_THONG_2026-09-15.md`](Report/DANH_GIA_KIEN_TRUC_HE_THONG_2026-09-15.md)
— quét trực tiếp dữ liệu sống (RawMaterial 1.939 dòng, FinishGood 194 dòng,
`/api/ledger` 184 dòng) 2 ngày sau khi Phần 1-4 ở trên được viết. 2 điểm
dưới đây là phần THẬT SỰ CÒN THIẾU so với thiết kế ở Phần 1-4 (không lặp
lại nội dung đã có — sổ cái bất biến/CCP/AI-chỉ-ở-lớp-1 đã thiết kế đúng
ở Phần 3, được xác nhận thêm bằng 2 sự cố thật bên dưới).*

### 5.1 Bổ sung quy trình bắt buộc: đo phân bố giá trị thật TRƯỚC khi khoá field

Chưa có ở Phần 1-4. Kinh nghiệm AVP_AI: gần như MỌI giả định ban đầu về ý
nghĩa 1 trường đều sai khi đo trên dữ liệu thật — `SHIPPED="0"` tưởng "đã
xuất" thực ra là "chưa xuất" (loại sạch 1.811/1.812 dòng); hậu tố Part#
tưởng "biến thể vô hại" thực ra "sản phẩm khác nhau thật"; LOT NO. tưởng
luôn là mã lot, đo ra 59% là chữ trạng thái; Pot# tưởng khoá 1:1 vĩnh
viễn, đo ra tái sử dụng 41,1%. **Quy tắc bổ sung cho dự án mới**: mỗi
field trong Lớp 3 (Ledger) — trước khi khoá kiểu dữ liệu/ràng buộc unique
— phải chạy 1 script đo phân bố giá trị thật trên toàn bộ dữ liệu lịch
sử đang có (giống cách đã đo Pot#/LOT NO./SHIPPED ở AVP_AI), không suy
diễn từ tên cột.

### 5.2 Bổ sung vào Lớp 1→2: bước "chuẩn hoá định dạng" + quét sức khoẻ dữ liệu định kỳ

Phần 3 đã tách Lớp 1 (Capture, AI chạy ở đây) và Lớp 2 (Core Rule Engine,
tất định) — đúng hướng, nhưng chưa nói rõ 1 bước RIÊNG ở giữa 2 lớp này:
**làm sạch/validate ĐỊNH DẠNG** (cắt tiền tố nhãn kiểu "Pot#"/"Lot" dính
vào giá trị, chặn độ dài/ký tự bất thường) TRƯỚC KHI Lớp 2 chạy các quy
tắc NGHIỆP VỤ. Bằng chứng thật (2026-09-15, AVP_AI): traveler 718643 bị
lưu `Pot#="Pot#313"` (nguyên cả nhãn), traveler 717544 bị lệch cả
Part#/Pot#/LOT# do 1 ô xuống 2 dòng trên giấy — cả 2 lọt qua vì hệ thống
hiện tại CHỈ có kiểm tra nghiệp vụ (đối chiếu RawMaterial, PartControl),
không có bước kiểm tra định dạng thuần tuý.

Thêm nữa: cần **1 job quét định kỳ chạy lại trên TOÀN BỘ dữ liệu đã lưu**
(không chỉ dòng mới) — 7 dòng FinishGood bị lỗi thật (Pot#=LOT# giống hệt
nhau, traveler 717397/718437/717772/717781/718646/718016/718014, cùng
tạo lúc `2026-09-14T21:06:05`, nguồn gốc **chưa xác định**) đã nằm im
trong Sheet cho tới khi tình cờ bị phát hiện hôm nay — không có cơ chế
nào tự báo nếu không có ai chủ động đi tìm.

### 5.3 Bằng chứng thật mới, củng cố thiết kế đã có (không cần đổi gì)

- **Sổ cái bất biến (§8.1) là cần thiết, không phải lý thuyết suông**:
  traveler 717544 hôm nay xuất hiện cả ở PS đã xuất (30101, ngày 9/9) và
  RawMaterial "mới về" (cũng ngày 9/9) — nhìn như trùng/lỗi, hoá ra là
  nhận rồi xuất ngay trong ngày. Với mô hình `stage` hiện tại (Phần 1),
  tình huống này dễ gây hiểu lầm; với sổ cái bút toán độc lập (mỗi lần
  nhận/xuất là 1 dòng có mốc thời gian riêng) thì tự nhiên rõ ràng, không
  cần suy luận thêm.
- **"Chỉ 1 đường ghi hợp lệ" (§8.3) là đúng, đã bị vi phạm 1 lần thật**:
  script nhập lịch sử PS 30098/30101 (chạy ở AVP_AI hiện tại) ghi thẳng
  vào Sheet, đi vòng qua hàm duyệt PS duy nhất — sinh ra 22 traveler bị
  sót đánh dấu đã xuất (đã sửa xong 2026-09-15). Dự án mới cần enforce
  điều này bằng CODE (vd migration/import script cũng phải gọi đúng hàm
  Lớp 2, không được ghi thẳng Lớp 3), không chỉ ghi trong tài liệu.

---

## PHẦN 6 — SỬA LẠI BẢN ĐỒ QUY TRÌNH THEO ĐÚNG 7 KHÂU THẬT (xác nhận trực tiếp với Owner, 2026-09-15)

*Nguồn: Owner mô tả trực tiếp trong phiên làm việc + đối chiếu chứng từ
thật trong `Data/` (đã xem tận mắt từng loại form) + Owner CHỈNH LẠI 2 lần
những suy luận sai của AI (xem cảnh báo ở cuối mục 6.3). Phần này THAY THẾ
cách chia 8 bước trừu tượng ở Phần 1 bằng đúng 7 khâu vật lý thật tại
xưởng, khớp cấu trúc thư mục `Data/` Owner đang tự tay sắp xếp lại.*

### 6.1 Bảng 7 khâu thật — chứng từ nào, AVP_AI đọc gì, còn thiếu gì

| # | Khâu | Chứng từ/dữ liệu thật (đường dẫn hiện tại) | AVP_AI hiện đọc gì | Còn thiếu |
|---|---|---|---|---|
| 1 | **PO / Email** — Infasco gửi PO qua email, có sẵn Traveler#/Part#/Pot#/Weight/Pieces | `Data/1.PO` | Có đọc (khâu "PO tham khảo") | — |
| 2 | **Good Receives** — nguyên liệu vít về trong tote/bin (Pot#), traveler còn "trắng" chưa có sticker QA | `Data/2. Good receives` | Có đọc (khâu "Kho nguyên liệu"), nhưng là lối tắt (không có checkpoint xác nhận vật lý riêng — GAP-015 cũ) | Bước Goods Receipt thật (xem GAP-015, chưa quyết định) |
| 3 | **Machine** — từng máy (BF1xx/MC0xx/TBx) ghi sản lượng theo ca | `Data/3.Machine` | Chỉ đọc Split Form (04_Work_Order_Quality_Check) | **Work Station log (Bin 1-4), Operator timesheet, Productivity/Operation Hour — CHƯA đọc gì (xem 6.2)** |
| 4 | **Traveler** — QC kiểm, dán sticker (Sorting Results, Operator initials, Lock QA) lên traveler gốc | `Data/4 Traveler` | Có đọc (Split Form/FinishGood) | — |
| 5 | **Labeling** — máy tính Excel tại trạm label (QC tự nhập sau khi lựa xong), tính tổng sản lượng + in nhãn từng thùng | `Data/5.Labeling` | Chưa đọc gì | **File Excel máy label (Owner sẽ gửi) — có Serial#/SSCC cấp thùng, xem 6.3** |
| 6 | **Scanning** — tổng hợp Scanning Sheet + Wrapping Summary thành bảng skid/pallet | `Data/6.Scanning` | Có đọc (Scanning Sheet → FinishGood) | Wrapping Summary (skid#, location) — chưa đọc riêng |
| 7 | **Parking/Packing Slip** — Customer Service lập PS từ "Daily Log Summary", đối chiếu Empty Bin Report, tài xế ký nhận | `Data/7.Parking slip` | Có đọc/tạo PS (khâu 4 webapp) | Chữ ký tài xế (proof of delivery) — chưa lưu; vòng Empty Bin — chưa model (xem 6.3) |

**Lưu ý cấu trúc thư mục `Data/` đang được Owner tự sắp xếp lại liên tục
trong ngày 2026-09-15** — tên/thứ tự thư mục ở bảng trên là trạng thái tại
thời điểm viết (14:22), có thể đổi tiếp. Đường dẫn chỉ mang tính tham
khảo, không phải hợp đồng cố định.

### 6.2 Nguồn dữ liệu MỚI phát hiện hôm nay, chưa từng đưa vào thiết kế trước đây

- **Work Station log** (`Data/3.Machine/Work station SEP 15-*.pdf`) — form giấy điền tay theo Bin 1-4 của từng máy, có "Not running waiting: bin f...to...", "M/C change over: f...to...", "M/C repair: f...to..." theo phút. Đây là dữ liệu downtime THẬT theo nguyên nhân — có thể trả lời câu hỏi "hao hụt do dùng chung máy" (62% nhóm Ngày+Máy có ≥2 traveler, đã đo ở BAO_CAO_TONG_HOP_AVP.md) vốn trước đây chỉ đo được HIỆN TƯỢNG, chưa biết NGUYÊN NHÂN.
- **Operator timesheet** (`Data/3.Machine/Operator SEP 15-1.pdf`) — time in/out + gán máy mỗi nhân viên theo ca. Nguồn CHÍNH THỨC cho "mã nhân viên" (GAP cũ chưa có danh sách chính thức).
- **Productivity / Operation Hour** (`Data/3.Machine/Productivity SEP 15-2.pdf`, `Operation Hour SEP 15-3.pdf`) — 2 báo cáo Excel pivot có sẵn (không phải AVP_AI làm ra): năng suất theo ngày/workstation, và giờ+giá theo Part#/Line. Xác nhận công ty đã tự đo năng suất/chi phí lao động bằng tay — không mù hoàn toàn như báo cáo trước giả định.

### 6.3 Đính chính 2 suy luận SAI của AI trong phiên này (Owner tự sửa, ghi lại để không lặp lại)

- **SAI**: AI đọc lưu đồ công ty năm 2020 (`Data/AVP Operations Flow Chart...pdf` trang 17 "How to Print Infasco Label") rồi suy diễn hệ thống in nhãn/Serial#-SSCC là do **Infasco** trang bị, tên hệ thống "IVACO-HO-770". **ĐÚNG theo Owner**: lưu đồ đó đã cũ (2020), không phản ánh thực tế. Máy label thật chỉ là **1 máy tính Excel bình thường tại xưởng, do AVP tự vận hành**, QC tự tay nhập số liệu sau khi lựa xong để tính tổng sản lượng và in từng label — không liên quan Infasco.
- Bài học: **không suy diễn hệ thống/quy trình từ tài liệu cũ (2020) khi chưa xác nhận với Owner còn đúng thực tế hay không** — đặc biệt các chi tiết kỹ thuật cụ thể (tên hệ thống, username/password, tên máy chủ) rất dễ đã lỗi thời.
- Việc cần làm thật (đã Owner xác nhận): khi có file Excel từ máy label (Owner sẽ gửi vào `Data/5.Labeling`), đọc trực tiếp file đó (Excel sạch, không cần OCR) để lấy Serial#/SSCC cấp thùng bổ sung vào FinishGood — không cần tìm hiểu thêm phần mềm/hệ thống nào khác.

---

## PHẦN 7 — KIẾN TRÚC PHÂN LỚP CẬP NHẬT: ĐIỂM LABEL LÀ ĐIỂM NHẬP KẾT QUẢ SẢN XUẤT (2026-09-15)

*Owner chỉ ra: máy Label ở khâu 5 không đơn thuần "in tem" — đây là điểm
QC gõ tay kết quả sản xuất cuối dây chuyền (Quantity, Lot#, Skid#,
Operator, Serial#) NGAY SAU khi lựa xong, TRƯỚC khi tem được in ra. Tức
là dữ liệu tại điểm này chính là "sự thật cuối cùng" của 1 mẻ sản xuất —
không phải dữ liệu phụ. Phần này vẽ lại Lớp 1 (Capture) của kiến trúc ở
Phần 3 để phản ánh đúng vai trò này.*

### 7.1 Vì sao đây là điểm có ưu thế lớn nhất trong 7 khâu

1. **Dữ liệu sạch, gõ tay trực tiếp bởi QC — không qua OCR.** Toàn bộ lỗi
   đọc nhầm chữ viết tay đã gặp thật ở AVP_AI (`Pot#="Pot#313"` dính nhãn,
   traveler 717544 lệch cả Part#/Pot#/LOT# do đọc nhầm dòng) là lỗi CỦA
   BƯỚC OCR giấy — nguồn Excel này không đi qua OCR nên không mắc lỗi
   cùng loại.
2. **Là nguồn DUY NHẤT trong 7 khâu có Serial#/SSCC cấp thùng** — không
   khâu nào khác (kể cả Scanning Sheet) có thông tin này.
3. **Là điểm hội tụ tự nhiên của 1 sự kiện sản xuất** — cùng 1
   Traveler#/Pot#/Lot# vừa được QC ghi PASS/HOLD trên giấy Traveler (khâu
   4), vừa được gõ Quantity/Skid/Serial# vào Excel Label (khâu 5) — 2
   NGUỒN ĐỘC LẬP cho cùng 1 sự thật, dùng để đối chiếu chéo thay vì tin mù
   1 nguồn (đúng nguyên tắc "đo dữ liệu thật, không suy diễn" đã áp dụng ở
   Phần 5.1).
4. **Giảm phụ thuộc Scanning Sheet** — hiện Scanning Sheet là "100% viết
   tay rồi gõ tay lại" (xem `BAO_CAO_TRINH_OWNER_QUY_TRINH_2026-09-15.md`
   mục 5, chưa có AI/máy đọc). Excel Label có thể thay thế phần lớn vai
   trò lấy Quantity/Lot/Skid chính xác của Scanning Sheet, vì đã là dữ
   liệu điện tử sẵn có, không cần OCR.

### 7.2 Lớp 1 (Capture) vẽ lại — 2 nhánh song song hội tụ, không còn 1 luồng thẳng

```
┌───────────────────────── LỚP 1 — CAPTURE (2 NGUỒN ĐỘC LẬP CHO CÙNG 1 SỰ KIỆN SẢN XUẤT) ─────────────────────────┐
│                                                                                                                    │
│   NHÁNH A — Giấy Traveler (đi theo hàng vật lý)        NHÁNH B — Excel máy Label (trạm cuối dây chuyền)          │
│  ┌──────────────────────────────────────┐             ┌───────────────────────────────────────────────┐         │
│  │ Split Form / Traveler sticker (khâu 4)│             │ Excel máy Label (khâu 5, QC tự nhập)           │         │
│  │  • QC PASS / HOLD / CONCESSION        │             │  • Traveler# / Part# / Pot# / Lot#             │         │
│  │  • Operator initials, ngày ký         │             │  • Skid# / Station# / Operator / Boxes         │         │
│  │  • Sorting Results, Defect count      │             │  • Quantity, Serial#/SSCC theo thùng           │         │
│  │  📷 Vision OCR — có sai số (viết tay)  │             │  📄 Đọc Excel trực tiếp — sạch, không OCR       │         │
│  └───────────────────┬────────────────────┘             └────────────────────┬──────────────────────────┘         │
│                       │                                                       │                                    │
│                       └───────────────────────┬───────────────────────────────┘                                   │
│                                                 │  KHOÁ ĐỐI CHIẾU: (Traveler#, Pot#, Lot#)                          │
└─────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┘
                                                    ▼
                     Trùng khớp cả 2 nguồn?  ──NO──▶  cảnh báo CCP-1 (không tự chọn nguồn nào, người xem lại)
                                                    │ YES
                                                    ▼
                          LỚP 2 — CORE: gộp 2 nguồn thành 1 bản ghi PACKED theo quy tắc ưu tiên field (xem 7.3)
                                                    ▼
                          LỚP 3 — LEDGER: stage=PACKED, Quantity + Serial#/SSCC + qc_status + operator
```

### 7.3 Quy tắc ưu tiên khi 2 nguồn cho cùng 1 field (Lớp 2, tất định — không LLM tự chọn)

| Field | Nguồn ưu tiên | Vì sao |
|---|---|---|
| Quantity, Skid#, Lot#, Serial#/SSCC | **Excel Label (Nhánh B)** | Gõ trực tiếp bởi QC lúc đó, không qua OCR — đây cũng là số liệu dùng để in tem thật, sai thì tem thật đã sai theo |
| QC PASS/HOLD/CONCESSION, chữ ký/Operator initials xác nhận chất lượng | **Traveler giấy (Nhánh A)** | Đây là hồ sơ chất lượng chính thức theo ISO 9001 8.7, có bút ký — không thể thay bằng số gõ Excel |
| Traveler# / Part# / Pot# (khoá đối chiếu) | **Phải khớp cả 2 nguồn** | Nếu lệch → đây chính là dấu hiệu lỗi nhập liệu ở 1 trong 2 phía (giống lỗi Pot#="Pot#313" đã gặp) — KHÔNG tự chọn nguồn nào, đẩy lên CCP-1 |

**Điểm mấu chốt**: Lớp 2 không "tin" 1 nguồn duy nhất — nó dùng 2 nguồn để
tự phát hiện lỗi nhập liệu ở CẢ HAI phía, việc mà hệ thống hiện tại (chỉ
đọc 1 nguồn giấy qua OCR) không làm được.

---

## PHẦN 8 — ĐÍNH CHÍNH LỚN: GAP-015 (Goods Receipt) KHÔNG PHẢI THIẾU SÓT — LÀ ĐẶC THÙ MÔ HÌNH KINH DOANH (2026-09-15)

*Owner giải thích trực tiếp lý do không cân/đếm nguyên liệu đầu vào — đây
là lý do NGHIỆP VỤ, không phải hệ thống thiếu bước. Ghi lại để không đề
xuất lại "thêm bước Goods Receipt" trong các phiên sau.*

### 8.1 Vì sao khâu 2 (Good Receives) không cân/đếm

AVP chỉ gia công **công đoạn CUỐI CÙNG** trong cả chuỗi sản xuất vít của
Infasco. Trên chính form [`Data/2. Good receives/TRAVELER SHEETS raw
material.jpg`](Data/2.%20Good%20receives/TRAVELER%20SHEETS%20raw%20material.jpg),
mục "ROUTING" liệt kê đủ các công đoạn: RECEIVE INFASCO NUT → WASH → LOCK
→ HEAT TREAT METEX → PLATE ANTI-FRICTION → FINAL INSPECTION → **SORT AND
PACK** — AVP chỉ chịu trách nhiệm ô cuối cùng này. AVP **ăn tiền theo
Pot#** (mỗi bin/tote xử lý), không theo kg hay số lượng nhận vào — nên
không có lý do nghiệp vụ nào để cân/đếm nguyên liệu lúc nhận. **Kết luận:
GAP-015 (Goods Receipt) được RÚT KHỎI danh sách việc cần làm — không phải
thiếu, mà là không cần với mô hình ăn công theo Pot# này.**

### 8.2 Số lượng chính thức thật sự nằm ở đâu — sửa lại ưu tiên nguồn dữ liệu

Vì không cân đầu vào, **số lượng đáng tin duy nhất là số ra ở đầu kia**:
tại khâu 3 (máy lựa), có **sensor đếm số ốc vít cho vào từng box** — đây
là số đo bằng máy, không phải ước lượng tay. Số này được ghi vào Work
Station log (`Data/3.Machine/Work station SEP 15-*.pdf`, các trường
"Machine count", "Scale count", "Total quantity") rồi truyền tới máy Label
(khâu 5) để in nhãn đúng số lượng theo từng máy/từng công nhân.

→ **Nâng mức ưu tiên của Work Station log** (đã nêu ở mục 6.2): không chỉ
là dữ liệu downtime phụ trợ như đánh giá trước — đây là **nguồn số lượng
sản xuất chính thức, đo bằng sensor**, đáng tin hơn cả số gõ tay trên
Traveler hay Scanning Sheet. Nếu làm được, nên ưu tiên đọc nguồn này
TRƯỚC cả Excel máy Label ở Phần 7.3 khi có sẵn.

### 8.3 Vai trò thật của form "Good Receives" — phân công, không phải xác nhận nhận hàng

Form khâu 2 thực chất là phiếu **routing/phân công**: Pot# nào chạy máy
nào, công nhân nào vận hành — nguyên liệu "chạy thẳng đến máy đó" theo lời
Owner, không dừng ở kho chờ xác nhận. Cần sửa lại tên gọi/vai trò của
node này trong thiết kế: không phải "Kho nguyên liệu — CCP xác nhận nhận
hàng" như Phần 1 mô tả, mà là **"Routing — phân công Pot#→Máy→Operator"**.

---

## PHẦN 9 — ĐỀ XUẤT TÍNH NĂNG: KIỂM SOÁT OEE (Input=Traveler, Output=số đếm sensor thật) — đề xuất của Owner, 2026-09-15

*Owner đề xuất trực tiếp: dùng đúng 3 nguồn đã xác định ở khâu 3 (Machine)
— Operator sheet, Productivity report, Work Station log — làm nền cho 1
tính năng kiểm soát OEE (Overall Equipment Effectiveness) thật, không cần
chờ dự án ERP mới, có thể làm ngay trên AVP_AI hiện tại.*

### 9.1 Cơ chế đọc Work Station log (đã xác nhận cách đọc số liệu)

Trên form [`Work station SEP 15-4.pdf`](Data/3.Machine/Work%20station%20SEP%2015-4.pdf),
mỗi Bin (1-4) của 1 máy có 1 dãy số 1→36 — công nhân **đánh dấu chéo dần
theo từng box hoàn thành**; số cuối cùng bị đánh chéo = tổng số box đã
đóng cho Part#/Traveler# đang chạy ở Bin đó (ví dụ Owner nêu: đánh chéo
tới ô "24" nghĩa là đã đóng 24 box). Số này đối chiếu chéo với "Machine
count"/"Scale count"/"Total quantity" (đều là số đo bằng sensor/cân điện
tử) đã ghi trên cùng form — 2 cách đếm độc lập cho cùng 1 sự thật.

### 9.2 Công thức Input → Output cho OEE

- **Input**: Traveler# được phân công vào 1 Bin của 1 Máy tại 1 thời điểm
  (M/C Start) — lấy từ chính Work Station log, khớp chéo với routing ở
  khâu 2 (Pot#→Máy→Operator).
- **Output**: số box/quantity đếm được thật bằng sensor tại M/C Stop
  (Machine count, Scale count, Total quantity) — không phải ước lượng.
- **Availability** = thời gian máy chạy thật ÷ thời gian có sẵn (đã trừ
  "Not running waiting: bin", "M/C change over", "M/C repair" — đều có
  mốc thời gian ghi sẵn trên form).
- **Performance** = Output thật ÷ Output lý thuyết (dựa tốc độ định mức
  của máy đó, nếu có số liệu định mức).
- **Quality** = (Output tốt) ÷ (Output tổng) — đối chiếu với Defect count
  trên Split Form (khâu 4) cho cùng Traveler#.

### 9.3 Việc cần làm (khi được xác nhận triển khai)

1. Đọc Work Station log (OCR/nhập tay như các form khác) → 1 bản ghi theo
   (Traveler#, Máy, Bin, ca) gồm: giờ bắt đầu/kết thúc, thời gian
   waiting/changeover/repair, Machine count/Scale count/Total quantity.
2. Khớp Traveler# với FinishGood đã có (Defect count) để tính Quality.
3. Dashboard OEE theo Máy × Ca × Part# — trả lời đúng câu hỏi "hao hụt do
   dùng chung máy" đã đo hiện tượng (62% nhóm Ngày+Máy ≥2 traveler,
   `BAO_CAO_TONG_HOP_AVP.md`) nhưng chưa biết nguyên nhân — giờ có Bin
   nào đợi đổi lô bao lâu, máy nào hỏng bao lâu, theo đúng phút.

*Chưa code — đây là đặc tả tính năng, chờ Owner xác nhận có triển khai
ngay trên AVP_AI hay để dành.*

---

## PHẦN 10 — BỔ SUNG: NĂNG SUẤT LAO ĐỘNG (Output ÷ đầu người) — Máy vs Tay (2026-09-15)

*Owner đề xuất thêm 1 chỉ số: sản lượng đếm được ở mỗi nhóm chia cho số
người vận hành bình quân — để so sánh công bằng giữa 2 phương thức sản
xuất (Máy điện vs Bàn tay), khác với chỉ so tổng sản lượng thô (luôn
nghiêng về máy).*

### 10.1 Bằng chứng thật đã đối chiếu (từ `Productivity SEP 15-2.pdf`
+ `Operator SEP 15-1.pdf`, 5 ngày 2026-09-08→09-14)

- **Tỷ lệ số LÔ (Pot#) qua tay/tổng** ("% of Manual", đã cộng khớp
  100% với TOTAL POT of TA + TOTAL POT of MC = TOTAL POT mỗi ngày):
  42,86% / 31,58% / 44,93% / 52,00% / 47,89% — trung bình **~44% số lô**.
- **Tỷ lệ SẢN LƯỢNG qua tay/tổng** (TB1-INF→TB5-INF ÷ TOTAL): **~29%**
  (3.610.315 / 12.376.955 trong 5 ngày) — thấp hơn tỷ lệ số lô, nghĩa là
  lô ở tay trung bình NHỎ HƠN lô ở máy, nhưng không phải luôn luôn (có
  ngày TB3-INF 1 lô ra tới 594.450 — lớn hơn hầu hết lô máy cùng ngày).
- **Theo đầu người** (dùng đúng số người ghi trên Operator sheet ngày
  2026-09-15: máy 4 nhóm × ~1-2 người/nhóm + 2 người phụ máy riêng ≈ 6
  người; tay 4 bàn × 2 người ≈ 8 người): sản lượng/người ở MÁY cao hơn
  TAY khoảng **~3 lần**. Ngược lại, theo $ (Operation Hour cùng giai
  đoạn): TOTAL MANUAL ($5.864) > TOTAL LINE ($4.156) — tay tạo $/người
  cao hơn máy dù sản lượng/người thấp hơn.

### 10.2 Công thức chỉ số (nguồn số người: đếm thật từ Operator sheet mỗi
ngày — Owner đã xác nhận, KHÔNG dùng giả định cố định)

```
Năng suất/người (theo sản lượng) = Total Quantity (nhóm) ÷ số người ghi trong cột
                                     "Work Station" của Operator sheet CÙNG NGÀY,
                                     đếm theo đúng nhóm (L-1..L-4 = Máy, TBx-INFASCO = Tay)

Năng suất/người (theo $)         = Total Price (nhóm, từ Operation Hour) ÷ số người
                                     cùng nhóm, cùng ngày
```

**Lý do dùng đếm thật mỗi ngày thay vì số cố định**: số người ở mỗi nhóm
có thể đổi theo ngày (vắng/tăng ca) — đếm thật từ Operator sheet tự động
đúng, không cần giả định "máy 2 người/tay 3-4 người" phải sửa tay mỗi khi
nhân sự thay đổi.

### 10.3 Việc cần làm (gộp chung vào đặc tả OEE ở Phần 9, cùng 1 lần đọc
Work Station log + Operator sheet + Operation Hour)

1. Đọc Operator sheet mỗi ngày → đếm số người theo từng nhóm (L-1..L-4,
   TB1..TB5-INFASCO) bằng cách nhóm theo cột "Work Station".
2. Ghép với Total Quantity (Productivity/Work Station log) và Total Price
   (Operation Hour) CÙNG nhóm, CÙNG ngày → ra 2 chỉ số năng suất/người ở
   10.2.
3. Dashboard so sánh Máy vs Tay theo cả 2 chiều (sản lượng/người VÀ
   $/người) — không dùng 1 chiều duy nhất để tránh kết luận sai (máy luôn
   thắng nếu chỉ nhìn sản lượng, tay có thể thắng nếu chỉ nhìn $).

*Chưa code — gộp vào đặc tả Phần 9, chờ Owner xác nhận triển khai.*

### 10.4 Đính chính: cột "Rate" trên Operator sheet KHÔNG phân biệt lương máy/tay

Rà lại đúng cột "Rate" trên `Operator SEP 15-1.pdf` (Owner khoanh đỏ xác
nhận): **cả 4 người trông Line máy điện (Tuyen-004, TrangNg-275,
Noel-290, Vuong-280) VÀ 2 người phụ máy riêng (Gagandeep-285,
Pardeep-240) VÀ 8 người ở bàn tay (TB1-4) đều ghi CÙNG 1 mức "General
Labour"** — không có dòng nào ghi riêng "Machine Operator $30" hay "Phụ
máy $28". Nếu "General Labour" = 1 đơn giá duy nhất (vd $20/giờ), thì
**không có chênh lệch lương giữa trông máy và lựa tay trong dữ liệu
thật** — khi đó chi phí/1 đơn vị sản phẩm ở Máy còn rẻ hơn Tay nhiều hơn
mức đã tính ở 10.1 (vì Máy không còn "gánh" đơn giá cao hơn).

**Việc còn treo, cần Owner xác nhận**: đơn giá $30 (vận hành máy)/$28
(phụ máy) Owner nêu trước đó áp dụng cho vai trò nào — nếu không phải
6 người đang trông Line/máy riêng trong ca này, thì 2 mức đó có thể là
đơn giá CHÍNH THỨC cho 1 chức danh khác chưa xuất hiện trong ca
2026-09-15, hoặc bảng chấm công "Rate" hiện tại chưa cập nhật đúng đơn
giá thật của từng người — **không suy diễn thêm, chờ Owner xác nhận
nguồn đơn giá thật trước khi tính lại chi phí/đơn vị chính xác.**

---

## PHẦN 11 — ĐỀ XUẤT TÍNH NĂNG: HAO HỤT (PCS) THEO TỪNG MÁY (2026-09-15)

*Nối tiếp GAP đã đo ở `BAO_CAO_TRINH_OWNER_QUY_TRINH_2026-09-15.md` bước
3: "124/1.911 traveler (6,5%) sản lượng đóng gói lệch >50% so với nguyên
liệu nhận vào — không biết mất ở đâu vì không tách được theo mẻ". Work
Station log (đã xác nhận ở Phần 8.2 là nguồn sản lượng chính thức, đo
bằng sensor) giải quyết ĐÚNG chỗ "không tách được theo mẻ" này — vì mỗi
dòng Work Station log đã có sẵn Traveler#+Máy+Bin, tách theo mẻ tự
nhiên.*

### 11.1 Công thức hao hụt theo máy

```
Hao hụt (pcs) 1 mẻ = Pieces nhận vào (RawMaterial, theo Traveler#)
                      − Total quantity sản xuất ra (Work Station log, sensor, theo Traveler#+Máy+Bin)

Hao hụt theo máy = tổng hao hụt tất cả mẻ đã chạy qua máy đó, trong 1 khoảng thời gian
% hao hụt/máy    = Hao hụt theo máy ÷ tổng Pieces nhận vào đã chạy qua máy đó
```

### 11.2 Vì sao tách theo MÁY (không chỉ theo Traveler) là bước tiến thật

Cách đo hiện tại (so Pieces nhận vào của CẢ Traveler với Quantity đóng
gói cuối cùng) không biết hao hụt xảy ra ở máy nào — vì 1 Traveler có
thể bị **chia nhỏ chạy qua NHIỀU máy khác nhau** (đã xác nhận qua hiện
tượng "62% nhóm Ngày+Máy chạy chung ≥2 traveler" và ngược lại 1 Pot#
cũng có thể tách sang nhiều Bin/máy). Work Station log ghi đúng **Máy
nào, Bin nào, chạy Traveler# nào, ra bao nhiêu** — cộng dồn theo máy sẽ
lộ ra máy nào đang có tỷ lệ hao hụt cao bất thường so với các máy khác
cùng chạy 1 loại Part#, thay vì đổ chung 1 con số mơ hồ cho "hao hụt do
dùng chung máy" như hiện nay.

### 11.3 Việc cần làm (dùng chung nguồn đọc Work Station log ở Phần 9)

1. Khi đọc Work Station log (Phần 9.3 bước 1), giữ đúng khoá (Traveler#,
   Máy, Bin) cho mỗi dòng Total quantity.
2. Lấy Pieces nhận vào tương ứng từ RawMaterial theo Traveler# (đã có sẵn
   trong AVP_AI).
3. Nếu 1 Traveler chạy qua nhiều Máy/Bin (nhiều dòng Work Station log
   cùng Traveler#) → cộng dồn Total quantity các dòng đó trước khi so
   với Pieces nhận vào — tránh báo hao hụt giả vì mới đọc 1 phần mẻ.
4. Dashboard % hao hụt theo Máy × Part# × Ca — xếp hạng máy nào hao hụt
   cao nhất, để ưu tiên kiểm tra/bảo trì đúng chỗ thay vì đoán mò.

*Chưa code — đặc tả tính năng, gộp chung nhóm việc với Phần 9 (đọc Work
Station log 1 lần, dùng cho cả OEE lẫn hao hụt theo máy).*

---

## PHẦN 12 — QUYẾT ĐỊNH: GỘP PHẦN 9-11 THÀNH 1 MODULE RIÊNG "NHÂN CÔNG & NĂNG SUẤT" (2026-09-15)

Owner xác nhận: toàn bộ Phần 9 (OEE), Phần 10 (năng suất/người), Phần 11
(hao hụt theo máy) gộp chung thành **1 module riêng trong AVP_AI**, KHÔNG
gắn vào Dashboard hiện có, KHÔNG phải 1 khâu duyệt hàng (không có CCP/nút
xác nhận như PO/Kho/FinishGood/PackingList) — chỉ là trang xem/phân tích.

### Phạm vi module (gộp 3 nguồn dữ liệu)

| Nguồn | Đưa vào Sheet mới | Dùng để tính |
|---|---|---|
| Operator sheet (giờ công) | `LaborLog` (Date, Person, TimeIn, TimeOut, WorkStation, Rate) | Số người/nhóm/ngày (Phần 10) |
| Work Station log (sensor) | `MachineLog` (Date, Traveler#, Máy, Bin, giờ chạy, waiting/changeover/repair, Total quantity) | OEE (Phần 9), hao hụt theo máy (Phần 11) |
| RawMaterial (đã có sẵn) | — (dùng thẳng, không cần Sheet mới) | Mẫu số hao hụt (Phần 11) |

### Triển khai theo 3 giai đoạn đã thống nhất trước đó (chưa đổi)

1. **GĐ1**: đọc Operator sheet → `LaborLog`, chưa tính gì.
2. **GĐ2**: ghép `LaborLog` với FinishGood.Máy (đã có, ghi hồi tố) → năng
   suất/người gần đúng, dùng ngay không cần chờ Work Station log.
3. **GĐ3**: đọc Work Station log → `MachineLog` → nâng cấp lên OEE + hao
   hụt theo máy chính xác (Phần 9, 11).

*Chưa code — Owner mới xác nhận PHẠM VI module, chưa xác nhận bắt đầu từ
giai đoạn nào.*

---

## PHẦN 13 — ĐỀ BÀI: MODULE ĐỐI CHIẾU TRẠNG THÁI TRƯỚC PACKING SLIP (2026-09-15)

*Viết lại theo đúng quy trình THẬT đã xác nhận trong phiên này bằng cách
đọc trực tiếp file thật — không suy diễn. Đây là điểm nghẽn thời gian
thật đã tìm ra, KHÔNG phải 3 bước đầu của chuỗi.*

### 13.1 Quy trình thật (đã xác nhận bằng file thật, không phải giả định)

```
[Khâu 3: Máy lựa] → [Khâu 5: Máy Label — QC nhập tay, sensor đếm]
        → tạo ra "FINISHED PALLETS ARCHIVE"
        → in nhãn dán lên từng box
        → [Khâu 6: Scanning/Wrapping] — TỰ ĐỘNG xuất báo cáo từ
          CHÍNH database khâu 5 (không gõ tay lại số — đã xác nhận:
          "WRAPPING SUMMARY" và "SCANNING" là CÙNG 1 form, cùng cột)
        → [Điểm nghẽn thật]: người in báo cáo Scanning ra GIẤY, tự tay
          tích ✓ từng dòng, tự tay cộng lại bằng tay ở lề giấy — dù số
          đã đúng sẵn trong database điện tử
        → [Khâu cuối]: QC/nhân viên PS tự tra xem Traveler# nào đã đủ
          điều kiện xuất, tự chuyển trạng thái SHIPPED — đây là bước
          KHÔNG tự động, đã gây ra sự cố thật (PS 30102, 2 dòng
          Quantity=0 xuất thật cho khách)
```

### 13.2 Vấn đề cụ thể cần AVP_AI giải quyết

Không phải "tạo dữ liệu mới" — 3 khâu đầu (nhập liệu máy, in nhãn, xuất
báo cáo Scanning) đã nhanh, tự động, không cần sửa. Vấn đề nằm ở đúng
2 chỗ:

1. **Report đã đúng số nhưng vẫn bị in giấy + tích tay lại** — vì không
   có màn hình nào cho phép NGƯỜI XEM TRỰC TIẾP trạng thái đã đủ điều
   kiện xuất hay chưa, phải tự tra bằng mắt trên giấy.
2. **Trạng thái SHIPPED không nhất quán giữa các sổ** — đã đo được
   43% traveler lệch trạng thái SHIPPED giữa 2 sổ song song (Phần báo
   cáo `BAO_CAO_TRINH_OWNER_QUY_TRINH_2026-09-15.md` bước 7) — không ai
   biết tin sổ nào.

### 13.3 Việc cần làm — chi tiết

1. **Đọc đúng 4 nguồn hiện có, dùng Traveler# làm khoá nối duy nhất**
   (không cần thêm cột nào ở nguồn gốc):
   - FINISHED PALLETS ARCHIVE (khâu 5) — Quantity/Serial#/Skid#/Operator
   - Sheet **CHECKING SUMMARY** trong `DAILY LOG CHECK SHEET.xlsm`
     (khâu 6) — Machine/MC#/SKID#/Reject/SHIPPED — đọc thẳng từ Excel,
     KHÔNG cần đọc bản PDF đã in ra (PDF chỉ là export để in giấy tích
     tay, không phải nguồn gốc)
   - Sheet **SHIPPED** riêng trong cùng file (hiện đang lệch 43% với
     CHECKING SUMMARY)
   - Sheet **WORK ORDER** (có PO, dùng cho mục đích khác — không trộn
     chung logic SHIPPED vào đây)
2. **Đối chiếu tự động mỗi Traveler#** phải có ĐÚNG 1 trạng thái SHIPPED
   nhất quán giữa CHECKING SUMMARY và sheet SHIPPED — lệch thì báo ra
   màn hình, KHÔNG tự ý chọn tin sổ nào (đúng nguyên tắc CCP đã dùng ở
   Phần 7).
3. **Chặn cứng trước khi tạo Packing Slip** nếu Traveler# đó: còn thiếu
   Quantity (=0), còn HOLD chưa xử lý, hoặc đang lệch trạng thái giữa
   2 sổ — đúng nguyên nhân gốc của sự cố PS 30102.
4. **Bỏ hẳn bước in giấy + tích tay ✓** — thay bằng 1 màn hình duy nhất:
   nhân viên PS gõ/quét Traveler#, hệ thống tự trả lời "đủ điều kiện
   xuất hay chưa, vì sao chưa" — không cần mở nhiều Excel, không cần tự
   cộng bằng tay ở lề giấy như hiện nay.

### 13.4 Có cần chỉnh sửa form/database hiện có không?

**Không cần sửa cấu trúc cột của form gốc nào.** Chỉ cần:
- Xin Owner file Excel gốc của máy Label (khâu 5) — như đã ghi ở Phần
  6.3, thay vì đọc bản PDF in ra (giảm sai số do OCR chữ viết tay/serial
  bị dính chữ).
- Đọc trực tiếp sheet CHECKING SUMMARY + SHIPPED trong
  `DAILY LOG CHECK SHEET.xlsm` — 2 sheet này đã tồn tại sẵn, không cần
  tạo mới.
- Traveler# đã có mặt ở cả 4 nguồn dưới dạng số nguyên/text thống nhất
  — đủ để làm khoá nối, không cần thêm trường nào.

*Chưa code — đây là đề bài/đặc tả, chờ Owner xác nhận bắt đầu triển
khai.*

---

## PHẦN 14 — QUY TRÌNH CẬP NHẬT MỚI NHẤT (2026-09-16, Owner tự viết lại,
đã đối chiếu file thật)

*Owner viết lại toàn bộ quy trình theo cấu trúc thư mục `Data/` mới nhất
(đã đơn giản hoá: bỏ hẳn "Good receives", gộp "Machine" vào Traveler,
đổi "Labeling"→"FINISHED PALLET", gộp "Scanning" vào "WRAPPING"). Đã đối
chiếu TỪNG bước với file thật, không suy diễn.*

### 14.1 6 bước — đã xác nhận đúng file thật (1, 2, 3, 4, 6)

| # | Bước | File thật đã đối chiếu | Kết quả đối chiếu |
|---|---|---|---|
| 1 | PO từ nhà cung cấp qua email/file | [`Data/1.PO/PO PO SEP 8_193853.pdf`](Data/1.PO/PO%20PO%20SEP%208_193853.pdf) | Tồn tại thật, đúng tên |
| 2 | Nguyên liệu về kèm Pot#+Traveler, số liệu tạm nhận theo PO | [`Data/2. Traveler/TRAVELER SHEETS SEP 9.pdf`](Data/2.%20Traveler/TRAVELER%20SHEETS%20SEP%209.pdf) | Tồn tại thật |
| 3 | Lựa tại từng station, đầu ra đếm bằng sensor | (đã xác nhận ở Phần 8.2 — sensor tại khâu lựa là nguồn số lượng chính thức) | Khớp với phát hiện trước |
| 4 | Công nhân ghi phiếu → nhập vào database **WorkStationArchive** | [`Data/3.FINISHED PALLET/FINISHED PALLET REPORT_final.xlsm`](Data/3.FINISHED%20PALLET/FINISHED%20PALLET%20REPORT_final.xlsm) sheet **WorkStationArchive** | **Xác nhận đúng 100%** — đây chính là "FINISHED PALLETS ARCHIVE" đã đọc bản PDF ở Phần 7, nay có **file Excel gốc thật** (3.103 dòng, cột: Traveler/Part Number/Pot/Serial#/Lot/Skid#/Station#/OPERATOR/# of boxes/Unit Quantity/Total Quantity/Date/Notes) — **GAP "chưa có Excel gốc" ở Phần 6.3 đã ĐÓNG**, không cần đợi Owner gửi nữa. |
| 6 | Xuất Packing Slip — tạo tay (chọn Traveler/PO/Part, hàng HOLD không được xuất) hoặc tạo tự động đề xuất FIFO | [`Data/5.Parking slip/PACKING SLIP- SEP 09.pdf`](Data/5.Parking%20slip/PACKING%20SLIP-%20SEP%2009.pdf) | **Khớp đúng tính năng ĐÃ CÓ SẴN trong AVP_AI** (bảng picker chọn nhiều + "Tạo PS tự động" FIFO, xem SES-20260915-002 mục 2.1) — không cần thiết kế thêm, chỉ cần đảm bảo chặn HOLD đúng như Phần 13. |

### 14.2 Bước 5 — CỘT STATUS (good/hold/concession): ĐÃ XÁC NHẬN LÀ ĐỀ XUẤT MỚI, chưa tồn tại

Owner mô tả: sau khi in bảng Wrapping
([`Data/4.WRAPPING/Wrapping_final.xlsm`](Data/4.WRAPPING/Wrapping_final.xlsm))
và đi đối chiếu thực tế, nhân viên **cập nhật vào 1 cột STATUS** với 3 giá
trị: **good** (sẵn sàng xuất), **hold** (giữ lại), **concession** (nhượng
bộ khi có nhu cầu xuất sớm — đúng ISO 9001 §8.7 đã dùng trong thiết kế
trước).

Đã rà toàn bộ sheet **CHECKING SUMMARY** của `Wrapping_final.xlsm` (2.147
dòng, 29 cột) — **KHÔNG tìm thấy cột STATUS hay bất kỳ giá trị
good/hold/concession nào**. Owner xác nhận trực tiếp: **đây là đề xuất
MỚI cho AVP_AI, chưa tồn tại trong Excel hiện tại** — không phải hiện
trạng bị bỏ sót.

→ Việc cần làm (đặc tả, chưa code): thêm 1 trường **`qc_status`** (giá
trị `good` / `hold` / `concession`) vào bản ghi theo Traveler#, do nhân
viên cập nhật SAU KHI đối chiếu thực tế với bảng Wrapping in ra — đây
chính là điểm **CCP con người** thay thế cho việc "in giấy tích tay ✓"
đã mô tả ở Phần 13.2 — nhân viên vẫn đi đối chiếu thực tế (không bỏ bước
kiểm tra vật lý), nhưng kết quả được **gõ 1 lần vào đúng 1 trường**, thay
vì tích tay trên giấy rồi không ai tổng hợp lại được.

### 14.3 Cập nhật sơ đồ chuỗi (thay thế sơ đồ cũ ở Phần 6.1)

```
[1.PO] → [2.Traveler — nhận Pot#/nguyên liệu] → [3.Máy/tay lựa, sensor đếm]
   → [4.FINISHED PALLET — WorkStationArchive: Serial#/Skid#/Qty/Date]
   → [5.WRAPPING — in bảng, đối chiếu thực tế, gõ qc_status: good/hold/concession] (MỚI)
   → [6.Parking slip — tạo tay (chặn HOLD) hoặc tự động FIFO] (đã có sẵn trong AVP_AI)
```

*Chưa code phần 14.2 (trường `qc_status` mới) — đây là đặc tả, chờ Owner
xác nhận triển khai cùng lúc với module Phần 13.*
