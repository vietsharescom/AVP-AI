# SESSION REPORT
## SES-20260912-001 — AVP Packing Flow

> **ĐỌC FILE NÀY TRƯỚC** khi bắt đầu phiên làm việc tiếp theo.
> Đây là nguồn thông tin duy nhất để AI nhớ lại ngữ cảnh giữa các phiên
> (theo đúng nguyên tắc của khung `D:\16.ISO_CA`: "AI không có trí nhớ
> giữa các phiên — session report thay thế trí nhớ đó").

---

## 1. THÔNG TIN PHIÊN

| Trường | Giá trị |
|---|---|
| Session | SES-20260912-001 |
| Ngày | 2026-09-11 → 2026-09-12 |
| Chủ dự án | Andy Phan (Viet), Maple Leaf Group |
| Thư mục | `D:\AVP_AI` |
| Web app | `D:\AVP_AI\webapp` (Next.js + TypeScript) |
| Database | Google Sheets, ID `1TWYKdRHv3MioEBf2DHAvUGlCvZxsr-KXQKLCR_3BkJ4` |
| Git | Repo có sẵn (`aecea88` initial commit), **TOÀN BỘ code phiên này CHƯA COMMIT** |
| Link demo | https://believe-acrobat-purchased-agencies.trycloudflare.com (tunnel tạm, chết khi tắt máy) |
| Trạng thái lúc đóng phiên | Server `localhost:3000` **còn chạy** (200 OK). Tunnel: process `cloudflared` còn sống nhưng link **không phản hồi** lúc kiểm tra cuối (DNS fail) — phiên sau cần kiểm tra lại link, có thể phải khởi động lại tunnel. |

---

## 2. TRẠNG THÁI KIỂM THỬ

Không có test suite tự động (dự án không dùng pytest/jest). Xác nhận bằng:
`npm run build` + `npm run lint` — **PASS** ở lần build cuối cùng trong phiên.
Test thực tế qua ảnh/PDF thật (traveler form, scanning sheet, split form) —
xem mục 3.

---

## 3. ĐÃ HOÀN THÀNH TRONG PHIÊN NÀY

### Xây từ đầu: web app 4 khâu thay thế 2 file Excel
`DAILY LOG CHECK SHEET.xlsm` + `PACKING SLIPS.xlsm` (chỉ 1 người mở được 1
lúc) → thay bằng Next.js + Google Sheets (nhiều người dùng cùng lúc).

- **1. Email/Dự báo** (`/inbox`) — upload traveler form (ảnh/PDF) → Gemini
  đọc → người xem lại → lưu vào RawMaterial. Có đối chiếu barcode thật.
- **2. Kho Nguyên Vật Liệu** (`/warehouse`) — MỚI, khâu còn thiếu hoàn toàn
  lúc đầu. Xác nhận số **thực nhận** (khác dự báo email), có quét Split
  Form để tìm nhanh đúng dòng traveler.
- **3. Finish Good** (`/scan`) — quét tờ Scanning viết tay HOẶC Split Form
  (lấy đúng dữ liệu đóng gói thật S4/S6/S11/S13/S14/S16/S18). Tự tách file
  PDF nhiều trang thành từng trang xử lý riêng (giảm lỗi lẫn cột từ ~30%
  xuống ~2.4%). Có nút nhập tay khi AI không đọc được.
- **4. Packing List** (`/packing-list`) — ghép Raw Material + Finish Good
  theo Traveler#, tự tính DESCRIPTION + QUANTITY, đối chiếu chéo với TTL
  QNT lúc scan, phát hiện traveler bị scan trùng trước khi duyệt.
- **PartControl** (`/part-control`) — đã import 1235 dòng thật từ
  `PACKING SLIPS.xlsm`. Có form thêm/sửa + danh sách Part# thiếu cần chủ
  xác nhận.
- **Đối chiếu Traveler** (`/reconciliation`) — Forecast → Received →
  Scanned → Shipped, số tuyệt đối + %, ngưỡng cảnh báo chỉnh được.
- **Truy vết Traveler** (`/trace`) — nhập 1 Traveler#, xem lịch sử đủ 4
  khâu, tự cảnh báo khi 1 khâu có >1 bản ghi (dấu hiệu trùng lặp).

### Quyết định kỹ thuật quan trọng (đừng làm lại/tranh luận lại)
- **Groq → Gemini**: Groq free tier chỉ 15 request/phút + 1000 token/phút
  → không đủ cho 1 trang traveler form. Gemini free tier rộng hơn nhiều.
  Model đang dùng: `gemini-3.5-flash-lite`.
- **OAuth thay Service Account**: org Google Workspace của AVP chặn
  `iam.disableServiceAccountKeyCreation` → dùng OAuth (tài khoản Google
  của Andy), refresh token lấy 1 lần qua `scripts/get-refresh-token.mjs`.
- **Tách PDF nhiều trang, xử lý tuần tự (không Promise.all)**: Gemini
  free tier chỉ 15 request/phút — xử lý song song nhiều trang sẽ bị 429
  ngay lập tức. Đã thêm retry+backoff cho lỗi 429.
- **Mọi bước lưu đều "partial save"**: không bao giờ chặn cả batch chỉ vì
  1-2 dòng lỗi (traveler trùng/không tồn tại) — chỉ skip đúng dòng lỗi,
  lưu phần còn lại, báo rõ dòng nào bị skip.
- **Không tự clear dữ liệu form khi upload/lưu thất bại**: sự cố thật đã
  gặp (mất 171 dòng scan) do code cũ xóa `rows` trước khi biết kết quả —
  đã sửa: chỉ thay dữ liệu cũ SAU khi có dữ liệu mới thành công.
- **Model 4 khâu (không phải 3)**: Traveler form email chỉ là DỰ BÁO
  (forecast), không phải tồn kho thật — xác nhận trực tiếp từ AVP. Khâu
  "Kho Nguyên Vật Liệu" là khâu riêng, độc lập.
- **Split Form ≠ nguồn số lượng nhận kho**: trường "Pieces" trên Split
  Form thường để trống — KHÔNG dùng form này để đối chiếu số nhận kho với
  email. Phần "PACKAGING" ở góc dưới trái mới là số liệu thật, và nó thuộc
  về Finish Good (khâu 3), không phải Kho NVL (khâu 2).

### File quan trọng đã tạo ngoài code
- `BRS_TRS.md` — business/technical requirements, xác minh với dữ liệu
  thật (WORK ORDER, PartControl trong 2 file .xlsm gốc).
- `ISO_AI_CONTROL.md` — mô hình 5 lớp kiểm soát AI (bản rút gọn).
- `CLAUDE.md` (project-local, mẫu đơn giản tham khảo `D:\Ops_Ai`) — mục
  SESSION START/END viết thẳng trong file, không dùng skill/lệnh tự động
  (đã thử đề xuất kiểu LifeOS, Owner từ chối vì "nặng", đổi sang mẫu này).
- File này (`LATEST_SESSION.md`).

---

## 4. TRẠNG THÁI HIỆN TẠI & LỖ HỔNG ĐÃ BIẾT

**Giai đoạn:** MVP hoạt động được với dữ liệu thật, đã test nhiều vòng qua
ảnh/PDF thật (traveler form, scanning sheet, split form, packing slip cũ
để đối chiếu). CHƯA sẵn sàng dùng thật hàng ngày.

### Lỗ hổng đã biết (GAP)
- **GAP-001**: Cổng email tự động (Gate 1 automation — tự đọc hộp thư
  Infasco) CHƯA làm — vẫn phải upload tay ảnh/PDF. Cần xác nhận Infasco
  gửi qua email nào + cách cấp quyền đọc hộp thư trước khi làm.
- **GAP-002**: Packing List đánh dấu Shipped theo TRAVELER#, không theo
  từng dòng Finish Good riêng biệt → nếu 1 traveler có nhiều dòng (trùng
  lặp hoặc chia nhiều lần đóng gói hợp lệ), TẤT CẢ bị đánh dấu Shipped
  cùng lúc. Đã bàn thiết kế sổ cái theo ID riêng (`SCAN-xxx`) nhưng
  **CHƯA triển khai** — mới dừng ở tầng phát hiện/cảnh báo trùng lặp.
- **GAP-003**: Không lưu file ảnh/PDF gốc kèm mỗi dòng dữ liệu → không
  trả lời được "traveler này lấy từ file nào" sau khi dữ liệu đã lưu (gặp
  phải khi debug traveler 717671 bị trùng).
- **GAP-004**: 2 dòng trùng thật của traveler 717671 trong FinishGood
  (do lỗi mạng lúc test) **CHƯA XÓA** — anh đã đồng ý an toàn để xóa
  (không liên quan chứng từ thật nào) nhưng chưa thực hiện.
- **GAP-005**: Chưa có test tự động, chỉ xác nhận bằng build/lint + test
  tay.
- **GAP-006**: `git commit` CHƯA làm cho toàn bộ code phiên này (đúng
  theo luật "không commit khi chưa confirm" — nhưng có rủi ro mất việc
  nếu máy có sự cố trước khi commit).
- **GAP-007**: PartControl "missing parts" (21 mã Part# thật từ lịch sử
  chưa có Quantity/box) — đã liệt kê trong `BRS_TRS.md` mục 6, **chưa có
  ai (chủ) xác nhận số Quantity/box** cho các mã này.

### Việc đang mở (chưa quyết định)
- Anh hỏi có muốn nâng Groq lên Dev Tier không → đã chuyển hẳn sang
  Gemini nên câu hỏi này không còn áp dụng, có thể bỏ qua.
- Câu hỏi "Box QTY: 31.000" trên Split Form có phải = số thực nhận kho
  hay không — **CHƯA có câu trả lời từ Andy**, đang treo.

---

## 5. BẮT ĐẦU PHIÊN SAU TỪ ĐÂY

### Việc ưu tiên tiếp theo
0. Kiểm tra lại link tunnel (lúc đóng phiên link không phản hồi dù
   process cloudflared vẫn sống) — có thể cần `cloudflared tunnel --url
   http://localhost:3000` lại, link mới sẽ khác link cũ.
1. Hỏi Andy: xóa 2 dòng trùng traveler 717671 trong Google Sheet luôn
   chưa (đã xác nhận an toàn, chỉ còn chờ thực hiện).
2. Trả lời câu hỏi "Box QTY 31.000" trên Split Form trước khi quyết định
   có dùng số đó cho khâu Kho NVL không.
3. Nếu muốn dùng thật: cần server chạy 24/7 (hiện chỉ chạy khi máy Andy
   bật + cloudflared tunnel tạm) — bàn phương án deploy thật (Vercel,
   VPS...) khi sẵn sàng.
4. Cân nhắc triển khai sổ cái theo ID riêng cho mỗi dòng Finish Good
   (GAP-002) nếu việc chia nhiều lần đóng gói/xuất hàng là phổ biến thực
   tế (Split Form đã cho bằng chứng thật là CÓ xảy ra).
5. `git add` + `git commit` khi Andy xác nhận (chưa tự ý làm).

### Cảnh báo cho phiên sau
- ⚠ KHÔNG tự commit/push git — Andy chưa xác nhận.
- ⚠ Server dev hiện có thể đã tắt (chạy `npm run start` trong
  `D:\AVP_AI\webapp`, port 3000) — tunnel cloudflared cũng cần chạy lại
  nếu đã tắt máy (`cloudflared tunnel --url http://localhost:3000`), link
  sẽ ĐỔI nếu tunnel khởi động lại (trừ khi dùng named tunnel).
- ⚠ File `.env.local` trong `webapp/` chứa API key thật (Gemini +
  Google OAuth) — không commit file này (đã có trong `.gitignore` mặc
  định của Next.js, nhưng double-check trước khi `git add`).
- ⚠ 2 dòng trùng traveler 717671 vẫn còn trong Sheet — sẽ tiếp tục làm
  méo báo cáo Đối chiếu Traveler cho tới khi xóa.

---

## 6. QUYẾT ĐỊNH ĐÃ CHỐT (KHÔNG BÀN LẠI)

- DESCRIPTION = Oprtr + Machine + MC# + S/P + Special Notes. **KHÔNG**
  gồm Type/Nut-Bolt (đã thử thêm "BOLTS" theo suy đoán, Andy xác nhận sai,
  đã rút lại — xem `BRS_TRS.md` mục 4).
- QUANTITY = BOX × Quantity/box (PartControl), **không** copy thẳng TTL
  QNT lúc scan — TTL QNT chỉ dùng đối chiếu chéo.
- Part# hậu tố mạ/vật liệu (`-L`, `-HT`, `-A`...) không ảnh hưởng
  Quantity/box — an toàn để bỏ hậu tố khi tra PartControl không khớp
  thẳng.
- Ngưỡng cảnh báo mặc định cho mọi đối chiếu số lượng: **10%**.

---

*Session Report — tham khảo mẫu của D:\16.ISO_CA/docs/records/*
*Ghi lúc: 2026-09-12*
*Đọc lại đầu phiên sau; cập nhật lại file này ở cuối mỗi phiên làm việc tiếp theo.*
