## 1. Vấn đề đặt ra

[[Debezium]] đọc thay đổi từ transaction log, nhưng một pipeline CDC còn phải chạy connector ở đâu, lưu vị trí đã đọc thế nào, đưa event vào Kafka ra sao và khôi phục khi một máy dừng. Tự viết chương trình riêng cho từng nguồn và đích sẽ lặp lại các việc kết nối, retry, quản lý offset, cấu hình và giám sát.

Kafka Connect là runtime chuẩn hóa phần **tích hợp dữ liệu** đó. Nó phù hợp khi cần di chuyển dữ liệu giữa Kafka và database, SaaS, message broker, object storage hoặc search engine bằng connector có sẵn. Nó không phải Kafka broker, không thay thế Apache Kafka, và không phải nơi thích hợp để chứa transform nghiệp vụ có state phức tạp.

## 2. Kafka Connect là gì?

Kafka Connect là framework và runtime của Apache Kafka để vận hành hai loại connector:

```text
Database / SaaS / file ── source connector ──> Kafka topic
Kafka topic ─────────────── sink connector ──> warehouse / S3 / Elasticsearch
```

| Khái niệm | Ý nghĩa |
| --- | --- |
| **Worker** | Tiến trình JVM chạy Kafka Connect. Worker kết nối Kafka, nhận cấu hình và thực thi task. |
| **Connect cluster** | Nhóm worker cùng chia sẻ trạng thái trong chế độ distributed. |
| **Plugin** | Thư mục JAR chứa implementation của connector, converter hoặc transformation. |
| **Connector** | Một instance cấu hình của plugin, mô tả tích hợp cần thực hiện: nguồn/đích nào, topic/bảng nào. |
| **Task** | Đơn vị thực thi do connector tạo và worker chạy. Task mới thực sự đọc hoặc ghi dữ liệu. |
| **Source connector** | Đưa record từ hệ ngoài vào Kafka. Debezium là một ví dụ. |
| **Sink connector** | Đọc record từ Kafka rồi ghi ra hệ ngoài. |
| **Converter** | Chuyển mô hình dữ liệu Kafka Connect thành bytes trong Kafka và ngược lại: JSON, Avro, Protobuf… |
| **SMT** | Single Message Transform: biến đổi nhẹ trên từng record, ví dụ đổi tên topic, lọc/mask cột. |

Connector không trực tiếp sao chép dữ liệu; nó chia công việc thành task để Connect phân phối cho worker. `tasks.max` chỉ là mức tối đa mong muốn: connector có thể tạo ít task hơn nếu nguồn hoặc đích không song song hóa được.

### 2.1. Kafka Connect khác gì Kafka broker và ứng dụng consumer?

| Thành phần | Trách nhiệm chính | Không chịu trách nhiệm cho |
| --- | --- | --- |
| Kafka broker | Lưu, replicate và phân phối record theo topic/partition | Đọc WAL, gọi API SaaS, ghi dữ liệu vào sink |
| Kafka Connect | Kết nối hệ ngoài với Kafka, quản lý task/cấu hình/offset | Quy tắc nghiệp vụ, join hay aggregate phức tạp |
| Consumer application | Xử lý nghiệp vụ, tạo read model, ra quyết định | Vận hành chung connector cho nhiều hệ ngoài |
| Kafka Streams/Flink/Spark | Xử lý stream có state, cửa sổ thời gian, join | Thay connector để tích hợp hệ ngoài |

Ví dụ, Connect có thể dùng Debezium đưa thay đổi `orders` vào Kafka và một sink connector đưa event đến Elasticsearch. Nhưng tính “doanh thu ròng theo giờ” cần join, quy tắc hủy đơn và dữ liệu muộn nên thuộc stream processor hoặc lớp ELT, không nên nhét vào SMT.

## 3. Kiến trúc và luồng xử lý

### 3.1. Source connector

Với source connector, task đọc dữ liệu từ nguồn, tạo `SourceRecord`, Kafka Connect tuần tự hóa record bằng converter rồi ghi vào topic. Sau khi record được ghi thành công, Connect/connector ghi nhận **source offset**: vị trí đã xử lý ở hệ nguồn.

```text
PostgreSQL WAL → Debezium source task → converter → Kafka topic
                    │
                    └── source offset: LSN đã xử lý
```

Offset không phải lúc nào cũng là Kafka offset. Với Debezium PostgreSQL đó là vị trí LSN/WAL; với file source có thể là vị trí byte/dòng; với API có thể là cursor hoặc timestamp do connector định nghĩa.

### 3.2. Sink connector

Sink task subscribe topic hoặc pattern topic, đọc record bằng consumer group của connector, biến bytes thành dữ liệu Kafka Connect rồi ghi sang đích. Khi sink xác nhận ghi thành công, Connect commit Kafka offset của record đã xử lý.

```text
Kafka topic → converter → sink task → data warehouse / S3 / Elasticsearch
                   │
                   └── Kafka topic-partition-offset đã xử lý
```

Nghĩa là độ an toàn của sink phụ thuộc vào cả thao tác ghi đích và việc commit Kafka offset. Nếu ghi thành công nhưng Connect dừng trước khi commit offset, record có thể được ghi lại sau restart. Sink nên idempotent, chẳng hạn `upsert` theo khóa nghiệp vụ hoặc lưu version/offset nguồn đã áp dụng.

### 3.3. Debezium trong Kafka Connect

Debezium thường chạy như source connector trong một dịch vụ Kafka Connect riêng với Kafka broker. PostgreSQL connector snapshot khi cần, đọc logical replication stream, phát change event vào topic, và lưu LSN trong offset store của Connect. Kafka Connect không đọc WAL thay Debezium; nó cung cấp runtime, quản lý task, REST API và persistence state chung. [Debezium Architecture](https://debezium.io/documentation/reference/stable/architecture.html)

Một connector PostgreSQL thông thường chỉ có một task đọc luồng WAL. Vì vậy tăng `tasks.max` không tự làm capture nhanh hơn. Khả năng mở rộng phải được đánh giá theo từng connector, partition của topic, năng lực nguồn và năng lực sink.

## 4. Hai chế độ chạy

| Chế độ | Dùng khi | Lưu state | Ưu điểm | Hạn chế |
| --- | --- | --- | --- | --- |
| Standalone | Local development, test, agent đơn lẻ | File local | Cài đơn giản | Không HA; worker mất thì workload dừng; không phù hợp CDC production |
| Distributed | Production | Kafka internal topics | HA, scale worker, REST API quản lý connector | Cần thiết kế Kafka, internal topic và quan sát vận hành |

Standalone chạy mọi connector/task trong một process. Offset thường ở `offset.storage.file.filename`; không nên dùng file local cho luồng CDC cần phục hồi bền vững.

Distributed mode là lựa chọn thông thường cho production. Worker có cùng `group.id` và cùng internal topics tạo thành một Connect cluster. Khi worker rời cluster, Connect rebalance các task sang worker còn lại. Worker có thể chạy trong VM, container hoặc Kubernetes; Connect không tự quyết định autoscaling/restart process, việc đó thuộc orchestrator bên ngoài.

## 5. Internal topics và khả năng khôi phục

Connect distributed lưu state vào Kafka, không lưu riêng trên worker:

| Internal topic | Nội dung | Quy tắc quan trọng |
| --- | --- | --- |
| `config.storage.topic` | Cấu hình connector/task | Compaction; chính xác một partition; replication phù hợp SLA |
| `offset.storage.topic` | Offset nguồn hoặc trạng thái tiến độ | Compaction; replication đủ cao; không xóa tùy tiện |
| `status.storage.topic` | Trạng thái connector/task | Compaction; phục vụ API/monitoring |

Mọi worker trong cùng cluster phải dùng cùng ba topic. Một cluster khác phải có `group.id` **và cả ba topic** riêng; chỉ đổi `group.id` là chưa đủ. Các internal topic là control plane: mất topic offset có thể khiến source connector snapshot/đọc lại hoặc không thể tiếp tục đúng vị trí; mất config topic làm mất định nghĩa connector.

Với Debezium relational connector, ngoài offset của Connect còn có thể có schema history riêng. Schema history không thay thế offset: offset trả lời “đọc tiếp từ đâu”, còn history giúp connector diễn giải schema nguồn tại vị trí log tương ứng. [Debezium state storage](https://debezium.io/documentation/reference/stable/configuration/storage.html)

## 6. Delivery semantics: không suy diễn quá mức

Mặc định an toàn nhất là coi pipeline Connect là **at-least-once**:

1. Source task phát record thành công vào Kafka nhưng dừng trước khi offset nguồn được flush → record có thể phát lại.
2. Sink ghi đích thành công nhưng dừng trước commit Kafka offset → record có thể ghi lại.
3. Retry lỗi tạm thời cũng có thể tạo bản ghi lặp nếu đích không idempotent.

Vì vậy thiết kế downstream cần trả lời rõ khóa idempotency là gì, update/delete được áp dụng thế nào, event cũ đến muộn xử lý ra sao, và khi nào được phép replay/backfill.

Kafka Connect có hỗ trợ exactly-once cho connector **có triển khai khả năng này** và khi cluster/configuration đáp ứng điều kiện; đây không phải thuộc tính tự động của mọi plugin. Source exactly-once chỉ có ở distributed mode và cần bật `exactly.once.source.support` theo quy trình nâng cấp an toàn. Với sink, connector và hệ đích cũng phải hỗ trợ ngữ nghĩa tương ứng. [Apache Kafka Connect user guide](https://kafka.apache.org/40/kafka-connect/user-guide/)

## 7. Converter, schema và SMT

### 7.1. Converter

Kafka topic lưu bytes; converter quyết định biểu diễn của key, value và header.

| Lựa chọn | Phù hợp | Đánh đổi |
| --- | --- | --- |
| JSON Converter | Debug nhanh, hệ đơn giản | Cần quy ước schema rõ nếu nhiều consumer |
| Avro + Schema Registry | Event contract dài hạn, compatibility | Cần vận hành registry và quy trình schema |
| Protobuf + Schema Registry | Hệ sinh thái service dùng Protobuf | Cần kiểm tra mapping kiểu từ Connect |
| JSON Schema + Schema Registry | Consumer hướng JSON, vẫn cần schema governance | Cần quản trị compatibility như các lựa chọn khác |

Đặt converter mặc định ở worker khi đa số connector dùng chung; connector có thể override nếu cần. Khi override, phải khai báo đầy đủ cấu hình phụ thuộc của converter, ví dụ URL Schema Registry, thay vì giả định worker config sẽ được kế thừa.

### 7.2. SMT

SMT chạy từng record theo chuỗi cấu hình. Các ứng dụng hợp lý gồm routing topic, chọn/đổi tên field, mask PII, thêm metadata và làm phẳng envelope Debezium. Ví dụ `ExtractNewRecordState` có thể đưa `after` thành value cho sink chỉ cần trạng thái hiện tại.

Đổi lại, làm phẳng quá sớm có thể mất `before`, `op`, transaction metadata hoặc tombstone cần để xử lý delete/audit. SMT không phù hợp cho join nhiều topic, aggregate theo cửa sổ, tra cứu trạng thái ngoài Kafka hay logic có retry nghiệp vụ.

## 8. Cấu hình, triển khai và quản lý

Worker config là cấu hình hạ tầng dùng chung: `bootstrap.servers`, `group.id`, internal topics, `plugin.path`, converter mặc định, TLS/SASL, REST listener, quota và logging. Connector config mô tả một pipeline: `name`, `connector.class`, endpoint/credential, bảng hoặc topic, `tasks.max`, retry, converter override và SMT.

Trong distributed mode, tạo/sửa/pause/resume connector bằng REST API. Ví dụ tối giản của một Debezium PostgreSQL connector:

```json
{
  "name": "postgres-orders",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "postgres.internal",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "${secret:...}",
    "topic.prefix": "app",
    "plugin.name": "pgoutput",
    "slot.name": "dbz_orders",
    "publication.name": "dbz_publication",
    "table.include.list": "public.orders"
  }
}
```

Đây chỉ là khung minh họa. Cấu hình production còn cần quyền PostgreSQL, TLS, secret provider, chiến lược snapshot, converter, replication slot/publication và policy lỗi. Plugin phải được cài với cùng phiên bản trên mọi worker có thể nhận task; `plugin.path` là nơi Connect phát hiện và cô lập thư viện plugin.

## 9. Lỗi, retry và dead-letter queue

Phân biệt ba nhóm lỗi:

| Nhóm lỗi | Ví dụ | Cách xử lý chính |
| --- | --- | --- |
| Hạ tầng tạm thời | Kafka, database, DNS, sink timeout | Retry có giới hạn, backoff, alert và kiểm tra lag |
| Record không hợp lệ | Sai kiểu dữ liệu, schema không tương thích | Log ngữ cảnh, gửi DLQ hoặc dừng theo mức độ nghiêm trọng |
| Cấu hình/quyền | Sai credential, thiếu ACL, plugin không có | Sửa cấu hình/quyền; retry đơn thuần không giải quyết |

Connect hỗ trợ `errors.retry.timeout`, `errors.retry.delay.max.ms`, logging lỗi và dead-letter queue cho nhiều tình huống xử lý record. `errors.tolerance=all` không phải mặc định an toàn: nó chỉ phù hợp khi có DLQ, quy trình xử lý bản ghi bị cô lập và chấp nhận rõ tác động thiếu dữ liệu. [Apache Kafka Connect error handling](https://kafka.apache.org/35/kafka-connect/user-guide/)

DLQ cũng không thay thế monitoring. Cần alert khi DLQ xuất hiện/tăng nhanh, vì một connector “RUNNING” vẫn có thể làm mất dữ liệu nghiệp vụ nếu lỗi bị dung thứ mà không được xử lý.

## 10. Vận hành production

Theo dõi tối thiểu gồm:

- Trạng thái connector/task, số lần restart và failed task.
- Source lag, sink consumer lag, throughput record/byte và tuổi dữ liệu ở đích.
- CPU, heap, GC, network, queue/buffer và file descriptor của worker.
- Lag/retention replication slot và WAL khi dùng Debezium PostgreSQL.
- Dung lượng, replication, compaction và quyền truy cập của internal topics.
- Retry rate, DLQ volume, lỗi converter/schema và tỷ lệ update/delete bị từ chối ở sink.

Về bảo mật, dùng TLS/SASL cho Kafka, ACL tối thiểu theo topic, replication user riêng cho database, secret provider thay vì password trong Git, và mask/loại PII trước khi event tới consumer không được phép truy cập. REST API Connect cũng phải được giới hạn mạng và xác thực như một control plane.

## 11. Khi nào nên dùng và khi nào không?

Kafka Connect phù hợp khi connector đã trưởng thành, tích hợp chủ yếu là di chuyển record, cần cấu hình và vận hành nhất quán, hoặc nhiều source/sink dùng chung Kafka làm event log.

Không nên chọn chỉ vì dữ liệu “có Kafka”. Một ứng dụng custom hoặc stream processor phù hợp hơn nếu phải gọi API theo workflow nghiệp vụ, xử lý state phức tạp, join/aggregate, cần UX điều phối riêng, hoặc connector không đáp ứng ngữ nghĩa source/sink bắt buộc. Trước khi đưa connector vào production, proof of concept phải thử snapshot, update, delete, schema change, retry, restart worker, sink outage, replay và backfill trên dữ liệu đại diện.

## 12. Nguồn tham khảo

1. Apache Kafka, [Kafka Connect User Guide](https://kafka.apache.org/40/kafka-connect/user-guide/), truy cập 04/09/2026.
2. Apache Kafka, [Kafka Connect configuration](https://kafka.apache.org/38/configuration/kafka-connect-configs/), truy cập 04/09/2026.
3. Apache Kafka, [Connector Development Guide](https://kafka.apache.org/25/kafka-connect/connector-development-guide/), truy cập 04/09/2026.
4. Debezium, [Architecture](https://debezium.io/documentation/reference/stable/architecture.html), truy cập 04/09/2026.
5. Debezium, [Storing state of a Debezium connector](https://debezium.io/documentation/reference/stable/configuration/storage.html), truy cập 04/09/2026.
6. Confluent, [Get started with Kafka Connect](https://docs.confluent.io/platform/current/connect/userguide.html), truy cập 04/09/2026.
