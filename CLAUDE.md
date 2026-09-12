# CLAUDE.md -- AVP PACKING FLOW
# Project identity + session rules (mau don gian, tham khao D:\Ops_Ai)

## IDENTITY
Project:      AVP Packing Flow
Owner:        Andy Phan (Viet), Maple Leaf Group
Path:         D:\AVP_AI (web app: D:\AVP_AI\webapp)
Stack:        Next.js + TypeScript + Google Sheets (database) + Gemini API (vision OCR)
Muc tieu:     Thay 2 file Excel (DAILY LOG CHECK SHEET.xlsm, PACKING SLIPS.xlsm)
              bang web app 4 khau: Email/Du bao -> Kho Nguyen Vat Lieu ->
              Finish Good (Scan) -> Packing List, cho quy trinh gia cong
              oc vit AVP <-> Infasco.

## SESSION START (bat buoc moi phien moi)
1. Doc file nay (tu dong).
2. Doc LATEST_SESSION.md (o thu muc goc D:\AVP_AI) -- tom tat da lam gi /
   con lai gi / co GAP nao dang mo.
3. Neu co lam viec voi web app: kiem tra server con chay khong
   (`curl http://localhost:3000`) va tunnel cloudflared con song khong
   (link doi neu tunnel khoi dong lai).
4. Neu LATEST_SESSION.md co "Viec dang mo (chua quyet dinh)" -- hoi lai
   Owner truoc khi tu suy dien va lam tiep.

## SESSION END (truoc khi ket thuc mot hang muc cong viec lon)
1. Cap nhat lai LATEST_SESSION.md: hoan thanh gi / GAP nao con / uu tien
   phien sau la gi.
2. KHONG tu commit/push git khi Owner chua xac nhan (quy tac toan cuc).

## NGUYEN TAC TUYET DOI
- KHONG de AI (Gemini) ghi thang vao Google Sheets -- moi output AI phai
  qua nguoi xem lai + bam nut xac nhan (xem ISO_AI_CONTROL.md).
- KHONG tu doan/dien so lieu nghiep vu (vi du: da tung bia sai truong
  "Oprtr" cua traveler 705988 -- xem BRS_TRS.md muc 4, khong lap lai).
- KHONG xoa du lieu that trong Google Sheet ma chua hoi Owner truoc.

## FILE THAM KHAO
- LATEST_SESSION.md -- bao cao/tri nho giua cac phien (doc dau tien).
- ISO_AI_CONTROL.md -- mo hinh 5 lop kiem soat AI.
- BRS_TRS.md -- business/technical requirements, xac minh voi du lieu that.
