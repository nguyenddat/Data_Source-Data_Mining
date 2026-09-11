## 1. Vấn đề đặt ra

Trong **Execution Plan**, optimizer phải chọn cách đọc bảng, dùng index, join và cấp tài nguyên **trước** khi câu SQL chạy. Không thể đọc toàn bộ dữ liệu để quyết định; vì vậy PostgreSQL cần một bản tóm tắt nhỏ, đủ đại diện để ước lượng số dòng thỏa điều kiện và chi phí của từng phương án.

Nếu bản tóm tắt cũ hoặc không phản ánh phân bố dữ liệu, optimizer có thể đoán sai cardinality, rồi chọn scan, thứ tự join hoặc thuật toán join không phù hợp. Đây là một nguyên nhân thường gặp khi `EXPLAIN` ước lượng rất khác `EXPLAIN ANALYZE`.

## 2. Planner statistics là gì?
Là các số liệu **xấp xỉ** về nội dung bảng, cột và biểu thức. Lệnh `ANALYZE` lấy mẫu dữ liệu, lưu kết quả trong system catalog `pg_statistic`; query planner dùng chúng để ước lượng selectivity và chọn execution plan.[^pg-analyze]

`pg_statistic` bị hạn chế quyền đọc vì có thể làm lộ đặc điểm của dữ liệu. **pg_stats** là view công khai, dễ đọc hơn, chỉ hiển thị statistics của các bảng mà người dùng có quyền đọc.[^pg-stats]

Đừng nhầm planner statistics với các view giám sát như `pg_stat_activity`, `pg_stat_user_tables` hoặc extension `pg_stat_statements`. Các view giám sát ghi nhận hoạt động và tải hệ thống; `pg_stats` mô tả phân bố dữ liệu để optimizer dự đoán **trước** khi chạy query.

## 3. PostgreSQL dùng statistics như thế nào?
Với truy vấn sau, planner cần ước lượng số dòng thỏa từng predicate rồi quyết định dùng `Seq Scan`, `Index Scan`, `Bitmap Scan`, hay một kiểu join phù hợp:

```sql
SELECT *
FROM orders
WHERE status = 'pending'
  AND created_at >= DATE '2026-09-01';
```

Ví dụ, nếu `pending` chiếm 70% bảng, index scan có thể đắt hơn sequential scan vì phải lấy quá nhiều row từ heap. Ngược lại, một giá trị hiếm có thể phù hợp với index scan. Ước lượng số dòng là nguyên liệu chính để planner tính cost của các plan.[^pg-planner]

```text
ANALYZE
  → lấy mẫu dữ liệu
  → ghi tóm tắt vào pg_statistic
  → pg_stats trình bày dữ liệu dễ đọc
  → planner ước lượng cardinality và cost
  → EXPLAIN ANALYZE kiểm chứng estimate với thực tế
```

## 4. Statistics một cột trong pg_stats
Mỗi hàng trong `pg_stats` mô tả một cột đã được phân tích.

| Trường                                                                 | Nội dung                                                                                                             | Ứng dụng trong planning                                                                       |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `null_frac`                                                            | Tỷ lệ giá trị `NULL`.                                                                                                | Ước lượng `IS NULL`, `IS NOT NULL` và predicate có `NULL`.                                    |
| `avg_width`                                                            | Kích thước trung bình của giá trị, theo byte.                                                                        | Ước lượng I/O, bộ nhớ cho sort/hash và kích thước kết quả.                                    |
| `n_distinct`                                                           | Số giá trị khác nhau ước lượng. Nếu âm, trị tuyệt đối là tỷ lệ distinct trên tổng số row; `-1` thường là cột unique. | Ước lượng equality predicate, `GROUP BY`, `DISTINCT` và số nhóm aggregate.                    |
| `most_common_vals`                                                     | Các giá trị xuất hiện nhiều nhất (MCV).                                                                              | Nhận biết dữ liệu lệch; ước lượng chính xác hơn cho equality predicate trên giá trị phổ biến. |
| `most_common_freqs`                                                    | Tần suất tương ứng của MCV.                                                                                          | Tính tỷ lệ row khớp với giá trị MCV.                                                          |
| `histogram_bounds`                                                     | Các mốc chia phần dữ liệu không phải MCV thành nhóm có số row gần bằng nhau.                                         | Ước lượng điều kiện `<`, `>`, `BETWEEN` và range predicate.                                   |
| `correlation`                                                          | Tương quan từ `-1` đến `1` giữa thứ tự vật lý của row và thứ tự giá trị cột.                                         | Giá trị gần `-1` hoặc `1` giúp index range scan rẻ hơn do giảm truy cập disk ngẫu nhiên.      |
| `most_common_elems`, `most_common_elem_freqs`, `elem_count_histogram`  | Giá trị phần tử phổ biến và phân bố số phần tử của cột array.                                                        | Ước lượng các toán tử array, chẳng hạn `tags @> ARRAY['postgres']`.                           |
| `range_length_histogram`, `range_empty_frac`, `range_bounds_histogram` | Độ dài, tỷ lệ rỗng và cận của dữ liệu kiểu range.                                                                    | Ước lượng các predicate overlap, containment và range khác.                                   |

Các trường thống kê cho array và range chỉ có ý nghĩa với kiểu dữ liệu tương ứng; với kiểu scalar chúng thường là `NULL`.[^pg-stats]

### 4.1. Kiểm tra statistics của một cột
```sql
SELECT
  null_frac,
  n_distinct,
  most_common_vals,
  most_common_freqs,
  histogram_bounds,
  correlation
FROM pg_stats
WHERE schemaname = 'public'
  AND tablename = 'orders'
  AND attname = 'status';
```

Kết quả như dưới đây cho thấy `pending` chiếm khoảng 70% dữ liệu. Vì predicate này trả về phần lớn bảng, việc planner chọn `Seq Scan` có thể hoàn toàn hợp lý.

```text
most_common_vals  = {pending,completed,cancelled}
most_common_freqs = {0.70,0.28,0.02}
```

## 5. Extended statistics cho nhiều cột và biểu thức

Statistics mặc định chủ yếu mô tả từng cột riêng lẻ. Khi có nhiều predicate, planner thường giả định chúng độc lập. Giả định này dễ sai với dữ liệu liên quan, chẳng hạn `country = 'VN'` và `city = 'Hanoi'`.

Tạo extended statistics để mô tả mối quan hệ đó:

```sql
CREATE STATISTICS customer_country_city_stats
  (dependencies, ndistinct, mcv)
ON country, city
FROM customers;

ANALYZE customers;
```

| Loại           | Dùng khi                                                             | Tác dụng                                                      |
| -------------- | -------------------------------------------------------------------- | ------------------------------------------------------------- |
| `dependencies` | Một cột được xác định hoặc phụ thuộc mạnh vào cột khác.              | Tránh nhân selectivity như các predicate hoàn toàn độc lập.   |
| `ndistinct`    | `GROUP BY`, `DISTINCT` hoặc aggregate trên nhiều cột/biểu thức.      | Ước lượng số tổ hợp giá trị riêng biệt.                       |
| `mcv`          | Các tổ hợp giá trị xuất hiện rất thường xuyên hoặc không hề tồn tại. | Ước lượng chính xác selectivity của nhiều equality predicate. |

`CREATE STATISTICS` cũng có thể thu thập statistics cho biểu thức, ví dụ `date_trunc('month', created_at)`. Extended statistics hiện chưa được dùng cho selectivity estimation của table join.[^pg-create-statistics]

## 6. Thu thập và điều chỉnh statistics

Autovacuum thường tự chạy `ANALYZE` sau khi nội dung bảng thay đổi đủ nhiều. Sau bulk load, `INSERT`/`UPDATE` lớn, thay đổi phân bố dữ liệu, hoặc với bảng partitioned, có thể cần chạy thủ công:

```sql
ANALYZE public.orders;
```

Có thể chỉ phân tích các cột liên quan đến lọc, join, sort hoặc group:

```sql
ANALYZE public.orders (status, customer_id, created_at);
```

Với cột có phân bố lệch mạnh, tăng statistics target riêng cho cột đó:

```sql
ALTER TABLE public.orders
  ALTER COLUMN status
  SET STATISTICS 500;

ANALYZE public.orders;
```

`default_statistics_target` mặc định là `100`. Target cao hơn cho nhiều MCV và histogram bucket hơn, nhưng làm `ANALYZE` chậm hơn và tăng dung lượng `pg_statistic`. Vì `ANALYZE` lấy mẫu ngẫu nhiên, statistics luôn gần đúng và có thể thay đổi nhẹ qua các lần chạy dù dữ liệu không đổi.[^pg-analyze]

## 7. Ứng dụng khi tối ưu truy vấn

Khi `EXPLAIN ANALYZE` cho thấy chênh lệch lớn, ví dụ `rows=10` nhưng `actual rows=500000`, hãy kiểm tra statistics trước khi vội thêm index:

```sql
SELECT *
FROM pg_stats
WHERE schemaname = 'public'
  AND tablename = 'orders';
```

Quy trình thực tế:

1. Chạy `ANALYZE` nếu bảng vừa thay đổi lớn hoặc statistics lỗi thời.
2. Tăng target cho cột lọc có phân bố rất lệch nếu sample hiện tại không đủ đại diện.
3. Tạo extended statistics khi sai lệch xuất hiện ở predicate, `GROUP BY` hoặc biểu thức có quan hệ với nhau.
4. Chạy lại `EXPLAIN (ANALYZE, BUFFERS)` trong điều kiện tương đương để xác nhận `estimated rows` đã gần `actual rows` hơn và plan đã cải thiện.

[^pg-analyze]: [PostgreSQL 18 — ANALYZE](https://www.postgresql.org/docs/18/sql-analyze.html)
[^pg-stats]: [PostgreSQL 18 — pg_stats](https://www.postgresql.org/docs/18/view-pg-stats.html)
[^pg-planner]: [PostgreSQL 18 — How the Planner Uses Statistics](https://www.postgresql.org/docs/18/planner-stats-details.html)
[^pg-create-statistics]: [PostgreSQL 18 — CREATE STATISTICS](https://www.postgresql.org/docs/18/sql-createstatistics.html)
