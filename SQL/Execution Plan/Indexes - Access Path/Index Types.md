## 1. Vấn đề đặt ra

Một index chỉ giúp khi cấu trúc của nó phù hợp với predicate, thứ tự trả kết quả và phân bố vật lý của dữ liệu. B-tree có thể đáp ứng phần lớn truy vấn quan hệ, nhưng array, `jsonb`, full-text, dữ liệu không gian và bảng time-series rất lớn cần các access method khác.

Mỗi index cũng làm `INSERT`, `UPDATE`, `DELETE`, `VACUUM` và dung lượng lưu trữ tốn thêm chi phí. Tạo index theo query thực tế, rồi xác nhận bằng `EXPLAIN (ANALYZE, BUFFERS)` thay vì tạo index cho mọi cột.[^pg-indexes]

## 2. Index access method là gì?

PostgreSQL có sáu loại index tích hợp sẵn: B-tree, Hash, GiST, SP-GiST, GIN và BRIN. Mỗi loại dùng thuật toán khác nhau và chỉ hỗ trợ các operator phù hợp với operator class của index. `CREATE INDEX` không có `USING` mặc định tạo B-tree.[^pg-index-types]

```sql
CREATE INDEX idx_orders_customer_id
ON orders (customer_id);

-- Tương đương với:
CREATE INDEX idx_orders_customer_id_btree
ON orders USING btree (customer_id);
```

| Loại | Phù hợp nhất | Truy vấn/operator thường dùng | Không phù hợp |
|---|---|---|---|
| B-tree | Giá trị scalar: ID, timestamp, số, text, enum. | `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `ORDER BY`, prefix `LIKE`. | Substring như `'%foo%'`, tìm thành phần trong `jsonb`/array. |
| Hash | Equality lookup thuần túy. | `=` | Range, sort, `ORDER BY`. |
| GiST | Không gian, range, geometric, PostGIS, nearest-neighbor. | Overlap, contains, contained-by, distance `<->`. | Equality/range scalar thông thường. |
| SP-GiST | Dữ liệu phân vùng theo không gian hoặc tiền tố. | Tùy operator class; có thể nearest-neighbor. | Query scalar phổ thông nếu B-tree phù hợp. |
| GIN | Giá trị có nhiều thành phần: array, `jsonb`, full-text. | Containment, overlap, key presence, `@@`. | Equality lookup đơn giản trên bảng write-heavy. |
| BRIN | Bảng rất lớn có dữ liệu gần theo thứ tự vật lý. | Equality/range trên time, date hoặc ID tăng dần. | Giá trị phân bố ngẫu nhiên trên disk. |

## 3. B-tree

B-tree là loại mặc định và nên là lựa chọn đầu tiên cho phần lớn bảng nghiệp vụ. Nó giữ key theo thứ tự, nên hỗ trợ equality, range và có thể trả row theo thứ tự cho `ORDER BY`.[^pg-index-types]

```sql
CREATE INDEX idx_orders_created_at
ON orders (created_at);

SELECT *
FROM orders
WHERE created_at >= TIMESTAMPTZ '2026-09-01'
  AND created_at <  TIMESTAMPTZ '2026-10-01'
ORDER BY created_at;
```

B-tree dùng được với `LIKE 'foo%'` hoặc regex có neo ở đầu như `~ '^foo'`, nhưng không dùng được với `LIKE '%foo%'`. Với locale khác `C`, prefix matching có thể cần operator class như `text_pattern_ops`.[^pg-index-types]

Primary key và unique constraint thường tạo unique B-tree index:

```sql
CREATE TABLE customers (
  customer_id bigint PRIMARY KEY,
  email text UNIQUE
);
```

Không mặc định xem `Seq Scan` là lỗi. Nếu predicate trả về phần lớn bảng, planner có thể đánh giá quét tuần tự rẻ hơn index scan cộng nhiều heap fetch.

## 4. Hash

Hash index lưu hash code 32-bit từ giá trị cột, nên chỉ hỗ trợ simple equality comparison.[^pg-index-types]

```sql
CREATE INDEX idx_sessions_token_hash
ON sessions USING hash (token);

SELECT *
FROM sessions
WHERE token = 'abc123';
```

B-tree cũng hỗ trợ `=`, lại dùng được cho range và sort. Vì vậy Hash chỉ nên cân nhắc khi workload thực sự chỉ có equality lookup trên cột đó.

## 5. GiST và SP-GiST

GiST và SP-GiST là framework cho nhiều operator class, không phải một index có tập operator cố định.

GiST phù hợp với quan hệ không gian/range và nearest-neighbor:

```sql
CREATE INDEX idx_places_location
ON places USING gist (location);

SELECT *
FROM places
ORDER BY location <-> point '(101,456)'
LIMIT 10;
```

Các use case thường gặp là PostGIS, geometric type và range như `daterange`, `tsrange`:

```sql
CREATE INDEX idx_bookings_period
ON bookings USING gist (period);

SELECT *
FROM bookings
WHERE period && daterange('2026-09-01', '2026-09-08');
```

SP-GiST hỗ trợ các cấu trúc phân vùng không cân bằng như quadtree, k-d tree và radix tree/trie. Nó phù hợp với dữ liệu có không gian hoặc prefix có thể chia nhánh rõ rệt, chẳng hạn point hoặc network, tùy operator class.[^pg-index-types]

## 6. GIN

GIN là *inverted index*: nó lưu entry cho từng thành phần của giá trị có nhiều thành phần, thay vì coi toàn bộ array hay document là một key duy nhất.[^pg-index-types]

```sql
CREATE INDEX idx_articles_tags
ON articles USING gin (tags);

SELECT *
FROM articles
WHERE tags @> ARRAY['postgres'];
```

GIN thường được dùng cho array, `jsonb` và full-text search:

```sql
-- Array chứa tất cả phần tử chỉ định
WHERE tags @> ARRAY['postgres', 'sql']

-- Array có ít nhất một phần tử chung
WHERE tags && ARRAY['postgres', 'database']

-- JSONB containment
WHERE metadata @> '{"status":"active"}'::jsonb

-- Full-text search
WHERE search_vector @@ plainto_tsquery('postgres planner')
```

GIN thường lớn hơn và tốn chi phí cập nhật hơn B-tree, nên không phải lựa chọn tốt cho equality lookup đơn giản hoặc write-heavy table không có query theo thành phần.

## 7. BRIN

BRIN (*Block Range INdex*) không lưu entry cho từng row. Nó lưu summary, thường là `min`/`max`, cho một dải block vật lý liên tiếp. Vì vậy index rất nhỏ và hiệu quả khi giá trị cột tương quan cao với thứ tự vật lý của row.[^pg-index-types]

```sql
CREATE INDEX idx_events_created_at_brin
ON events USING brin (created_at);

SELECT *
FROM events
WHERE created_at >= TIMESTAMPTZ '2026-09-01'
  AND created_at <  TIMESTAMPTZ '2026-09-02';
```

BRIN hợp với event/log append-only, time-series và bảng hàng trăm triệu hoặc hàng tỷ row, khi `created_at` hoặc ID tăng dần gần theo thứ tự insert. Nó không tốt với UUID ngẫu nhiên hoặc timestamp được insert lộn xộn, vì rất nhiều block range sẽ phải được kiểm tra.

Với truy vấn equality trên primary key unique:

```sql
SELECT *
FROM target
WHERE id = :id
  AND created_at >= TIMESTAMPTZ '2026-01-01';
```

B-tree primary key trên `id` phù hợp hơn BRIN: planner tìm tối đa một row theo ID rồi kiểm tra điều kiện thời gian. Composite index `(id, created_at)` không đem lại nhiều lợi ích nếu `id` unique.

Ngược lại, nếu cột đầu không unique và thường lọc equality rồi range, dùng composite B-tree:

```sql
CREATE INDEX idx_events_customer_created_at
ON events (customer_id, created_at);

SELECT *
FROM events
WHERE customer_id = :customer_id
  AND created_at >= TIMESTAMPTZ '2026-01-01';
```

BRIN `created_at` vẫn có thể hữu ích song song cho query báo cáo quét một khoảng thời gian lớn trên toàn bảng.

## 8. Chọn loại index và xác minh

```text
ID, foreign key, timestamp, status, email, amount
  → B-tree

Equality lookup duy nhất, không range/sort
  → B-tree trước; cân nhắc Hash khi có bằng chứng workload phù hợp

Array, JSONB, full-text search
  → GIN

Geospatial, range, nearest-neighbor
  → GiST

Dữ liệu point/prefix/network có cách phân vùng phù hợp
  → SP-GiST

Log/time-series cực lớn, dữ liệu gần theo thứ tự insert
  → BRIN
```

Planner chỉ dùng index nếu [[Statistics|statistics]] và cost model cho thấy index path rẻ hơn sequential scan. Xác minh bằng:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

[^pg-indexes]: [PostgreSQL 18 — Indexes](https://www.postgresql.org/docs/18/indexes.html)
[^pg-index-types]: [PostgreSQL 18 — Index Types](https://www.postgresql.org/docs/18/indexes-types.html)
