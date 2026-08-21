## Vấn đề đặt ra

Khi cần đưa dữ liệu từ database giao dịch sang hệ thống phân tích, cache hoặc search index, việc quét lại toàn bộ bảng theo lịch vừa chậm vừa tạo tải lên hệ thống đang phục vụ người dùng. Cách lọc theo `updated_at` còn có thể bỏ sót bản ghi bị xóa hoặc các thay đổi trung gian.

## Change Data Capture là gì?

**CDC** (Change Data Capture) là kỹ thuật:
- B1: Phát hiện và ghi nhận mọi thay đổi dữ liệu (INSERT / UPDATE / DELETE) trong một database
- B2: Đẩy các thay đổi đó sang hệ thống khác gần như real-time — thay vì định kỳ quét lại toàn bộ bảng.

## Vì sao cần?

ETL truyền thống thường full load hoặc query theo updated_at mỗi vài giờ: nặng, chậm, và bỏ sót bản ghi bị xóa. 

CDC chỉ chuyển phần delta, độ trễ tính bằng giây và gần như không gây tải lên DB nguồn.

CDC là cầu nối giữa hai thế giới đối nghịch nhau: [[OLTP - OnLine Transaction Processing]] tối ưu để ghi từng dòng nhanh, còn [[OLAP - OnLine Analytical Processing]] tối ưu để đọc/tổng hợp hàng triệu dòng. Chạy thẳng query phân tích (SUM, GROUP BY...) lên OLTP sẽ quét quá nhiều dòng, giết hiệu năng transaction đang chạy sản xuất. CDC giải quyết bằng cách bắt thay đổi ngay tại OLTP rồi đẩy sang OLAP — mỗi hệ làm đúng việc nó mạnh nhất, không hệ nào phải gánh việc của hệ kia.

## Triển khai thế nào?

Ba cách chính, đánh đổi nhau giữa độ đầy đủ, tải lên DB nguồn, và độ dễ triển khai:

| Tiêu chí                | [[Log-Based CDC]]            | [[Trigger-Based CDC]]                  | [[Query-Based CDC]]             |
| ----------------------- | ---------------------------- | -------------------------------------- | ------------------------------- |
| Bắt được DELETE         | Có                           | Có                                     | Không (trừ soft-delete)         |
| Có giá trị cũ (before)  | Có                           | Có                                     | Không                           |
| Mất thay đổi trung gian | Không                        | Không                                  | Có                              |
| Tải lên DB nguồn        | Rất thấp                     | Cao (thêm ghi mỗi write)               | Trung bình-cao, tùy chu kỳ poll |
| Độ trễ                  | Thấp, gần real-time          | Thấp, đồng bộ theo transaction         | Phụ thuộc chu kỳ poll           |
| Quyền cần có            | Replication                  | Tạo trigger/function                   | Chỉ cần SELECT                  |
| Xâm lấn schema/DB nguồn | Không                        | Có (trigger, shadow table)             | Có thể cần thêm cột mốc         |
| Portable qua nhiều DB   | Thấp (mỗi DB một log format) | Trung bình (cú pháp trigger khác nhau) | Cao (SQL chuẩn)                 |
| Bắt được DDL            | Một số tool hỗ trợ           | Không                                  | Không                           |

**Công cụ thường dùng:** 
- Debezium (chạy trên Kafka Connect, được dùng nhiều nhất)
- AWS DMS
- Maxwell
- Oracle GoldenGate

## Ứng dụng điển hình
- Đồng bộ [[OLTP - OnLine Transaction Processing]] → [[OLAP - OnLine Analytical Processing|data warehouse]] / data lake — ứng dụng phổ biến nhất: OLTP xử lý giao dịch, CDC bắt thay đổi, OLAP nhận data để phân tích mà không đụng vào DB sản xuất
- Cập nhật search index (Elasticsearch) và invalidate cache
- Giao tiếp giữa microservices qua **outbox pattern** — ghi event vào bảng outbox trong cùng transaction với business data, CDC đọc bảng đó và publish, giải quyết bài toán dual-write
- Audit trail, replicate cross-region, migration không downtime
