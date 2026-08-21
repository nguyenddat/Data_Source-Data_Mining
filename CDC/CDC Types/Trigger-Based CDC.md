## Vấn đề đặt ra

Bạn có một database nghiệp vụ [[OLTP - OnLine Transaction Processing]] và muốn biết "có gì vừa thay đổi" để đẩy sang [[OLAP - OnLine Analytical Processing|kho phân tích]], cache, search engine... Đây là nhu cầu [[Change Data Capture|CDC]]; câu hỏi là: làm sao biết được?

Trigger-based CDC là cách: không quét lại bảng, cũng không cần đọc log — bắt thay đổi ngay tại thời điểm nó xảy ra, bằng chính cơ chế trigger có sẵn trong database.

## Ý tưởng của trigger-based CDC

![Trigger-based CDC technique](assets/trigger-based%20cdc.png)

Gắn trigger vào bảng nguồn cho cả ba thao tác INSERT/UPDATE/DELETE. Mỗi khi có transaction ghi vào bảng nguồn, trigger tự động chạy trong cùng transaction đó, ghi lại thay đổi vào một bảng phụ — gọi là shadow table (hay change log table). Tiến trình CDC đọc shadow table, đẩy sang bảng đích.

=> Trigger-based CDC bắt thay đổi bằng cách **để database tự báo ngay khi ghi**, không quét lại bảng, không đọc log.

## Cơ chế hoạt động

Ba bước:
1. Transaction ghi INSERT/UPDATE/DELETE vào bảng nguồn
2. Trigger fire ngay trong transaction đó, ghi một dòng vào shadow table: loại thao tác, timestamp, khóa chính, giá trị cũ/mới
3. Tiến trình CDC đọc shadow table (poll định kỳ hoặc theo sự kiện), apply thay đổi sang bảng đích, rồi đánh dấu đã xử lý

Vì trigger chạy trong cùng transaction với thao tác gốc, ghi shadow table thành công đồng nghĩa transaction gốc cũng thành công — không có transaction nào lọt lưới.

```sql
CREATE TABLE change_log (
  id SERIAL PRIMARY KEY,
  table_name TEXT,
  op_type CHAR(1),
  pk_value TEXT,
  old_data JSONB,
  new_data JSONB,
  changed_at TIMESTAMP DEFAULT now()
);

CREATE OR REPLACE FUNCTION log_change() RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO change_log(table_name, op_type, pk_value, old_data, new_data)
  VALUES (
    TG_TABLE_NAME,
    LEFT(TG_OP,1),
    COALESCE(NEW.id, OLD.id)::TEXT,
    to_jsonb(OLD),
    to_jsonb(NEW)
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_cdc
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION log_change();
```

Có cả `old_data` lẫn `new_data` — giống log-based, khác query-based (chỉ thấy trạng thái hiện tại).

## Ưu điểm

| Ưu điểm | Giải thích |
|---------|------------|
| Bắt được DELETE | Trigger fire cả trên DELETE, ghi lại trước khi dòng biến mất |
| Có giá trị cũ | `old_data`/`new_data` đầy đủ, giống log-based |
| Không bỏ sót | Mỗi thao tác ghi một dòng log, đúng thứ tự, không gộp như query-based |
| Không cần quyền replication | Chỉ cần quyền tạo trigger/function, không cần đụng vào WAL/binlog |
| Chạy trên mọi DB hỗ trợ trigger | Oracle, MySQL, PostgreSQL, SQL Server... đều dùng được |
| Đồng bộ, không mất dữ liệu | Trigger chạy chung transaction, ghi log fail thì transaction gốc cũng rollback |

## Nhược điểm

- **Tăng tải write ngay trên bảng nguồn**: mỗi INSERT/UPDATE/DELETE cõng thêm một lần ghi vào shadow table, tăng latency của chính transaction gốc — khác log-based (đọc log riêng, không chặn write).
- **Xâm lấn schema**: phải tạo thêm trigger, function, shadow table cho từng bảng cần theo dõi; đổi schema bảng nguồn phải sửa trigger theo.
- **Không bắt được DDL**: chỉ bắt DML (INSERT/UPDATE/DELETE), đổi cấu trúc bảng (ALTER TABLE...) trigger không thấy.
- **Rủi ro lock/contention**: nhiều transaction cùng ghi bảng nguồn đồng thời tranh nhau ghi vào một shadow table, có thể thành điểm nghẽn.
- **Trigger logic dễ vỡ**: viết sai trigger có thể làm transaction gốc lỗi theo, hoặc bỏ sót cột khi bảng có thay đổi.
- **Gắn với engine cụ thể**: cú pháp trigger/function mỗi DB một kiểu (PL/pgSQL, T-SQL, PL/SQL...), không portable như query-based (SQL chuẩn).
