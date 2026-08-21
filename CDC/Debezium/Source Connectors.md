## Vấn đề đặt ra

[[Debezium]] có thể ghi nhận thay đổi từ nhiều hệ quản trị cơ sở dữ liệu, nhưng mỗi hệ có transaction log và giao thức riêng: MySQL dùng binlog, PostgreSQL dùng logical replication, còn MongoDB có change stream. Kafka Connect cần một thành phần hiểu cơ chế riêng đó để lấy thay đổi ra khỏi database và đưa chúng vào Kafka một cách có thể cấu hình, theo dõi và khôi phục.

**Debezium source connector** là thành phần đảm nhận việc này. Từ *source* được gọi theo góc nhìn của Kafka Connect: database là nơi dữ liệu đi **vào** Kafka.

## Định nghĩa cơ bản

Kafka Connect là framework chạy các tích hợp truyền dữ liệu giữa Kafka và hệ thống bên ngoài. Nó phân loại connector theo chiều dữ liệu:

| Loại                 | Chiều dữ liệu                | Ví dụ                                                             |
| -------------------- | ---------------------------- | ----------------------------------------------------------------- |
| **Source connector** | Hệ thống ngoài → Kafka topic | Debezium đọc thay đổi từ PostgreSQL rồi ghi event vào Kafka.      |
| **Sink connector**   | Kafka topic → hệ thống ngoài | Elasticsearch sink connector đọc Kafka rồi cập nhật search index. |

Debezium cung cấp các **plugin source connector** cho từng database. Khi chạy trong Kafka Connect, một plugin Debezium đọc snapshot và/hoặc log thay đổi của database, tạo **change event**, rồi để Kafka Connect tuần tự hóa và ghi event vào topic.

```text
Database nguồn → Debezium source connector → Kafka Connect → Kafka topic
```

Trong sơ đồ này, Debezium không phải Kafka broker và cũng không tự là một server độc lập. Nó là logic CDC được cài vào Kafka Connect; Kafka Connect là tiến trình/framework thực thi và quản lý connector.

## Plugin, connector instance và task

Từ “connector” có thể chỉ hai cấp khác nhau:

- **Plugin connector:** mã đã cài vào Kafka Connect, ví dụ plugin Debezium PostgreSQL. Plugin chứa logic đọc WAL của PostgreSQL.
- **Connector instance:** một công việc được tạo từ plugin kèm cấu hình cụ thể, ví dụ kết nối tới `postgres-prod`, chỉ theo dõi bảng `public.orders`, và ghi vào các topic có tiền tố `inventory`.

Một connector instance có thể quản lý một hoặc nhiều **task**. Kafka Connect phân task cho các worker để thực thi và lưu offset bền vững. Offset ở đây là vị trí đã đọc trong log nguồn (chẳng hạn PostgreSQL LSN hoặc MySQL binlog position), nhờ đó connector có thể tiếp tục gần đúng vị trí cũ sau khi khởi động lại.

## Connector Debezium làm gì?

1. Kết nối tới database và, nếu được cấu hình, chụp snapshot dữ liệu hiện có.
2. Đọc log thay đổi của database để nhận `INSERT`, `UPDATE` và `DELETE`.
3. Chuyển mỗi thay đổi thành change event có dữ liệu và metadata nguồn.
4. Gửi event cho Kafka Connect để ghi vào Kafka topic; ứng dụng consumer hoặc sink connector có thể tiêu thụ tiếp.

Các connector Debezium hỗ trợ MongoDB, MariaDB, MySQL, PostgreSQL, SQL Server, Oracle, Db2, Cassandra, Spanner và một số connector đang ở trạng thái incubating. Dù cách đọc log khác nhau, Debezium cố gắng tạo event có cấu trúc tương tự để downstream consumer ít phụ thuộc vào database nguồn.

## Khi nào cần source connector Debezium?

Dùng khi database là nguồn thay đổi và Kafka là event log trung gian: đồng bộ sang data warehouse, cập nhật search index/cache, phát event theo outbox pattern, hoặc cho nhiều hệ thống tiêu thụ thay đổi độc lập. Việc chọn connector phải khớp database nguồn; không thể dùng PostgreSQL connector để đọc MySQL vì cơ chế log hoàn toàn khác nhau.

Đây mới là lớp **đọc CDC**. Các lựa chọn như JSON hay Avro chỉ quyết định cách Kafka Connect mã hóa event sau khi connector đã tạo nó.

## Nguồn tham khảo

- [Debezium — Source Connectors](https://debezium.io/documentation/reference/stable/connectors/index.html)
- [Confluent — Kafka Connect concepts](https://docs.confluent.io/platform/current/connect/index.html)
