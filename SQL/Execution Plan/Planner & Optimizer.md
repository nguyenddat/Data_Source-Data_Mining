## 1. Vấn đề đặt ra

Trong **Execution Plan**, cùng một câu SQL có thể được thực thi bằng nhiều cách: quét bảng hoặc index, join theo thứ tự khác nhau, rồi dùng `Nested Loop`, `Hash Join` hoặc `Merge Join`. Các cách này trả cùng kết quả nhưng có chi phí rất khác nhau. PostgreSQL cần chọn một phương án trước khi biết chính xác câu SQL sẽ trả về bao nhiêu dòng.

## 2. Planner/Optimizer là gì?

Planner và optimizer là cùng một thành phần của PostgreSQL. Sau giai đoạn parse và rewrite, nó tạo các execution plan có thể có, ước lượng cost của từng phương án, rồi chọn phương án được dự đoán là chạy nhanh nhất.[^pg-planner]

Planner không thử chạy tất cả phương án. Nó làm việc với các cấu trúc rút gọn gọi là *path*; khi chọn được path rẻ nhất, PostgreSQL mới tạo plan tree đầy đủ cho executor chạy.[^pg-planner]

```text
SQL
  → parser / rewriter
  → planner/optimizer
      → tạo scan path và join path có thể dùng
      → ước lượng cardinality và cost
      → chọn path rẻ nhất
  → executor chạy plan đã chọn
```

Với query có rất nhiều join, duyệt mọi thứ tự join có thể tốn quá nhiều CPU và memory. Khi số quan hệ trong `FROM` đạt ngưỡng `geqo_threshold` (mặc định 12), PostgreSQL dùng Genetic Query Optimizer (GEQO) để tìm một plan hợp lý trong thời gian chấp nhận được; kết quả không được bảo đảm là plan tối ưu tuyệt đối.[^pg-geqo]

## 3. Planner tạo và so sánh plan như thế nào?

Planner bắt đầu với từng relation, xét các scan path dựa trên index sẵn có và điều kiện query. Sau đó nó ghép các relation thành các join path, đồng thời xét thứ tự join và thuật toán join. Cardinality và cost của đầu vào quyết định phương án nào rẻ hơn.[^pg-planner]

| Quyết định        | Ví dụ path được cân nhắc                                 | Thông tin cần ước lượng                                    |
| ----------------- | -------------------------------------------------------- | ---------------------------------------------------------- |
| Đọc bảng          | `Seq Scan`, `Index Scan`, `Bitmap Scan`                  | Số row khớp, số page phải đọc, tính chọn lọc của điều kiện |
| Join              | `Nested Loop`, `Hash Join`, `Merge Join`                 | Cardinality hai đầu vào và số row sau join                 |
| Thứ tự join       | `(orders JOIN customers) JOIN items` hoặc cách ghép khác | Kích thước intermediate result của mỗi thứ tự              |
| Sort và aggregate | Sort/Hash aggregate hoặc dùng thứ tự từ index            | Số row, kích thước row và số nhóm dự kiến                  |

Cost là đơn vị nội bộ để so sánh các path, không phải milliseconds. Nó phụ thuộc vào số row/page ước lượng và các hằng số cost của server, như chi phí đọc tuần tự, đọc ngẫu nhiên, CPU theo row và CPU theo operator.[^pg-query-config]

## 4. Statistics đi vào planner ở đâu?

[[Statistics|Planner statistics]] là dữ liệu đầu vào chính để planner ước lượng cardinality. `ANALYZE` thu thập các tóm tắt gần đúng về bảng, cột và biểu thức trong `pg_statistic`; `pg_stats` là view dễ đọc để kiểm tra chúng.[^pg-statistics]

Luồng đầy đủ:

```text
ANALYZE
  → pg_class: reltuples, relpages
  → pg_statistic: phân bố giá trị theo cột
  → planner tính selectivity cho predicate
  → estimated rows tại scan, filter, join và aggregate
  → cost cho từng path
  → execution plan được chọn
```

`pg_class.reltuples` và `pg_class.relpages` cung cấp quy mô row/page ước lượng cho bảng và index. Với `WHERE`, planner dùng statistics cột để ước lượng *selectivity*: tỷ lệ row thỏa một điều kiện.[^pg-statistics]

```text
estimated_rows = input_rows × selectivity
```

Ví dụ:

```sql
SELECT *
FROM orders
WHERE status = 'pending'
  AND created_at >= DATE '2026-09-01';
```

| Bước                    | Statistics planner dùng                                            | Kết quả                                          |
| ----------------------- | ------------------------------------------------------------------ | ------------------------------------------------ |
| `status = 'pending'`    | `most_common_vals`, `most_common_freqs`, `n_distinct`, `null_frac` | Ước lượng tỷ lệ đơn có trạng thái `pending`.     |
| `created_at >= ...`     | Most-common values và `histogram_bounds`                           | Ước lượng tỷ lệ row thuộc khoảng ngày.           |
| Hai điều kiện cùng đúng | Selectivity riêng lẻ hoặc extended statistics                      | Ước lượng số row sau filter.                     |
| Chọn scan               | Estimated rows, số page, correlation và index khả dụng             | So sánh `Seq Scan`, `Index Scan`, `Bitmap Scan`. |

Nếu `pending` chiếm 70% bảng, `Index Scan` có thể đắt hơn `Seq Scan` vì phải đọc heap cho phần lớn row. Nếu giá trị chỉ chiếm 0,2%, index scan thường rẻ hơn. Vì vậy statistics không trực tiếp làm PostgreSQL “dùng index”; chúng cho planner cơ sở để đánh giá lợi ích của index.[^crunchy-stats]

### 4.1. Statistics cho một predicate

Với equality predicate, planner ưu tiên frequency của giá trị trong danh sách most-common values (MCV). Nếu giá trị không có trong MCV list, nó giả định phần giá trị còn lại phân bố đều dựa trên `n_distinct`, phần tần suất MCV và `null_frac`.[^pg-row-estimation]

Với range predicate như `<`, `>`, `BETWEEN`, planner cộng phần MCV thỏa điều kiện với phần ước lượng từ histogram cho dữ liệu không thuộc MCV:

```text
selectivity = mcv_selectivity
            + histogram_selectivity × non_mcv_fraction
```

`correlation` cho biết thứ tự vật lý của row có gần thứ tự giá trị của cột hay không. Khi nó gần `-1` hoặc `1`, planner ước lượng index range scan rẻ hơn vì số heap page cần đọc ngẫu nhiên giảm.[^pg-stats-view]

### 4.2. Nhiều predicate, join và extended statistics

Không có extended statistics, planner giả định các predicate ở cột khác nhau độc lập:

```text
selectivity(A AND B) = selectivity(A) × selectivity(B)
```

Giả định này dễ sai với `country = 'VN' AND city = 'Hanoi'`. Nếu estimate quá thấp, planner có thể chọn `Nested Loop`; số row thực tế lớn sẽ khiến bước trong bị lặp nhiều lần và query chậm.

Với equality join, planner lấy statistics của cột join ở cả hai bảng để ước lượng số row sau join. Với join hay filter nhiều cột có quan hệ, tạo extended statistics có chọn lọc:[^pg-row-estimation]

```sql
CREATE STATISTICS customer_location_stats
  (dependencies, ndistinct, mcv)
ON country, city
FROM customers;

ANALYZE customers;
```

| Loại           | Planner dùng để làm gì                                                |
| -------------- | --------------------------------------------------------------------- |
| `dependencies` | Điều chỉnh selectivity của equality predicate trên các cột phụ thuộc. |
| `ndistinct`    | Ước lượng số nhóm của `GROUP BY` hoặc `DISTINCT` nhiều cột.           |
| `mcv`          | Ước lượng các tổ hợp giá trị phổ biến hoặc không tồn tại.             |

Extended statistics hiện không được dùng cho selectivity estimation của table join. Functional dependencies cũng chỉ áp dụng cho equality với hằng số và `IN` hằng số, không áp dụng cho range predicate, `LIKE`, so sánh cột với cột hay cột với biểu thức.[^pg-create-statistics]

## 5. Từ sai statistics đến plan chậm

```text
Statistics cũ hoặc không đủ chi tiết
  → planner estimate 100 row
  → thực tế 500.000 row
  → một path như Nested Loop trông rẻ một cách sai lệch
  → executor thực thi với số vòng lặp và I/O lớn
  → query chậm
```

Khi chẩn đoán, dùng `EXPLAIN (ANALYZE, BUFFERS)` và tìm node đầu tiên, từ lá lên gốc, có `rows` lệch xa `actual rows`. Sau đó kiểm tra [[Statistics|planner statistics]], chạy `ANALYZE`, tăng statistics target cho cột phân bố lệch hoặc tạo extended statistics cho nhóm cột thực sự cùng xuất hiện trong filter/grouping.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Không nên ép một loại plan bằng các biến `enable_*` như giải pháp lâu dài. Tài liệu PostgreSQL khuyến nghị cải thiện statistics hoặc cost constants để nâng chất lượng plan thay vì dựa vào cách ép planner chọn một phương án.[^pg-query-config]

[^pg-planner]: [PostgreSQL 18 — Planner/Optimizer](https://www.postgresql.org/docs/18/planner-optimizer.html)
[^pg-geqo]: [PostgreSQL 18 — Genetic Query Optimizer](https://www.postgresql.org/docs/18/geqo-pg-intro.html)
[^pg-statistics]: [PostgreSQL 18 — Statistics Used by the Planner](https://www.postgresql.org/docs/18/planner-stats.html)
[^pg-stats-view]: [PostgreSQL 18 — pg_stats](https://www.postgresql.org/docs/18/view-pg-stats.html)
[^pg-row-estimation]: [PostgreSQL 18 — Row Estimation Examples](https://www.postgresql.org/docs/18/row-estimation-examples.html)
[^pg-create-statistics]: [PostgreSQL 18 — CREATE STATISTICS](https://www.postgresql.org/docs/18/sql-createstatistics.html)
[^pg-query-config]: [PostgreSQL 18 — Query Planning Configuration](https://www.postgresql.org/docs/18/runtime-config-query.html)
[^crunchy-stats]: [Crunchy Data — Indexes, Selectivity and Statistics](https://www.crunchydata.com/blog/indexes-selectivity-and-statistics)
