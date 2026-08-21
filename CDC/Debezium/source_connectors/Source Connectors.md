## Vấn đề đặt ra

[[Debezium]] cần đọc thay đổi từ nhiều database, nhưng mỗi database có cơ chế riêng: MySQL dùng binlog, PostgreSQL dùng logical replication, MongoDB dùng change stream. Vì vậy không thể dùng một bộ đọc log chung cho mọi nguồn.

## Source connector là gì?

**Debezium source connector** là plugin chạy trong Kafka Connect, đọc snapshot và/hoặc log thay đổi của một database rồi tạo change event để Kafka Connect ghi vào Kafka topic. *Source* được gọi theo chiều dữ liệu: database → Kafka.

```text
Database nguồn → Debezium source connector → Kafka Connect → Kafka topic
```

Debezium là logic CDC cài vào Kafka Connect, không phải Kafka broker hay một server độc lập. Các connector khác nhau theo database nhưng cố gắng tạo event có cấu trúc tương tự, giúp consumer ít phụ thuộc vào nguồn.

## Plugin, instance và task

- **Plugin** là mã đã cài, ví dụ Debezium PostgreSQL Connector.
- **Connector instance** là một plugin kèm cấu hình cụ thể: database nào, bảng nào và tên topic nào.
- **Task** là đơn vị thực thi do Kafka Connect quản lý. Connect lưu offset — vị trí đã đọc trong log nguồn, như LSN hoặc binlog position — để instance tiếp tục sau khi khởi động lại.

## Chọn và dùng connector

Chọn connector đúng với database nguồn; PostgreSQL connector không thể đọc MySQL. Debezium hiện có connector cho các database phổ biến như MySQL, PostgreSQL, SQL Server, Oracle, MongoDB, MariaDB và Db2; một số connector khác ở trạng thái incubating, nên API có thể thay đổi.

Source connector chỉ phụ trách **đọc CDC và tạo event**. JSON, Avro hay Protobuf là lựa chọn mã hóa event, không thay đổi cách connector đọc log.

## Nguồn tham khảo

- [Debezium — Source Connectors](https://debezium.io/documentation/reference/stable/connectors/index.html)
- [Confluent — Kafka Connect plugins and workers](https://docs.confluent.io/platform/current/connect/userguide.html)
