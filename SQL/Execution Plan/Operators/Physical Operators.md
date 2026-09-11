## 1. Vấn đề đặt ra

Một câu SQL cho biết *kết quả cần có*, nhưng không chỉ ra phải đọc dữ liệu, ghép bảng, nhóm hay sắp xếp bằng thuật toán nào. Cùng một phép `JOIN` có thể rất nhanh với một kế hoạch và chậm đáng kể với kế hoạch khác, dù kết quả không đổi. [[Logical Operators|Logical operators]] mô tả yêu cầu theo đại số quan hệ; physical operator mới là đơn vị triển khai cụ thể quyết định dữ liệu được đọc và biến đổi như thế nào.

Vì vậy, cần đọc execution plan theo physical operator để tìm đúng nguyên nhân: quét quá nhiều dữ liệu, ước lượng số dòng sai, sort/hash bị tràn bộ nhớ, hoặc chiến lược join không phù hợp. Tên node là đặc thù của từng DBMS, nên cần hiểu vai trò và điều kiện áp dụng thay vì học thuộc tên.

## 2. Physical operator là gì?

Physical operator (toán tử vật lý) là thuật toán hoặc routine thực thi một bước trong physical execution plan. Optimizer ánh xạ một logical operation sang một hoặc nhiều cách triển khai vật lý, ước lượng chi phí của các phương án rồi chọn phương án có chi phí dự kiến thấp hơn. Ví dụ, logical `Join` có thể thành `Nested Loop`, `Hash Join` hoặc `Merge Join`; logical `Aggregate` có thể thành hash aggregate hoặc stream aggregate.

Plan thường là một cây: node lá lấy dữ liệu từ bảng/index; node giữa lọc, ghép hoặc biến đổi; node gốc trả kết quả hay thực hiện thao tác ghi. PostgreSQL mô tả các scan node ở đáy và các node join/aggregate/sort ở phía trên; DuckDB cũng biểu diễn physical plan là cây operator chạy theo thứ tự để tạo kết quả.

## 3. Các nhóm toán tử phổ biến

Các DBMS không dùng một từ vựng hoàn toàn giống nhau: `Seq Scan` là tên PostgreSQL/DuckDB, SQL Server có `Clustered Index Scan`, còn Oracle nói về access path. Bảng dưới quy về vai trò chung.

| Nhóm | Ví dụ physical operator |
|---|---|
| Truy cập dữ liệu | sequential/table scan, index seek/scan, bitmap index + bitmap heap scan, index-only scan |
| Lọc và tính toán | filter, compute scalar, projection |
| Ghép | nested loop, hash join, merge join |
| Gom nhóm/loại trùng | hash aggregate, stream aggregate, distinct |
| Sắp xếp và giới hạn | sort, top/limit, incremental sort |
| Lưu kết quả trung gian | materialize, spool, worktable |
| Phân phối song song | exchange, repartition, gather streams |

### 3.1. Ba thuật toán join quan trọng

| Thuật toán | Cách chạy | Thường phù hợp | Điều kiện/rủi ro |
|---|---|---|---|
| Nested loop | Lấy từng dòng outer, tìm các dòng khớp ở inner. | Outer rất nhỏ và inner có index/selective access theo join key; truy vấn OLTP điểm. | Nếu inner bị scan lặp lại cho nhiều outer row, chi phí tăng mạnh. `Materialize` có thể được thêm để tránh chạy lại inner. |
| Hash join | Build hash table từ input nhỏ hơn, rồi probe bằng input còn lại. | Equi-join với tập dữ liệu vừa/lớn, không sẵn thứ tự. | Cần bộ nhớ; input build lớn hay skew có thể spill/batch. Không phù hợp trực tiếp với predicate không phải bằng. |
| Merge join | Đọc hai input đã sắp theo join key và di chuyển đồng thời. | Hai input đã có thứ tự nhờ index/sort, dữ liệu lớn, hoặc cần giữ thứ tự. | Nếu cần sort cả hai bên, tổng chi phí có thể lớn hơn hash join. |

Db2 và PostgreSQL đều tài liệu hóa ba họ join này. Quy tắc “OLTP dùng nested loop, phân tích dùng hash/merge” chỉ là điểm xuất phát: độ chọn lọc, index, kích thước thực tế và kiểu predicate mới quyết định lựa chọn.

### 3.2. Ví dụ: một logical yêu cầu, nhiều physical plan

```sql
SELECT c.name, SUM(o.total) AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.region = 'VN'
GROUP BY c.name
ORDER BY revenue DESC;
```

Một plan có thể là `Index/Seq Scan customers` → `Filter region` → `Nested Loop` + `Index Scan orders(customer_id)` → `Hash Aggregate` → `Sort`. Nếu số khách hàng tại Việt Nam lớn, optimizer có thể đổi sang `Hash Join`, build hash từ `customers` đã lọc rồi scan `orders`. Cả hai vẫn thực hiện cùng logical filter, join, group và order, nhưng lượng I/O, bộ nhớ và thời điểm trả dòng khác nhau.

## 4. Optimizer chọn operator dựa trên gì?

Optimizer không chọn theo một luật cố định; nó so sánh chi phí ước lượng của tổ hợp access path, join order và operator. Các yếu tố chính là:

| Yếu tố | Ảnh hưởng đến lựa chọn |
|---|---|
| Cardinality và độ chọn lọc | Quyết định kích thước input/output dự kiến; sai số ở node thấp lan truyền lên toàn cây. |
| Thống kê bảng/index và histogram | Cơ sở để ước lượng; stale statistics hoặc dữ liệu lệch (skew) dễ đưa đến join/scan sai. |
| Index, thứ tự và covering columns | Có thể cho seek, scan có thứ tự, index-only access; đồng thời ảnh hưởng chi phí lookup. |
| Bộ nhớ và I/O | Quyết định hash/sort có in-memory, phải chia batch hay spill; chi phí random so với sequential I/O khác nhau theo hệ thống. |
| Join predicate và ràng buộc ngữ nghĩa | Equi-join mở đường hash join; outer join, `NULL`, thứ tự bắt buộc hoặc predicate phức tạp giới hạn reorder/thuật toán. |
| Song song và phân phối dữ liệu | Chi phí exchange, số worker, partitioning và skew quyết định lợi ích thực. |

Chi phí hiển thị trong plan là mô hình tương đối của engine, không phải thời gian thực. Db2 gọi đơn vị ước lượng là *timeron* và nêu rõ nó thay đổi theo thống kê/nội bộ; vì thế không nên so `cost` giữa hai DBMS, hay coi chi phí thấp hơn là bằng chứng đo được nhanh hơn.

## 5. Đọc và xác minh execution plan

### 5.1. Estimated plan và actual plan

`EXPLAIN` thường cho plan biên dịch cùng estimates mà không chạy câu lệnh; DuckDB ghi nhận đây là estimated cardinality. `EXPLAIN ANALYZE` (hay Actual Execution Plan tùy DBMS) chạy truy vấn và thêm actual rows, thời gian, loops, buffer/memory hoặc spill. PostgreSQL cảnh báo `EXPLAIN ANALYZE` thực sự thực thi statement, nên với `INSERT`/`UPDATE`/`DELETE` cần dùng transaction rồi `ROLLBACK` khi chỉ muốn phân tích.

| Cần so sánh | Dấu hiệu | Hướng xử lý |
|---|---|---|
| Estimated rows với actual rows | Chênh lệch lớn từ node đầu | Cập nhật statistics; xem histogram, correlation, predicate/parameter và dữ liệu skew. |
| Số loops và row của inner node | Scan/seek inner bị gọi rất nhiều trong nested loop | Cân nhắc index phù hợp hoặc xem lại cardinality/join order. |
| Thời gian, buffer/I/O, memory/spill | Sort/hash chậm hoặc dùng temporary disk | Giảm input trước node, tận dụng thứ tự index, chỉnh memory đúng chính sách engine. |
| Rows removed/filter placement | Nhiều dòng bị loại ở node cao | Viết predicate để optimizer pushdown được; chỉ thêm index sau khi xác minh selectivity. |

### 5.2. Quy trình chẩn đoán thực tế

1. Lấy estimated plan và actual plan cho cùng câu SQL, tham số và dữ liệu đại diện.
2. Đi từ node có actual time/I/O lớn hoặc cardinality lệch mạnh, rồi lần xuống node con gây ra input đó.
3. Kiểm tra predicate, join key, statistics và index trước; chỉ sau đó mới thử rewrite, index hay hint.
4. Thay đổi một giả thuyết mỗi lần, đo benchmark lặp lại và so plan trước/sau. Plan mới tốt trên `cost` chưa chắc nhanh hơn dưới workload thật.
5. Theo dõi hồi quy sau khi dữ liệu, thống kê, phiên bản DBMS hay cấu hình thay đổi; plan cache và tham số có thể làm kết quả khác giữa các lần chạy.

## 6. Giới hạn và các hiểu nhầm thường gặp

- Physical operator không phải cú pháp SQL. Người dùng diễn đạt logical operation; engine có thể đổi operator sau `ANALYZE`, thay đổi dữ liệu, index hoặc phiên bản.
- Cùng tên không đảm bảo cùng cơ chế: cách build/probe, vectorization, spill và parallelism là chi tiết triển khai của DBMS. Đọc tài liệu đúng phiên bản đang vận hành.
- Không ép một loại join chỉ vì “hash join luôn nhanh”. Hints có thể giúp thử nghiệm, nhưng có thể khóa plan vào giả định nhanh lỗi thời khi dữ liệu đổi.
- Không chẩn đoán từ một node đơn lẻ. `Sort`, `Materialize` hoặc table scan có thể là hệ quả tối ưu của yêu cầu `ORDER BY`, tỷ lệ đọc cao hay cần reuse; phải xem toàn bộ input/output và metric thực tế.
- `EXPLAIN ANALYZE` có overhead và có thể thực thi thao tác ghi. Với production, dùng bản sao, transaction rollback hoặc công cụ quan sát phù hợp.

## 7. Nguồn tham khảo

- [Microsoft Learn — Logical and physical Showplan operator reference](https://learn.microsoft.com/en-us/sql/relational-databases/showplan-logical-and-physical-operators-reference?view=sql-server-2017): phân biệt logical/physical operator, mô hình iterator và mô tả operator SQL Server.
- [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) và [EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html): cấu trúc plan tree, scan/join node, estimated/actual metrics và lưu ý khi chạy `ANALYZE`.
- [Oracle Database — Joins](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/joins.html): access path và lựa chọn join row source.
- [MySQL 8.0 Reference Manual — EXPLAIN output](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html): access/join type xuất hiện trong `EXPLAIN`.
- [IBM Db2 — Explain data operators](https://www.ibm.com/docs/en/db2/11.1.0?topic=information-explain-data-operators) và [guidelines for analyzing explain information](https://www.ibm.com/docs/en/db2/12.1.x?topic=facility-guidelines-analyzing-explain-information): operator, chi phí, join method và cách dùng plan để tuning.
- [DuckDB — Inspect query plans](https://duckdb.org/docs/current/guides/meta/explain) và [Profiling queries](https://duckdb.org/docs/current/sql/statements/profiling): physical plan, cardinality estimate và số đo runtime theo operator.
