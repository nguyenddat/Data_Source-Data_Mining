## Vấn đề đặt ra

Bạn có một database nghiệp vụ [[OLTP - OnLine Transaction Processing]] và muốn biết "có gì vừa thay đổi" để đẩy sang [[OLAP - OnLine Analytical Processing|kho phân tích]], cache, search engine... Đây là nhu cầu [[Change Data Capture|CDC]]; câu hỏi là: làm sao biết được?

Naive: Định kỳ vào hỏi lại bảng dữ liệu. Nhưng một dòng bị xóa thì nó biến mất, bạn không có cách nào biết nó từng tồn tại. Một dòng bị sửa 5 lần trong 1 phút thì bạn chỉ thấy kết quả cuối cùng.

Cách thông minh hơn: Thay vì chạy vào "kho hàng" đếm lại từ đầu thì thực hiện đọc "sổ nhập kho" - nơi mọi thao tác CRUD đều được ghi lại đúng thứ tự.
## Ý tưởng của log-based CDC

![WAL](assets/wal.jpg)

"Sổ nhập kho" ghi chép mọi thay đổi về dữ liệu trong database gọi là:

| Database | Tên gọi  |
| -------- | -------- |
| MySQL    | binlog   |
| Postgres | WAL      |
| Oracle   | Redo Log |

=> Log-based CDC đọc log thay đổi của dữ liệu, rồi áp dụng cho ứng dụng cần.

## Cơ chế hoạt động

Công cụ CDC (Debezium, Maxwell...) đăng ký với database với danh nghĩa là **một replica** do Database vốn đã stream log cho các replica, nên với nó đây là việc quen thuộc — chỉ là có thêm một "bản sao" nữa xin nhận log.

Ba bước:
1. CDC connector kết nối, xin nhận log từ một vị trí xác định
2. Database stream log về liên tục
3. Connector dịch log → message có nghĩa (dạng row-level), đẩy đi

![log-based cdc example](assets/log-based%20cdc%20example.jpg)

Một message CDC có định dạng:

|Thành phần|Ý nghĩa|
|----------|-------|
|`before`|Giá trị **trước** khi thay đổi|
|`after`|Giá trị **sau** khi thay đổi|
|`op`|Loại thao tác: `c` create, `u` update, `d` delete, `r` read (từ snapshot)|
|`ts_ms`|Thời điểm thay đổi|
|`source`|Metadata: database, table, vị trí trong log|

Có `before` là điều query-based **không bao giờ làm được** — vì lúc bạn query thì giá trị cũ đã bị ghi đè mất rồi.

## Ưu điểm

| Ưu điểm | Giải thích |
|---------|------------|
| Bắt được DELETE | Log có ghi thao tác xóa, dù dòng đã biến mất khỏi bảng |
| Có giá trị cũ | Biết đơn hàng đổi pending → paid, không chỉ biết giờ là paid |
| Không bỏ sót | Sửa 5 lần = 5 message, đúng thứ tự |
| Tải rất nhẹ | Đọc file tuần tự, rẻ hơn nhiều so với quét bảng liên tục |
| Không xâm lấn | Không sửa schema, không thêm cột `updated_at`, không cài trigger — app không hề biết |
| Độ trễ thấp | Gần real-time, không phải chờ chu kỳ quét |

## Nhược điểm

- **Cần quyền ở tầng database**: phải bật chế độ log phù hợp, cấp quyền replication.
- **Gắn chặt với từng loại DB**: mỗi hệ một định dạng log riêng, phải dùng đúng connector.
- **Log bị giữ lại chờ bạn đọc**: database không dám xóa log cho tới khi bạn xác nhận đã đọc xong. Connector chết mà không ai biết → log chất đống → đầy đĩa → database sập. Đây là sự cố kinh điển nhất của Postgres CDC.
- **Log chỉ giữ có hạn**: tụt lại quá xa thì phần cần đọc có thể đã bị xóa, buộc phải snapshot lại từ đầu.
