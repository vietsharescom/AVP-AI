# ISO_AI_CONTROL — AVP Packing Flow
## Bản rút gọn, tham khảo ISO/IEC 42001:2023 (theo khung D:\16.ISO_CA)

> Đây là bản ĐƠN GIẢN HÓA cho 1 web app nội bộ (Next.js), không phải hệ
> thống agent tự hành — nên không áp dụng đủ 11 stage/10 clause của khung
> gốc. Chỉ giữ lại đúng nguyên lý cốt lõi: **AI chỉ tạo ra bản nháp, con
> người luôn là người quyết định cuối cùng.**

---

## Mô hình 5 lớp kiểm soát AI (thực tế đã xây, không phải lý thuyết)

```
┌────────────────────────────────────────────────────────────┐
│ LỚP 1 — AI SINH DỮ LIỆU (Generation)                        │
│ Gemini vision đọc: Traveler form, Scanning sheet, Split Form│
│ File: src/lib/extract.ts — ĐÂY LÀ NƠI DUY NHẤT GỌI AI       │
└────────────────────────────────────────────────────────────┘
                    ↓ output thô, CHƯA lưu
┌────────────────────────────────────────────────────────────┐
│ LỚP 2 — KIỂM TRA RULE-BASED (Validation, không cần AI)      │
│ • Ép enum: O/C chỉ "O"/"C", Machine chỉ "Tbl"/"BF"/"MC"     │
│ • Tự phát hiện + loại dòng trùng lặp y hệt trong 1 lần đọc  │
│ • Đối chiếu barcode thật (traveler#) khi ảnh đủ nét         │
│ Sai rule → tự hạ "low confidence", không tự sửa/đoán        │
└────────────────────────────────────────────────────────────┘
                    ↓ escalate nếu bất thường
┌────────────────────────────────────────────────────────────┐
│ LỚP 3 — GIÁM SÁT RỦI RO (Governance/Risk)                   │
│ • Đối chiếu Traveler: Forecast → Received → Scanned →       │
│   Shipped, cảnh báo lệch > ngưỡng (mặc định 10%)            │
│ • Phát hiện Part# chưa có trong PartControl                 │
│ • Phát hiện traveler bị scan trùng / duplicate trong        │
│   Packing List trước khi duyệt                              │
└────────────────────────────────────────────────────────────┘
                    ↓ CCP (Critical Control Point)
┌────────────────────────────────────────────────────────────┐
│ LỚP 4 — CỔNG DUYỆT CỦA NGƯỜI (Human Gate) — CCP             │
│ KHÔNG có bước nào tự động lưu vào database.                 │
│ Mọi ghi dữ liệu đều qua 1 nút bấm tường minh của người dùng:│
│  "Xác nhận & Lưu" (Raw Material, Finish Good),              │
│  "Xác nhận" (Kho NVL), "Duyệt & Tạo Packing Slip" (P.List)  │
└────────────────────────────────────────────────────────────┘
                    ↓ sau khi đã duyệt
┌────────────────────────────────────────────────────────────┐
│ LỚP 5 — LƯU VẾT / TRUY VẾT (Audit & Traceability)           │
│ • Google Sheets = sổ cái dùng chung, mỗi dòng có createdAt  │
│ • Trang "Truy vết Traveler": xem lại toàn bộ lịch sử 1      │
│   traveler qua cả 4 khâu bất kỳ lúc nào                     │
│ • PS# gắn vào mọi dòng RawMaterial/FinishGood khi duyệt     │
└────────────────────────────────────────────────────────────┘
```

**Nguyên tắc cốt lõi (không thương lượng):** AI (Lớp 1) không bao giờ được
phép ghi thẳng vào Google Sheets. Mọi output của AI bắt buộc phải đi qua ít
nhất Lớp 2 (rule-check) rồi tới Lớp 4 (người bấm nút) mới thành dữ liệu
thật. Đây là lý do mọi trang trong app đều có bước "xem lại trước khi lưu".

## Đối chiếu với khung ISO gốc (D:\16.ISO_CA)

| Khung gốc (4 layer, 11 stage) | AVP Packing Flow (5 layer, rút gọn) |
|---|---|
| LAYER 1 — AI Generation (L6_AGENT) | Lớp 1 — `src/lib/extract.ts` |
| LAYER 2 — AI Validation (Rule Engine, Anomaly Detector...) | Lớp 2 — enum check, dedupe, barcode check |
| LAYER 3 — Governance/Risk (RiskEngine, CCPRegistry) | Lớp 3 — Reconciliation, PartControl missing, duplicate detection |
| LAYER 4 — Human + Audit (HumanGate + RTMEngine) | Lớp 4 (Human Gate) + Lớp 5 (Trace) — tách riêng vì đây là 2 việc khác nhau trong thực tế AVP |
| ImmutableLedger (hash-chain) | Google Sheets + `createdAt` — **chưa có hash-chain thật**, đây là điểm yếu hơn bản gốc, chấp nhận được vì quy mô nhỏ |

## Giới hạn đã biết (không giấu, ghi rõ để không tự nhận sai)

- Chưa có test tự động (khung gốc yêu cầu `pytest` 100% PASS — AVP hiện xác
  nhận bằng cách chạy `npm run build` + `npm run lint` + test tay qua file
  thật, chưa có test suite chính thức).
- ImmutableLedger (Lớp 5) mới là Google Sheets thường — sửa/xóa được, không
  có hash-chain chống giả mạo như khung gốc yêu cầu cho hệ thống lớn.
- Lớp 2 (Validation) mới cover: O/C, Machine, Type, trùng lặp, barcode.
  Chưa cover: Part# format, số lượng âm, ngày tháng phi lý...

---

*ISO_AI_CONTROL v1.0 — AVP Packing Flow — rút gọn từ ISO/IEC 42001:2023*
*Ngày viết: 2026-09-12*
