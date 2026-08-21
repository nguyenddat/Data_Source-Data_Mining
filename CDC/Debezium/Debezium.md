## Vấn đề đặt ra

[[Log-Based CDC|Log-based CDC]] cho phép đọc thay đổi từ transaction log mà không phải liên tục quét các bảng OLTP. Tuy nhiên, mỗi database có cơ chế và định dạng log riêng: MySQL có binlog, PostgreSQL có logical replication stream. Tự viết một chương trình để đọc, theo dõi vị trí đã xử lý, chuyển đổi bản ghi thành sự kiện và khôi phục sau khi lỗi là phức tạp và khó vận hành.

**Debezium** cung cấp các connector sẵn có để chuẩn hóa phần việc này: lấy thay đổi theo từng dòng từ database nguồn và phát chúng thành một luồng sự kiện để các hệ thống khác tiêu thụ.

## Debezium là gì?

Debezium là một nền tảng CDC mã nguồn mở gồm các dịch vụ phân tán và connector cho nhiều database. Nó ghi nhận các thay đổi `INSERT`, `UPDATE`, `DELETE` ở cấp dòng theo thứ tự xảy ra rồi phát chúng thành **change event**.

Debezium không thay database nguồn thành message broker. Nó là lớp chuyển đổi giữa transaction log của database và hệ thống nhận sự kiện. Các connector phổ biến gồm PostgreSQL, MySQL, SQL Server, MongoDB và Oracle.

## Khả năng chính

- **CDC dựa trên log:** ghi nhận thay đổi với độ trễ thấp mà không cần polling dày đặc, thêm cột `updated_at` hay sửa data model; có thể capture cả `DELETE`.
- **Snapshot:** tạo trạng thái ban đầu khi cần, rồi chuyển sang đọc log; một số connector hỗ trợ incremental snapshot được kích hoạt lúc đang chạy để backfill dữ liệu theo từng phần.
- **Event giàu ngữ cảnh:** tùy database và cấu hình, event có `before`/`after`, loại thao tác, vị trí log, transaction ID hoặc truy vấn gây ra thay đổi. Schema history giúp diễn giải event đúng theo cấu trúc bảng tại thời điểm thay đổi.
- **Kiểm soát dữ liệu:** lọc schema, bảng và cột bằng include/exclude list; mask cột nhạy cảm trước khi event rời connector.
- **Tích hợp và vận hành:** có SMT sẵn cho routing, lọc và làm phẳng event; đa số connector xuất metric qua JMX. Debezium có thể chạy với Kafka Connect, Debezium Server hoặc được nhúng bằng Engine.

Khả năng và giới hạn không hoàn toàn giống nhau giữa các connector. Ví dụ, khả năng lấy trạng thái cũ, metadata transaction, snapshot hoặc mức song song phụ thuộc vào cơ chế log mà database cung cấp. Các connector phổ biến có ngữ nghĩa **at-least-once**, vì vậy hệ thống nhận cần chịu được event trùng lặp.

## Khi nào nên dùng?

- Đồng bộ dữ liệu từ OLTP sang kho phân tích, search index hoặc cache gần thời gian thực.
- Xây dựng read model riêng cho dashboard mà không đặt truy vấn tổng hợp nặng lên database giao dịch.
- Triển khai **outbox pattern**: ứng dụng ghi business data và outbox record trong cùng transaction; Debezium phát outbox record thành event để tránh dual-write.
- Theo dõi thay đổi để audit, tích hợp hệ thống hoặc migration với thời gian dừng thấp.

## Học tiếp

Xem [[Debezium Architecture|kiến trúc Debezium]] để hiểu các cách triển khai bằng Kafka Connect, Debezium Server hoặc Debezium Engine. Các chủ đề nên tách thành ghi chú riêng: connector cho từng database; cấu trúc change event; snapshot; và vận hành connector.

## Nguồn tham khảo

- [Debezium Documentation](https://debezium.io/documentation/reference/stable/)
- [Debezium Features](https://debezium.io/documentation/reference/stable/features.html)
- [Debezium Architecture](https://debezium.io/documentation/reference/stable/architecture.html)
- [Apache Kafka Connect](https://kafka.apache.org/documentation/#connect)
- [Confluent: Debezium PostgreSQL Source Connector](https://docs.confluent.io/kafka-connectors/debezium-postgres-source/current/overview.html)
