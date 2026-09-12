Các rule 
cơ bản là đúng nhưng phải tìm ra mối liện hệ ràng buộc giữa các fiel trước khi làm tiếp. viết ra requirement technical và business. quan hệ 1 PO to many traveler, 1 traveler to 1 part numer , 1 part number to many traveler, 1 part number khác nhau mỗi Pot Number, 1 part number individual có quantity tương ứng trong sheet partcontrol, cột quantity = box* số quantities/ 
1.	Bảng traveler: ViẾt tắt T, các cột tương ứng sẽ đếm từ 1 từ trái sang phải: T1,T2..
T1 TRAVELER = S4 = P3 , S2 PART NUMBER = P2, T3= Pot number =p4, 
2.	Gọi bảng Scanning còn gọi finish good viết tắt S, các cột tương ứng sẽ đếm từ 1 từ trái sang phải S1,S2
S5=P2=T2
S6=P4=T3
S14=P6
S16=P7

3.	Bảng packing slip, viết tắt P , các cột tương ứng sẽ đếm từ 1 từ trái sang phải: P1,P2..
P1=PO Number, P2= Part numbe, P3 traveler number, P4 Pot number (
P6=box, 
P7=Quantity=p6* E (E =each part number has individual quantity and partcontrol of packing slip)
P5=S3+S11+S12+S8+S10
CÓ NHIỀU PART NUMBER ĐANG VIẾT BỊ ĐƯ HỌC THIẾU MÃ GỐC BỊ BIẾN ĐỖ. LIỆT KÊ HẾT RA NHỮNG LOẠI NÀO CHƯA CÓ TRONG DATA. HOẶC TÌM MỐI QUAN HỆ LIÊN QUAN.
Ý NGHĨA BÀI TÁN: TỪ BẢNG TRAVELER, VÀ BẢNG SCANNING XÂY DỰNG BẢNG PACKING 

