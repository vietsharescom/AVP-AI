# BÁO CÁO TỔNG HỢP — HIỆN TRẠNG HỆ THỐNG SẢN XUẤT AVP SOLUTIONS
## (Gộp: Đánh giá hệ thống + So sánh ERP + Câu hỏi cần Owner + Bảng mã cần duyệt)

*Đối tượng đọc: Chủ doanh nghiệp / Ban điều hành AVP*
*Người thực hiện: Andy Phan (Viet), phân tích cùng Claude — dựa trên đọc
trực tiếp mã nguồn thật (`webapp/`), 2 file Excel vận hành thật
(`PACKING SLIPS.xlsm` — 1.853 dòng WORK ORDER, `DAILY LOG CHECK
SHEET.xlsm` — 1.811 dòng CHECKING SUMMARY), Google Sheets đang chạy thật
(RawMaterial 1.939 dòng), 4 form giấy thật, đối chiếu chuẩn ERP (Odoo 17
thật đang chạy) + ISO 9001 Cl.8.7.*
*Ngày: 2026-09-14 — gộp lại từ 4 tài liệu rời (`Bao_Cao_Danh_Gia_He_Thong_AVP`,
`SO_SANH_QUY_TRINH_AVP_VS_ERP`, `CAU_HOI_CHO_OWNER_THEO_CHUAN_ERP`,
`BANG_MA_CAN_OWNER_DUYET`) thành 1 bản đọc liền mạch.*

> **Nguyên tắc**: mọi con số trong báo cáo đều lấy từ dữ liệu THẬT (đọc
> trực tiếp file, chạy công thức thật, đối chiếu nhiều file với nhau) —
> không suy đoán. Chỗ nào là ước tính (thời gian lao động) ghi rõ giả
> định, không trình bày như số đo chính xác. Chỗ nào chưa có số liệu thật
> (vd vít dư trả Infasco) ghi rõ là CẦN Owner cung cấp, không tự bịa.

---

## MỤC LỤC

1. Bảng tổng hợp chính — Hiện trạng / Số liệu chứng minh / Giải pháp chi tiết
2. Bằng chứng riêng: chênh lệch hao hụt theo máy
3. AVP đang quản lý được bao nhiêu % nguồn lực chuẩn ERP
4. Chi phí lao động ước tính
5. Lộ trình triển khai — theo mức độ phụ thuộc
6. Câu hỏi cần Owner trả lời (rút gọn, ưu tiên cao → thấp)
7. Bảng mã cần Owner duyệt/bổ sung

---

## 1. BẢNG TỔNG HỢP CHÍNH — HIỆN TRẠNG / SỐ LIỆU CHỨNG MINH / GIẢI PHÁP CHI TIẾT

| # | Khâu quy trình (Input→Output) | Hiện trạng hiện nay (AVP đang làm) | Số liệu chứng minh (dữ liệu thật) | Giải pháp chi tiết — làm gì |
|---|---|---|---|---|
| 1 | **Nhận PO email Infasco → lập Traveler (RawMaterial)** | 100% gõ tay, chỉ là DỰ BÁO qua email — chưa phải hàng đã về kho thật. Không có bước Goods Receipt tách riêng. | 2 bản WORK ORDER lệch nhau thật (`PACKING SLIPS.xlsm` 1.853 dòng vs `DAILY LOG CHECK SHEET.xlsm` 1.811 dòng CHECKING SUMMARY) — **797/1.847 traveler lệch SHIPPED/PS** giữa 2 file (vd traveler 704978: file 1 nói PS 29979, file 2 nói PS 30072). | (1) Thêm bước **Goods Receipt** độc lập — chờ Owner trả lời Câu hỏi 2 (mục 6) trước khi code. (2) Không dùng 2 file Excel cũ làm "sự thật tuyệt đối" khi đối chiếu số AVP_AI trong giai đoạn chạy song song. |
| 2 | **Nhận nguyên liệu vật lý thật** — GAP LỚN NHẤT | **Không ghi nhận ở đâu cả.** Bản redesign khâu 2 trước đây bỏ hẳn form nhập tay, thay bằng bảng đối chiếu tự động Forecast↔Thành phẩm — vô tình xoá luôn bước xác nhận "đã về kho thật". Tab `Warehouse` được tạo đúng mục đích này nhưng đang "chết" (không ai ghi vào). | Quy trình thật (Owner xác nhận 2026-09-13): nhân viên kho scan barcode Pot# → tự động phân Pieces về đúng máy → sinh Split Form riêng cho máy đó. | Cần Owner trả lời 3 câu hỏi trước khi code (Giai đoạn 0, mục 5): ô nào trên Split Form là số nhận thật (Pieces hay Box QTY/"SL=PLT"), ai chụp lúc nào, lưu vào `Warehouse` hay thêm field `RawMaterial`. Sau đó: hồi sinh tab `Warehouse`, đổi công thức "nguyên liệu vào" ở Khâu 2 dùng số Goods Receipt thay vì RawMaterial dự báo. |
| 3 | **Đưa vào sản xuất — chạy trên máy/trạm** | **Không có trường nào ghi nhận traveler nào chạy máy nào, lúc nào, chạy bao lâu.** Cột Machine/MC# trong Scanning Sheet chỉ ghi được SAU khi đóng gói xong (hồi tố). | Đo trên 1.811 dòng CHECKING SUMMARY: **381/614 (62%) nhóm (Ngày+Máy+Trạm) có ≥2 traveler khác nhau chạy chung, cao nhất 13 traveler/máy/1 ngày** (xem chi tiết mục 2). Đây chính là lý do quản lý thấy "chênh lệch hao hụt quá lớn" — thực ra là do gộp nhiều mẻ chung 1 máy, không hẳn là hao hụt thật. | Xây **Work Center Master** (danh sách 25 mã máy từ sheet `Status`, chính thức hoá) + bắt đầu ghi giờ máy chạy/dừng (OEE) — xếp ở Giai đoạn 3 (cần thu thập dữ liệu mới, không chỉ code). Trước mắt: xác nhận với quản đốc xưởng máy dùng chung là BÌNH THƯỜNG (nhiều lô nhỏ nối tiếp) hay CẦN tách riêng để dễ truy vết hơn. |
| 4 | **QC/phân loại — Split Form giấy → gõ lại Scanning Sheet** | 100% trên giấy, số hoá thủ công. Cột `DEFECTS` tồn tại nhưng **0/1.811 dòng có dữ liệu** — chưa từng dùng thật. Cột `Reject` ghi chuỗi số ngăn dấu phẩy không đơn vị (vd thật `"5, 154, 59, 3"`) — máy không đọc được. | 19/20 cột Scanning Sheet gõ tay 100% (chỉ cột "Off" có công thức). | Thêm **Non-Conformance Record (NCR)** có cấu trúc: mã lỗi + số lượng + công đoạn phát hiện, tách khỏi cột Reject dạng chuỗi tự do. Thêm nhánh **CONCESSION** (ISO 9001 Cl.8.7) — hiện chỉ có PASS/HOLD, thiếu trạng thái "lỗi nhưng chấp nhận xuất có điều kiện, cần chữ ký duyệt". |
| 5 | **Vít dư đạt chất lượng (surplus tốt, không phải lỗi)** | **Không có trường nào trong toàn bộ schema** (FinishGood/PackingList) ghi nhận số dư đạt chuẩn giữ lại rồi gộp trả Infasco cuối tháng. | Theo quản lý: mỗi lần giao dư ~100-200 con/SKU, nhiều đơn cộng dồn — **hiện KHÔNG đo được** vì chưa có số liệu thật do Owner/quản lý cung cấp. | Thêm mã giao dịch riêng kiểu **RMA chiều ngược (hàng tốt dư thừa trả nhà cung cấp)**, tách khỏi Reject/Scrap — theo dõi tồn theo Part#, đối chiếu khi gộp trả cuối tháng. **Cần Owner/quản lý cung cấp số liệu ít nhất 1 tháng gần nhất trước khi thiết kế field.** |
| 6 | **Vít hư/lỗi (Scrap thật)** | Có 2 cột `scrapQty`/`scrapDetail` (thêm 2026-09-13, đọc từ bảng "Sorting Results/Defect" trên Split Form) — nhưng chỉ phủ lỗi phát hiện LÚC PHÂN LOẠI, có thể bỏ sót lỗi phát hiện lúc chạy máy. | Dữ liệu đã lưu được trong Sheet nhưng **chưa có màn hình tổng hợp Yield%** (= Thành phẩm tốt / Nguyên liệu vào theo Part#/máy/ca). | Giai đoạn 1 (làm ngay, rủi ro thấp): dựng 1 màn hình tính Yield% từ `scrapQty` đã có sẵn — chưa cần theo máy (vì Work Center Master chưa có ở mục #3). Xác nhận với Owner có nguồn scrap nào khác ngoài Split Form chưa được ghi. |
| 7 | **Đóng gói — gán LOT NO. / Pot# tái sử dụng** | 1 traveler có thể sinh nhiều LOT NO. (đúng chuẩn ERP: LOT = 1 Work Order). Nhưng Pot# và LOT NO. đang dùng LẪN 2 vai trò khác nhau trong cùng 1 trường. | **41,1% Pot#** (469/1.141) bị tái sử dụng cho ≥2 traveler khác nhau theo thời gian (chỉ 1:1 trong nhóm CÒN MỞ/chưa Shipped). **59% giá trị LOT NO.** thực ra là chữ trạng thái (`TRAVELERRECEIVED` 53,8%, `SORT&RETURN` 1,7%, `SPLIT FROM TR#` 3,6%), không phải mã lot thật. | Đã sửa code (`isLotPlaceholder`) loại 3 giá trị placeholder khỏi cảnh báo "LOT đổi khác thường" — xong. Còn thiếu: xây **vòng đời container (Pot#)** riêng biệt (nhận→dùng→rỗng→trả→nhận lại), tách khỏi vai trò "khoá lot" — xếp Giai đoạn 3. |
| 8 | **Traveler quay lại xử lý (rework/return)** — MỚI PHÁT HIỆN HÔM NAY | **Không có quy trình rõ ràng.** Traveler có thể xuất hiện 2 lần trong lịch sử với Part#/Pot#/PO khác nhau, hệ thống không phân biệt được đâu là bản ghi cuối/đúng. | **6/1.847 traveler thật** bị lặp kiểu này trong WORK ORDER (712880, 714061, 714057, 712883, 716969, 716970 — mẫu hình chung: dòng đầu LOT "SORT&RETURN", dòng sau có LOT thật). Đã lan sang RawMaterial đang chạy thật (1.939 dòng, cùng 6 traveler bị trùng do vừa import). | Thêm khái niệm **Rework Order** liên kết ngược về Work Order gốc (parent-child) thay vì dùng lại traveler# gốc ghi đè mập mờ. Trước mắt: **quyết định ngay** với 6 traveler đang trùng thật trong RawMaterial — giữ dòng có LOT thật (khuyến nghị) hay giữ cả 2 (xem Câu hỏi mục 6). |
| 9 | **Xuất hàng — Packing List/PS** | Packing Slip = 1 chuyến xe, không gắn 1 PO cụ thể (đúng, đã xác nhận). Xuất lẻ (partial shipment) đã code đúng chuẩn — tách dòng gốc thành phần đã xuất + phần còn lại, mỗi phần 1 PS# số thật. | 2 Packing Slip thật kiểm chứng: PS 30098 gộp 9 PO khác nhau, PS 30101 gộp 11 PO khác nhau. | Không có gap lớn ở khâu này — chỉ cần theo dõi thêm **container tái sử dụng** (Pot# rỗng, pallet) khi có Asset Master (Giai đoạn 3). |
| 10 | **Lưu trữ lịch sử (ARCHIVE) + Báo cáo KPI (SUMMARY IN&OUT)** | `ARCHIVE`/`ARCHIVE2` 100% gõ tay, KHÔNG link gì với WORK ORDER, trộn lẫn return/internal-move/sample không phân loại. `SUMMARY IN&OUT` đếm tay để báo cáo quản lý mỗi ngày. | Lệch PS: 25/1.468 traveler giữa ARCHIVE và WORK ORDER. **SUMMARY IN&OUT: 0/30 ngày mẫu khớp với WORK ORDER thật**, có ngày lệch tới 105 dòng. | Thêm field **loại giao dịch bắt buộc chọn** (chuẩn Odoo `stock.picking.picking_type_id`). Số KPI phải **tính tự động từ sổ cái thật**, không đếm tay — hệ quả nghiêm trọng nhất: quản lý đang ra quyết định dựa trên số liệu SAI mà không biết. |

---

## 2. BẰNG CHỨNG RIÊNG — CHÊNH LỆCH HAO HỤT THEO MÁY

Quản lý nói đúng: phần lớn "chênh lệch quá lớn" là do nhiều mẻ (traveler)
khác nhau chạy CHUNG 1 máy/trạm cùng lúc, không phải hao hụt vật lý thật.
Đo trực tiếp trên `CHECKING SUMMARY` (1.811 dòng thật, nhóm theo
Ngày + Machine + MC#):

| Chỉ số | Số liệu thật |
|---|---|
| Tổng số nhóm (Ngày, Máy, Trạm) | 614 |
| Số nhóm có **từ 2 traveler khác nhau trở lên** dùng chung | **381 (62%)** |
| Số traveler nhiều nhất dùng chung 1 máy/1 ngày | **13** |

Hệ thống hiện đối chiếu "nguyên liệu vào – thành phẩm ra" theo TỪNG
TRAVELER riêng lẻ (Khâu 2, `warehouse/list`). Khi 13 traveler cùng chạy 1
máy 1 ngày, sản lượng/hao hụt thật của máy đó bị "xé lẻ" gán nhầm cho
từng traveler theo tỷ lệ không rõ ràng. **Đây là lỗ hổng OEE (Overall
Equipment Effectiveness) — quản lý theo máy — mà AVP hiện chưa có.**

> **Ghi chú thêm 2026-09-14 (trao đổi trực tiếp với Andy, chưa xác minh
> bằng số liệu)**: nguyên nhân thực tế rất có thể là **PO/nguyên liệu mới
> có ngày về trễ tới 12h trưa** — công nhân "gối đầu" làm nốt phần còn
> lại của PO/traveler hôm trước trên CÙNG máy đó trong lúc chờ, rồi
> traveler mới nạp lên máy khi PO về buổi trưa. Đây có thể là cách vận
> hành HỢP LÝ, không phải lỗi ghi nhận — nhưng hệ thống hiện tại không có
> ranh giới rõ ràng "lúc nào máy chuyển từ traveler A sang traveler B",
> nên hao hụt vẫn bị tính lẫn giữa 2 lô. **Cần Owner/quản lý xác nhận**:
> việc gối đầu này do công nhân tự quyết khi thấy PO chưa về, hay có
> quản lý/kế hoạch chỉ định rõ mỗi sáng? Nếu xác nhận đúng, đây là 1 lợi
> ích cụ thể thêm của Goods Receipt (mục 1 hàng #2) — hiển thị "PO hôm
> nay đã về hay chưa" giúp kế hoạch/công nhân chủ động gối đầu thay vì tự
> đoán.

```
AVP hiện tại:     Traveler# ──► Part#/Pot#/Weight/Pieces ──► LOT ──► Ship
                  (1 trục duy nhất — không biết máy nào, lúc nào)

ERP/ISO chuẩn:    Traveler# ──► Part#/Pot#/Weight/Pieces ──► LOT ──► Ship
                       │
                       └──► Máy/Trạm ──► Giờ chạy ──► Sản lượng/Hao hụt
                            theo máy (OEE) ──► Chi phí/Bảo trì
```

---

## 3. AVP ĐANG QUẢN LÝ ĐƯỢC BAO NHIÊU % NGUỒN LỰC CHUẨN ERP?

ERP đúng nghĩa quản lý toàn bộ nguồn lực: Vật liệu, Máy móc, Nhân lực,
Tài chính, Chất lượng, Khách hàng.

| Nguồn lực (chuẩn ERP/Odoo) | AVP hiện tại | Bằng chứng |
|---|---|---|
| **Vật liệu (Material/Inventory)** | ⚠ Có, nhưng phân mảnh 2+ file, không đồng bộ | Mục 1 hàng #1, #10 |
| **Chất lượng (Quality)** | ❌ Có Ý ĐỊNH, thực thi thất bại | Cột DEFECTS 0% dùng, Reject không cấu trúc (mục 1 hàng #4) |
| **Máy móc/Thiết bị (Maintenance/OEE)** | ❌ Không có gì | Không 1 cột nào ghi giờ chạy/dừng máy trong toàn bộ 2 file (mục 1 hàng #3) |
| **Container/Tài sản tái sử dụng (Asset)** | ❌ Không có | Pot#/Pallet không track vòng đời (mục 1 hàng #7) |
| **Nhân lực (HR/Labor)** | ❌ Không có | Không file nào đo giờ công, năng suất nhân viên |
| **Tài chính/Chi phí (Costing)** | ❌ Không có | Không có cột chi phí, giá vốn nào trong toàn bộ 2 file |
| **Khách hàng (CRM)** | ❌ Không có (dù có hàng chục khách hàng thật — `PartControl.client`: Jobal, Thompson Fasteners, Ultra Form, Dana Canada...) | Không sheet nào quản lý theo khách hàng |
| **Báo cáo quản trị (BI/Reporting)** | ❌ Có làm nhưng SAI số liệu | Mục 1 hàng #10 |

**→ AVP hiện chỉ thực sự quản lý được ~1/8 nguồn lực chuẩn ERP** (Vật
liệu — và ngay phần này cũng lệch nhau giữa các file).

---

## 4. CHI PHÍ LAO ĐỘNG ƯỚC TÍNH

| Thành phần | Ước tính |
|---|---|
| Gõ tay WORK ORDER (10 cột × ~78,5 traveler/ngày) | ~2,2 giờ/ngày |
| Thu gom Split Form + cập nhật | ~0,5-0,75 giờ/ngày |
| Đối chiếu kho + gõ Scanning Sheet (~15 cột × 78,5) | ~3,3 giờ/ngày |
| Gõ Packing List + đếm Pallets/Empty | ~0,5 giờ/ngày |
| **Tổng gõ thuần** | **~6,5-7,3 giờ/ngày** |
| + Hệ số "mò tìm số liệu" (2-3 lần — dò VLOOKUP 3 tầng, giải mã chữ tự do, đối chiếu file lệch nhau) | **~21-28 giờ/ngày** (tổng, chia nhiều nhân viên) |
| **Quy năm (250 ngày làm việc)** | **~5.250-7.000 giờ/năm** — chỉ phần hành chính, CHƯA tính công lựa/đóng gói vật lý |

⚠ Đây là ước tính có giả định rõ ràng (tốc độ gõ 8-15 giây/ô, hệ số mò
2-3 lần) — nên đo thời gian thật (bấm giờ 1 ngày làm việc thật) trước khi
dùng số này thuyết trình chính thức.

---

## 5. LỘ TRÌNH TRIỂN KHAI — THEO MỨC ĐỘ PHỤ THUỘC (không phải theo độ khó)

*Nguyên tắc: việc nào cần Owner/Infasco trả lời trước thì làm trước, dù
nhỏ — vì mọi việc code phía sau đều bị chặn bởi nó (bài học từ chính
PartControl — làm rồi mới phát hiện 85% dựa trên giả định chưa xác minh).*

**Giai đoạn 0 — Owner/Infasco trả lời trước, KHÔNG code được nếu thiếu**
1. Ý nghĩa hậu tố Part# (`-L`/`-HT`/`-A`...) — ảnh hưởng 85% Part# đang chạy.
2. Ô nào trên Split Form là số nhận kho THẬT (Pieces hay Box QTY/"SL=PLT").
3. MC# là máy độc quyền hay trạm dùng chung (quản đốc xưởng trả lời ngay được).

**Giai đoạn 1 — Làm ngay được, rủi ro thấp, không phụ thuộc Giai đoạn 0**
- Màn hình Yield/hao phí theo Part# từ `scrapQty` đã có sẵn.
- Dashboard tổng quan tức thời (traveler đang HOLD, lệch >10%, đã xuất hôm nay).
- Sửa PartControl theo hướng Owner đã chọn (đã xong 2026-09-13).

**Giai đoạn 2 — Cần quyết định kiến trúc, phụ thuộc trực tiếp Giai đoạn 0**
- Thêm bước Goods Receipt (chụp Split Form lúc nhận, hồi sinh tab `Warehouse`).
- Sửa công thức "nguyên liệu vào" ở Khâu 2 dùng số Goods Receipt thay vì RawMaterial dự báo.

**Giai đoạn 3 — Cần THU THẬP dữ liệu mới ngoài đời, không chỉ code**
- Work Center Master (25 mã máy) + trạng thái AVAILABLE/DOWN/MAINTENANCE.
- Resource Master (danh sách mã nhân viên hợp lệ).
- Container/Asset Master (Pot#) + quy trình xác nhận trả Pot# rỗng.
- OEE (hiệu suất máy) — khó nhất, chưa từng tồn tại kể cả ở Excel cũ.

> **Chi tiết đề xuất "ma trận Máy × Part#" (trao đổi 2026-09-14)** — Andy
> nêu tình huống thật: 10 máy (1-10), mỗi máy tính chất khác nhau (lựa
> được part nhỏ/lớn khác nhau, công suất khác nhau, số người vận hành
> khác nhau, sự cố không biết máy nào ảnh hưởng); 1 PO có 6 traveler
> (A-F), có traveler cùng Part#, có traveler khác Part# — gọi đây là "ma
> trận trong sản xuất", nghi đây là lý do công ty chưa dám áp dụng hệ
> thống. Đề xuất giải quyết KHÔNG cần thuật toán tối ưu (APS) ngay từ
> đầu — chỉ cần 3 bảng dữ liệu đơn giản, con người vẫn quyết định, hệ
> thống chỉ hỗ trợ nhìn rõ + cảnh báo sai:
>
> 1. **Work Center Master (Bảng máy)**: mỗi máy 1 dòng — MC#, loại
>    size lựa được (nhỏ/lớn/cả hai), công suất (pcs/giờ), số người vận
>    hành cần, trạng thái hiện tại (Đang chạy/Dừng/Bảo trì) + lý do dừng.
> 2. **Part↔Machine Compatibility**: Part# (hoặc nhóm theo size) → danh
>    sách máy tương thích — trả lời "máy nào lựa loại nhỏ, máy nào lựa
>    loại lớn". Đây là phần duy nhất cần công sức nhập liệu ban đầu (1
>    lần, ít thay đổi sau đó).
> 3. **Traveler↔Machine Assignment (log thực tế)**: traveler nào chạy
>    máy nào, lúc nào — cột MC# đã có sẵn trong Scanning Sheet, chỉ cần
>    ghi NGAY LÚC BẮT ĐẦU chạy thay vì hồi tố sau khi đóng gói xong như
>    hiện tại.
>
> Với cách này: hệ thống KHÔNG tự động xếp máy cho 6 traveler A-F — nhân
> viên/kế hoạch vẫn chọn máy như bình thường, hệ thống chỉ tra bảng #2 để
> cảnh báo nếu chọn máy không tương thích Part# đó (bắt lỗi con người,
> không thay con người quyết định). Khi sự cố máy: đổi trạng thái máy đó
> ở bảng #1 kèm lý do — kế hoạch tra bảng #2 biết ngay máy tương thích
> nào khác đang rảnh để chuyển traveler qua, không cần đoán. Có thể bắt
> đầu chỉ với bảng #1 (quản đốc cung cấp 1 lần) — bảng #3 tự có từ việc
> ghi Scanning Sheet đang làm sẵn.

**Giai đoạn 4 — Nền tảng lớn, làm sau cùng, chỉ khi chắc chắn cần**
- Gộp RawMaterial/FinishGood/PackingList thành 1 sổ cái kiểu `stock.move`.
- Sổ cái "còn tồn chưa xuất" + mã truy vết `SCAN-{Traveler}-{n}`.
- Lưu ảnh/PDF gốc kèm mỗi dòng dữ liệu (truy vết có bằng chứng giấy tờ thật).
- Truy vết hàng loạt (`/trace` hiện chỉ 1 traveler/lần → nhận cả PO/Pot#).

---

## 6. CÂU HỎI CẦN OWNER TRẢ LỜI (rút gọn, ưu tiên cao → thấp)

| # | Câu hỏi | Vì sao ưu tiên | Ví dụ dễ hiểu |
|---|---|---|---|
| ~~✅~~ | ~~Hậu tố Part# là sản phẩm khác nhau hay biến thể vô hại?~~ | **ĐÃ CHỐT 2026-09-13** — sản phẩm khác nhau thật, đã sửa code | `11546437-MY-AK-L` giờ báo đúng "thiếu trong PartControl" |
| 1 | Ý nghĩa CỤ THỂ từng hậu tố Part# (`-L`, `-HT`, `-A`, `-T`, `-B`, `-X`...)? | Cần để mô tả đúng khi bổ sung 36 Part# thiếu (mục 7.2) | — |
| 2 | Ô nào trên Split Form là số nhận kho THẬT — Pieces hay Box QTY? ("SL=PLT" là gì?) | Chặn toàn bộ việc sửa công thức đối chiếu Khâu 2 | Phiếu traveler 717365: ô Pieces để trống, ô Box QTY ghi 31.000 — không biết lấy số nào |
| 3 | Đồng ý hồi sinh tab `Warehouse` để ghi "đã về kho thật" không? | Quyết định kiến trúc, ảnh hưởng nhiều code | Sheet có sẵn 1 sổ tên "Warehouse" nhưng từ lúc redesign không ai ghi vào nữa |
| 4 | **[MỚI]** 6 traveler bị trùng 2 bản ghi khác nhau (mục 1 hàng #8) — giữ dòng nào? | Đang tồn tại thật trong RawMaterial đang chạy (1.939 dòng) | 712880: dòng 1 LOT "SORT&RETURN", dòng 2 LOT thật `6-244-20-A` — cùng traveler, khác Pot#/PO |
| 5 | **[MỚI]** Vít dư đạt chuẩn (~100-200 con/SKU/lần giao) — có số liệu tháng gần nhất không? | Chưa đo được gì — cần số thật để thiết kế field theo dõi | Cuối tháng gộp trả Infasco — hiện không ai biết tổng đang "treo" bao nhiêu |
| 6 | **[MỚI]** Máy/trạm dùng chung nhiều traveler (mục 2) — bình thường hay nên tách? | Quyết định có cần OEE theo máy ngay hay không | 13 traveler/máy/1 ngày — nhiều lô nhỏ xen kẽ hay lỗi ghi nhận? |
| 7 | Traveler 717671 — có chứng từ giấy gốc để đối chiếu không? | Quyết định có phải sửa/xoá dữ liệu thật trong Sheet | 1 traveler, 2 dòng thành phẩm LOT gần giống nhau (`6-253-13-A`/`6-251-13-A`) |
| 8 | MC# là máy độc quyền hay trạm dùng chung? | Ảnh hưởng độ tin cậy cảnh báo "trùng máy" | MC#078 cùng ngày cùng ca gắn 2 traveler khác nhau |
| 9 | finishedPartNo hay partNo (đầu form) — cái nào ưu tiên dùng thật? | Owner nói sẽ hỏi Infasco — chưa có trả lời | Đầu phiếu ghi 1 Part#, sticker đóng gói in Part# khác |
| 10 | Đọc thêm Serial#/SSCC (mã từng box) vào hệ thống? | Hoàn thiện truy vết tới mức box, chuẩn GS1 | Nhãn đóng gói đã in sẵn dải serial liên tục — chỉ cần đọc thêm |
| 11 | 36 Part# đang hoạt động thiếu Quantity/box — ai điền? | Chặn tính QUANTITY cho các Part# này | Xem danh sách đủ ở mục 7.1 |
| 12 | **[MỚI 2026-09-15]** 2 nguồn Part# lệch hậu tố (RawMaterial lúc nhận email vs FinishGood lúc scan) — khi làm Packing List, ưu tiên nguồn nào? | Đang chọn RawMaterial-hay-FinishGood tuỳ ý (ưu tiên FinishGood hiện tại) — sai thì lấy nhầm sản phẩm khác (hậu tố = sản phẩm khác nhau thật, đã chốt mục trên) | Traveler 718064: RawMaterial ghi Part# `7151143403-L`, nhưng FinishGood lúc scan chỉ ghi `7151143403` (mất hậu tố `-L`) — PartControl không khớp được, Quantity không tự tính ra |

---

## 7. BẢNG MÃ CẦN OWNER DUYỆT/BỔ SUNG

### 7.1 Mã Part# — 36 mã đang hoạt động thật, cần Owner cung cấp Quantity/box

Xác nhận 2026-09-13: mỗi hậu tố là 1 sản phẩm riêng, không dùng chung
Quantity/box với mã gốc. Cột "Gợi ý" chỉ để tham khảo tốc độ điền —
KHÔNG tự dùng nếu Owner chưa xác nhận 2 mã thực sự cùng Quantity/box.

| Part# cần thêm | Số traveler đang dùng | Quantity/box mã GỐC (chỉ tham khảo) |
|---|---|---|
| `11546437-MY-AK-L` | 20 | gốc `11546437` = 1500 |
| `11546367-MY-AK-L` | 15 | gốc `11546367` = 700 |
| `11546370-IN-PN-T` | 15 | gốc `11546370` = 220 |
| `11549204-T` | 11 | gốc `11549204` = 210 |
| `06104388AA-HT` | 8 | gốc `06104388AA` = 660 |
| `11546462-A` | 8 | gốc `11546462` = 650 |
| `11602326-HT` | 8 | gốc `11602326` = 120 |
| `40161567-HT` | 8 | gốc `40161567` = 175 |
| `06509967AA-HT` | 7 | gốc `06509967AA` = 825 |
| `11546370-MY-AK-L` | 5 | gốc `11546370` = 220 |
| `11546389-T` | 4 | gốc `11546389` = 4500 |
| `11561645-HT` | 4 | gốc `11561645` = 4000 |
| `11609826-X` | 4 | gốc `11609826` = 175 |
| `11610091-MY-AK-T` | 4 | gốc `11610091` = 2500 |
| `W711953-S900-A` | 4 | gốc không có sẵn — **mã hoàn toàn mới** |
| `11511672AA-HT`* | 3 | gốc `11511672AA`* = 1400 |
| `11502717-TH-TC-T` | 3 | gốc `11502717` = 1300 |
| `11546566-HT` | 3 | gốc `11546566` = 1800 |
| `11602292-A` | 3 | gốc `11602292` = 1000 |
| `HL3W-4320-AA-L` | 3 | gốc không có sẵn — **mã hoàn toàn mới** |
| `06508200AA-HT` | 2 | gốc không có sẵn — **mã hoàn toàn mới** |
| `06511637AA-A` | 1 | gốc `06511637AA` = 1200 |
| `06512349AA-HT` | 2 | gốc `06512349AA` = 400 |
| `06513555AA-HT` | 2 | gốc `06513555AA` = 440 |
| `11514596-CA-IN-B` | 2 | gốc `11514596` = 2500 |
| `11546703-HT` | 2 | gốc `11546703` = 4000 |
| `11549170-L` | 2 | gốc `11549170` = 400 |
| `40110160-L` | 2 | gốc `40110160` = 200 |
| `1K2GZ0-00-A` | 2 | gốc không có sẵn — **mã hoàn toàn mới** |
| `BL3W-4320-BA-L` | 2 | gốc không có sẵn — **mã hoàn toàn mới** |
| `W520113-S440-L` | 2 | gốc không có sẵn — **mã hoàn toàn mới** |
| `11549168-CA-IN-B` | 1 | gốc `11549168` = 1000 |
| `11610128-BR-HA-HT` | 1 | gốc `11610128` = 1125 |
| `11611960-HT` | 1 | gốc `11611960` = 8000 |

*(`11511672AA-HT` — kiểm tra lại chính tả, có thể là `06511672AA-HT`, cần đối chiếu chứng từ gốc.)*

Loại khỏi danh sách: `TEST-PART` (7 traveler) — dữ liệu test nội bộ.

**Đề xuất duyệt**: Owner chỉ cần xác nhận Quantity/box thật cho mỗi dòng
— không cần làm hết 1 lần, ưu tiên theo cột "Số traveler đang dùng"
(nhiều nhất trước).

### 7.2 Mã hậu tố Part# — chưa biết ý nghĩa từng loại

```
-L        -HT       -A        -T        -B        -X
-MY-AK    -CA-IN    -IN-PN    -TH-TC    -BR-HA
```

### 7.3 Mã máy (Machine/MC#)

Nguồn: sheet `Status` trong `PACKING SLIPS.xlsm`:
- **BF 102 → BF 112** (11 mã)
- **MC 4 → MC 112** (nhiều mã, không liên tục — cần Owner xác nhận đủ dải)
- **TABLE 1, TABLE 2**
- **PCKY**

→ Tổng 25 mã hợp lệ theo nguồn này — cần Owner xác nhận còn đúng/đủ không.

### 7.4 Mã nhân viên (Operator code)

Chưa có danh sách chính thức — chỉ quan sát được trên chứng từ thật:

```
275   290   279   391   292   285   275   157   297   280
```

Owner cần cung cấp danh sách mã nhân viên hợp lệ kèm tên thật (export từ
hệ chấm công nếu có).

### 7.5 Mã thiết bị QC (Hardness/Tensile/Proof Load)

Khu FINAL INSPECTION trên Split Form KHÔNG có ô mã máy/thiết bị, chỉ có
"Stamp & Date" — khác khu Sorting Results có ghi rõ "Sorting M/C#:
MC078". Chưa biết phòng QC có đánh số máy đo riêng ở đâu khác không.

### 7.6 Quy ước LOT NO.

| Dạng | Ý nghĩa |
|---|---|
| `6-DDD-SS-L` (vd `6-247-03-b`) | Mã lot thật — cấp bởi Infasco (đôi khi giao trễ) |
| `TRAVELERRECEIVED` | Traveler đã nhận, ĐANG CHỜ Infasco cấp LOT# — không phải lỗi |
| `SORT&RETURN` | Hàng trả lại, không ra lot |
| `SPLIT FROM TR#{traveler}` | Ghi chú tách từ traveler khác |

Cần Owner xác nhận: cấu trúc `6-DDD-SS-L` — số "6" cố định là gì (mã nhà
máy/dòng sản phẩm?), "DDD" có phải ngày trong năm không?

### 7.7 Mã Serial#/SSCC (mức thùng/carton)

Mỗi carton có 1 số serial liên tục (vd `...19929` → `...19964` cho đúng
36 carton) — chuẩn GS1 SSCC, đề xuất thêm vào hệ thống (Giai đoạn 4).

---

## KẾT LUẬN

1. **Không phải "làm chưa tới" mà là "nhiều nơi tự làm, không ai đồng
   bộ"** — bằng chứng: 797 traveler lệch giữa 2 file lịch sử, SUMMARY
   IN&OUT sai 0/30 ngày khớp — lỗi ĐÃ XẢY RA trong dữ liệu thật, đang
   dùng để ra quyết định mỗi ngày.
2. **Thiếu hẳn trục "theo máy"** (mục 2) là lý do gốc khiến không truy
   vết được thất thoát chất lượng đến từng công đoạn, và chênh lệch hao
   hụt nhìn "quá lớn" giả tạo do gộp nhiều traveler chung 1 máy.
3. **Vít dư đạt chuẩn và vít hư không có nơi ghi nhận tách biệt** khỏi
   luồng traveler chính — dễ thất lạc khi gộp trả Infasco cuối tháng.
4. **7/8 nguồn lực ERP chuẩn chưa được quản lý** — công ty vận hành như
   xưởng thủ công ghi sổ dù quy mô thật đã tương đương doanh nghiệp cần
   ERP thật sự.
5. **AVP_AI (dự án đang xây) đã giải quyết 1 phần đáng kể** các lỗi cụ
   thể ở trên — nhưng để giải quyết TOÀN DIỆN, cần Owner trả lời các câu
   hỏi Giai đoạn 0 (mục 6) trước, theo đúng lộ trình phụ thuộc (mục 5).

---

*Gộp từ 4 tài liệu 2026-09-13/14: `Bao_Cao_Danh_Gia_He_Thong_AVP`,
`SO_SANH_QUY_TRINH_AVP_VS_ERP`, `CAU_HOI_CHO_OWNER_THEO_CHUAN_ERP`,
`BANG_MA_CAN_OWNER_DUYET`. Chi tiết kỹ thuật đầy đủ hơn (bảng quan hệ dữ
liệu/cardinality, sơ đồ truy vết 2 chiều): `BRS_TRS.md`, `ARCHITECTURE.md`.
Mọi số liệu đối chiếu đều chạy trực tiếp trên file/Sheet thật, có thể tái
kiểm chứng lại bất cứ lúc nào.*
