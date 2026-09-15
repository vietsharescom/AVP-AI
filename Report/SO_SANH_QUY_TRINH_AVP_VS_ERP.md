# SO SÁNH QUY TRÌNH: AVP HIỆN TẠI vs. CHUẨN ERP/ISO

*Đối tượng đọc: Chủ doanh nghiệp / Ban điều hành AVP*
*Mục đích: cho thấy rõ vì sao hệ thống hiện tại CHƯA truy vết được thất
thoát chất lượng, CHƯA theo dõi được chênh lệch hao hụt theo từng máy —
so với cách 1 hệ thống ERP/ISO chuẩn sẽ làm.*
*Ngày: 2026-09-14*

> **Nguyên tắc**: mọi số liệu trong báo cáo này lấy từ dữ liệu THẬT — đọc
> trực tiếp `PACKING SLIPS.xlsm` (1.853 dòng WORK ORDER), `DAILY LOG CHECK
> SHEET.xlsm` (1.811 dòng CHECKING SUMMARY), và Google Sheets đang chạy
> thật (RawMaterial 1.939 dòng). Không suy đoán số liệu nghiệp vụ — chỗ
> nào chưa rõ ý nghĩa, ghi rõ là câu hỏi cần Owner xác nhận, không tự
> đoán. Đây là bản mở rộng của `BAO_CAO_DANH_GIA_HE_THONG.md` (2026-09-13)
> — tập trung riêng vào 2 điểm Owner/quản lý mới nêu ra: **truy vết thất
> thoát chất lượng** và **chênh lệch hao hụt theo từng máy**.

---

## 1. Bảng so sánh chính — theo từng khâu quy trình (Input → Output)

| # | Quy trình (Input→Output) | AVP đang làm hiện nay | ERP/ISO chuẩn nên làm | Bằng chứng thật |
|---|---|---|---|---|
| 1 | **Nhận nguyên liệu (Pot#) từ Infasco** | Ghi vào RawMaterial (dự báo qua email) — traveler#, Part#, Pot#, Weight, Pieces, PO, Date. Không có bước xác nhận "đã nhận thật" tách riêng (GAP-015, chưa làm). | **Goods Receipt** độc lập: cân/đếm thật lúc nhận, có mã người nhận, thời gian nhận, đối chiếu tự động với forecast — sai lệch >X% tự chặn, không cho vào kho ảo. | Tab `Warehouse` đã tạo (đúng ý định) nhưng đang "chết" — chưa có nút nhập số thật. |
| 2 | **Đưa vào sản xuất — chạy trên máy/trạm** | **KHÔNG có trường nào ghi nhận traveler nào chạy trên máy nào, lúc nào, chạy bao lâu.** Cột `Machine`/`MC#` trong Scanning Sheet chỉ ghi được SAU khi đóng gói xong (hồi tố), không phải lúc đang chạy. | **Work Order Routing theo máy** (WIP tracking): mỗi lệnh sản xuất gắn 1 máy/trạm cụ thể, có giờ bắt đầu/kết thúc, sản lượng theo ca — cho phép tính **hao hụt/hiệu suất theo từng máy** (OEE), không chỉ theo traveler. | Xem mục 2 bên dưới — **62% số nhóm (ngày, máy, trạm) có ≥2 traveler khác nhau chạy chung**, cao nhất 13 traveler/máy/ngày. |
| 3 | **QC/phân loại — Split Form giấy** | 100% trên giấy, số hoá thủ công sau đó. Cột `DEFECTS` tồn tại trong Scanning Sheet nhưng **0/1.811 dòng có dữ liệu** — chưa từng dùng thật. Cột `Reject` ghi chuỗi số ngăn dấu phẩy không đơn vị (vd `"5, 154, 59, 3"`), máy không đọc được. | **Non-Conformance Record (NCR)** có cấu trúc: mã lỗi + số lượng + công đoạn phát hiện, tách riêng khỏi sản lượng tốt — truy vết được thất thoát chất lượng theo mã lỗi, theo máy, theo Part#, theo thời gian. | `BAO_CAO_DANH_GIA_HE_THONG.md` mục 1 hàng #4 (đã đo trên dữ liệu thật). |
| 4 | **Vít dư đạt chất lượng (surplus tốt, không phải lỗi)** | **KHÔNG có trường nào trong toàn bộ schema** (FinishGood/PackingList) ghi nhận số lượng dư đạt chuẩn giữ lại rồi gộp trả Infasco cuối tháng. Theo quản lý: mỗi lần giao dư ~100-200 con/SKU, nhiều đơn cộng dồn — hiện **không ai theo dõi được tổng số đang "treo"**. | **Return Merchandise (RMA) — chiều ngược, hàng tốt dư thừa**: 1 mã giao dịch riêng (khác Reject/Scrap), theo dõi tồn theo Part#, đối chiếu khi gộp trả cuối tháng — biết chính xác đang giữ bao nhiêu, đã trả bao nhiêu, còn thiếu bao nhiêu. | Quan sát mới 2026-09-14 (qua trao đổi Andy-quản lý) — **CHƯA đo được số liệu thật, cần Owner/quản lý cung cấp số cụ thể ít nhất 1 tháng gần nhất để định lượng**. |
| 5 | **Vít hư/lỗi (Scrap thật)** | Có 2 cột `scrapQty`/`scrapDetail` (thêm 2026-09-13, đọc từ bảng "Sorting Results/Defect" trên Split Form) — nhưng đây là lỗi phát hiện lúc PHÂN LOẠI, có thể KHÔNG phải toàn bộ scrap thực tế (vd lỗi phát hiện lúc chạy máy không qua bảng này thì không ghi được). | **Scrap/Yield tracking** gắn với từng công đoạn (không chỉ lúc phân loại cuối) — mỗi công đoạn có hệ số hao hụt chuẩn (standard scrap rate), chênh lệch vượt ngưỡng tự động cảnh báo, quy ra chi phí. | 2 cột đã thêm là bước đúng hướng nhưng mới phủ 1 điểm trong quy trình, cần xác nhận với Owner có nguồn scrap nào khác chưa được ghi. |
| 6 | **Đóng gói — gán LOT NO. (~Work Order)** | 1 traveler có thể sinh nhiều LOT NO. — đã xác nhận đúng theo mô hình ERP (LOT = 1 Work Order). Nhưng **41,1% Pot# bị tái sử dụng** qua nhiều traveler khác nhau (theo thời gian) và **59% giá trị LOT NO. thực ra là CHỮ TRẠNG THÁI** (`TRAVELERRECEIVED`, `SORT&RETURN`...), không phải mã lot thật. | Container (Pot#) có **vòng đời riêng** (nhận→dùng→rỗng→tái sử dụng) tách biệt khỏi Lot/Work Order — không dùng chung 1 trường cho cả "định danh vật lý" lẫn "trạng thái quy trình". | `BRS_TRS.md` mục 3b (đo trên toàn bộ 1.853 dòng WORK ORDER thật). |
| 7 | **Traveler quay lại xử lý (rework/return)** | **Không có quy trình rõ ràng.** Phát hiện thật hôm nay: traveler có thể xuất hiện **2 lần trong lịch sử** với Part#/Pot#/PO khác nhau (vd traveler 712880: lần 1 Pot#751/PO192852/LOT "SORT&RETURN", lần 2 Pot#1359/PO193803/LOT thật) — hệ thống hiện KHÔNG phân biệt được đâu là bản ghi cuối/đúng. | **Rework Order** liên kết ngược về Work Order gốc (parent-child), có trạng thái riêng (PENDING REWORK/RETURNED TO WIP/SCRAPPED) — không dùng lại traveler# gốc để ghi đè mập mờ. | 6/1.847 traveler thật bị lặp kiểu này trong WORK ORDER (712880, 714061, 714057, 712883, 716969, 716970) — đã lan sang RawMaterial đang chạy thật (1.939 dòng, cùng 6 traveler bị trùng). |
| 8 | **Xuất hàng — Packing List/PS** | Traveler xuất lẻ 2 lần ghi bằng cách sửa chữ tay thành chuỗi (`"29577/30018"`), không phải số. | **Partial shipment** có cấu trúc: 1 dòng gốc tách thành N dòng con, mỗi dòng 1 PS# riêng dạng số, tổng luôn khớp dòng gốc. | `BAO_CAO_DANH_GIA_HE_THONG.md` mục 1 hàng #5. |
| 9 | **Báo cáo KPI cho quản lý** | `SUMMARY IN&OUT` đếm tay, **0/30 ngày mẫu khớp với WORK ORDER thật**, lệch tới 105 dòng/ngày. | Số liệu KPI tính **tự động** từ sổ cái thật, không đếm tay — quản lý luôn thấy số đúng thời gian thực. | `BAO_CAO_DANH_GIA_HE_THONG.md` mục 1 hàng #7. |

---

## 2. Bằng chứng định lượng mới — chênh lệch hao hụt theo máy

Quản lý nói đúng: **chênh lệch (hao hụt) quá lớn phần lớn là do nhiều mẻ
(traveler) khác nhau chạy CHUNG 1 máy/trạm cùng lúc**, không phải hao hụt
vật lý thật. Đo trực tiếp trên `CHECKING SUMMARY` (1.811 dòng thật, nhóm
theo Ngày + Machine + MC#):

| Chỉ số | Số liệu thật |
|---|---|
| Tổng số nhóm (Ngày, Máy, Trạm) | 614 |
| Số nhóm có **từ 2 traveler khác nhau trở lên** dùng chung | **381 (62%)** |
| Số traveler nhiều nhất dùng chung 1 máy/1 ngày | **13** |

**Hệ quả**: hệ thống hiện tại đối chiếu "nguyên liệu vào – thành phẩm ra"
theo TỪNG TRAVELER riêng lẻ (Khâu 2, `warehouse/list`). Khi 13 traveler
cùng chạy 1 máy 1 ngày, sản lượng/hao hụt thật của máy đó bị "xé lẻ" gán
nhầm cho từng traveler theo tỷ lệ không rõ ràng — khiến chênh lệch nhìn
"quá lớn" ở traveler này, "quá nhỏ" ở traveler khác, dù tổng thể của máy
có thể vẫn hợp lý. **Đây chính là lỗ hổng "không theo dõi được hao hụt
theo từng máy" mà 1 hệ thống ERP/ISO chuẩn phải có (OEE — Overall
Equipment Effectiveness) mà AVP hiện chưa có.**

---

## 3. Tổng kết — AVP đang quản lý theo Traveler#, ERP/ISO cần quản lý theo CẢ 2 trục

```
AVP hiện tại:     Traveler# ──► Part#/Pot#/Weight/Pieces ──► LOT ──► Ship
                  (1 trục duy nhất — không biết máy nào, lúc nào)

ERP/ISO chuẩn:    Traveler# ──► Part#/Pot#/Weight/Pieces ──► LOT ──► Ship
                       │
                       └──► Máy/Trạm ──► Giờ chạy ──► Sản lượng/Hao hụt
                            theo máy (OEE) ──► Chi phí/Bảo trì
```

Thiếu hẳn **trục thứ 2** (theo máy/thiết bị) là lý do gốc rễ khiến:
- Không truy vết được thất thoát chất lượng đến từng công đoạn/máy cụ thể
  (chỉ biết traveler nào có Reject, không biết máy nào gây ra).
- Chênh lệch hao hụt nhìn có vẻ lớn nhưng thực ra là do gộp nhiều traveler
  chung 1 máy, không phải hao hụt thật.
- Vít dư đạt chuẩn và vít hư không có nơi ghi nhận tách biệt khỏi luồng
  traveler chính — dễ thất lạc số liệu khi gộp trả Infasco cuối tháng.

---

## 4. Câu hỏi cần Owner/quản lý xác nhận trước khi thiết kế tiếp

1. Vít dư đạt chuẩn (~100-200 con/SKU/lần giao) — có số liệu tháng gần
   nhất để đối chiếu không (bao nhiêu SKU, tổng bao nhiêu con, đã trả
   Infasco đợt nào chưa)?
2. Máy/trạm dùng chung nhiều traveler (mục 2) — đây là **cách vận hành
   bình thường** (nhiều lô nhỏ xen kẽ cùng máy) hay **nên tách riêng** để
   dễ truy vết hơn?
3. 6 traveler bị lặp 2 bản ghi khác nhau (mục 1 hàng #7) — dòng nào là
   đúng/cuối cùng, xem chi tiết trong hội thoại phiên làm việc hôm nay?

---

*Bản mở rộng của `BAO_CAO_DANH_GIA_HE_THONG.md` (2026-09-13). Số liệu mục
2 chạy trực tiếp trên `DAILY LOG CHECK SHEET.xlsm` (sheet `CHECKING
SUMMARY`), có thể tái kiểm chứng lại bất cứ lúc nào.*
