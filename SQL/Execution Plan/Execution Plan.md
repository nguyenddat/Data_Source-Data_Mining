## 1. Vấn đề đặt ra

Một câu SQL có thể trả cùng kết quả qua nhiều cách đọc dữ liệu, join, sắp xếp và aggregate, nhưng thời gian cùng lượng I/O có thể chênh lệch rất lớn. Execution plan là bằng chứng cho biết PostgreSQL dự định làm gì hoặc đã thực sự làm gì, để việc tối ưu dựa trên số liệu thay vì suy đoán.

## 2. Execution plan là gì?
Execution plan là cây các operator mà PostgreSQL dùng để tạo kết quả query. Các node ở đáy lấy row từ bảng, index hoặc nguồn dữ liệu khác; node phía trên lọc, join, sort hoặc aggregate các row đó; node trên cùng trả kết quả.[^pg-using]

```text
Hash Join
  → Seq Scan on orders
  → Hash
       → Index Scan on customers
```

Trong cây này, `Seq Scan on orders` và `Index Scan on customers` tạo hai luồng row; `Hash` xây cấu trúc tra cứu từ một luồng; `Hash Join` ghép chúng. Đọc node từ dưới lên để theo dõi dữ liệu đi qua plan, nhưng xem node trên cùng để biết tổng kết quả.

| Loại plan                  | Cách lấy         | Dùng để làm gì                          |
| -------------------------- | ---------------- | --------------------------------------- |
| [[Estimated Execution Plan]] |  `EXPLAIN SELECT ...`                    | Xem phương án planner dự định dùng mà không chạy query. |
| [[Actual Execution Plan]]    |  `EXPLAIN (ANALYZE, BUFFERS) SELECT ...` | So sánh dự đoán với runtime, I/O và số row thực tế.     |

## 3. Lấy execution plan an toàn

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 42;
```

Thêm `ANALYZE` để PostgreSQL thực thi query và đưa runtime statistics vào output:

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT *
FROM orders
WHERE customer_id = 42;
```

`ANALYZE` trong `EXPLAIN` khác với lệnh `ANALYZE table_name`: tùy chọn của `EXPLAIN` chạy query để đo kết quả thực tế, còn lệnh `ANALYZE` thu thập [[Statistics|planner statistics]]. `EXPLAIN ANALYZE` thực thi cả DML; với lệnh thay đổi dữ liệu, dùng transaction rồi rollback nếu chỉ muốn quan sát plan:[^pg-explain]

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS)
UPDATE orders
SET status = 'archived'
WHERE order_date < DATE '2020-01-01';
ROLLBACK;
```

## 4. Cách đọc một node

```text
Index Scan using orders_customer_id_idx on orders
  (cost=0.29..8.31 rows=3 width=120)
  (actual time=0.024..0.031 rows=4 loops=1)
```

| Trường | Ý nghĩa | Cách dùng khi phân tích |
|---|---|---|
| `Index Scan` | Loại operator. | Biết PostgreSQL đọc dữ liệu bằng cách nào. |
| `using ...` | Index đang dùng. | Xác nhận index nào tham gia plan. |
| `cost=a..b` | Startup cost và total cost dự kiến. | Chỉ so sánh tương đối giữa các plan; không diễn giải thành milliseconds. |
| `rows=n` | Số row planner dự đoán node xuất ra. | So với `actual rows` để phát hiện cardinality estimate sai. |
| `width=n` | Kích thước trung bình ước lượng của row đầu ra, theo byte. | Giải thích chi phí sort, hash, memory và truyền dữ liệu. |
| `actual time=a..b` | Thời gian tới row đầu tiên và tới khi node hoàn tất, theo ms. | Xác định phần việc runtime tốn kém. |
| `actual rows=n` | Số row thực tế node xuất ra trong mỗi loop. | Nhân với `loops` khi cần ước lượng tổng row tạo ra. |
| `loops=n` | Số lần node chạy. | Tìm inner node bị lặp nhiều trong `Nested Loop`. |

Cost của node cha bao gồm cost node con và giả định node chạy đến hết. Với `LIMIT`, node cha có thể dừng sớm, nên total cost không phải luôn là lượng công việc thực tế sẽ xảy ra.[^pg-using]

`actual time` và `actual rows` là số liệu trung bình trên mỗi loop. Vì vậy node có `actual rows=1 loops=100000` đã tạo xấp xỉ 100.000 row và được gọi 100.000 lần; đây là dấu hiệu quan trọng khi tìm nested loop đắt.[^pg-using]

## 5. Các node thường gặp

| Node | Chức năng | Dấu hiệu cần xem tiếp |
|---|---|---|
| `Seq Scan` | Đọc tuần tự bảng. | Chỉ đáng nghi khi bảng lớn, predicate chọn ít row, nhưng vẫn đọc nhiều buffer. |
| `Index Scan` | Dò index rồi lấy row từ heap. | Có thể đắt nếu trả nhiều row hoặc dữ liệu vật lý phân tán. |
| `Index Only Scan` | Lấy dữ liệu từ index khi visibility map cho phép. | `Heap Fetches` cao làm lợi ích của nó giảm. |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | Gom vị trí row bằng bitmap rồi đọc heap theo page. | Phù hợp với số row ở giữa index scan và seq scan; xem `Recheck Cond` và buffer. |
| `Nested Loop` | Với mỗi row nhánh ngoài, chạy nhánh trong. | Xem `loops` của nhánh trong và cardinality nhánh ngoài. |
| `Hash Join` | Hash một đầu vào, quét đầu kia để ghép. | Kiểm tra hash có spill sang temporary file không. |
| `Merge Join` | Ghép hai đầu vào đã sắp theo join key. | Kiểm tra chi phí `Sort` hoặc index cung cấp sẵn thứ tự. |
| `Sort` | Sắp xếp row. | `Sort Method: external merge` cho biết đã dùng temporary disk. |
| `HashAggregate` / `GroupAggregate` | Gom nhóm và aggregate. | Kiểm tra số group estimate, memory và temp I/O. |
| `Materialize` | Lưu tạm output của node con để tái sử dụng. | Hợp lý khi bị đọc lại; cần xem số row và số lần đọc lại. |

Đừng xem `Seq Scan` như lỗi mặc định. Nếu query cần phần lớn dữ liệu, sequential scan thường rẻ hơn index scan kèm nhiều heap fetch.[^pg-using]

## 6. Đọc buffers và runtime

Với `BUFFERS`, PostgreSQL hiển thị số lần truy cập buffer tại node:

| Giá trị | Ý nghĩa |
|---|---|
| `shared hit` | Block đã có trong PostgreSQL shared buffer. |
| `shared read` | Block phải đọc vào shared buffer. |
| `shared dirtied` | Block bị thay đổi trong buffer. |
| `shared written` | Block được ghi từ buffer ra storage. |
| `temp read` / `temp written` | I/O temporary file, thường do sort, hash hoặc materialize không vừa memory. |

Buffer là số lần truy cập block, không phải số page riêng biệt. `shared read` cao thường hướng điều tra sang I/O hoặc lượng dữ liệu đọc; `temp read`/`temp written` chỉ ra spill ra disk.[^pg-explain]

## 7. Quy trình chẩn đoán

1. Chạy `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)` trong điều kiện có thể đại diện cho query cần điều tra.
2. Xem `Execution Time`, sau đó đọc node từ lá lên gốc.
3. Tìm node đầu tiên có `rows` lệch xa `actual rows`; đây thường là nguồn lỗi ước lượng lan truyền.
4. Kiểm tra node đó và node cha có `loops` cao, `shared read` cao hoặc temporary I/O không.
5. Đối chiếu [[Planner & Optimizer]] để hiểu lựa chọn scan/join, rồi kiểm tra [[Statistics]]: `ANALYZE`, MCV/histogram, correlation hoặc extended statistics.
6. Sửa nguyên nhân và chạy lại cùng điều kiện để xác nhận estimate, I/O và execution time đã cải thiện.

Ví dụ dưới đây không có nghĩa `Nested Loop` tự nó là lỗi:

```text
Nested Loop                        rows=20       actual rows=400000
  → Index Scan customers           rows=20       actual rows=400000
  → Index Scan orders              rows=1        actual rows=3 loops=400000
```

Planner tin nhánh `customers` chỉ trả 20 row, nên nested loop trông rẻ. Thực tế node đó tạo 400.000 row, làm index scan trên `orders` bị gọi 400.000 lần. Điều tra bắt đầu ở predicate và statistics của `customers`, không phải bằng cách ép tắt nested loop.

[^pg-using]: [PostgreSQL 18 — Using EXPLAIN](https://www.postgresql.org/docs/18/using-explain.html)
[^pg-explain]: [PostgreSQL 18 — EXPLAIN](https://www.postgresql.org/docs/18/sql-explain.html)
