## 1. Vấn đề đặt ra

Nhiều query lọc đồng thời nhiều cột, chẳng hạn một khách hàng trong một khoảng thời gian. Hai index đơn có thể được kết hợp, nhưng composite index thường định vị dữ liệu và giữ thứ tự hiệu quả hơn cho một query shape cụ thể. Đổi lại, thứ tự key column quyết định phần index mà PostgreSQL có thể bỏ qua.

```sql
SELECT *
FROM events
WHERE customer_id = :customer_id
  AND created_at >= :from_time
  AND created_at < :to_time;
```

## 2. Multicolumn index là gì?

Multicolumn index, hay composite index, có nhiều key column:

```sql
CREATE INDEX idx_events_customer_created
ON events (customer_id, created_at);
```

PostgreSQL hỗ trợ nhiều key column cho B-tree, GiST, GIN và BRIN. Tối đa có 32 cột, kể cả `INCLUDE` column, nhưng index hơn ba key column hiếm khi hữu ích nếu workload không rất chuyên biệt.[^pg-multicolumn]

Phần này kết nối ba mục liên tiếp trong tài liệu PostgreSQL:

| Mục | Nội dung |
|---|---|
| §11.3 | Cách key column và thứ tự cột ảnh hưởng khả năng tìm kiếm của multicolumn index. |
| §11.4 | Cách B-tree index cung cấp `ORDER BY` để tránh node `Sort`. |
| §11.5 | Khi dùng các index riêng và Bitmap AND/OR thay vì một composite index. |

## 3. B-tree: thứ tự cột quyết định đường tìm kiếm

B-tree `(a, b, c)` được sắp theo `a`, rồi theo `b` trong mỗi nhóm `a`, rồi theo `c` trong mỗi nhóm `(a, b)`:

```text
customer_id = 1
  → 2026-01-01
  → 2026-01-02

customer_id = 2
  → 2026-01-01
  → 2026-01-03
```

Quy tắc chính xác là: **equality condition trên các cột đầu, cộng với inequality/range condition trên cột đầu tiên không có equality, luôn giới hạn phần index phải quét**. Các condition ở bên phải range vẫn có thể được kiểm tra trong index và giảm heap fetch, nhưng thường không thu hẹp đoạn leaf page cần quét.[^pg-multicolumn]

Vì thế, cấu hình sau phù hợp với query ở phần 1:

```sql
CREATE INDEX idx_events_customer_created
ON events (customer_id, created_at);
```

| Cột index | Predicate | Tác dụng |
|---|---|---|
| `customer_id` | `= :customer_id` | Định vị nhóm key của một khách hàng. |
| `created_at` | `>= :from_time AND < :to_time` | Chỉ quét khoảng thời gian cần thiết trong nhóm đó. |

Đảo thứ tự thành `(created_at, customer_id)` thường kém hiệu quả hơn cho cùng query: planner phải đi vào toàn bộ khoảng thời gian trước, rồi mới kiểm tra `customer_id` trong khoảng đó.

### 3.1. Equality trước, range sau

Với query có nhiều equality predicate rồi một range predicate, bố trí B-tree thường là:

```sql
CREATE INDEX idx_orders_customer_status_created
ON orders (customer_id, status, created_at);
```

```sql
SELECT *
FROM orders
WHERE customer_id = :customer_id
  AND status = 'pending'
  AND created_at >= :from_time
  AND created_at < :to_time;
```

Không nên diễn giải thành “cột selectivity cao nhất phải đứng đầu” như một quy tắc tuyệt đối. Khi các predicate đều là equality, B-tree có thể định vị tổ hợp key; thứ tự nên được chọn theo query shape thường gặp, range predicate, `ORDER BY` và các query chỉ dùng một phần prefix của index.

### 3.2. Cột bên phải không có điều kiện

Index `(customer_id, created_at)` vẫn hữu ích cho:

```sql
WHERE customer_id = :customer_id
```

Nhưng query chỉ có:

```sql
WHERE created_at >= :from_time
```

không tận dụng được leftmost prefix theo cách thông thường. PostgreSQL 18 có B-tree *skip scan*: planner có thể thử tạo equality constraint nội bộ cho từng giá trị distinct của `customer_id`, rồi tìm `created_at` trong mỗi nhóm. Điều này chỉ hữu ích khi cột đầu có ít giá trị distinct; khi có rất nhiều giá trị, planner thường không dùng index đó.[^pg-multicolumn]

## 4. Composite B-tree và ORDER BY

B-tree là index type có thể trả output theo thứ tự. Một composite index phù hợp có thể vừa lọc, vừa tránh `Sort`:[^pg-ordering]

```sql
CREATE INDEX idx_events_customer_created_desc
ON events (customer_id, created_at DESC);

SELECT *
FROM events
WHERE customer_id = :customer_id
ORDER BY created_at DESC
LIMIT 50;
```

Planner có thể định vị nhóm `customer_id`, đọc 50 row đầu theo `created_at DESC`, rồi dừng. Với bảng event lớn, đây thường tốt hơn đọc toàn bộ row phù hợp rồi sort.

Thứ tự và chiều sort của các cột trong index phải phù hợp với `ORDER BY`. Ví dụ `(x ASC, y DESC)` có ích cho `ORDER BY x ASC, y DESC`, vì index quét tiến hoặc lùi đơn thuần không thể tạo mọi tổ hợp chiều sort.[^pg-ordering]

Nếu query không khóa cột đầu:

```sql
SELECT *
FROM events
ORDER BY created_at DESC
LIMIT 50;
```

index `(customer_id, created_at DESC)` không tạo global order theo `created_at`; cần index riêng `(created_at DESC)` nếu query này phổ biến.

## 5. Một composite index hay nhiều index đơn?

Với workload có ba query shape:

```sql
WHERE customer_id = :customer_id
WHERE created_at >= :from_time
WHERE customer_id = :customer_id AND created_at >= :from_time
```

có hai lựa chọn chính.

| Thiết kế | Ưu điểm | Đánh đổi |
|---|---|---|
| Hai index đơn `(customer_id)` và `(created_at)` | Mỗi query một cột có index trực tiếp; planner có thể Bitmap AND cho query có hai điều kiện. | Quét hai index, tạo bitmap, mất thứ tự index; `ORDER BY` thường cần sort. |
| Composite `(customer_id, created_at)` | Thường hiệu quả hơn cho query dùng cả hai điều kiện; có thể đáp ứng `ORDER BY created_at` khi khóa `customer_id`. | Không tốt cho query chỉ có `created_at`, trừ khi skip scan có hiệu quả. |

PostgreSQL có thể kết hợp index riêng qua `BitmapAnd` hoặc `BitmapOr`. Nó tạo bitmap vị trí row từ từng index rồi kết hợp các bitmap. Row sau đó được đọc theo thứ tự vật lý, nên thứ tự sắp xếp của index không còn và query có `ORDER BY` có thể cần `Sort`.[^pg-bitmap]

Trong hệ thống đọc nhiều, có nhiều query thuộc cả ba dạng, `(customer_id, created_at)` cùng một index riêng `(created_at)` có thể hợp lý. Ba index gồm `(customer_id)`, `(created_at)`, `(customer_id, created_at)` chỉ phù hợp khi table ít update và mọi query shape đều phổ biến.[^pg-bitmap]

## 6. Multicolumn index ngoài B-tree

| Loại index | Tác động của thứ tự key column |
|---|---|
| B-tree | Rất quan trọng; equality ở đầu, range sau. |
| GiST | Cột đầu quan trọng; index tương đối kém hiệu quả nếu cột đầu có rất ít distinct value. |
| GIN | Có thể dùng với query condition trên bất kỳ subset cột nào; hiệu quả tìm kiếm không phụ thuộc thứ tự cột. |
| BRIN | Có thể dùng với query condition trên bất kỳ subset cột nào; hiệu quả tìm kiếm không phụ thuộc thứ tự cột. |

Multicolumn BRIN không thay thế B-tree `(tenant_id, created_at)` cho lookup hẹp theo tenant và thời gian. BRIN chỉ loại block range không liên quan. Nó hợp khi các cột đều có correlation với vị trí vật lý, chẳng hạn bảng log append-only; lý do chính để dùng BRIN riêng thay vì multicolumn BRIN là cần `pages_per_range` khác nhau.[^pg-multicolumn]

## 7. Kiểm chứng thiết kế

Tạo index theo query phổ biến và kiểm chứng trong điều kiện đại diện:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM events
WHERE customer_id = :customer_id
  AND created_at >= :from_time
  AND created_at < :to_time
ORDER BY created_at DESC
LIMIT 50;
```

Kiểm tra:

1. Index condition có dùng các cột đúng thứ tự không.
2. Có `Sort` không và liệu index có thể đáp ứng `ORDER BY` không.
3. `rows` có gần `actual rows` không; nếu không, đối chiếu [[Statistics]].
4. Buffer read, heap fetch và execution time có tốt hơn thiết kế cũ không.

[^pg-multicolumn]: [PostgreSQL 18 — Multicolumn Indexes (§11.3)](https://www.postgresql.org/docs/18/indexes-multicolumn.html)
[^pg-ordering]: [PostgreSQL 18 — Indexes and ORDER BY (§11.4)](https://www.postgresql.org/docs/18/indexes-ordering.html)
[^pg-bitmap]: [PostgreSQL 18 — Combining Multiple Indexes (§11.5)](https://www.postgresql.org/docs/18/indexes-bitmap-scans.html)
