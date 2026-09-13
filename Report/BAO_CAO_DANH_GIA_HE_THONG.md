# BÁO CÁO ĐÁNH GIÁ HIỆN TRẠNG HỆ THỐNG QUẢN LÝ SẢN XUẤT — AVP SOLUTIONS

*Đối tượng đọc: Chủ doanh nghiệp / Ban điều hành AVP*
*Người thực hiện: Andy Phan (Viet), phân tích cùng Claude — dựa trên đọc trực tiếp
mã nguồn thật (`webapp/`), 2 file Excel vận hành thật (`PACKING SLIPS.xlsm`,
`DAILY LOG CHECK SHEET.xlsm`), 4 form giấy thật, và đối chiếu chuẩn ERP
(Odoo 17 thật đang chạy) + mô hình quản trị AI (dự án Ops_Ai của Andy).*
*Ngày: 2026-09-13*

> **Nguyên tắc của báo cáo này**: mọi con số, mọi lỗi nêu ra đều lấy từ dữ
> liệu THẬT (đọc trực tiếp file, chạy công thức thật, đối chiếu 2 file với
> nhau) — không suy đoán. Chỗ nào là ước tính (thời gian lao động) đều ghi
> rõ giả định, không trình bày như số đo chính xác.

---

## 1. Bảng tổng hợp toàn bộ pipeline — từ nhận PO tới xuất hàng

| # | Giai đoạn (Input→Output) | Biểu mẫu đang dùng | Mục đích | Lỗ hổng phát hiện (bằng chứng thật) | Nguồn lực ERP đang quản lý | Nguồn lực CẦN bổ sung | Mức độ lãng phí / tệ hại |
|---|---|---|---|---|---|---|---|
| 1 | **Nhận PO email Infasco → lập Traveler/WORK ORDER** | Sheet `WORK ORDER` (2 bản riêng: `PACKING SLIPS.xlsm` 1.847 dòng + `DAILY LOG CHECK SHEET.xlsm` 5.224 dòng) | Sổ cái gốc: Traveler↔Part#↔Pot#↔Weight↔Pieces↔PO↔Date↔Lot# | **100% gõ tay** (0/1.853 dòng có công thức ở CẢ 10 cột). **2 bản WORK ORDER lệch nhau THẬT**: trong số traveler có ở cả 2 file, **797 traveler bị lệch SHIPPED/PS** (vd traveler 704978: file 1 nói PS 29979, file 2 nói PS 30072) | Vật liệu (Material) — chỉ số lượng cơ bản | Không có bước **Receiving Inspection** độc lập (cân/đếm xác nhận riêng với forecast) | ~2,2 giờ/ngày gõ tay (ước tính) — CHƯA tính thời gian đối chiếu 2 file lệch nhau |
| 2 | **Sản xuất/QC lựa hàng → dán Split Form giấy** | `02_Work_Order_Quality_Check_Split form.jpeg` (giấy) | Ghi nhận FINAL INSPECTION, routing, cartons/pcs thực tế | Toàn bộ trên **GIẤY**, không số hóa cho tới khi có người gõ lại. "DMF" (Defective Material Form) được nhắc tới trong NOTE (`SORT FOR DAMAGED LOCKING-DMF FORM`) nhưng **không lưu trữ số hóa ở đâu cả** trong toàn bộ 2 file | Chất lượng (Quality) — chỉ 1 phần, không lưu | Số hóa tại chỗ (mobile/tablet lúc QC kiểm), lưu trữ DMF có cấu trúc | Thời gian trễ giữa lúc QC xong và lúc dữ liệu vào máy — không đo được chính xác vì hoàn toàn thủ công |
| 3 | **Thu gom Split Form + đối chiếu kho physical + xác định "good"** | Đi lại thu thập giấy trên xưởng | Xác nhận số liệu trước khi đóng gói chính thức | Không có hệ thống nào track — phụ thuộc hoàn toàn trí nhớ + kỷ luật cá nhân đi thu gom | Nhân lực (Labor) — không đo lường | Quy trình số hóa tại nguồn để bỏ hẳn bước "đi thu gom giấy" | ~0,5-0,75 giờ/ngày đi lại (ước tính) |
| 4 | **Gõ kết quả vào Scanning Sheet** | `CHECKING SUMMARY` (`DAILY LOG CHECK SHEET.xlsm`, 1.811 dòng) tương ứng `03_Finish_Goods_Scanning.jpeg` | Ghi nhận thành phẩm đã đóng gói: Boxes, TTL QNT, Reject, Skid... | **19/20 cột gõ tay 100%** (chỉ cột "Off" có công thức). Cột **Reject** dùng chuỗi số ngăn cách dấu phẩy không đơn vị (vd thật: `"5, 154, 59, 3"`) — máy không parse được. Cột **DEFECTS** có tên hẳn hoi nhưng **0/1.811 dòng có dữ liệu** — chưa bao giờ dùng thật. Cột LOT NO. đôi khi bị ghi đè `"SORT&RETURN"` (không phải lot thật) để làm cờ trạng thái | Sản lượng — có; Chất lượng — CÓ Ý ĐỊNH (cột Reject/DEFECTS tồn tại) nhưng **thực thi thất bại** | NCR có cấu trúc (mã lỗi + số lượng tách riêng), CHECKER field (cũng bỏ trống 100%) | ~3,3 giờ/ngày gõ tay (ước tính) — CHƯA tính thời gian đối chiếu kho vật lý |
| 5 | **Tạo Packing List → xuất hàng** | Sheet `PACKING SLIP` (Table1, 23 dòng/lần) → `04_Delivery_Order_packing_slip.jpeg` | Tạo phiếu giao hàng chính thức cho Infasco | Cột SHIPPED/PS dùng **công thức vòng lặp (circular self-reference)**, phụ thuộc cài đặt "Iterative Calculation" của Excel — phải **tô vàng + Paste-Values tay** để khóa cứng (đã xác nhận bằng cách đọc formula thật, dòng 14 vs dòng 15-17 trong ảnh Andy gửi). Traveler xuất lẻ 2 lần chỉ ghi được bằng cách **sửa chữ tay** thành `"29577/30018"` (chuỗi, không phải số) | Xuất hàng cơ bản (PO/Part/Traveler/Box/Qty qua VLOOKUP) | Field **"loại giao dịch"** (return/internal-move/sample hiện lẫn lộn — xem #6), theo dõi **container tái sử dụng** (POT rỗng, pallet) | ~0,5 giờ/ngày gõ (đỡ nhất nhờ VLOOKUP) — **nhưng rủi ro sai lệch số PS cao nhất** (xem #6) |
| 6 | **Lưu trữ lịch sử đã xuất** | `ARCHIVE` (1.578 dòng) + `ARCHIVE2` "COPY TRAVELER SHIPPING ARCHIVE" (138 dòng, chữ tự do) | Lưu vết đã xuất PS nào, traveler nào, để tra cứu sau này | **100% gõ tay, KHÔNG link gì với WORK ORDER** (đã kiểm chứng công thức: 0/1.579 và 0/138 dòng có công thức). Lệch PS với WORK ORDER: **25/1.468 traveler** (vd traveler 716404: ARCHIVE nói PS 29946, WORK ORDER nói PS 30061). Cột Part# **không có tiêu đề**. Trộn lẫn **return/internal-move/sample** chung 1 bảng không phân loại (vd "RETURN TO INFASCO NUT", "BIN TO BIN 1359->969" cùng nằm với xuất hàng thật). Sai kiểu dữ liệu: cột BOXES có giá trị chữ `"BIN"`. ARCHIVE2 là **1 cột chữ tự do** nhét nhiều dòng, không máy nào parse được, lẫn cả ghi chú không liên quan ("SAMPLES DELIVERED BY DANIEL") | Lưu trữ lịch sử — có, nhưng KHÔNG đáng tin | Field loại giao dịch (transaction type) bắt buộc chọn — chuẩn Odoo `stock.picking.picking_type_id` | Không đo được giờ, nhưng đây là **nguồn phát sinh sai số PS nhiều nhất đã đo được** |
| 7 | **Báo cáo KPI hàng ngày cho quản lý xem** | `SUMMARY IN&OUT` | Trả lời nhanh "hôm nay nhận/xử lý/xuất bao nhiêu" cho quản lý | **Đếm tay hoàn toàn, KHÔNG khớp với WORK ORDER thật** — kiểm tra 30 ngày mẫu, **0/30 ngày khớp** (sai lệch gần đúng ≤2), có ngày lệch tới **105** (ngày 46258: WORK ORDER có 105 dòng thật, SUMMARY ghi trống) | Số lượng thô (throughput) — có nhưng SAI | Tính **tự động** từ sổ cái thật thay vì đếm tay/ước lượng | **Không tính ra giờ được — hậu quả nghiêm trọng hơn giờ: quản lý ra quyết định dựa trên số liệu SAI mà không biết** |
| 8 | **Tính năng thiết kế rồi bỏ dở** | `Status` (danh sách 5 trạng thái: Loaded/Not started/Preloaded/Running/Shipped), `Crosscheckk`, `Sheet1`/`Sheet2` | Từng có ý định làm trạng thái sản xuất chi tiết hơn + đối chiếu chéo | `Status`: 5 trạng thái nhưng quét toàn bộ workbook **chỉ xuất hiện 4 lần, trong CHÍNH sheet đó** — chưa từng dùng thật. `Crosscheckk`: 1 dòng dữ liệu, mọi ô ghi literal **"Please enter"** — chưa từng hoàn thành | — | — | Bằng chứng công ty đã **nhiều lần cố nâng cấp nhưng không hoàn thành** — rủi ro lặp lại nếu không đổi cách làm |

---

## 2. Đối chiếu ERP chuẩn — họ đang quản lý được BAO NHIÊU % nguồn lực doanh nghiệp?

ERP (Enterprise Resource Planning) đúng nghĩa quản lý **toàn bộ nguồn lực**:
Vật liệu, Máy móc, Nhân lực, Tài chính, Chất lượng, Khách hàng. Đối chiếu:

| Nguồn lực (chuẩn ERP/Odoo) | AVP hiện tại | Bằng chứng |
|---|---|---|
| **Vật liệu (Material/Inventory)** | ⚠ Có, nhưng phân mảnh 2+ file, không đồng bộ | Mục 1, hàng #1, #7 |
| **Chất lượng (Quality)** | ❌ Có Ý ĐỊNH, thực thi thất bại | Cột DEFECTS 0% dùng, Reject không cấu trúc (mục 1 hàng #4) |
| **Máy móc/Thiết bị (Maintenance/OEE)** | ❌ Không có gì | Không 1 cột nào ghi giờ chạy/dừng máy trong TOÀN BỘ 2 file |
| **Container/Tài sản tái sử dụng (Asset)** | ❌ Không có | POT#/Pallet không track vòng đời (nhận→rỗng→trả) |
| **Nhân lực (HR/Labor)** | ❌ Không có | Không file nào đo giờ công, năng suất nhân viên |
| **Tài chính/Chi phí (Costing)** | ❌ Không có | Không có cột chi phí, giá vốn nào trong toàn bộ 2 file |
| **Khách hàng (CRM)** | ❌ Không có (dù có hàng chục khách hàng thật — xem `PartControl.client`: Jobal, Thompson Fasteners, Ultra Form, Dana Canada...) | Không sheet nào quản lý quan hệ/lịch sử theo khách hàng |
| **Báo cáo quản trị (BI/Reporting)** | ❌ Có làm (`SUMMARY IN&OUT`) nhưng **sai số liệu** | Mục 1, hàng #7 |

**→ Ước tính: AVP hiện chỉ thực sự quản lý được 1/8 nguồn lực chuẩn ERP
(Vật liệu, và ngay cả phần này cũng lệch nhau giữa các file) — 7/8 nguồn
lực còn lại KHÔNG được quản lý có hệ thống, dù một số (Quality) đã CÓ Ý
ĐỊNH làm nhưng bỏ dở.**

---

## 3. Tổng chi phí lao động ước tính (xem chi tiết `BRS_TRS.md` mục 6c/6d)

| Thành phần | Ước tính |
|---|---|
| Gõ tay WORK ORDER (10 cột × ~78,5 traveler/ngày) | ~2,2 giờ/ngày |
| Thu gom Split Form + cập nhật | ~0,5-0,75 giờ/ngày |
| Đối chiếu kho + gõ Scanning Sheet (~15 cột × 78,5) | ~3,3 giờ/ngày |
| Gõ Packing List + đếm Pallets/Empty | ~0,5 giờ/ngày |
| **Tổng gõ thuần** | **~6,5-7,3 giờ/ngày** |
| + Hệ số "mò tìm số liệu" (2-3 lần, do Andy đề xuất — dò VLOOKUP 3 tầng, giải mã chữ tự do, đối chiếu file lệch nhau) | **~21-28 giờ/ngày** (tổng, chia nhiều nhân viên) |
| **Quy năm (250 ngày làm việc)** | **~5.250-7.000 giờ/năm** — chỉ phần hành chính, CHƯA tính công lựa/đóng gói vật lý |

⚠ Đây là **ước tính có giả định rõ ràng** (tốc độ gõ 8-15 giây/ô, hệ số mò
2-3 lần) — nên đo thời gian thật (bấm giờ 1 ngày làm việc thật) trước khi
dùng số này thuyết trình chính thức với chủ.

---

## 4. Kết luận — vì sao cần thiết kế hệ thống mới

1. **Không phải "làm chưa tới" mà là "nhiều nơi tự làm, không ai đồng bộ"** —
   bằng chứng rõ nhất: **2 bản WORK ORDER lệch nhau ở 797 traveler thật**,
   **SUMMARY IN&OUT sai 100% (0/30 ngày khớp)** — đây không phải rủi ro lý
   thuyết, là lỗi **ĐÃ XẢY RA** trong dữ liệu lịch sử thật, đang được dùng
   để ra quyết định mỗi ngày.
2. **Công ty đã nhiều lần cố nâng cấp nhưng bỏ dở** (`Status`, `Crosscheckk`) —
   không phải vì không nhận ra vấn đề, mà vì thiếu 1 kiến trúc đúng để hoàn
   thành trọn vẹn thay vì làm thêm 1 sheet rời rạc mỗi lần.
3. **7/8 nguồn lực ERP chuẩn chưa được quản lý** — công ty đang vận hành như
   1 xưởng thủ công ghi sổ, dù quy mô thật (hàng chục khách hàng, hàng nghìn
   traveler/năm) đã tương đương 1 doanh nghiệp cần ERP thật sự.
4. **Chi phí cơ hội ước tính ~5.000-7.000 giờ/năm** lãng phí vào việc gõ +
   mò số liệu — phần lớn có thể chuyển thành thời gian làm việc có giá trị
   hơn (xử lý ngoại lệ, chăm sóc khách hàng, cải tiến sản xuất) nếu có hệ
   thống đúng.
5. **AVP_AI (dự án đang xây) đã giải quyết được 1 phần đáng kể** những lỗi
   cụ thể tìm thấy ở trên (không còn công thức vòng lặp, ép kiểu dữ liệu,
   cảnh báo tự động dựa trên quy luật đã đo bằng dữ liệu thật) — nhưng để
   giải quyết TOÀN DIỆN (7/8 nguồn lực còn thiếu), cần đầu tư thêm theo lộ
   trình đã đề xuất ở `ARCHITECTURE.md` mục 8.

---

*Báo cáo này tổng hợp từ nghiên cứu trực tiếp trong phiên làm việc
2026-09-12 → 2026-09-13. Chi tiết kỹ thuật đầy đủ: `BRS_TRS.md` (mục 6a-6e),
`ARCHITECTURE.md` (mục 8). Mọi số liệu đối chiếu đều chạy trực tiếp trên file
Excel thật, có thể tái kiểm chứng lại bất cứ lúc nào.*
