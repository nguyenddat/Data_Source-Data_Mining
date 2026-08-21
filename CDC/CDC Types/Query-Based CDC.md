## Vấn đề đặt ra

Bạn có một database nghiệp vụ [[OLTP - OnLine Transaction Processing]] và muốn biết "có gì vừa thay đổi" để đẩy sang [[OLAP - OnLine Analytical Processing|kho phân tích]], cache, search engine... Đây là nhu cầu [[Change Data Capture|CDC]]; câu hỏi là: làm sao biết được?

Query-based CDC là cách: vẫn hỏi bằng SQL bình thường, nhưng hỏi khôn hơn — chỉ lấy phần vừa đổi, không quét lại từ đầu.

## Ý tưởng của query-based CDC

![Query-Based CDC](assets/query-based%20cdc.png)

Định kỳ chạy một câu SELECT vào bảng nguồn, lọc theo điều kiện "mới hơn lần trước", rồi đẩy kết quả sang bảng đích qua data pipeline.

=> Query-based CDC phát hiện thay đổi bằng cách **hỏi lại bảng dữ liệu theo chu kỳ**, không đọc log.

## Cơ chế hoạt động

Ba bước:
1. Lưu checkpoint của lần quét trước (timestamp hoặc ID lớn nhất đã lấy)
2. Chạy SELECT với điều kiện `WHERE cột_theo_dõi > checkpoint`
3. Đẩy row lấy được đi, cập nhật checkpoint mới

Cần một cột làm mốc để so sánh:

| Cách đánh dấu | Ví dụ query |
|---|---|
| Timestamp cập nhật | `WHERE updated_at > '2026-08-20 10:00:00'` |
| ID tự tăng | `WHERE id > 10234` |
| Cột version/sequence | `WHERE version > last_version` |
| So sánh toàn bảng (checksum/diff) | Tính hash từng row, so với lần trước — không cần cột riêng nhưng tốn nhất |

Không có cột nào trong 3 loại đầu → không làm được, hoặc phải fallback về diff toàn bảng.

## Ưu điểm

| Ưu điểm | Giải thích |
|---------|------------|
| Dễ triển khai | Chỉ cần quyền SELECT, không cần đụng vào replication/log của database |
| Chạy trên mọi DB | Bất kỳ engine nào hỗ trợ SQL đều dùng được, không phụ thuộc connector riêng theo từng loại DB |
| Không cần hạ tầng thêm | Không cần bật WAL/binlog, không cần Debezium/Kafka Connect |

## Nhược điểm

- **Không bắt được DELETE**: dòng bị xóa biến mất khỏi bảng, câu SELECT không còn thấy nó — trừ khi dùng soft-delete (`deleted_at`) thay vì xóa thật.
- **Mất thay đổi trung gian**: dòng sửa 5 lần giữa 2 lần quét thì chỉ thấy giá trị cuối cùng, không như log-based thấy đủ 5 lần.
- **Không có giá trị cũ**: chỉ biết trạng thái hiện tại (`after`), không biết trước đó là gì — khác với log-based có cả `before`.
- **Tải lên database nguồn**: mỗi chu kỳ là một câu quét bảng, chu kỳ càng ngắn tải càng nặng, ảnh hưởng hệ thống production.
- **Độ trễ phụ thuộc chu kỳ**: không real-time, khoảng cách giữa 2 lần quét là độ trễ tối thiểu.
- **Cần cột hỗ trợ**: bảng phải có sẵn `updated_at`/ID tăng dần/version — schema cũ nhiều khi thiếu, phải sửa schema mới dùng được.
