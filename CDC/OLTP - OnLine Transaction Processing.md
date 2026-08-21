## Vấn đề đặt ra

Các thao tác như chuyển khoản, đặt hàng và cập nhật hồ sơ phải hoàn tất nhanh, không được mất dữ liệu hoặc để hai giao dịch ghi đè sai lên nhau. Hệ thống lưu dữ liệu vận hành vì vậy cần tối ưu cho nhiều lần đọc/ghi nhỏ, đồng thời bảo toàn tính đúng đắn của từng giao dịch.

## OLTP là gì?

**OLTP** (Online Transaction Processing) là kiểu hệ thống database phục vụ các giao dịch nghiệp vụ hàng ngày. Đặc trưng: rất nhiều truy vấn nhỏ, ngắn, đọc/ghi vài dòng mỗi lần, và phải trả lời trong mili-giây.

Các đặc điểm chính:
- Nhiều thao tác **ghi**, mỗi thao tác chạm ít bản ghi
- Yêu cầu **ACID** chặt chẽ — không được mất tiền, không được đặt trùng ghế
- Dữ liệu **chuẩn hóa** (normalized, 3NF) để tránh trùng lặp và bất nhất
- Lưu trữ theo **dòng** (row-oriented), đánh index nhiều để tra cứu theo khóa nhanh
- Chỉ giữ dữ liệu hiện tại, không lưu lịch sử dài

**Đối lập với nó là [[OLAP - OnLine Analytical Processing]]** — hệ thống phân tích: báo cáo doanh thu theo quý, phân tích hành vi người dùng. Ở đây truy vấn ít nhưng mỗi truy vấn quét hàng triệu dòng, chủ yếu đọc, dữ liệu **denormalized** (star schema), lưu theo **cột** để nén tốt và chỉ đọc cột cần thiết.

|            | OLTP                 | OLAP                        |
| ---------- | -------------------- | --------------------------- |
| Mục đích   | Chạy nghiệp vụ       | Phân tích dữ liệu           |
| Truy vấn   | Nhiều, Nhỏ, Nhanh    | Ít, Lớn, Rộng               |
| Ghi/Đọc    | Cân bằng, ghi nhiều  | Chủ yếu đọc                 |
| Lưu trữ    | Theo dòng            | Theo cột                    |
| Người dùng | App/ Người dùng cuối | Analyst, BI, Data Scientist |
