## 1. Vấn đề đặt ra

[[Debezium]] có thể biến thay đổi trong database thành event, nhưng nếu mỗi hệ đích nhận event trực tiếp từ nguồn thì thêm một consumer mới sẽ làm tăng điểm tích hợp và khó phát lại dữ liệu. Cần một event log bền vững, cho phép nhiều hệ độc lập đọc cùng dữ liệu theo tốc độ riêng, vẫn chịu được lỗi máy và có thể đọc lại khi sửa consumer.

Apache Kafka giải quyết bài toán đó bằng cách lưu event theo log phân tán. Kafka không phải database giao dịch thay thế OLTP và cũng không tự hiểu nghiệp vụ của event; nó là hạ tầng lưu, phân phối và phát lại luồng record. [[Kafka Connect]] thường là lớp đưa dữ liệu vào/ra Kafka, còn consumer hoặc stream processor chịu trách nhiệm dùng dữ liệu.

## 2. Apache Kafka là gì?

Kafka là nền tảng event streaming phân tán. Producer ghi record vào **topic**; broker lưu và replicate record; consumer đọc record theo **offset**. Một topic có thể có nhiều producer và nhiều consumer, nên Kafka tạo được mô hình publish–subscribe thay vì chỉ là hàng đợi một người nhận.

```text
PostgreSQL → Debezium / Kafka Connect → Kafka topic → consumer A: search
                                            ├──────→ consumer B: warehouse
                                            └──────→ consumer C: fraud service
```

Ba đặc tính làm Kafka phù hợp cho CDC:

- Ghi và đọc tuần tự trên partition cho throughput cao.
- Lưu event theo retention; consumer có thể đọc chậm, bắt đầu muộn hoặc replay.
- Replicate từng partition để chịu lỗi broker.

Kafka chỉ đảm bảo thứ tự trong **một partition**, không có thứ tự toàn cục cho toàn topic nhiều partition. Đây là quyết định kiến trúc quan trọng nhất khi thiết kế event key và consumer.

## 3. Các thành phần cốt lõi

| Thành phần | Vai trò | Ví dụ CDC |
| --- | --- | --- |
| Broker | Server Kafka lưu partition và phục vụ client | Một node trong cluster Kafka |
| Cluster | Tập broker cùng quản lý metadata và dữ liệu | Cluster 3 broker production |
| Topic | Tên logic của một luồng record | `app.public.orders` |
| Partition | Log có thứ tự nằm trong topic | `app.public.orders-2` |
| Record | Đơn vị dữ liệu: key, value, header, timestamp | Event `UPDATE orders` từ Debezium |
| Offset | Số tăng dần, định danh vị trí record trong một partition | Offset `81452` của partition 2 |
| Producer | Client ghi record | Kafka Connect source task |
| Consumer | Client đọc record | Sink connector hoặc ứng dụng Java/Python |
| Consumer group | Nhóm consumer cùng chia việc của một subscription | Nhóm materialize Elasticsearch |
| Controller | Thành phần điều phối metadata/leader partition | Quorum controller ở chế độ KRaft |

### 3.1. Record, key và partition

Record gồm key và value dạng bytes ở tầng Kafka; schema/serialization do producer và consumer quy ước. Key không bắt buộc, nhưng rất quan trọng cho CDC:

```text
key   = order_id = 42
value = { op: "u", before: {...}, after: {...} }
```

Producer dùng key ổn định để gửi các thay đổi của cùng `order_id` vào cùng partition. Nhờ đó consumer đọc các thay đổi của đơn hàng đó theo đúng thứ tự đã ghi. Nếu không có key, partitioner có thể phân tán record và consumer không còn thứ tự theo entity.

Offset chỉ có ý nghĩa trong cặp `(topic, partition)`. Offset `100` của partition 0 không thể so sánh thời gian hoặc thứ tự với offset `100` của partition 1. Offset cũng không phải ID nghiệp vụ và không thay thế LSN/WAL position từ nguồn CDC.

## 4. Topic, partition và thứ tự

Topic được chia thành một số partition cố định tại thời điểm tạo (có thể tăng về sau nhưng không giảm trực tiếp). Partition là append-only log:

```text
orders-0:  offset 0 → 1 → 2 → 3 → ...
orders-1:  offset 0 → 1 → 2 → 3 → ...
orders-2:  offset 0 → 1 → 2 → 3 → ...
```

Partition vừa là đơn vị song song hóa, vừa là biên giới thứ tự:

- Nhiều partition tăng throughput và cho phép nhiều consumer trong một group xử lý song song.
- Một partition chỉ được một consumer trong cùng group xử lý tại một thời điểm.
- Record trong cùng partition luôn được đọc theo thứ tự ghi; record giữa các partition không có thứ tự tổng.

Số partition là đánh đổi. Quá ít partition giới hạn throughput và số consumer active. Quá nhiều tạo thêm file, replication traffic, metadata và chi phí rebalance. Chọn dựa trên throughput, số task/consumer cần song song, key cardinality, retention và năng lực broker; không chỉ dựa vào “dữ liệu lớn”.

Tăng số partition về sau có thể thay đổi kết quả băm key → partition của record mới. Vì thế không được giả định thứ tự của cùng key xuyên suốt trước và sau thay đổi partition nếu partitioner/mapping không bảo toàn điều đó; cần thiết kế và kiểm thử trước khi tăng.

## 5. Replication, leader và độ bền

Kafka replicate theo **topic-partition**. Mỗi partition có một leader và các follower replica trên broker khác. Producer ghi vào leader; follower kéo dữ liệu từ leader để theo kịp log. Khi leader lỗi, Kafka bầu replica phù hợp làm leader mới.

| Khái niệm | Ý nghĩa vận hành |
| --- | --- |
| Replication factor (RF) | Tổng số bản sao cho mỗi partition, gồm cả leader |
| ISR | In-sync replicas: các replica đang bắt kịp leader đủ điều kiện |
| `acks` | Producer chờ mức xác nhận nào trước khi coi record thành công |
| `min.insync.replicas` | Số replica ISR tối thiểu broker phải có để nhận ghi với `acks=all` |
| Unclean leader election | Cho phép replica lạc hậu lên leader; có thể mất record đã commit theo kỳ vọng cũ |

Với dữ liệu CDC quan trọng, mục tiêu thường là kết hợp RF phù hợp, `acks=all`, `min.insync.replicas` phù hợp và **không** bật unclean leader election. Không có một con số đúng cho mọi hệ thống: RF=3 là mẫu production phổ biến, nhưng SLA, số broker/zone và khả năng chấp nhận mất dữ liệu mới quyết định cấu hình thực tế.

Replication không thay thế backup, DR hay kiểm tra restore. Nó bảo vệ trước lỗi broker trong cùng cluster; không tự bảo vệ khi xóa topic nhầm, cấu hình retention sai, lỗi ứng dụng phát dữ liệu sai, hoặc mất cả vùng triển khai.

## 6. Producer: ghi dữ liệu đúng cách

Producer gửi record bất đồng bộ và thường batch/compress record để tăng throughput. Các tham số chính là key/partitioner, `acks`, retry, idempotence, batch size, linger, compression và timeout.

Độ trễ và throughput là đánh đổi: batch lớn hoặc chờ lâu hơn giúp I/O hiệu quả hơn nhưng tăng latency. Với CDC, cần đo latency end-to-end thay vì chỉ tối ưu tốc độ producer.

Producer idempotent giúp broker tránh tạo duplicate do retry trong phạm vi producer session. Transactional producer có thể atomically ghi nhiều record và commit offset của consumer khi xây dựng luồng consume–transform–produce. Tuy nhiên, idempotence/transaction của Kafka không tự làm thao tác ghi vào database, HTTP API hay Elasticsearch trở thành exactly-once.

Khi dùng source connector, nhiều chi tiết producer do Kafka Connect quản lý. Dù vậy, đội vận hành vẫn phải hiểu `acks`, retry, quota, topic ACL và khả năng downstream chịu event lặp.

## 7. Consumer, offset và consumer group

Consumer gửi fetch request cho leader của partition và cho biết offset muốn đọc. Nó có thể commit offset đã xử lý để khởi động lại đúng tiến độ. Offset commit thuộc về **consumer group**, không thuộc một consumer process đơn lẻ.

```text
Topic orders có 6 partition

Group search-index: 3 consumer  → mỗi consumer nhận khoảng 2 partition
Group warehouse:    1 consumer  → nhận cả 6 partition
Group audit:        6 consumer  → mỗi consumer nhận 1 partition
```

Các group khác nhau đọc độc lập cùng topic. Đây là lý do Kafka hợp với một luồng CDC có nhiều đích. Trong **cùng** group, một partition chỉ gán cho một consumer active; thêm consumer vượt số partition không tăng song song hóa.

Khi consumer join/rời group hoặc số partition thay đổi, group rebalance phân công lại partition. Consumer cần dừng nhận assignment cũ, hoàn tất/commit an toàn phần xử lý đang dở và khởi tạo lại state cần thiết. Rebalance thường không phải lỗi, nhưng rebalance liên tục làm tăng lag và có thể bộc lộ xử lý chậm, timeout không phù hợp hoặc deploy không an toàn.

### 7.1. Commit offset và xử lý lặp

Hai thứ tự phổ biến đều có đánh đổi:

| Trình tự | Hệ quả khi process chết |
| --- | --- |
| Commit offset rồi ghi đích | Có thể mất record nếu chết sau commit trước khi ghi đích |
| Ghi đích rồi commit offset | Có thể ghi lặp nếu chết sau khi ghi đích trước commit |

Đa số consumer chọn ghi đích trước rồi commit, chấp nhận **at-least-once** và làm sink idempotent. Ví dụ warehouse `MERGE` theo `(order_id, version)`; Elasticsearch index theo document ID; cache áp dụng update khi version mới hơn. Không commit offset chỉ vì record đã được “đọc”; chỉ commit sau khi hiệu ứng nghiệp vụ bền vững.

Consumer có thể `seek` về offset cũ để replay. Nhưng replay chỉ khả dụng khi record vẫn còn trong retention và sink/logic chịu được chạy lại.

## 8. Retention, log compaction và tombstone

Kafka không tự xóa record khi consumer đã đọc. Topic giữ record theo chính sách retention; do đó consumer chậm hay group mới vẫn có thể đọc trong thời hạn dữ liệu còn tồn tại.

| Chính sách | Dùng cho | Điểm cần nhớ |
| --- | --- | --- |
| `delete` | Event lịch sử, audit, stream theo thời gian | Xóa segment khi quá thời gian/kích thước retention |
| `compact` | Trạng thái mới nhất theo key, changelog, cache rebuild | Giữ ít nhất giá trị cuối cho mỗi key, không giữ toàn bộ lịch sử |
| `delete,compact` | Cần cả trạng thái gần đây lẫn giới hạn lịch sử | Cả hai chính sách đều tác động; phải kiểm tra kỹ retention |

Log compaction không phải database snapshot tức thì. Cleaning chạy nền, record cũ vẫn có thể xuất hiện một thời gian; consumer đang bắt kịp vẫn thấy mọi record mới. Sau compaction, consumer replay từ đầu được bảo đảm thấy ít nhất trạng thái cuối còn lại theo key, chứ không phải mọi update lịch sử.

Record có **key** và value `null` là tombstone. Nó báo xóa key trong compacted topic; Kafka xóa các value cũ của key khi compact, nhưng tombstone cũng chỉ được giữ trong thời gian `delete.retention.ms`. Consumer hoặc sink cần xử lý tombstone trước khi nó bị làm sạch. Debezium có thể phát cả envelope `DELETE` và tombstone; hai record có ý nghĩa khác nhau.

`retention.ms` là SLA về thời gian tối đa consumer có thể chậm/replay với topic dùng `delete`, không phải “mốc chính xác record bị xóa”. Retention và compaction diễn ra theo segment/background process, nên thiết kế backfill phải có nguồn snapshot hoặc archive riêng khi cần lịch sử dài hạn.

## 9. Delivery semantics và transactions

| Ngữ nghĩa | Ý nghĩa | Rủi ro/chú ý |
| --- | --- | --- |
| At-most-once | Không lặp, nhưng có thể mất | Commit trước xử lý hoặc bỏ record lỗi |
| At-least-once | Không chủ đích mất, nhưng có thể lặp | Cần idempotency ở consumer/sink |
| Exactly-once processing | Một lần trong phạm vi transaction được hỗ trợ | Không tự mở rộng sang mọi external side effect |

Kafka transaction cho phép producer atomically ghi output record và commit offset đầu vào trong luồng consume–transform–produce. Consumer cần `isolation.level=read_committed` để bỏ qua transaction bị abort. Kafka Streams là cách đơn giản hơn để có ngữ nghĩa này khi xử lý hoàn toàn trong Kafka.

Với CDC end-to-end, đừng gắn nhãn “exactly-once” nếu chưa chứng minh cả connector nguồn, Kafka, consumer/sink và thao tác ghi đích cùng đáp ứng. Ví dụ, Kafka có thể chỉ phát một event committed nhưng HTTP sink timeout sau khi server đã áp dụng request vẫn tạo bài toán duplicate ở đích.

## 10. Kafka trong pipeline CDC

Một thiết kế điển hình:

```text
OLTP PostgreSQL
  → Debezium source connector
  → Kafka topic theo bảng, key = primary key
  → [tùy chọn] stream processor chuẩn hóa / outbox routing
  → consumer group hoặc sink connector
  → warehouse, search index, cache, service khác
```

Các quyết định phải chốt trước:

- Topic là raw CDC, domain event hay derived state? Không trộn lẫn mục đích mà không có contract.
- Key nào giữ đúng thứ tự theo entity? Có yêu cầu thứ tự cross-table/cross-transaction không?
- Retention đủ lâu cho outage, replay, backfill và điều tra sự cố chưa?
- Topic có cần compaction để dựng lại trạng thái, hay cần lịch sử đầy đủ theo thời gian?
- Delete/tombstone, schema change và PII được consumer xử lý thế nào?
- Ai sở hữu schema, ACL, quota, SLO lag và chi phí lưu trữ?

Kafka không tự biến raw row-level CDC thành event nghiệp vụ. Với tích hợp giữa service, outbox pattern thường rõ contract hơn việc để consumer suy luận “đơn đã thanh toán” từ mọi update ở bảng `orders`.

## 11. Vận hành, bảo mật và khôi phục thảm họa

Theo dõi ở ba lớp:

| Lớp | Chỉ báo quan trọng |
| --- | --- |
| Broker/cluster | Under-replicated partitions, offline partitions, ISR shrink, disk, network, controller health |
| Topic | Throughput, partition skew, retention, compaction lag, disk growth, producer errors |
| Consumer | Lag theo group/partition, rebalance, commit latency, processing failure, DLQ volume |

Phân vùng lệch (*hot partition*) thường xảy ra khi một key hoặc tập key chiếm phần lớn traffic. Không thể khắc phục chỉ bằng thêm consumer nếu hot key vẫn luôn phải nằm một partition để giữ thứ tự. Cần xem lại key, shard nghiệp vụ khi có thể, hoặc chấp nhận giới hạn throughput của entity đó.

Bảo mật gồm TLS khi truyền, SASL/mTLS để xác thực, ACL theo topic và consumer group, mã hóa dữ liệu lưu trữ theo hạ tầng, quản lý secret, audit thao tác quản trị và phân tách topic PII. Tránh cấp quyền wildcard rộng cho Kafka Connect hay consumer service.

DR cần xác định RPO/RTO, backup cấu hình/ACL/schema, phương án replicate đa cluster khi yêu cầu, và quy trình restore có kiểm thử. Replication trong một cluster không tự là geo-replication; xóa topic hoặc event sai vẫn có thể được replicate hoàn hảo.

## 12. Khi nào nên dùng Kafka?

Kafka phù hợp khi cần nhiều consumer độc lập, replay, throughput cao, event streaming liên tục, tách producer khỏi consumer hoặc làm event backbone cho CDC. Nó đặc biệt hữu ích khi số hệ đích thay đổi theo thời gian và không nên làm nguồn OLTP tích hợp trực tiếp với từng đích.

Kafka không mặc định phù hợp cho request–response đồng bộ, command cần phản hồi tức thời, một queue đơn giản có ít workload, hoặc khi tổ chức không có năng lực vận hành/managed service phù hợp. Nó cũng không thay thế OLTP cho truy vấn theo khóa, data warehouse cho phân tích ad-hoc, hay object storage cho archive rẻ dài hạn.

## 13. Nguồn tham khảo

1. Apache Kafka, [Introduction](https://kafka.apache.org/documentation/), truy cập 04/09/2026.
2. Apache Kafka, [Design](https://kafka.apache.org/40/design/design/), truy cập 04/09/2026.
3. Apache Kafka, [Topic-level configurations](https://kafka.apache.org/40/configuration/topic-level-configs/), truy cập 04/09/2026.
4. Apache Kafka, [Kafka protocol and partitioning](https://kafka.apache.org/40/design/protocol/), truy cập 04/09/2026.
5. Confluent, [Kafka consumer groups](https://docs.confluent.io/platform/current/clients/consumer.html), truy cập 04/09/2026.
