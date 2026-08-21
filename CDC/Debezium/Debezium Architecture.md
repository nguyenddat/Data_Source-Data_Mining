## Vấn đề đặt ra

[[Debezium]] chuẩn hóa việc đọc transaction log từ nhiều database, nhưng một pipeline CDC hoàn chỉnh còn phải đưa event đến đâu, lưu chúng ra sao và khôi phục ở vị trí nào khi tiến trình gặp lỗi. Nếu để ứng dụng tự đọc log rồi ghi thẳng đến từng Elasticsearch, cache hay kho dữ liệu, số điểm tích hợp tăng nhanh, khó mở rộng và khó phát lại dữ liệu.

Debezium có thể chạy qua Kafka Connect, dưới dạng Debezium Server hoặc được nhúng vào ứng dụng qua Debezium Engine. Kafka Connect là phương án phổ biến, không phải thành phần bắt buộc của Debezium. Việc chọn cách triển khai quyết định liệu pipeline có event log Kafka ở giữa hay phát event trực tiếp đến đích.

## Các phương án triển khai

| Phương án       | Phù hợp khi                                                                                    | Đặc điểm chính                                                                                                |
| --------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Kafka Connect   | Cần Kafka làm event log, nhiều consumer hoặc nhiều sink có thể thay đổi theo thời gian.        | Debezium là source connector; Kafka Connect quản lý connector/task và sink connector đưa event ra đích.       |
| Debezium Server | Muốn gửi CDC trực tiếp đến một hạ tầng đích được hỗ trợ mà không vận hành Kafka Connect/Kafka. | Ứng dụng độc lập, cấu hình sẵn; chạy source connector và một sink cho Redis, Kinesis, Pulsar, Google Pub/Sub… |
| Debezium Engine | Ứng dụng cần tự kiểm soát vòng đời, xử lý hay cách phát event.                                 | Thư viện Java được nhúng; ứng dụng tự triển khai consumer và cơ chế vận hành.                                 |

## Debezium với Kafka Connect

![[../assets/debezium-kafka architecture.png|Kiến trúc Debezium với Kafka Connect và Apache Kafka]]

Trong cách triển khai thông dụng, Debezium chạy như **source connector** trong Kafka Connect — một dịch vụ độc lập với Kafka broker. Connector kết nối tới database nguồn, chuyển từng thay đổi thành change event và Kafka Connect ghi event vào Kafka. Sau đó, ứng dụng consumer, stream processor hoặc **sink connector** đọc các topic này để cập nhật Elasticsearch, cache, data warehouse hay hệ thống khác.

| Lớp                          | Vai trò                                                                                                                                |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Database nguồn               | Ghi giao dịch nghiệp vụ và duy trì log thay đổi. MySQL dùng binlog; PostgreSQL cung cấp logical replication stream.                    |
| Debezium source connector    | Đọc snapshot và/hoặc log thay đổi, tạo event CDC có dữ liệu trước/sau thay đổi cùng metadata nguồn.                                    |
| Kafka Connect                | Chạy và quản lý connector/task; ở chế độ distributed, các worker phối hợp phân công task và lưu cấu hình, trạng thái, offset bền vững. |
| Apache Kafka                 | Event log trung tâm, đệm giữa producer và consumer, lưu event theo topic/partition để nhiều downstream system tiêu thụ độc lập.        |
| Consumer hoặc sink connector | Đọc topic rồi materialize dữ liệu vào đích, ví dụ Elasticsearch, Infinispan hay kho phân tích.                                         |

## Luồng dữ liệu

1. Connector kết nối đến database và, tùy cấu hình, chụp **snapshot** dữ liệu hiện có để downstream có trạng thái ban đầu.
2. Sau đó connector theo dõi log của database. Một thao tác `INSERT`, `UPDATE` hoặc `DELETE` ở mức dòng được chuyển thành change event.
3. Theo mặc định, thay đổi của một bảng được ghi vào một Kafka topic tương ứng. Có thể dùng topic-routing SMT để đổi tên topic hoặc gộp event của nhiều bảng; cần cân nhắc vì việc gộp làm giảm sự tách biệt và có thể tạo hot partition.
4. Nhiều consumer có thể đọc cùng event log theo consumer group riêng. Sink connector chỉ là một cách tiêu thụ; ứng dụng cũng có thể xử lý event trực tiếp hoặc qua Kafka Streams/Flink trước khi ghi ra đích.

Change event mặc định là một **envelope**, thường gồm `before`, `after`, loại thao tác, thời điểm và thông tin vị trí trong log nguồn. Vì thế event có đủ ngữ cảnh để audit hoặc xử lý `DELETE`, nhưng không phải sink nào cũng chấp nhận cấu trúc này. SMT *Extract New Record State* có thể chỉ chuyển phần `after` cho sink; đổi lại sink mất metadata thao tác và dữ liệu `before` nếu không được cấu hình giữ lại.

## Khả năng khôi phục và mở rộng với Kafka Connect

Kafka Connect ghi **offset** — vị trí connector đã xử lý trong log nguồn — vào lưu trữ bền vững. Sau khi worker hoặc connector khởi động lại, connector đọc lại từ offset đã lưu thay vì quét từ đầu. Offset này do connector định nghĩa: có thể là vị trí binlog, LSN hoặc định danh transaction, không phải Kafka offset thông thường.

Trong chế độ distributed, nhiều worker phân phối connector và task để tăng khả năng sẵn sàng. Tuy nhiên, mức song song thực tế phụ thuộc từng connector và cách database cung cấp log; không phải cứ tăng `tasks.max` là tăng được tốc độ capture. Kafka duy trì thứ tự trong **một partition**, không tạo thứ tự toàn cục giữa nhiều bảng hoặc partition.

## Giới hạn và lưu ý vận hành

- Delivery thường là **at-least-once**: khi lỗi xảy ra giữa lúc phát event và lúc lưu offset, một số event có thể được phát lại. Consumer/sink nên idempotent, chẳng hạn upsert theo khóa chính hoặc lưu vị trí nguồn đã xử lý.
- Snapshot và luồng log phải được phối hợp đúng để không bỏ sót thay đổi xảy ra trong lúc snapshot. Chế độ snapshot, quyền truy cập log và thời gian lưu giữ log phải phù hợp với từng database connector.
- Kafka tạo độ tách rời và khả năng phát lại, nhưng cũng bổ sung hạ tầng cần vận hành: broker, Kafka Connect, internal topics, giám sát độ trễ và cảnh báo khi log nguồn sắp hết retention.
- Chỉ capture các bảng/cột cần thiết, bảo vệ thông tin nhạy cảm trong event, và kiểm tra schema compatibility trước khi cho downstream tiêu thụ thay đổi schema.

## Debezium Server

![[../assets/debezium-server-architecture.jpeg|Kiến trúc Debezium Server phát CDC trực tiếp đến các hệ thống đích]]

Debezium Server là ứng dụng cấu hình sẵn, chứa các source connector của Debezium và phát change event trực tiếp đến một sink được hỗ trợ, như Redis, Amazon Kinesis, Apache Pulsar hoặc Google Pub/Sub. Kiến trúc này phù hợp khi tổ chức đã dùng sẵn một hạ tầng messaging đích, hoặc chỉ có một đường phân phối CDC rõ ràng.

Đổi lại, Debezium Server không tự tạo event log Kafka hay hệ sinh thái Kafka Connect trong hình trên. Nếu nhiều hệ thống cần tiêu thụ độc lập, cần khả năng replay dài hạn, hoặc muốn dễ dàng bổ sung sink mới, Kafka Connect kết hợp Kafka thường phù hợp hơn. Debezium Server vẫn cần một nơi lưu offset bền vững để tiếp tục từ đúng vị trí trong log nguồn sau khi khởi động lại.

## Nguồn tham khảo

- [Debezium Architecture](https://debezium.io/documentation/reference/stable/architecture.html)
- [Debezium Server](https://debezium.io/documentation/reference/stable/operations/debezium-server.html)
- [Kafka Connect Architecture — Confluent Documentation](https://docs.confluent.io/platform/current/connect/design.html)
- [Debezium SQL Server Source Connector — Confluent Documentation](https://docs.confluent.io/kafka-connectors/debezium-sqlserver-source/current/overview.html)
