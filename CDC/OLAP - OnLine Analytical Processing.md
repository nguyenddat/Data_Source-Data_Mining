## Vấn đề đặt ra

Phân tích doanh thu nhiều năm, hành vi khách hàng hay tỷ lệ rời bỏ thường phải quét và tổng hợp hàng triệu đến hàng tỷ dòng. Chạy các truy vấn này trên [[OLTP - OnLine Transaction Processing|OLTP]] có thể cạnh tranh tài nguyên với giao dịch đang phục vụ sản xuất.

## OLAP là gì?

**OLAP** (Online Analytical Processing) là kiểu hệ thống database được tối ưu cho việc **phân tích** dữ liệu lớn. Khác với OLTP xử lý từng giao dịch, OLAP quét và tổng hợp hàng triệu đến hàng tỷ dòng cho mỗi truy vấn.

Tại sao nó nhanh với dữ liệu lớn:
- Lưu theo cột thay vì theo dòng. Query `SELECT SUM(revenue) FROM sales` chỉ đọc đúng cột `revenue`, bỏ qua 50 cột còn lại. 
- Dữ liệu cùng cột lại cùng kiểu nên nén rất tốt (thường 5–10x).
- **Vectorized execution** — xử lý cả khối vài nghìn giá trị một lúc thay vì từng dòng.
- **Xử lý song song (MPP)** — chia query ra nhiều node cùng chạy.
- **Denormalization** — dùng star schema (bảng fact ở giữa, các bảng dimension xung quanh) để giảm số lần JOIN.

**Tradeoffs:** 
- Ghi từng dòng rất kém hiệu quả, UPDATE/DELETE đắt đỏ, không phù hợp cho truy vấn điểm kiểu "lấy đơn hàng có id = 12345".
- Thường không đảm bảo ACID chặt như OLTP.

**Các dạng OLAP:**
- **ROLAP** — chạy trực tiếp trên quan hệ (Snowflake, BigQuery, ClickHouse, Redshift). Phổ biến nhất hiện nay.
- **MOLAP** — tính trước và lưu vào cube đa chiều (SSAS, Essbase). Query cực nhanh nhưng cứng nhắc, tốn công build.
- **HOLAP** — kết hợp cả hai.
