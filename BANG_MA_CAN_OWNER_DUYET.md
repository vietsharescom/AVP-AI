# BẢNG MÃ HIỆN CÓ — CẦN OWNER DUYỆT/BỔ SUNG

*Tổng hợp 2026-09-13 theo yêu cầu Owner (qua Andy): "thống kê lại toàn bộ
bảng mã hiện có cho owner duyệt". Mỗi bảng ghi rõ nguồn (đo trên dữ liệu
thật hay chỉ là quan sát), để Owner biết cái nào cần xác nhận, cái nào chỉ
cần điền số.*

---

## 1. Mã Part# — 36 mã đang hoạt động thật, CẦN Owner cung cấp Quantity/box

Xác nhận 2026-09-13: mỗi hậu tố (`-L`, `-HT`, `-A`...) là **1 sản phẩm
riêng**, không dùng chung Quantity/box với mã gốc. Đo trên RawMaterial hiện
tại: 36/39 Part# đang dùng thật chưa có dòng riêng trong PartControl.

Cột "Gợi ý" chỉ để tham khảo tốc độ điền (số của mã GỐC, KHÔNG được tự ý
dùng nếu Owner chưa xác nhận 2 mã thực sự cùng Quantity/box):

| Part# cần thêm | Số traveler đang dùng | Quantity/box mã GỐC (chỉ để tham khảo, CHƯA xác nhận đúng cho mã có hậu tố) |
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

Loại khỏi danh sách: `TEST-PART` (7 traveler) — dữ liệu test nội bộ, không phải Part# thật.

**Đề xuất quy trình duyệt**: với mỗi dòng, Owner chỉ cần xác nhận **Quantity/box thật** (có thể giống hoặc khác mã gốc) — không cần làm hết 1 lần, ưu tiên theo cột "Số traveler đang dùng" (nhiều nhất trước).

---

## 2. Mã hậu tố Part# — CÁC LOẠI hậu tố đã quan sát được (chưa biết nghĩa từng loại)

Quan sát được trong dữ liệu thật, **chưa biết ý nghĩa cụ thể từng mã** —
cần Owner giải thích (vd `-HT` = Heat Treat? `-L` = Locked/Long/loại
khác? `-A` = ?):

```
-L        -HT       -A        -T        -B        -X
-MY-AK    -CA-IN    -IN-PN    -TH-TC    -BR-HA
```

---

## 3. Mã máy (Machine/MC#)

Nguồn: sheet `Status` trong `PACKING SLIPS.xlsm` (đã có sẵn, chưa đưa vào hệ thống để validate):

- **BF 102 → BF 112** (11 mã)
- **MC 4 → MC 112** (nhiều mã, không liên tục — cần Owner xác nhận đủ dải)
- **TABLE 1, TABLE 2**
- **PCKY**

→ Tổng cộng 25 mã hợp lệ theo nguồn này. **Cần Owner xác nhận danh sách này còn đúng/đủ không** (có máy mới thêm hay máy cũ đã bỏ chưa cập nhật).

**Câu hỏi còn treo (GAP-012)**: mỗi mã máy có phải 1 máy vật lý ĐỘC QUYỀN hay 1 TRẠM dùng chung nhiều traveler/ca? (5,66% dữ liệu thật cho thấy dùng chung — chưa rõ bình thường hay lỗi).

---

## 4. Mã nhân viên (Operator code)

**CHƯA có danh sách chính thức** — chỉ quan sát được các mã xuất hiện trên
chứng từ thật, dạng số 2-3 chữ số, đôi khi ghi gộp 2 mã (vd `"275/190"`):

```
275   290   279   391   292   285   275   157   297   280
```

Owner cần cung cấp: **danh sách mã nhân viên hợp lệ kèm tên thật** (nếu có
sẵn trong hệ chấm công/nhân sự riêng, chỉ cần export ra) — hiện hệ thống
không có gì để đối chiếu, AI đọc ra số nào cũng chấp nhận.

---

## 5. Mã thiết bị QC (Hardness/Tensile/Proof Load)

Xem lại đúng khu **FINAL INSPECTION** trên Split Form thật (Nut/Bolt
Hardness Spec, Bolt Tensile Strength Spec, Nut/Bolt Proof Load Spec) —
**KHÔNG có ô mã máy/thiết bị nào cả**, chỉ có "Stamp & Date". Khác hẳn khu
Sorting Results có ghi rõ "Sorting M/C#: MC078".

→ **Chưa biết được liệu phòng QC có đánh số máy đo (máy đo độ cứng, máy
kéo...) riêng hay không** — cần Owner xác nhận có tồn tại mã này ở đâu
khác không (có thể ghi trên tem hiệu chuẩn máy, không phải trên Split
Form).

---

## 6. Quy ước LOT NO.

Đã xác nhận 2026-09-13 (xem `BRS_TRS.md` mục 3b) — LOT NO. có 4 trạng thái:

| Dạng | Ý nghĩa |
|---|---|
| `6-DDD-SS-L` (vd `6-247-03-b`) | Mã lot thật — cấp bởi Infasco (đôi khi giao trễ) |
| `TRAVELERRECEIVED` | Traveler đã nhận, ĐANG CHỜ Infasco cấp LOT# — không phải lỗi |
| `SORT&RETURN` | Hàng trả lại, không ra lot |
| `SPLIT FROM TR#{traveler}` | Ghi chú tách từ traveler khác |

**Cần Owner xác nhận**: cấu trúc `6-DDD-SS-L` — số "6" cố định là gì (mã
nhà máy/dòng sản phẩm?), "DDD" có phải ngày trong năm không? (không bắt
buộc phải biết để hệ thống chạy đúng, nhưng giúp validate tốt hơn).

---

## 7. Mã Serial#/SSCC (mức thùng/carton — mới phát hiện, CHƯA đưa vào hệ thống)

Từ nhãn đóng gói thật: mỗi carton có 1 số serial liên tục (vd
`...19929` → `...19964` cho đúng 36 carton). Đây là mã chuẩn GS1 (SSCC) —
đề xuất thêm vào hệ thống, xem trao đổi trước đó trong `Notes.md`.

---

*Tài liệu này là danh sách LÀM VIỆC — cập nhật dần khi Owner xác nhận từng
mục, không phải làm 1 lần xong hết. Ưu tiên mục 1 (36 Part# thiếu Quantity/
box) vì đang ảnh hưởng trực tiếp tới số liệu Packing List thật.*
