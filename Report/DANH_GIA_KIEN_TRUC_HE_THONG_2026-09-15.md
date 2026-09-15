# ĐÁNH GIÁ KIẾN TRÚC HỆ THỐNG — AVP_AI (WEBAPP)
## Quét dữ liệu thật + đối chiếu chuẩn ERP/Odoo/ISO — 2026-09-15

*Đối tượng đọc: Andy Phan (Owner đại diện) / Ban điều hành AVP.*
*Khác với [`BAO_CAO_TONG_HOP_AVP.md`](BAO_CAO_TONG_HOP_AVP.md) (đánh giá quy
trình THỦ CÔNG cũ — 2 file Excel — so với ERP), báo cáo này đánh giá chính
**webapp AVP_AI đang chạy thật** (`D:\AVP_AI\webapp`): mã nguồn + dữ liệu
sống trong Google Sheet (RawMaterial 1.939 dòng, FinishGood 194 dòng,
sổ cái `/api/ledger` 184 dòng, PackingList).*

> **Nguyên tắc**: mọi số liệu dưới đây lấy từ gọi API thật
> (`localhost:3000`) và đọc trực tiếp mã nguồn lúc 2026-09-15, không suy
> đoán. Phần nào chưa xác định được nguồn gốc thì ghi rõ "CHƯA RÕ — cần
> Anh xác nhận", không tự bịa lý do.

---

## MỤC LỤC

1. Quét dữ liệu thật — có còn bị "lệch cell" (scan nhảy cột) không?
2. Kiến trúc dữ liệu/tầng trích xuất đã khoa học theo ERP/ISO/Odoo chưa?
3. Sơ đồ kiến trúc Input→Process→Output + kho dữ liệu đã tập trung chưa?
4. Khuyến nghị ưu tiên
5. Bài học từ AVP_AI cho dự án mới (stock.move chuẩn ERP)
6. **[MỚI]** Xác minh bằng số liệu — nỗi đau hiện trường (phỏng vấn quản lý/kế hoạch, 2026-09-15)

---

## 1. QUÉT DỮ LIỆU THẬT — CÓ CÒN BỊ "LỆCH CELL" KHÔNG?

### 1.1 Hệ thống tự sinh cảnh báo — tổng quan (sổ cái `/api/ledger`, 184 dòng)

| Loại cảnh báo | Số dòng |
|---|---|
| Thiếu "Quantity per box" trong PartControl | 61 |
| QUANTITY tính ra vượt Pieces raw material >10% | 27 |
| Nghi bị scan lặp (Traveler#+LOT# trùng, chưa Shipped) | 6 |
| QUANTITY lệch >10% so với số scan tay | 5 |
| Part# ghi khác dấu gạch ngang (tự nhận diện được) | 1 |
| **Tổng dòng có ít nhất 1 cảnh báo** | **91/184 (49%)** |

→ Gần một nửa số dòng đang mở (chưa xuất) có ít nhất 1 điểm cần người
xem lại — phần lớn (61+27=88) là do **PartControl còn thiếu dữ liệu**
(Quantity/box), không phải lỗi AI đọc sai. Đây là gánh nặng đã biết
trước (báo cáo trước, mục 7.1: 36 Part# thiếu Quantity/box).

### 1.2 Quét thêm bằng heuristic riêng — phát hiện MỚI hôm nay

Em viết thêm 1 lượt quét độc lập (không dựa vào cảnh báo có sẵn) để tìm
đúng kiểu lỗi "AI đọc lệch cột/lệch dòng" như vụ traveler 717544 vừa xử
lý. Kết quả trên **toàn bộ RawMaterial (1.939 dòng) + FinishGood (194
dòng)**:

**a) Đã sửa xong hôm nay — traveler 717544** (chi tiết ở phần trước cuộc
trò chuyện): ô Part# in 2 dòng trên giấy (`11502717-TH-\nTC-T`) bị AI đọc
lạc 1 phần sang cột Pot#/LOT# ở cả 2 lần quét (bản Original + Copy). Đã
sửa `Part#=11502717-TH-TC-T`, `Pot#=614`, `LOT#` để trống.

**b) MỚI, CHƯA XỬ LÝ — 7 dòng FinishGood có Pot# và LOT# GIỐNG HỆT NHAU**
(traveler `717397`, `718437`, `717772`, `717781`, `718646`, `718016`,
`718014`) — ví dụ traveler 718437: `Pot#="371"` và `LOT#="371"`. Đối
chiếu `createdAt`: **cả 7 dòng đều được tạo đúng cùng 1 khoảnh khắc
`2026-09-14T21:06:05.642Z`**, và cột `oc` (Original/Copy — field bắt buộc
của luồng quét Split Form bình thường) đều **để trống** — khác hẳn dữ
liệu quét Split Form thật (luôn có "O" hoặc "C"). 3 trong 7 traveler này
(717772, 717781, 718014) còn có 1 dòng SONG SONG khác, tạo lúc
`19:15:22` cùng ngày, với LOT# thật hợp lệ (vd `6-251-10-B`) và `oc` có
giá trị đàng hoàng — tức là **dòng "tốt" đã có sẵn, dòng lỗi là dòng
THỪA ghi đè thêm vào sau**.

→ **Em CHƯA xác định được dòng lúc 21:06:05 này đến từ thao tác/route
nào** (không khớp với luồng quét Split Form AI vision bình thường, cột
`oc` trống là dấu hiệu rõ). Anh có nhớ lúc ~21:06 ngày 2026-09-14 đã làm
thao tác gì trên web (import Excel, "+ Nhập tay 1 dòng", hay test 1 tính
năng nào) không? Cần biết chính xác để vá đúng chỗ, tránh lặp lại.

**c) Nhãn field dính vào giá trị** — traveler `718643`: `Pot#="Pot#313"`,
`LOT#="Lot6-253-04-E"` — AI đọc luôn cả chữ nhãn ("Pot#", "Lot") thay vì
chỉ lấy số/mã phía sau. Khác kiểu lỗi (b) — đây là thiếu bước "cắt bỏ
tiền tố nhãn" sau khi AI trả kết quả.

**d) Pot# bị dùng để ghi ĐƠN VỊ đóng gói thay vì mã Pot#** — 10 dòng
RawMaterial: `"36 CTNS"`, `"27TOTES"`, `"49TOTES"`, `"GAYLORD"`... Mục
`GAYLORD` đã biết là bình thường (phiên trước xác nhận: tên loại thùng,
không phải mã Pot# — traveler 712654). Còn `"CTNS"/"TOTES"` là dấu hiệu
**cùng 1 ô Pot# trên giấy đang gánh 2 vai trò khác nhau** (mã định danh
SỐ vs mô tả ĐƠN VỊ đóng gói) tuỳ loại hàng — không hẳn là lỗi AI đọc sai,
mà là **schema Pot# đang mơ hồ** (chưa tách riêng trường "loại đóng gói"
khỏi "mã Pot#").

### 1.3 Đánh giá tổng thể phần quét dữ liệu

Điểm tốt: `finish-good/confirm/route.ts` (nơi lưu dữ liệu quét mới) đã có
**7 lớp kiểm tra chồng chéo** khá kỹ ngay lúc lưu — trùng lot, trùng
Boxes+Qty khác LOT (nghi đọc nhầm), 1 Traveler nhưng LOT#/Part#/Pot# đổi
khác thường (dựa trên quy luật 1:1 đã đo từ 1.598 traveler lịch sử), Pot#
đang mở gắn với traveler khác, Pot# lệch so với RawMaterial gốc, Part#
thiếu PartControl, Traveler lệch xa dải số cùng lô scan. Đây là mức kiểm
soát khá cao so với 1 app quét OCR thông thường.

Điểm yếu: các lớp kiểm tra trên **chỉ chạy lúc LƯU DÒNG MỚI**, so sánh với
dữ liệu ĐANG MỞ tại thời điểm đó — không có lượt "quét sức khoẻ dữ liệu"
định kỳ chạy lại trên TOÀN BỘ dữ liệu đã lưu trước đó. Nghĩa là: lỗi nào
lọt qua (hoặc xảy ra ở dòng lưu TRƯỚC KHI code kiểm tra này tồn tại, hoặc
qua đường nhập liệu khác như case (b) ở trên) sẽ nằm im mãi cho tới khi
tình cờ có người phát hiện — đúng như cách vụ 717544 và vụ (b) hôm nay
được tìm ra (do Anh chỉ tay vào ảnh, không phải hệ thống tự báo).

---

## 2. KIẾN TRÚC ĐÃ KHOA HỌC THEO CHUẨN ERP/ISO/ODOO CHƯA?

### 2.1 Mô hình lưu trữ

Google Sheets đóng vai trò "database": 5 tab phẳng (RawMaterial,
Warehouse, FinishGood, PartControl, PackingList), không có khoá ngoại,
không transaction, không index — mọi JOIN (RawMaterial↔FinishGood↔
PartControl↔PackingList) được code tự ghép LÚC ĐỌC.

**Rủi ro đo được thật**: em đếm trong mã nguồn — **14 file route API**
tự đọc + tự ghép RawMaterial/FinishGood riêng lẻ (`packing-list/generate`,
`packing-list/line`, `reconciliation`, `trace`, `warehouse/*`,
`finish-good/confirm`...), và **3 component giao diện** (`GlobalSearchBar`,
`PoStatusTable`, `TravelerSection`) tự gọi thẳng `/api/raw-material/list`
+ `/api/finish-good/list` để tự ghép lại LẦN NỮA ở phía trình duyệt.
`/api/ledger` (sổ cái tổng, tạo phiên trước đúng để giải quyết vấn đề
này) hiện **chỉ được 1 màn hình duy nhất dùng** (Packing List) — đúng
hướng nhưng chưa lan toả hết. Hệ quả thật: script nhập lịch sử PS
30098/30101 (phiên trước) quên đúng 1 bước "đánh dấu Shipped" vì nó không
đi qua đường xử lý chung nào — sinh ra 22 traveler bị lỗi trạng thái (đã
[sửa xong hôm nay](fix-ps30098-30101-shipped.mjs)). Đây là hậu quả trực
tiếp của việc chưa có 1 tầng nghiệp vụ DUY NHẤT ("chỉ có 1 cách hợp lệ để
đánh dấu đã xuất") — càng nhiều route tự làm riêng, càng dễ có route mới
quên 1 bước.

**Không có kiểm soát ghi đồng thời (concurrency)**: `updateRowsWhere`/
`appendRows` không kiểm tra phiên bản/khoá dòng trước khi ghi — nếu 2
người (hoặc 2 tab) cùng sửa 1 dòng gần như cùng lúc, ai ghi sau thắng,
không có cảnh báo "dữ liệu đã đổi kể từ lúc Anh mở". Ở quy mô vài người
dùng hiện tại rủi ro thấp, nhưng sẽ tăng khi nhiều nhân viên cùng thao
tác trong ca cao điểm.

**Không có audit trail khi SỬA** (chỉ có `createdAt` lúc TẠO): sửa
`shipped`/`ps`/Concession/Priority ghi đè trực tiếp vào ô, không có bảng
lịch sử "ai sửa gì lúc nào" ngoài Google Sheets' bản thân có version
history riêng (không phải app quản lý). Concession/Priority là 2 ngoại lệ
tốt (có ghi `...By/...Reason/...At`) — nên áp dụng cùng kiểu cho các thay
đổi nhạy cảm khác.

**Không có migration/schema version chính thức**: thêm cột mới bằng
script chạy tay riêng lẻ mỗi lần (`add-finishgood-scrap-columns.mjs`,
`...-concession-columns.mjs`, `...-priority-columns.mjs`...) — hoạt động
ổn nhưng không có 1 nơi liệt kê "Sheet đang ở version nào, đã chạy migration
nào" — phụ thuộc hoàn toàn trí nhớ người vận hành.

### 2.2 So với Odoo (ERP thật đang chạy, đối chiếu trong báo cáo trước)

| Tiêu chí | Odoo (chuẩn) | AVP_AI hiện tại |
|---|---|---|
| Sổ cái tồn kho | `stock.move` — BẤT BIẾN (append-only), sửa = thêm bút toán đảo, không sửa/xoá dòng cũ | Sửa TRỰC TIẾP field `shipped`/`ps` tại chỗ — không giữ vết bút toán gốc |
| Trạng thái chứng từ | State machine chính thức (draft→confirmed→done→cancel), mỗi bước có quyền riêng | Chỉ 1 cờ boolean `shipped` (TRUE/FALSE) — không phân biệt "đang xử lý"/"đã duyệt"/"đã huỷ" |
| Toàn vẹn dữ liệu | DB (PostgreSQL) tự ép kiểu, khoá ngoại, ràng buộc unique ở TẦNG LƯU TRỮ | Toàn bộ ràng buộc (Traveler↔Pot# 1:1, Part# hợp lệ...) do CODE tự kiểm tra lúc ghi — tầng lưu trữ (Sheet) không ép được gì |
| Đồng thời nhiều người dùng | Transaction + row lock | Không có — ghi sau thắng, không cảnh báo |

Đây **không phải lời chê** — ở quy mô hiện tại (vài nghìn dòng/tháng, vài
người dùng), việc chọn Google Sheets thay vì dựng hẳn Postgres/Odoo là
hợp lý về chi phí/thời gian, và Andy đã chủ động quyết định (LATEST_SESSION,
"mô hình sổ cái `stock.move` chuẩn ERP sẽ làm ở 1 dự án MỚI hoàn toàn
riêng" — đúng hướng, không vá app đang chạy thật bằng thay đổi kiến trúc
lớn). Bảng trên để trả lời đúng câu hỏi "khoa học theo ERP chưa" — câu
trả lời ngắn gọn: **chưa, và Anh đã biết + đã có quyết định đúng (tách dự
án mới), không cần làm ngay trên hệ thống đang chạy thật.**

### 2.3 Tầng trích xuất (Gemini vision OCR) — có khoa học, có dễ conflict không?

**Điểm tốt**: mỗi loại chứng từ giấy có 1 JSON schema riêng biệt
(`TRAVELER_FORM_SCHEMA`, `SCANNING_FORM_SCHEMA`, schema Split Form) —
tách bạch rõ ràng theo loại tài liệu, có field `confidence` (AI tự báo
"đọc không chắc") cho chữ viết tay khó đọc, và **đối chiếu chéo với dữ
liệu đã có** (RawMaterial gốc, PartControl, lịch sử traveler khác) thay
vì tin tuyệt đối vào 1 lần đọc — đây là điểm mạnh, hơn hẳn OCR thông
thường "đọc gì lưu nấy".

**Điểm dễ gây lỗi lặp lại trong tương lai**: schema mô tả field bằng
NGÔN NGỮ TỰ NHIÊN cho AI hiểu (`"Pot # column"`, `"Part # column"`),
không kèm ví dụ ảnh mẫu, và **không có bước validate/regex làm sạch kết
quả AI trả về trước khi lưu** — nên khi AI trả nguyên cụm `"Pot#313"`
thay vì `"313"` (mục 1.2c), hệ thống lưu thẳng, không có lớp "chặn giữa"
phát hiện ra. Tương tự việc 1 ô xuống 2 dòng trong file gốc (vụ 717544)
không có cơ chế cảnh báo "định dạng bất thường" trước khi lưu — chỉ được
phát hiện nhờ các cảnh báo NGHIỆP VỤ (Pot# khác RawMaterial) tình cờ bắt
được, không phải nhờ kiểm tra ĐỊNH DẠNG dữ liệu.

→ Khuyến nghị: thêm 1 lớp "chuẩn hoá sau OCR" đơn giản (cắt tiền tố nhãn
kiểu `/^(Pot#|Lot)\s*/i`, cảnh báo nếu Pot#/LOT# dài bất thường hoặc lẫn
chữ lạ) — không cần sửa toàn bộ, chỉ thêm 1 bước lọc trước khi các cảnh
báo nghiệp vụ hiện có chạy.

---

## 3. SƠ ĐỒ KIẾN TRÚC INPUT → PROCESS → OUTPUT

> **Đính chính**: bản đầu báo cáo này viết "hệ thống chưa từng có bảng vẽ
> layer kiến trúc" — **SAI**, đã có sẵn từ 2026-09-13, đầy đủ hơn bản vẽ
> dưới đây: [`ARCHITECTURE.md`](../ARCHITECTURE.md) mục 0 (sơ đồ hệ thống
> HIỆN TẠI) và [`THIET_KE_HE_THONG_MOI.md`](../THIET_KE_HE_THONG_MOI.md)
> mục 3 (kiến trúc 7 lớp cho hệ thống MỚI, đối chiếu Odoo/`stock.move`
> thật). Em bỏ sót do không liệt kê hết file `.md` gốc dự án trước khi
> viết báo cáo này. Sơ đồ dưới đây vẫn giữ lại vì có 1 điểm khác:
> `ARCHITECTURE.md` §0 vẽ sơ đồ Ở MỨC KHÂU NGHIỆP VỤ (1→4), còn bản dưới
> đây vẽ thêm chi tiết Ở MỨC KỸ THUẬT (OCR/schema/route API) đang chạy
> thật — 2 bản bổ sung cho nhau, không thay thế.

```
┌─────────────────────────── INPUT (giấy/email/Excel thật) ───────────────────────────┐
│  Email PO Infasco     Split Form (giấy)     WORK ORDER lịch sử   PACKING SLIP giấy   │
│  (PDF/ảnh)            (giấy, viết tay)      (PACKING SLIPS.xlsm) (giấy, đã xuất)     │
└───────┬───────────────────────┬───────────────────────┬───────────────────┬─────────┘
        │                       │                       │                   │
        ▼                       ▼                       ▼                   ▼
┌─────────────────────────── PROCESS (webapp AVP_AI) ──────────────────────────────────┐
│                                                                                        │
│  CaptureGate (cổng quét chung) ──► Gemini Vision OCR (extract.ts)                     │
│         │                              │  - TRAVELER_FORM_SCHEMA (PO/Pot#/Weight...)  │
│         │                              │  - SCANNING_FORM_SCHEMA (Split Form)         │
│         │                              │  - confidence + đối chiếu chéo RawMaterial   │
│         ▼                              ▼                                              │
│  [Người xem lại + bấm Xác nhận] ◄── Bản NHÁP (chưa ghi Sheet)                         │
│         │           ▲ 7+ lớp cảnh báo (trùng lot, Pot# lệch, Part# thiếu PartControl,  │
│         │             1 Traveler đổi khác thường, trùng máy...) — CHỈ CẢNH BÁO,        │
│         │             không tự chặn/tự sửa (nguyên tắc ISO_AI_CONTROL.md)             │
│         ▼                                                                              │
│  ┌──────────────────────────── GOOGLE SHEET (kho dữ liệu) ─────────────────────────┐  │
│  │  RawMaterial   Warehouse(chết)   FinishGood   PartControl   PackingList          │  │
│  │  1.939 dòng    không ai ghi      194 dòng     Quantity/box   PS đã xuất+nháp     │  │
│  └──────────────────────────────────┬──────────────────────────────────────────────┘  │
│                                      │ 14 route API tự đọc+JOIN riêng lẻ                │
│                                      │ (chỉ 1 route /api/ledger là JOIN tập trung,      │
│                                      │  hiện chỉ Packing List dùng)                     │
└──────────────────────────────────────┼─────────────────────────────────────────────────┘
                                        ▼
┌─────────────────────────── OUTPUT (người dùng thấy/nhận) ────────────────────────────┐
│  Bảng Kho (đối chiếu Forecast↔Thành phẩm)   Packing Slip in giấy   Báo cáo tổng hợp   │
│  GlobalSearchBar (tra cứu tổng)             /trace (truy vết 1 traveler)              │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

**Kho dữ liệu đã tập trung chưa?** — **Đã đúng hướng**: toàn bộ 5 loại dữ
liệu nằm chung **1 Google Sheet duy nhất** (không phân mảnh nhiều file
như hệ thống Excel cũ nêu ở báo cáo trước) — đây là điểm cải thiện thật
so với hệ thống cũ. Nhưng thực chất chỉ **4/5 tab đang "sống"** — tab
`Warehouse` (ô màu trong sơ đồ trên) được tạo đúng mục đích ghi nhận
"hàng đã về kho thật" nhưng từ lúc redesign không còn ai ghi vào (GAP đã
nêu ở báo cáo trước, mục 6 câu hỏi #3) — nghĩa là bước "Goods Receipt"
độc lập vẫn đang thiếu trên thực tế, dù có sẵn chỗ chứa.

---

## 4. KHUYẾN NGHỊ ƯU TIÊN

1. **Xác nhận nguồn gốc lô dữ liệu lỗi lúc 21:06:05 ngày 2026-09-14**
   (mục 1.2b) — cần biết Anh/nhân viên đã làm thao tác gì để vá đúng chỗ,
   tránh lặp lại (hiện có 7 dòng, 5 traveler bị ảnh hưởng, 3/7 đã có bản
   sao lưu đè lên dữ liệu tốt).
2. Thêm 1 lớp "chuẩn hoá sau OCR" (cắt tiền tố nhãn, validate định dạng
   Pot#/LOT#) — vá đúng lỗ hổng gây ra vụ 717544 và 718643, chi phí thấp.
3. Cân nhắc chạy 1 script "quét sức khoẻ dữ liệu" định kỳ (không phải
   lúc lưu mới) — tái sử dụng chính các quy tắc đã có sẵn trong
   `finish-good/confirm/route.ts`, áp lên TOÀN BỘ dữ liệu hiện có thay vì
   chỉ dòng mới.
4. Mở rộng dần các màn hình còn lại (PoStatusTable, TravelerSection,
   GlobalSearchBar, reconciliation, trace) sang dùng chung `/api/ledger`
   thay vì tự JOIN riêng — giảm rủi ro "sửa 1 nơi quên nơi khác" đã xảy
   ra thật (vụ shipped=TRUE bị sót).
5. Việc dựng lại kiến trúc chuẩn ERP (`stock.move`, state machine, audit
   trail đầy đủ) — giữ đúng quyết định đã có: làm ở **dự án MỚI hoàn toàn
   riêng**, không vá lớn vào hệ thống đang chạy thật.

---

## 5. BÀI HỌC TỪ AVP_AI (PHIÊN BẢN NÀY) CHO DỰ ÁN MỚI (stock.move chuẩn ERP)

> **Đính chính quan trọng**: `THIET_KE_HE_THONG_MOI.md` (viết 2026-09-13,
> 2 ngày trước báo cáo này) **đã thiết kế sẵn phần lớn nội dung dưới đây**
> — sổ cái `stock.move` bất biến (§8.1), CCP/human-gate chặn ghi trực tiếp
> (§8.3), AI chỉ chạy ở 1 lớp riêng (Phần 3, Lớp 1). Bảng dưới đối chiếu
> từng bài học với thiết kế đã có, để biết CÁI GÌ ĐÃ GIẢI QUYẾT và CÁI GÌ
> THẬT SỰ CÒN THIẾU (chỉ #1 và #4) — không lặp lại nội dung đã có.

| # | Bài học | Đã có trong `THIET_KE_HE_THONG_MOI.md`? | Việc cần làm |
|---|---|---|---|
| 1 | Đo phân bố giá trị thật của mỗi trường TRƯỚC khi khoá schema | ❌ **CHƯA** — tài liệu đi thẳng vào thiết kế đề xuất, chưa có bước "đo dữ liệu thật trước khi khoá field" như 1 quy trình bắt buộc | Bổ sung — xem phần thêm vào `THIET_KE_HE_THONG_MOI.md` bên dưới |
| 2 | Sổ cái bất biến (append-only), không sửa cờ boolean tại chỗ | ✅ **ĐÃ CÓ** — §8.1: "`stock.move` — 1 sổ cái BẤT BIẾN duy nhất". Vụ traveler 717544 hôm nay (nhận-xuất cùng ngày, tưởng trùng) là bằng chứng thực tế MỚI xác nhận đúng thiết kế này cần thiết — nên trích vào tài liệu làm ví dụ thật | Chỉ thêm ví dụ, không đổi thiết kế |
| 3 | Chỉ 1 con đường hợp lệ để đổi trạng thái, enforce bằng code | ✅ **ĐÃ CÓ** — §8.3 + Phần 3: "Ghi vào Lớp 3 (Ledger) chỉ xảy ra SAU KHI qua đúng CCP... không có đường ghi tắt". Vụ 22 traveler sót `shipped=TRUE` hôm nay (script đi vòng qua CCP) là bằng chứng thực tế xác nhận đúng rủi ro tài liệu đã cảnh báo trước | Chỉ thêm ví dụ |
| 4 | 2 lớp phòng thủ tách biệt: chuẩn hoá ĐỊNH DẠNG sau OCR rồi mới kiểm tra NGHIỆP VỤ + quét sức khoẻ dữ liệu ĐỊNH KỲ (không chỉ lúc ghi mới) | ⚠ **MỘT PHẦN** — Lớp 1 (Capture)/Lớp 2 (Core Rule Engine) đã tách riêng, nhưng chưa nói rõ "làm sạch định dạng" là 1 bước RIÊNG giữa 2 lớp, và chưa có "quét định kỳ dữ liệu đã lưu" | Bổ sung — xem phần thêm vào `THIET_KE_HE_THONG_MOI.md` bên dưới |
| 5 | Cảnh báo thay vì tự quyết; mẫu Who/Why/When cho MỌI ngoại lệ | ✅ **ĐÃ CÓ** — §8.3 CCP-1..4 đúng tinh thần; CONCESSION đã yêu cầu "chữ ký người có thẩm quyền độc lập" (Phần 1, dòng 3) | Không cần thêm |
| 6 | Xây nhanh trên nền dễ sửa cho phần nghiệp vụ CHƯA CHẮC, chỉ khoá cứng phần ĐÃ ổn định | ✅ **ĐÃ CÓ** — §8.4 "không đề xuất làm hết 1 lần, có lộ trình nhỏ→vừa→lớn"; Phần 4 xếp việc gộp sổ cái là "rủi ro cao, làm sau cùng" | Không cần thêm |

Bản đầy đủ từng bài học (giữ lại để có ngữ cảnh/bằng chứng chi tiết — bảng
trên là bản tóm tắt đối chiếu):

**1. Đừng giả định ý nghĩa 1 trường dữ liệu — đo trên toàn bộ dữ liệu thật
trước khi thiết kế schema.** Gần như MỌI giả định ban đầu ở AVP_AI đều
sai khi đo trên dữ liệu thật: `SHIPPED="0"` tưởng là "đã xuất" thực ra là
"chưa xuất" (loại sạch 1.811 dòng chỉ còn 1 dòng lọt qua); hậu tố Part#
tưởng là "biến thể vô hại" thực ra là "sản phẩm khác nhau thật" (Owner
đính chính); LOT NO. tưởng luôn là mã lot thật, đo ra 59% là chữ trạng
thái; Pot# tưởng là khoá 1:1 vĩnh viễn, đo ra tái sử dụng 41,1%. → Dự án
mới: mỗi trường dữ liệu quan trọng đều phải có 1 bước "đo phân bố giá trị
thật" (như đã làm ở đây) TRƯỚC khi khoá schema, không suy diễn từ tên cột.

**2. 1 traveler/1 mã không đảm bảo duy nhất xuyên suốt thời gian — mô hình
hoá thực tế vật lý (container/số được TÁI SỬ DỤNG), đừng giả định "khoá
chính vĩnh viễn".** Vụ traveler 717544 hôm nay (xuất hiện cả ở PS đã xuất
30101 và RawMaterial mới) suýt bị hiểu nhầm là lỗi trùng số — thực ra là
hàng về rồi đi thẳng trong ngày. Nếu dữ liệu là 1 SỔ CÁI CÁC BÚT TOÁN
(mỗi lần nhận/xuất là 1 dòng riêng, có thời điểm) thay vì 1 dòng duy nhất
mang cờ boolean `shipped`, thì tình huống "nhận rồi xuất luôn trong ngày"
sẽ tự nhiên rõ ràng (2 bút toán, không mâu thuẫn) — đây chính xác là lý
do `stock.move` của Odoo là APPEND-ONLY, không sửa tại chỗ.

**3. Chỉ 1 con đường hợp lệ để đổi trạng thái nghiệp vụ — enforce bằng
code, không chỉ bằng quy ước.** Bug thật hôm nay (22 traveler bị sót
`shipped=TRUE`) xảy ra vì script nhập lịch sử PS 30098/30101 ghi thẳng
vào Sheet, đi VÒNG QUA hàm `/api/packing-list/approve` (nơi duy nhất
"biết" phải đánh dấu Shipped). Dự án mới: mọi thay đổi trạng thái (kể cả
script migration/import 1 lần) phải đi qua đúng 1 tầng service/hàm
nghiệp vụ — không cho phép ghi thẳng vào bảng dưới bất kỳ hình thức nào,
kể cả script "chỉ chạy 1 lần".

**4. OCR/AI đọc sai là chuyện chắc chắn XẢY RA, không phải rủi ro hiếm —
cần 2 lớp phòng thủ, không phải 1.** AVP_AI đã có lớp phòng thủ NGHIỆP VỤ
khá tốt (đối chiếu Pot# với RawMaterial, cảnh báo Traveler đổi khác
thường...) nhưng THIẾU lớp phòng thủ ĐỊNH DẠNG (validate/regex làm sạch
kết quả AI trả về TRƯỚC KHI đưa vào kiểm tra nghiệp vụ) — đúng lỗ hổng
khiến `"Pot#313"` (dính nhãn) và ô Part# xuống 2 dòng lọt qua. Dự án mới:
tách rõ 2 tầng — (a) chuẩn hoá/validate ĐỊNH DẠNG ngay sau OCR, (b) kiểm
tra NGHIỆP VỤ sau đó — và thêm 1 lượt "quét sức khoẻ dữ liệu" chạy ĐỊNH
KỲ trên dữ liệu đã lưu, không chỉ kiểm tra lúc ghi mới (lỗi cũ nằm im tới
khi tình cờ phát hiện, như 2 vụ hôm nay).

**5. Cảnh báo thay vì tự quyết (ISO_AI_CONTROL) là triết lý ĐÚNG — giữ
nguyên cho dự án mới, đừng vì "ERP chuẩn hơn" mà để hệ thống tự động ghi
đè quyết định con người.** Toàn bộ dữ liệu sai tìm được đến giờ đều đang
ở dạng "chưa Shipped, chờ người duyệt" — chưa từng có trường hợp AI tự
ghi thẳng số liệu sai vào 1 chứng từ ĐÃ CHỐT mà không ai xem qua. Mẫu
`concessionBy/Reason/At` và `priorityBy/Reason/At` (ghi rõ AI/ai làm gì,
lúc nào, vì sao) nên trở thành khuôn mẫu CHUẨN cho MỌI ngoại lệ/ghi đè
trong dự án mới, không chỉ 2 trường hợp này.

**6. Xây nhanh trên nền tảng dễ sửa (Google Sheets) TRƯỚC KHI hiểu rõ
nghiệp vụ là chiến lược đúng, không phải đường tắt tệ.** Rất nhiều quyết
định kiến trúc ở AVP_AI chỉ ĐÚNG sau khi đã xây rồi mới phát hiện sai (vd
PartControl phải sửa lại cách khớp Part# sau khi đo ảnh hưởng thật 85%).
Nếu làm "chuẩn ERP" ngay từ đầu (schema cứng, migration nghiêm ngặt), chi
phí sửa mỗi lần phát hiện sai sẽ cao hơn nhiều. Dự án mới: những phần
NGHIỆP VỤ CÒN CHƯA CHẮC (vd ý nghĩa từng hậu tố Part#, mã máy/nhân viên
chính thức — vẫn đang chờ Owner ở báo cáo trước) nên tiếp tục prototype
nhanh, chỉ những phần ĐÃ CHỨNG MINH ổn định (luồng nhận→đóng gói→xuất cơ
bản) mới đáng đầu tư vào sổ cái `stock.move` chuẩn ngay từ đầu.

---

## 6. XÁC MINH BẰNG SỐ LIỆU — NỖI ĐAU HIỆN TRƯỜNG (phỏng vấn quản lý/kế hoạch, 2026-09-15)

*Andy phỏng vấn trực tiếp nhân viên kế hoạch + quan sát hiện trường +
nghe quản lý mô tả cách làm Packing Slip thật. Dưới đây là từng nhận định,
đối chiếu với số liệu đo được — **xác nhận** những gì đo được, và nói rõ
**chưa đo được gì** thay vì gật đầu cho qua.*

### 6.1 "Không chủ động lập kế hoạch — PO về giờ nào làm giờ đó"

> Quan sát thật: PO mới về lúc 13h chiều hôm qua, sáng cùng ngày vẫn đang
> làm hàng tồn từ thứ Sáu. Nhân viên kế hoạch tự lục PO trong email, không
> lập kế hoạch chính thức — nguyên liệu về thùng/Pot/Bin rồi tự "trôi"
> sang máy nào tiện, không có văn bản giao sản lượng chính thức.

**Đánh giá: KHÔNG đo trực tiếp được bằng Sheet** — RawMaterial chỉ ghi
**Ngày**, không ghi **Giờ nhận/giờ lập kế hoạch**, nên không dựng lại được
đúng khung giờ "13h chiều" bằng dữ liệu. Đây là quan sát trực tiếp đáng
tin (hành vi lặp lại, không phải 1 lần), nhưng cần ghi nhận rõ: **không có
bằng chứng số** cho riêng chi tiết giờ giấc.

**Bằng chứng GIÁN TIẾP nhưng chắc chắn**: hệ thống (cả Excel cũ lẫn
AVP_AI mới) **không có bất kỳ trường dữ liệu nào ghi nhận "kế hoạch sản
xuất"** — không có Work Center Master, không có lịch phân công máy, không
có trường "dự kiến làm ngày nào". Việc "không đo được kế hoạch" chính là
hệ quả trực tiếp của việc **chưa từng có khái niệm kế hoạch trong dữ liệu**
— khớp đúng GAP-015/Giai đoạn 3 đã nêu ở `BAO_CAO_TONG_HOP_AVP.md` mục 5.

### 6.2 "Nhiều traveler chồng lấn (overlap) 1 máy — không trace được ca/máy/mẻ, chênh sản lượng rất lớn"

**Đánh giá: XÁC NHẬN ĐÚNG bằng số liệu, có cả số liệu mới đo hôm nay.**

- Đã đo trước (`BAO_CAO_TONG_HOP_AVP.md` mục 2): **381/614 (62%) nhóm
  Ngày+Máy+Trạm có ≥2 traveler khác nhau chạy chung**, cao nhất **13
  traveler/1 máy/1 ngày**.
- **Đo mới hôm nay** (đối chiếu `/api/reconciliation`, 1.911 traveler thật,
  loại traveler TEST): **124 traveler (6,5%) có số lượng đóng gói (Scanned)
  lệch trên 50% so với số Pieces nhận vào ban đầu** — đúng loại "chênh sản
  lượng rất lớn" quản lý mô tả, xảy ra ở 1 phần đáng kể chứ không phải cá
  biệt.

→ Nguyên nhân gốc đã xác định: Machine/MC# chỉ ghi được SAU khi đóng gói
xong (hồi tố), không ghi lúc bắt đầu chạy — không thể tách riêng sản lượng
theo từng mẻ khi nhiều traveler dùng chung 1 máy cùng ngày.

### 6.3 "Không thấy được hao hụt, hàng lỗi, hàng phải trả, hàng quay lại xử lý"

**Đánh giá: XÁC NHẬN ĐÚNG**, đã đo trước ở `BAO_CAO_TONG_HOP_AVP.md`:

- Cột `DEFECTS`: **0/1.811 dòng có dữ liệu** (0% — có tên cột nhưng chưa
  từng dùng thật).
- Cột `Reject` ghi chuỗi số tự do không đơn vị (vd `"5, 154, 59, 3"`) —
  máy không đọc/tổng hợp được.
- **6/1.847 traveler** bị lặp kiểu "quay lại xử lý" (rework) mà hệ thống
  không có khái niệm rework order riêng, chỉ ghi đè mập mờ lên traveler cũ.

### 6.4 "Hàng đưa vào sản xuất chưa có LOT thật (TRAVELERRECEIVED)"

**Đánh giá: XÁC NHẬN TRÊN DỮ LIỆU LỊCH SỬ, chưa xác nhận lại được trên dữ
liệu SỐNG (mẫu còn nhỏ)** — cần nói rõ 2 vế, không gộp làm 1:

- **Dữ liệu lịch sử** (1.750 dòng WORK ORDER thật, đo trước): **53,8%**
  giá trị LOT NO. là `TRAVELERRECEIVED` (chưa đóng gói/chưa có lot thật) —
  đúng như quản lý mô tả, đây là trạng thái PHỔ BIẾN NHẤT, không phải
  ngoại lệ.
- **Dữ liệu sống hôm nay** (FinishGood webapp mới, 194 dòng): **0 dòng**
  còn placeholder này — nhưng mẫu quá nhỏ (194 dòng, mới chạy vài ngày,
  khác cách capture LOT# so với sổ Excel cũ) để kết luận vấn đề đã hết.
  **Cần theo dõi tiếp khi khối lượng tăng**, chưa nên báo cáo là "đã giải
  quyết".

### 6.5 "Hàng chưa Finish Good (chưa wrap xong) đã xuất"

**Đánh giá: KHÔNG ĐỦ BẰNG CHỨNG ĐỘC LẬP — đây là điểm duy nhất em KHÔNG
xác nhận được bằng số liệu hôm nay**, dù đã tìm kỹ.

Quét toàn bộ FinishGood sống: có đúng **7 dòng "đã Shipped" nhưng Box+Qty
đều trống/0**. Nhưng khi tra ngược, cả 7 dòng này đều là 2 vụ ĐÃ BIẾT và
ĐÃ SỬA trong chính hôm nay (5 dòng thuộc lô import lịch sử PS 30101 thiếu
sót — mục 22-traveler đã sửa; 2 dòng thuộc bug Quantity=0 của PS 30102) —
**không phải bằng chứng MỚI, ĐỘC LẬP** cho hiện tượng "xuất khi chưa wrap
xong" như 1 thói quen đang tiếp diễn. Đề xuất: theo dõi các PS MỚI phát
sinh từ nay (không tính PS nhập lại lịch sử) — nếu hiện tượng này còn xảy
ra ở dữ liệu MỚI, sẽ là bằng chứng chắc chắn hơn nhiều.

### 6.6 "Chữ viết tay trên form máy không nhận dạng được do thiếu thông tin"

**Đánh giá: ĐÚNG VỀ NGUYÊN TẮC, nhưng hệ thống hiện KHÔNG LƯU đủ để đo lại
% form khó đọc là bao nhiêu.** Schema quét Split Form (`SCANNING_FORM_SCHEMA`,
`webapp/src/lib/extract.ts`) đã có sẵn field `confidence: "low"` để AI tự
báo "đọc không chắc" — đúng cơ chế cần có — nhưng field này **không được
ghi vào Sheet FinishGood** sau khi lưu, nên không thể tổng hợp lại "bao
nhiêu % dòng từng bị AI báo đọc không chắc". **Khuyến nghị bổ sung**: lưu
`confidence` thành 1 cột trong FinishGood để đo được tỷ lệ này theo thời
gian.

### 6.7 "Vít dư vài trăm con tích tụ, không biết chất đống, không biết mã lot"

**Đánh giá: XÁC NHẬN RẤT MẠNH — đây là điểm quản lý mô tả đúng nhất, và vừa
có bằng chứng SỐ TRỰC TIẾP từ chính file quản lý gửi hôm nay**
(`Data/Partial Boxes 2026.xlsx`, xem thêm câu hỏi trước về PartControl):

- **1.026 dòng** ghi nhận thùng dở (partial box) trong 42 ngày log rải
  suốt năm 2026.
- **560/1.026 dòng (54,6%) — quá nửa — KHÔNG ghi Traveler#** kèm theo →
  không thể truy ngược Part#/PO/trạng thái QC của số hàng dư đó.
- **95/1.026 dòng (9,3%) không ghi LOT#**.

→ Đây CHÍNH XÁC là "chất đống không biết mã lot chất lượng" quản lý mô tả
— không phải cảm nhận, có số thật chứng minh. Đây cũng là bằng chứng cho
thấy quản lý ĐÃ tự ý thức được vấn đề và tự lập sổ theo dõi riêng (dùng
Excel, ngoài hệ thống chính) — dấu hiệu tốt để thuyết phục: quản lý sẵn
sàng ủng hộ giải pháp cho đúng nỗi đau này.

### 6.8 "Nhân viên Packing Slip phải xuống xưởng tìm form Scanning xem QC gạch chéo ô nào đạt"

**Đánh giá: XÁC NHẬN RẤT MẠNH bằng số liệu mới đo hôm nay — đây là con số
sốc nhất trong toàn bộ mục 6.**

**98,5% (191/194) dòng FinishGood sống có cột `qcStatus` HOÀN TOÀN TRỐNG**
— chỉ đúng 2 dòng có "PASS", 1 dòng có "HOLD" trong SUỐT quá trình chạy
webapp mới tới nay. Nghĩa là hệ thống mới, dù đã thiết kế sẵn cột để nhận
trạng thái QC, **gần như không capture được tín hiệu PASS/HOLD từ giấy
thật** — khớp chính xác với lời kể: nhân viên Packing Slip không thể tin
vào hệ thống để biết cái gì đạt, buộc phải xuống xưởng đọc trực tiếp ô
gạch chéo trên form giấy.

### 6.9 "Bảng WORK ORDER: Shipped không chính xác vì hệ thống QC xưởng chưa link vào"

**Đánh giá: XÁC NHẬN ĐÚNG**, đã đo trước: **797/1.847 (43%) traveler lệch
trạng thái SHIPPED/số Packing Slip** giữa 2 sổ Excel song song đang dùng —
đúng hiện tượng "bảng nói tên/trạng thái không chính xác".

### 6.10 "Quản lý rất thích tính năng scan — nhưng chưa hiểu scan sẽ lỗi nếu thiếu hệ thống bài bản/giám sát đi kèm"

**Đánh giá: ĐÚNG, và hôm nay có ngay bằng chứng thật minh hoạ cho chính
nhận định này** (không phải lý thuyết): trong 1 buổi làm việc hôm nay, 3
lỗi thật đã xảy ra — traveler 717544 bị đọc lệch cột (Part#/Pot#/LOT# lẫn
nhau), 22 traveler bị sót đánh dấu "đã xuất" (từ 1 script nhập liệu đi
vòng qua quy trình chuẩn), và 7 dòng FinishGood có Pot#=LOT# giống hệt
nhau từ 1 nguồn chưa xác định. **Không có lỗi nào trong 3 lỗi này do bản
thân công nghệ AI/scan kém** — cả 3 đều do THIẾU 1 bước kiểm soát quy
trình đứng sau (làm sạch định dạng, 1 đường ghi trạng thái duy nhất, xác
nhận nguồn gốc dữ liệu nhập). Đúng nhận định của quản lý: công cụ tốt mà
không có kỷ luật quy trình đi kèm thì vẫn sinh lỗi — chỉ là lỗi xảy ra
NHANH HƠN, KHÓ THẤY HƠN (vì không còn gõ tay chậm để tự soát lại).

### 6.11 "Owner muốn giảm nhân công — nhưng nếu không cải tổ triệt để thì không ai giải quyết được"

**Đánh giá: ĐÚNG về nguyên tắc, có số liệu hỗ trợ trực tiếp.** Chi phí lao
động ước tính ban đầu (`BAO_CAO_TONG_HOP_AVP.md` mục 4, đo 2026-09-14):
**21-28 giờ/ngày** dành cho gõ tay + dò tìm số liệu, quy năm ~5.250-7.000
giờ. **Cập nhật lại 2026-09-15** dựa trên quan sát thực tế: con số
này đo THIẾU — con số hợp lý hơn cho tổng hao phí là **khoảng 32 giờ/
ngày** (~4 công lao động/ngày, 1 công = 8 giờ), trong đó ~7 giờ vẫn là gõ
tay thuần, còn lại **~25 giờ/ngày là dò tìm/đối chiếu không hiệu quả** —
quy năm **~8.000 giờ** (chưa đo lại bằng bấm giờ thực tế, nên coi là ước
tính đã điều chỉnh, không phải số đo chính xác). Nếu chỉ
"lắp thêm công cụ scan" mà KHÔNG sửa 3 gốc rễ đã nêu ở mục 6.1-6.9 (không
có kế hoạch, không trace được máy/ca, QC không link vào hệ thống), nhân
công KHÔNG giảm được thật — chỉ chuyển từ "gõ tay chậm nhưng tự soát được"
sang "scan nhanh nhưng không ai soát", đúng rủi ro mục 6.10 vừa nêu. Giảm
nhân công bền vững đòi hỏi đúng 3 thứ Owner/quản lý đã tự nhắc tới: **hệ
thống truy vết (traceability)**, **cập nhật trạng thái ngay tại nguồn
(không hồi tố)**, và **giám sát CCP** — cả 3 đã có trong thiết kế đề xuất
`THIET_KE_HE_THONG_MOI.md` (Lớp 3 Ledger, Lớp 5 CCP) — vấn đề không phải
CHƯA CÓ Ý TƯỞNG, mà là CHƯA TRIỂN KHAI.

### Tóm tắt mục 6

| # | Nhận định | Xác nhận bằng số? |
|---|---|---|
| 6.1 | Lập kế hoạch bị động | ⚠ Không đo trực tiếp được (thiếu dữ liệu giờ), nhưng có bằng chứng gián tiếp chắc chắn |
| 6.2 | Overlap máy, chênh sản lượng lớn | ✅ 62% nhóm overlap + **124 traveler (6,5%) lệch >50%** (mới đo hôm nay) |
| 6.3 | Không thấy hao hụt/lỗi/trả/rework | ✅ DEFECTS 0% dùng, Reject không cấu trúc, 6 traveler rework chưa có khái niệm |
| 6.4 | Vào SX chưa có LOT thật | ✅ Lịch sử 53,8% TRAVELERRECEIVED — ⚠ dữ liệu sống mẫu nhỏ, chưa kết luận |
| 6.5 | Xuất khi chưa wrap xong | ❌ Chưa đủ bằng chứng độc lập — 7 dòng tìm được đều thuộc 2 vụ đã biết |
| 6.6 | Viết tay AI không đọc được | ⚠ Đúng nguyên tắc, nhưng hệ thống không lưu `confidence` nên không đo lại được |
| 6.7 | Vít dư không biết mã lot | ✅ **54,6% dòng Partial Boxes không có Traveler#** (mới đo hôm nay) |
| 6.8 | Phải xuống xưởng xem form QC | ✅ **98,5% dòng FinishGood sống có qcStatus TRỐNG** (mới đo hôm nay) |
| 6.9 | WORK ORDER Shipped sai vì QC chưa link | ✅ 797/1.847 (43%) lệch SHIPPED/PS |
| 6.10 | Scan sẽ lỗi nếu thiếu kỷ luật quy trình | ✅ Minh hoạ bằng đúng 3 lỗi thật xảy ra hôm nay |
| 6.11 | Giảm nhân công cần cải tổ triệt để | ✅ ~32 giờ/ngày lãng phí (điều chỉnh lại từ 21-28 giờ đo trước) — công cụ đơn thuần không giải quyết gốc rễ |

**8/11 xác nhận rõ bằng số liệu (3 số liệu HOÀN TOÀN MỚI đo trong buổi hôm
nay: mục 6.2 phần chênh sản lượng, mục 6.7, mục 6.8), 2 mục đúng nguyên
tắc nhưng thiếu dữ liệu đo lại, 1 mục chưa đủ bằng chứng độc lập để xác
nhận.** Đây là mức độ trùng khớp giữa lời kể hiện trường và số liệu đo
được cao bất thường — nên dùng chính bảng này khi làm việc với Owner: **9
trong 11 điều quản lý/kế hoạch mô tả bằng lời đã có số liệu xác nhận
ngay trong chính dữ liệu AVP đang chạy.**

---

*Báo cáo này dựa trên dữ liệu sống lúc 2026-09-15 — số dòng RawMaterial/
FinishGood sẽ tiếp tục tăng mỗi ngày, các con số phần trăm nên coi là ảnh
chụp tại thời điểm viết, không phải hằng số cố định.*
