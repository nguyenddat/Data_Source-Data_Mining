## 1. Vấn đề đặt ra

[[Source Connectors]] cần đưa thay đổi của PostgreSQL thành event mà không bỏ sót khoảng thời gian giữa dữ liệu hiện có và các giao dịch mới. PostgreSQL connector phối hợp snapshot với luồng logical replication để consumer có cả trạng thái ban đầu lẫn các thay đổi tiếp theo.

## 2. Quyền truy cập

Connector kết nối như một PostgreSQL client có quyền đọc luồng replication. Nên dùng một tài khoản replication riêng với đúng các quyền cần thiết, thay vì cấp `superuser`; đây là nguyên tắc giới hạn quyền cho thành phần có thể đọc thay đổi dữ liệu.

## 3. Thu thập dữ liệu từ PostgreSQL

Connector thu thập dữ liệu theo hai cách bổ sung: snapshot đọc trạng thái của bảng, còn streaming đọc các thay đổi đã commit từ WAL. Snapshot tạo trạng thái ban đầu hoặc backfill; nó không phải là thao tác đọc WAL.

### 3.1. Snapshot

Lần chạy đầu, connector thường tạo **snapshot nhất quán** vì WAL không lưu toàn bộ lịch sử dữ liệu. Nó ghi nhận vị trí log, đọc dữ liệu tại thời điểm đó, rồi lưu việc snapshot đã hoàn tất. Sau snapshot, connector bắt đầu stream từ vị trí log đã ghi nhận, nhờ vậy không bỏ sót thay đổi xảy ra trong lúc chụp dữ liệu.

`snapshot.mode` quyết định khi nào làm việc này: `initial` là mặc định; `no_data` chỉ đọc log khi chắc dữ liệu cần thiết vẫn còn trong log; `when_needed` snapshot khi offset cũ không còn dùng được. Nếu snapshot ban đầu thất bại trước khi hoàn tất, connector làm lại snapshot khi khởi động.

### 3.2. Ad-hoc snapshot

Sau lần đầu, có thể yêu cầu snapshot lại một hay nhiều bảng bằng signal, chẳng hạn khi thêm bảng vào phạm vi capture hoặc cần dựng lại topic. Connector ghi dữ liệu snapshot bổ sung vào topic hiện có, nên consumer phải chịu được dữ liệu lặp hoặc dữ liệu được nạp lại.

### 3.3. Incremental snapshot

Incremental snapshot đọc từng bảng theo các chunk, thường dựa trên khóa chính, trong khi streaming vẫn tiếp tục. Debezium dùng watermark và bộ đệm để xử lý trường hợp một dòng vừa được stream thay đổi vừa xuất hiện trong chunk snapshot, rồi chỉ phát kết quả theo thứ tự hợp lý. Cách này phù hợp để backfill bảng lớn mà không dừng luồng thay đổi.

### 3.4. Streaming changes

Phần lớn thời gian connector nhận các giao dịch đã commit qua replication protocol và logical decoding của PostgreSQL. Mỗi thay đổi được chuyển thành event `create`, `update` hoặc `delete` cùng **LSN** — vị trí của nó trong transaction log.

Kafka Connect lưu LSN đã xử lý làm offset. Sau restart, connector yêu cầu PostgreSQL gửi các thay đổi ngay sau LSN đó; do đó snapshot không phải lặp lại sau mỗi lần dừng bình thường.

## 4. Phát event vào Kafka

Sau khi connector tạo change event, Kafka Connect ghi event vào topic. Tên topic và transaction metadata là đặc điểm của cách event được xuất sang Kafka, không thay đổi cách PostgreSQL được đọc.

### 4.1. Tên topic

Mặc định, change event của mỗi bảng đi vào một topic theo dạng `topicPrefix.schemaName.tableName`. Việc tách topic theo bảng giúp consumer chỉ đăng ký dữ liệu mình cần; có thể định tuyến lại tên topic khi mô hình tiêu thụ yêu cầu.

### 4.2. Metadata transaction (tùy chọn)

Khi bật transaction metadata, connector phát event `BEGIN` và `END`, đồng thời thêm thông tin transaction vào từng change event. Metadata gồm định danh transaction và thứ tự event, hữu ích khi consumer cần nhận biết ranh giới của một giao dịch nhiều thay đổi.

## 5. Data change events

Mỗi thay đổi cấp dòng được connector đưa vào Kafka dưới dạng một record gồm **key** và **value**. `INSERT`, `UPDATE` và `DELETE` tạo event thay đổi; snapshot tạo event đọc (`r`). Cấu trúc cụ thể khi tuần tự hóa phụ thuộc vào Kafka Connect converter: JSON có thể mang cả `schema` và `payload`, còn khi dùng schema registry, consumer lấy schema theo ID đã đăng ký. Vì thế consumer không nên phụ thuộc vào chuỗi JSON ví dụ mà nên phụ thuộc vào schema và các trường của envelope.

### 5.1. Key: định danh dòng

Key thường là khóa chính của dòng, nên các event của cùng một dòng có cùng key và phù hợp với Kafka log compaction. Nếu bảng không có khóa chính, connector có thể dùng khóa duy nhất khi `REPLICA IDENTITY` là `FULL` hoặc `USING INDEX`; nếu không có khóa định danh, key là `null`. Có thể cấu hình `message.key.columns` để chọn cột khác làm key, nhưng các cột khóa chính hoặc khóa duy nhất luôn được giữ trong key dù bị loại khỏi phần dữ liệu bằng danh sách include/exclude cột.

### 5.2. Value: envelope của thay đổi

Value chứa envelope với các trường quan trọng sau:

| Trường                    | Ý nghĩa                                                                                                                                    |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `before`                  | Trạng thái dòng trước thay đổi; là `null` khi tạo mới.                                                                                     |
| `after`                   | Trạng thái dòng sau thay đổi; là `null` khi xóa.                                                                                           |
| `source`                  | Metadata nguồn: connector, database, schema, bảng, `txId`, LSN, thời điểm thay đổi và cờ snapshot.                                         |
| `op`                      | Loại thao tác: `c` (create), `u` (update), `d` (delete), `r` (read từ snapshot), `t` (truncate) hoặc `m` (logical message).                |
| `ts_ms`, `ts_us`, `ts_ns` | Thời điểm task Kafka Connect xử lý event. So sánh `payload.source.ts_ms` với `payload.ts_ms` để ước tính độ trễ từ database đến connector. |

Ví dụ với `UPDATE` thay đổi tên khách hàng, `before` chứa thông tin định danh trước khi đổi, `after` chứa dòng mới và `op` là `u`. Khi snapshot đọc cùng dòng, `after` cũng là trạng thái đã đọc nhưng `op` là `r`; consumer cần coi đây là nạp trạng thái, không phải một lệnh `INSERT` mới.

### 5.3. `REPLICA IDENTITY` quyết định dữ liệu cũ

`REPLICA IDENTITY` là cấu hình theo bảng của PostgreSQL, quyết định phần giá trị cũ mà logical decoding có thể cung cấp cho `UPDATE` và `DELETE`, do đó trực tiếp giới hạn trường `before` của Debezium:

- `DEFAULT` (mặc định): chỉ có các cột khóa chính; nếu bảng không có khóa chính thì tương đương `NOTHING`.
- `USING INDEX index_name`: dùng các cột trong unique index được chọn.
- `FULL`: có giá trị cũ của mọi cột.
- `NOTHING`: không có dữ liệu cũ.

Nếu consumer cần biết toàn bộ ảnh cũ của một dòng, đặt `ALTER TABLE ... REPLICA IDENTITY FULL`; nếu chỉ cần định danh để cập nhật projection, `DEFAULT` thường đủ. Với bảng không có khóa chính, cần `FULL` để event `DELETE` có `before` cho consumer nhận diện dòng bị xóa.

### 5.4. Xóa, đổi khóa và thao tác cấp bảng

Với `DELETE`, event có `before`, `after: null` và `op: "d"`. Ngay sau đó connector mặc định phát **tombstone**: cùng key nhưng value `null`. Tombstone cho phép log compaction xóa các record cũ của key đó; consumer phải phân biệt tombstone với envelope delete.

Đổi khóa chính không được biểu diễn bằng một event `UPDATE`: connector phát `DELETE` và tombstone theo key cũ, rồi `CREATE` theo key mới. Header `__debezium.newkey` trên event delete và `__debezium.oldkey` trên event create liên kết hai phía của lần đổi khóa này.

`TRUNCATE` tạo event `op: "t"` không có key; một lệnh tác động nhiều bảng tạo một event cho mỗi bảng. Vì record không có key, topic nhiều partition không bảo đảm thứ tự giữa truncate và các event của bảng; cần một partition nếu thứ tự đó là yêu cầu nghiệp vụ. Event `m` là logical message do `pg_logical_emit_message` ghi trực tiếp vào WAL; Debezium chỉ hỗ trợ nó khi dùng `pgoutput` trên PostgreSQL 14 trở lên.

## 6. Data type mappings

Khi [[PostgreSQL Connector]] biến một thay đổi thành event, nó giữ hình dạng của dòng: mỗi cột thành một field trong `before` hoặc `after`. Nhưng field không mang nguyên kiểu PostgreSQL; connector ánh xạ nó thành **literal type** của Kafka Connect (`INT64`, `STRING`, `BYTES`, `MAP`, `STRUCT`...) và, khi cần, gắn **semantic type** để nói rõ ý nghĩa của giá trị. Cơ chế này giúp consumer biết một `INT64` là số nguyên, microsecond kể từ epoch hay một khoảng thời gian.

### 6.1. Đọc kiểu dữ liệu trong event

| Nhóm cột nguồn | Dạng trong event | Ý nghĩa cho consumer |
| --- | --- | --- |
| Boolean, số nguyên, số thực | `BOOLEAN`, `INT16`/`INT32`/`INT64`, `FLOAT32`/`FLOAT64` | Ánh xạ trực tiếp; `SERIAL` được xem là số nguyên, không mang ý nghĩa tự tăng riêng. |
| Chuỗi (`CHAR`, `VARCHAR`, `CITEXT`) | `STRING` | Giá trị văn bản. |
| Nhị phân (`BYTEA`, bit string) | `BYTES` hoặc chuỗi mã hóa | Bit string dài có semantic type `Bits`; consumer không nên coi byte của nó là text. |
| `JSON`/`JSONB`, `XML`, `UUID`, `LTREE`, range, địa chỉ mạng | Chủ yếu `STRING` | Đây là biểu diễn chuỗi; semantic type (nếu có) giúp phân biệt JSON, UUID hay cây `LTREE`. `JSONB` không tự trở thành object lồng nhau trong Kafka Connect. |
| `ENUM` | `STRING` + semantic type `Enum` | Schema có danh sách giá trị được phép theo thứ tự logic của PostgreSQL; hữu ích để consumer nhận biết enum thay vì text tự do. |
| `POINT`, PostGIS | `STRUCT` | `POINT` có `x`, `y`; `GEOMETRY`/`GEOGRAPHY` có `srid` và `wkb`. Consumer cần hiểu cấu trúc này để dùng được tọa độ. |
| pgvector | `ARRAY` hoặc `STRUCT` | `VECTOR`/`HALFVEC` là mảng số; `SPARSEVEC` gồm kích thước và map chỉ số–giá trị. |
| `TSVECTOR` | `STRING` + semantic type `Tsvector` | Chuỗi lexeme đã chuẩn hóa, kèm vị trí cho full-text search. |

Vì converter quyết định cách schema được tuần tự hóa ra JSON, Avro hay Protobuf, consumer nên dựa vào schema/literal type và semantic type thay vì chỉ nhìn vào biểu diễn cuối cùng. Cùng một chuỗi JSON có thể là text thường, UUID hoặc JSONB.

### 6.2. Thời gian và độ chính xác

Với `DATE`, `TIME` và `TIMESTAMP` không kèm timezone, connector biểu diễn thời gian dưới dạng số đếm từ epoch hoặc từ nửa đêm. Theo cách mặc định, độ chính xác của cột quyết định đơn vị: precision 1–3 dùng milliseconds, 4–6 dùng microseconds. Nhờ vậy event giữ được độ chính xác mà cột có.

Một số consumer chỉ hiểu logical type chuẩn của Kafka Connect hoặc chuỗi ISO-8601. Connector có các lựa chọn biểu diễn phù hợp, nhưng đánh đổi là chế độ chuẩn Kafka Connect chỉ giữ milliseconds, còn biểu diễn chuỗi chuyển trách nhiệm parse sang consumer. Chế độ nanoseconds chỉ đổi đơn vị truyền tải; PostgreSQL vẫn chỉ có độ chính xác tối đa microsecond.

`TIMESTAMPTZ` và `TIMETZ` được phát dưới dạng chuỗi có timezone ở GMT. `TIMESTAMP` không có timezone được quy đổi nhất quán theo UTC, không bị ảnh hưởng bởi timezone của JVM chạy connector. Giá trị `TIMESTAMP` vô cực của PostgreSQL được dùng sentinel số nguyên; consumer phải nhận biết chúng là vô cực, không phải mốc thời gian bình thường.

`INTERVAL` mặc định là số microsecond **xấp xỉ** vì một tháng được quy đổi theo số ngày trung bình. Khi nghiệp vụ cần phân biệt chính xác tháng với số ngày, connector có thể biểu diễn interval bằng chuỗi ISO-8601 để giữ nguyên cấu trúc năm–tháng–ngày.

### 6.3. Số thập phân và cấu trúc linh hoạt

Mặc định, `DECIMAL`, `NUMERIC` và `MONEY` dùng logical type Kafka Connect `Decimal`: literal type là `BYTES`, còn `scale` nằm trong schema. Cách này bảo toàn độ chính xác — lựa chọn quan trọng với tiền và số liệu kế toán. Nếu `NUMERIC`/`DECIMAL` không giới hạn scale, mỗi giá trị có thể có scale khác nhau; event khi đó là `STRUCT` `VariableScaleDecimal` gồm `scale` và `value`.

Connector cũng có thể phát các số này thành `FLOAT64` hoặc `STRING`. `FLOAT64` đơn giản cho consumer nhưng có thể làm tròn; chuỗi giữ được dạng số nhưng consumer phải parse. `NaN` vì thế cũng xuất hiện theo dạng của lựa chọn biểu diễn, không phải một số hữu hạn.

`HSTORE` có thể xuất hiện như chuỗi JSON hoặc `MAP`. Domain được diễn giải theo kiểu cơ sở, nên consumer thường xử lý được như cột nguyên thủy; tuy vậy ràng buộc length/scale của domain lồng nhau có thể không còn trong schema. Các composite/custom type không có ánh xạ mặc định cần converter riêng — đây là ranh giới giữa kiểu mà connector hiểu sẵn và kiểu do ứng dụng tự định nghĩa.

### 6.4. Khi event không có toàn bộ giá trị

Giá trị lớn có thể được PostgreSQL lưu bằng TOAST. Với replica identity thông thường, nếu một cột TOAST không đổi trong `UPDATE`, PostgreSQL không gửi lại giá trị đó qua logical decoding; Debezium không đọc bù trực tiếp từ bảng vì có thể lấy nhầm phiên bản cạnh tranh. Thay vào đó, event dùng placeholder `unavailable.value.placeholder`. Vì vậy placeholder có nghĩa là **giá trị không được nguồn cung cấp**, không phải giá trị mới của cột. Khi event cần đầy đủ ảnh trước/sau cho các cột lớn, replica identity của bảng phải cho phép PostgreSQL gửi các giá trị đó.

Connector cũng cố đưa default của cột vào Kafka schema để hỗ trợ schema evolution. Default này là metadata, có thể cập nhật sớm, muộn hoặc bị lỡ nhịp so với schema nguồn; giá trị thật trong event mới là nguồn chân lý. Consumer không nên tự điền trường thiếu bằng default từ schema mà không xét ngữ nghĩa của event.

## 7. Custom converters

Các ánh xạ mặc định chỉ bao phủ những kiểu mà Debezium đã biết. Với kiểu do ứng dụng tự tạo — điển hình là composite type — connector không tự diễn giải được và mặc định phát `null`. Custom converter là điểm mở rộng để định nghĩa cách biến giá trị chưa biết thành schema và value Kafka Connect.

Luồng xử lý vẫn không đổi: logical decoding cung cấp giá trị cột, converter nhận giá trị đó và trả về biểu diễn Kafka Connect, rồi connector đặt kết quả vào `before`/`after` của change event như một cột bình thường. Converter chỉ được gọi cho kiểu chưa biết khi connector cho phép đưa unknown datatype vào xử lý. Dạng đầu vào phụ thuộc plug-in logical decoding: `pgoutput` cung cấp chuỗi, còn `decoderbufs` cung cấp `byte[]`. Vì thế converter nên tạo schema ổn định và tách việc parse định dạng đầu vào khỏi hợp đồng dữ liệu mà consumer sử dụng.

Chỉ dùng converter khi kiểu nghiệp vụ thật sự cần xuất hiện trong CDC. Nếu kiểu đó có thể được biểu diễn rõ ràng bằng cột nguyên thủy hoặc `JSONB`, cách đó thường giúp schema event dễ tiêu thụ và vận hành hơn.

## 8. Connector setup

### 8.1. Setting up PostgreSQL

Để [[PostgreSQL Connector]] đọc thay đổi, PostgreSQL phải tạo được luồng **logical decoding → replication slot → output plug-in → publication**. Slot giữ vị trí LSN và WAL mà connector chưa xử lý, nên đây vừa là cơ chế không bỏ sót event, vừa là nguồn rủi ro đầy đĩa khi connector bị chậm.

| Thành phần | Yêu cầu chính |
| --- | --- |
| Logical decoding | `wal_level=logical`; đủ số slot và WAL sender cho các connector. |
| Replication slot | Mỗi connector có slot riêng, không dùng chung; theo dõi độ trễ LSN và WAL. |
| Plug-in | Slot và connector dùng cùng plug-in; `pgoutput` có sẵn từ PostgreSQL 10. |
| Publication | Với `pgoutput`, publication phải chứa đúng bảng cần capture. |
| Quyền và mạng | Replication user có `LOGIN`, `REPLICATION`, `SELECT` cần thiết; `pg_hba.conf` chỉ mở cho host connector. |

Nên dùng replication user riêng thay vì `superuser`. Nếu Debezium tự tạo publication, user cần thêm quyền `CREATE` và quyền sở hữu bảng; để least privilege, DBA nên tạo publication trước.

Với cloud, kiểm tra dịch vụ có hỗ trợ logical decoding và `pgoutput`. Khi có nhiều connector, mỗi connector cần **slot và publication riêng**. Khi upgrade hoặc failover, không được tùy tiện xóa slot: slot mới chỉ có thay đổi kể từ lúc nó được tạo, nên có thể tạo lỗ hổng CDC.

### 8.2. Deploy Connector

Triển khai gồm: cài archive Debezium PostgreSQL vào `plugin.path` của mọi Kafka Connect worker, rồi restart worker để Connect nhận JAR. Với container bất biến, build image chứa plug-in thay vì chép JAR lúc chạy.

Sau đó đăng ký cấu hình qua Kafka Connect REST API. Các trường cốt lõi là:

| Trường                                         | Vai trò                                                                           |
| ---------------------------------------------- | --------------------------------------------------------------------------------- |
| `name`, `connector.class`                      | Định danh connector và chọn `io.debezium.connector.postgresql.PostgresConnector`. |
| `database.*`                                   | Điểm kết nối và credential của PostgreSQL.                                        |
| `plugin.name`, `slot.name`, `publication.name` | Khớp với plug-in, slot và publication đã chuẩn bị ở Bước 0.                       |
| `topic.prefix`                                 | Namespace cho các topic và schema mà connector tạo.                               |
| `table.include.list`                           | Giới hạn bảng cần capture; phải phù hợp với publication.                          |

Khi Kafka Connect nhận cấu hình, PostgreSQL connector luôn chạy **một task**; tăng `tasks.max` không làm đọc WAL song song. Task kết nối database, snapshot ban đầu (nếu cần), rồi stream event vào Kafka. Sau khi đăng ký, kiểm tra connector đã khởi động và các topic có event như phạm vi capture mong đợi.

### 8.3. Monitoring

Debezium xuất metric qua JMX, tách cho snapshot và streaming. Dùng `custom.metric.tags` để observability stack nhận diện connector ổn định, thay vì phụ thuộc MBean name có thể đổi khi cấu hình đổi.

| Theo dõi | Metric chính | Ý nghĩa |
| --- | --- | --- |
| Snapshot | `SnapshotRunning`, `SnapshotCompleted`, `RemainingTableCount`, `RowsScanned` | Snapshot có tiến triển hay bị dừng. |
| Streaming | `Connected`, `MilliSecondsBehindSource`, `LastEvent` | Connector còn kết nối và độ trễ so với nguồn. Lag chịu ảnh hưởng clock skew. |
| Lỗi và backpressure | `NumberOfErroneousEvents`, `QueueRemainingCapacity` | Event lỗi hoặc hàng đợi gần đầy. |
| Replication slot | `confirmed_flush_lsn`, `restart_lsn`, `wal_status` trong `pg_replication_slots` | Connector có bắt kịp không và WAL có nguy cơ bị giữ quá lâu/mất hay không. |

Xem realtime qua JMX của Kafka Connect; thông dụng nhất là JMX Exporter → Prometheus → Grafana. Kiểm tra nhanh bằng JConsole/JDK Mission Control. Trạng thái connector/task xem qua `GET /connectors/<name>/status`, còn trạng thái slot/WAL xem trong `pg_replication_slots`.

Các counter của Debezium reset khi task restart, nên cần alert theo xu hướng và kết hợp với trạng thái Kafka Connect. Nếu database đang nhàn rỗi, việc lâu không có event không tự nó là lỗi.

> Khi connector báo lỗi hoặc dừng, xem [Behavior when things go wrong](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-when-things-go-wrong) để xử lý các tình huống cấu hình, PostgreSQL, cluster và Kafka Connect không khả dụng.

## 9. Nguồn tham khảo

- [Debezium — How the PostgreSQL connector works](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#how-the-postgresql-connector-works)
- [PostgreSQL — Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [Debezium — Data change events](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-events)
- [Debezium — Data type mappings](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-data-types)
- [Debezium — Custom converters](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-custom-converters)
- [Debezium — Setting up PostgreSQL](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#setting-up-postgresql)
- [Debezium — Deployment](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-deployment)
- [Debezium — Monitoring](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-monitoring)
- [Debezium — Behavior when things go wrong](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-when-things-go-wrong)
- [Debezium — Monitoring with JMX, Prometheus, and Grafana](https://debezium.io/documentation/reference/3.2/operations/monitoring.html)
- [Apache Kafka — Kafka Connect REST API](https://kafka.apache.org/10/kafka-connect/user-guide/)
- [PostgreSQL — Logical decoding concepts](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html)
- [PostgreSQL — pg_replication_slots](https://www.postgresql.org/docs/current/view-pg-replication-slots.html)
- [PostgreSQL — Logical replication configuration](https://www.postgresql.org/docs/current/logical-replication-config.html)
- [PostgreSQL — Logical replication security](https://www.postgresql.org/docs/current/logical-replication-security.html)
- [PostgreSQL — Data Types](https://www.postgresql.org/docs/current/datatype.html)
- [Apache Kafka — Connect data API](https://kafka.apache.org/40/javadoc/org/apache/kafka/connect/data/package-summary.html)
- [PostgreSQL — ALTER TABLE: REPLICA IDENTITY](https://www.postgresql.org/docs/current/sql-altertable.html)
- [Apache Kafka — Log compaction](https://kafka.apache.org/documentation/#compaction)
