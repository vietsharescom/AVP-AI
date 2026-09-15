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
