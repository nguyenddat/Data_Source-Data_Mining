## 1. Vấn đề đặt ra

Trong [[Indexes]], application không thể bảo đảm một giá trị là duy nhất bằng cách `SELECT` trước rồi mới `INSERT`: hai transaction đồng thời vẫn có thể cùng thấy dữ liệu chưa tồn tại và cùng ghi duplicate. Quy tắc như email không trùng, mỗi dòng chi tiết chỉ xuất hiện một lần trong đơn hàng, hoặc mỗi user chỉ có một địa chỉ mặc định phải được database thực thi atomically.

## 2. Unique index là gì?

Unique index là B-tree index ngăn nhiều row có cùng giá trị key, hoặc cùng tổ hợp key. Khi `INSERT` hoặc `UPDATE`, PostgreSQL dùng index để kiểm tra uniqueness; nếu key trùng, câu lệnh lỗi và transaction không thể tạo duplicate.[^pg-unique]

```sql
CREATE UNIQUE INDEX users_email_uq_idx
ON users (email);

INSERT INTO users (email) VALUES ('dat@example.com');
INSERT INTO users (email) VALUES ('dat@example.com');
-- ERROR: duplicate key value violates unique constraint
```

Unique index không chỉ kiểm tra “nhanh”. Nó là điểm đồng bộ để PostgreSQL xử lý đúng khi concurrent transaction cùng cố ghi một key: tối đa một transaction có thể commit key đó; transaction còn lại phải chờ kết quả phù hợp hoặc nhận duplicate-key error.

## 3. Unique index, UNIQUE constraint và PRIMARY KEY

| Cách khai báo | Ý nghĩa dữ liệu | PostgreSQL tạo/thực thi |
|---|---|---|
| `CREATE UNIQUE INDEX` | Quy tắc uniqueness được biểu diễn trực tiếp bằng index. | Unique B-tree index. |
| `UNIQUE (email)` | Business rule: email phải duy nhất. | Tự tạo unique B-tree index. |
| `PRIMARY KEY (id)` | Identifier chính của row; duy nhất và không được `NULL`. Mỗi bảng chỉ có một primary key. | Tự tạo unique B-tree index. |

```sql
CREATE TABLE users (
  id bigint PRIMARY KEY,
  email text NOT NULL UNIQUE
);
```

Với một quy tắc dữ liệu thông thường, ưu tiên `UNIQUE` hoặc `PRIMARY KEY`: schema diễn đạt rõ domain rule, PostgreSQL tự tạo index cần thiết. Không tạo thêm index trên cùng key, vì đó là index trùng lặp.[^pg-unique]

`UNIQUE` constraint có thể khai báo `DEFERRABLE`, để kiểm tra uniqueness ở cuối transaction. Unique index tạo trực tiếp không có lựa chọn trì hoãn này.[^pg-constraints]

## 4. Có giúp query nhanh không?

Unique index vẫn là B-tree index, nên tăng tốc equality lookup như index thường:

```sql
SELECT *
FROM users
WHERE email = :email;
```

Lợi ích tốc độ lookup chủ yếu đến từ B-tree, không phải từ từ khóa `UNIQUE`. Tuy nhiên, uniqueness còn giúp planner biết equality predicate trên **toàn bộ unique key** trả tối đa một row. Cardinality estimate chính xác hơn có thể giúp planner chọn thứ tự join và join method phù hợp.

| Khía cạnh | Index thường | Unique index |
|---|---|---|
| Equality lookup | Nhanh nhờ B-tree. | Nhanh tương tự. |
| Cho phép duplicate | Có. | Không. |
| Planner biết tối đa một row khi equality đủ key | Không. | Có. |
| Khi ghi | Thêm entry index. | Kiểm tra duplicate rồi thêm entry index. |
| Mục đích chính | Tăng tốc query. | Enforce data integrity, đồng thời phục vụ lookup. |

Không tạo `UNIQUE` chỉ để mong query nhanh hơn nếu dữ liệu có thể duplicate. Ngược lại, nếu uniqueness là business rule, dùng constraint/index để nhận cả integrity và lợi ích truy vấn.

## 5. Unique key đa cột

```sql
CREATE UNIQUE INDEX order_line_order_product_uq_idx
ON order_lines (order_id, product_id);
```

Index này cấm hai row có cùng cả `order_id` và `product_id`, nhưng vẫn cho phép cùng product xuất hiện ở order khác hoặc cùng order có product khác.

```text
(order_id=1, product_id=10)  ✓
(order_id=1, product_id=11)  ✓
(order_id=2, product_id=10)  ✓
(order_id=1, product_id=10)  ✗ duplicate
```

Đây cũng là B-tree `(order_id, product_id)`, nên dùng tốt cho query theo `order_id` hoặc theo cả hai cột. Query chỉ theo `product_id` không tận dụng leftmost prefix theo cách thông thường; xem [[MultiColumn Indexes]].

## 6. NULL và uniqueness

Mặc định, `NULL` không được coi là bằng `NULL` trong unique index. Vì vậy nhiều `NULL` được phép:

```sql
CREATE TABLE users (
  email text UNIQUE
);

INSERT INTO users VALUES (NULL);
INSERT INTO users VALUES (NULL);
-- cả hai đều hợp lệ
```

Nếu business rule chỉ cho phép một `NULL`, khai báo `NULLS NOT DISTINCT`:

```sql
CREATE TABLE users (
  email text UNIQUE NULLS NOT DISTINCT
);

-- hoặc
CREATE UNIQUE INDEX users_email_uq_idx
ON users (email) NULLS NOT DISTINCT;
```

Với key đa cột, default behavior cũng chỉ từ chối duplicate khi toàn bộ key value được coi là bằng nhau; `NULLS NOT DISTINCT` làm `NULL` tham gia so sánh như một giá trị bằng chính nó.[^pg-unique]

## 7. Unique index trên expression

Unique index hữu ích khi uniqueness không nằm ở raw value của cột. Ví dụ email không phân biệt hoa thường:

```sql
CREATE UNIQUE INDEX users_email_lower_uq_idx
ON users (lower(email));
```

Các giá trị `Dat@example.com` và `dat@example.com` sẽ xung đột. Query nên dùng cùng expression để planner dùng index:

```sql
SELECT *
FROM users
WHERE lower(email) = lower(:email);
```

Expression và function trong index phải immutable: kết quả chỉ phụ thuộc argument, không phụ thuộc thời gian, session hay bảng khác.[^pg-create-index]

## 8. Partial unique index

Partial unique index thực thi uniqueness chỉ với row thỏa predicate. Đây là cách diễn đạt các quy tắc không thể viết bằng `UNIQUE` constraint thông thường.

```sql
CREATE UNIQUE INDEX addresses_one_default_uq_idx
ON addresses (user_id)
WHERE is_default;
```

Mỗi user có thể có nhiều địa chỉ, nhưng chỉ một row `is_default = true`.

Soft delete là một use case khác:

```sql
CREATE UNIQUE INDEX active_products_sku_uq_idx
ON products (sku)
WHERE deleted_at IS NULL;
```

Một SKU có thể tồn tại trong các row đã xóa mềm, nhưng chỉ được có một product active.[^pg-create-index]

## 9. Chọn cách khai báo

```text
ID chính của row
  → PRIMARY KEY

Giá trị/tổ hợp luôn duy nhất theo business rule
  → UNIQUE constraint

Uniqueness theo expression, ví dụ lower(email)
  → UNIQUE expression index

Uniqueness chỉ khi predicate đúng, ví dụ deleted_at IS NULL
  → UNIQUE partial index
```

[^pg-unique]: [PostgreSQL 18 — Unique Indexes](https://www.postgresql.org/docs/18/indexes-unique.html)
[^pg-constraints]: [PostgreSQL 18 — Constraints](https://www.postgresql.org/docs/18/ddl-constraints.html)
[^pg-create-index]: [PostgreSQL 18 — CREATE INDEX](https://www.postgresql.org/docs/18/sql-createindex.html)
