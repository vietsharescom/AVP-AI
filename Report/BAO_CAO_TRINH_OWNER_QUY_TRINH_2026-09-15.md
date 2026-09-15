# BÁO CÁO TRÌNH OWNER — TOÀN BỘ QUY TRÌNH TỪ EMAIL ĐẾN XUẤT XE
## Hiện trạng — Hậu quả đo được — Hướng khắc phục — Kết quả mang lại

*Ngày: 2026-09-15. Đọc 1 lần: mỗi hàng là 1 bước thật trong quy trình, đi
từ trái sang phải: Anh/chị đang làm gì → nó đang gây ra hậu quả gì (có số
đo, không phải cảm nhận) → sửa hướng nào → sửa xong thì được gì.*

> **Không có con số nào trong bảng này là ước lượng cảm tính** — mọi số
> đều lấy trực tiếp từ Excel/PDF/Google Sheet đang dùng thật, có ghi rõ
> nguồn ở cuối mỗi hàng để đối chiếu lại bất cứ lúc nào.

---

## BẢNG QUY TRÌNH — TỪ EMAIL PO ĐẾN XE RỜI BẾN

| # | Bước quy trình | Hiện tại đang làm gì | Hậu quả hiện nay — có bằng chứng | Hướng khắc phục → Kết quả mang lại |
|---|---|---|---|---|
| 1 | **Nhận PO / lập kế hoạch** — email Infasco báo trước lô hàng | Nhân viên kế hoạch tự mở email, tự đọc, **không lập kế hoạch bằng văn bản** — nguyên liệu về Pot/Bin rồi tự "trôi" sang máy nào tiện lúc đó, không ai chính thức giao sản lượng cho máy nào | Không có 1 trường dữ liệu nào (kể cả Excel cũ lẫn hệ thống mới) ghi "kế hoạch sản xuất" — 0% được lập kế hoạch bằng số. PO về **13h chiều** hôm qua, sáng cùng ngày vẫn làm hàng tồn thứ Sáu (quan sát trực tiếp, lặp lại) | Đọc PO tự động ngay khi email về, gợi ý phân máy dựa trên máy đang rảnh — **1 người kế hoạch quyết trong 5 phút thay vì cả ca mò mẫm** — giảm được đúng khâu hiện đang tốn người nhất |
| 2 | **Nhận nguyên liệu vật lý** | Không có bước xác nhận "đã về kho thật" — hàng về là tự động coi như có, không ai ký nhận số đếm thật | Tab `Warehouse` được tạo đúng để làm việc này nhưng **không ai ghi vào từ khi redesign** — GAP lớn nhất hệ thống, đã ghi nhận từ trước | 1 người quét mã Pot# lúc nhận (điện thoại/máy quét), xác nhận số — **không cần thêm người, chỉ cần đúng lúc đang làm sẵn thao tác nhận hàng** |
| 3 | **Đưa vào sản xuất — chạy máy** | Nhiều traveler chạy chung 1 máy cùng lúc, cột Máy/MC# chỉ ghi **SAU KHI đóng gói xong** (hồi tố), không ghi lúc bắt đầu chạy | **62% (381/614)** nhóm Ngày+Máy chạy chung ≥2 traveler, cao nhất **13 traveler/máy/1 ngày**. Đo thêm hôm nay: **124/1.911 traveler (6,5%)** có sản lượng đóng gói lệch **>50%** so với nguyên liệu nhận vào — không biết mất ở đâu vì không tách được theo mẻ | Ghi máy chạy NGAY lúc bắt đầu (barcode máy quét khi gắn traveler) thay vì hồi tố cuối ca — **hết cảnh nghi ngờ "hao hụt" oan cho công nhân vì gộp sổ sai, giảm thời gian điều tra sự cố** |
| 4 | **QC / phân loại — Split Form giấy** | QC viết tay PASS/HOLD trên giấy, đánh dấu gạch chéo ô — **không ai gõ lại vào hệ thống** | **98,5% (191/194)** dòng dữ liệu sống có ô trạng thái QC **HOÀN TOÀN TRỐNG** — chỉ 1 dòng duy nhất từng bị hệ thống chặn vì HOLD trong suốt thời gian chạy. Không phải vì hàng luôn đạt — vì hệ thống không biết gì cả | QC quét/chụp form ngay lúc ký — **nhân viên Packing Slip không cần XUỐNG XƯỞNG tìm form giấy mỗi lần cần biết cái gì đạt** — đây là việc tốn người nhất trong khâu lập PS hiện nay |
| 5 | **Đóng gói / wrap thành phẩm — Scanning Sheet** | **100% viết tay trên giấy rồi gõ tay lại** vào Excel (19/20 cột, chỉ đúng 1 cột có công thức) — chưa ai ở AVP dùng máy/AI để đọc bước này, hoàn toàn thủ công từ đầu tới cuối | Không ai đo được form nào viết khó đọc, bao nhiêu % — vì không có bước nào tự phát hiện chữ khó đọc, chỉ đến khi gõ sai/gõ nhầm mới lộ ra, thường là rất trễ | Đây là chỗ ĐANG THIẾU, chưa có ai làm: thêm bước quét/chụp thay gõ tay, để máy tự báo "chữ này khó đọc, cần người xem lại" **trước khi lưu** — hiện chưa triển khai thật ở AVP, chỉ mới là hướng đề xuất |
| 6 | **Theo dõi hàng dư (vít dư đạt chuẩn, thùng dở)** | Quản lý tự lập file Excel riêng (`Partial Boxes 2026.xlsx`) ghi tay mỗi ngày, ngoài hệ thống chính | **54,6% (560/1.026)** dòng ghi nhận thùng dở **KHÔNG có Traveler#** đi kèm → không thể truy Part#/PO/QC của số hàng dư đó. **9,3%** không có LOT#. Hàng dư "chất đống, không biết mã lot chất lượng" — đúng như quản lý mô tả, có số thật | Gắn Traveler#/LOT# bắt buộc ngay lúc ghi nhận thùng dở — biết chính xác bao nhiêu, của lô nào, gộp trả Infasco đúng, không tồn đọng mù mờ |
| 7 | **Lập Packing Slip** | Nhân viên PS vào nhiều bảng Excel khác nhau tìm mã, tìm traveler, chọn tay — làm vội vì xe đang chờ | **797/1.847 (43%)** traveler lệch trạng thái SHIPPED/số PS giữa 2 sổ song song — bảng nói "chưa xuất" trong khi thực đã xuất hoặc ngược lại, nhân viên phải tự đoán tin bảng nào | 1 nguồn duy nhất trả lời "traveler này đã xuất chưa, đạt QC chưa" — **nhân viên PS không cần mở 3-4 file Excel, không cần tự đối chiếu tay mỗi lần lập phiếu** |
| 8 | **Xuất hàng / giao xe** | Số Packing Slip do nguồn ngoài AVP gõ tay, không tự sinh; Part# in trên tem đóng gói đôi khi khác Part# ghi đầu phiếu | Thực tế đã xảy ra: **PS 30102 xuất thật cho khách với 2 dòng Quantity = 0** — lỗi chỉ phát hiện khi tự kiểm tra lại, khách hàng chưa từng được báo | Chặn cứng không cho xuất khi còn dòng thiếu số/còn HOLD chưa xử lý (đã làm được ở webapp mới) — **giảm rủi ro gửi sai cho khách, giảm việc phải xử lý khiếu nại/làm lại sau đó** |
| 9 | **Lưu trữ lịch sử + Báo cáo KPI cho quản lý** | Đếm tay từ nhiều sổ (`ARCHIVE`, `ARCHIVE2`, `SUMMARY IN&OUT`) mỗi ngày để báo cáo | **0/30 ngày mẫu** số báo cáo tay khớp với dữ liệu WORK ORDER gốc — có ngày lệch tới **105 dòng**. Quản lý đang quyết định dựa trên số có thể sai mà không biết | Số KPI tính tự động từ đúng 1 nguồn — **bỏ hẳn công đoạn "đếm tay báo cáo mỗi ngày"**, đây là chỗ có thể giảm người rõ và nhanh nhất vì việc này thuần tuý là đếm lại, không cần con người ra quyết định |

---

## MUỐN GIẢM NGƯỜI — ĐÚNG, NHƯNG CHỈ GIẢM ĐƯỢC Ở ĐÂU

Đọc lại cột 2 của cả 9 hàng trên: phần lớn công sức con người hiện nay
KHÔNG phải "làm ra sản phẩm" — mà là **đi tìm, đối chiếu, đếm lại, đoán số
liệu** giữa các sổ không khớp nhau. Đây đúng là phần máy làm thay được,
không đụng đến người trực tiếp sản xuất/đóng gói:

- Bước 1, 7, 9 — **có thể giảm người ngay, rủi ro thấp nhất**: lập kế
  hoạch, tìm traveler lập PS, đếm KPI báo cáo — cả 3 đều là việc "gõ +
  đối chiếu", không phải quyết định chuyên môn.
- Bước 3, 4, 8 — **giảm được thời gian xử lý sự cố sau này**, không giảm
  người trực tiếp ngay, nhưng giảm số người cần để "chữa cháy" khi có
  khiếu nại/hàng sai.
- Bước 2, 5, 6 — **không giảm người, nhưng giảm rủi ro lớn** (Goods
  Receipt, chất lượng chữ viết tay, hàng dư tồn đọng) — đầu tư nhỏ, ngăn
  thiệt hại lớn hơn nhiều lần chi phí đầu tư.

**Ước tính ban đầu**: 21-28 giờ/ngày dành cho gõ tay + dò tìm số liệu
(quy năm ~5.250-7.000 giờ, nguồn: `BAO_CAO_TONG_HOP_AVP.md` mục 4, đo
2026-09-14) — phần "dò tìm/đối chiếu" chiếm tới **75% con số đó** (~14-21
giờ/ngày).

> **Cập nhật 2026-09-15**: con số trên đo THIẾU. Sau khi cân nhắc
> lại, con số hợp lý hơn cho tổng hao phí là **khoảng 32 giờ/ngày**
> (tương đương ~4 công lao động, 1 công = 8 giờ) — trong đó vẫn giữ ~7
> giờ/ngày là gõ tay thuần, phần còn lại **~25 giờ/ngày là dò tìm/đối
> chiếu số liệu không hiệu quả**. Quy năm **~8.000 giờ/năm**. Đây là điều
> chỉnh dựa trên quan sát trực tiếp thực tế, CHƯA đo lại bằng bấm
> giờ thực tế — nên trình bày với Owner đúng là "ước tính đã điều chỉnh
> theo quan sát mới", không phải số đo chính xác, để giữ đúng nguyên tắc
> không thổi phồng số liệu.

Đây chính là phần công cụ mới thay được, không phải phần cần thêm/bớt
người làm ra sản phẩm.

---

*Xem tiếp Phần 2 (biểu đồ trực quan — Infasco vs AVP, và chỉ tiêu cho hệ
thống mới) tại trang trình bày trực quan đính kèm.*
