## 1. Vấn đề đặt ra

Một truy vấn có thể có nhiều cách thực thi khác nhau—quét bảng, dùng index, hoặc join theo nhiều thứ tự—với thời gian và mức tiêu thụ tài nguyên rất khác nhau. Vì vậy, trước khi chạy một truy vấn đắt tiền ặc có tác dụng thay đổi dữ liệu, cần biết optimizer *dự định* làm gì.

**Execution Plan** estimated là cách quan sát quyết định dự kiến này mà không thực thi câu SQL. Trong ghi chú này, “SQL” trong phần hướng dẫn sản phẩm được hiểu là Microsoft SQL Server.

## 2. Estimated Execution Plan là gì?

Estimated Execution Plan là kế hoạch do query optimizer tạo tại thời điểm biên dịch/lập kế hoạch. Nó mô tả:
- các operator dự kiến
- thứ tự xử lý
- số hàng ước lượng (*estimated rows/cardinality*)
- chi phí tương đối (*cost*).

SQL Server gọi đây là *compiled plan*; PostgreSQL tạo nó với `EXPLAIN` không kèm `ANALYZE`; Oracle tạo nó với `EXPLAIN PLAN`.

Plan này không thực thi truy vấn, nên không có thời gian thực tế, I/O thực tế, cảnh báo runtime hay số hàng thực tế. `Cost` cũng không phải milliseconds hoặc phần trăm CPU: đó là đơn vị nội bộ để optimizer so sánh các phương án. Chẳng hạn, PostgreSQL biểu diễn `cost=0.43..8.45` là chi phí khởi động và tổng chi phí ước lượng, còn `rows=3` là số hàng đầu ra ước lượng.[^pg-explain]

### 2.1. Estimated và Actual Plan

| Đặc điểm                          | [[Estimated Execution Plan]]                    | [[Actual Execution Plan]]                                          |
| --------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------ |
| Câu SQL có chạy không?            | Không                                           | Có                                                                 |
| Có số hàng/thời gian/I/O thực tế? | Không                                           | Có, tùy DBMS và tùy chọn                                           |
| Phù hợp nhất                      | Xem trước, phân tích an toàn, so sánh phương án | Xác minh nguyên nhân chậm và sai lệch ước lượng                    |
| Rủi ro với DML                    | Không làm đổi dữ liệu                           | Có thể thay đổi dữ liệu; cần transaction/rollback nếu chỉ kiểm tra |

Estimated plan là điểm bắt đầu tốt, nhưng không chứng minh hiệu năng thực tế. Khi truy vấn có thể chạy an toàn, cần đối chiếu `estimated rows` với `actual rows` theo từng operator. Sai lệch lớn thường làm các bước phía sau chọn join, memory grant hoặc chiến lược truy cập kém phù hợp.[^sqlserver-architecture]

## 3. Tại sao cần sử dụng?
- **Phân tích không gây tác động:** xem kế hoạch trước khi chạy truy vấn lớn.
- **Hiểu lựa chọn của optimizer:** xác định index/table scan, kiểu join (`Nested Loop`, `Hash`, `Merge`), sort, aggregate, filter và thứ tự join dự kiến.
- **Phát hiện nguyên nhân tiềm ẩn:** optimizer dùng cardinality và cost model để chọn phương án. Statistics/histogram cũ, dữ liệu phân bố lệch, predicate phức tạp, tương quan giữa cột hoặc bind parameter có thể làm ước lượng sai và khiến plan kém hiệu quả.[^sqlserver-ce]
- **Đánh giá thay đổi có cơ sở:** so sánh plan trước/sau khi cập nhật statistics, thêm/sửa index hoặc viết lại truy vấn, thay vì thay đổi theo cảm tính.

Không nên mặc định `Table Scan` là lỗi: với bảng nhỏ hoặc khi cần trả về phần lớn dữ liệu, scan có thể rẻ hơn index seek cộng với nhiều lần truy cập dữ liệu.

## 4. Cách sử dụng trên SQL Server

### 4.1. Trong SQL Server Management Studio

Mở câu truy vấn trong SSMS, chọn **Query → Display Estimated Execution Plan** hoặc nhấn `Ctrl+L`. SSMS hiển thị graphical plan trong tab **Execution Plan** mà không chạy query. Người dùng cần quyền chạy câu T-SQL tương ứng và quyền `SHOWPLAN` trên mọi database được tham chiếu.[^sqlserver-display]

### 4.2. Bằng T-SQL

```sql
SET SHOWPLAN_XML ON;
GO

SELECT o.OrderID, c.CustomerName
FROM dbo.Orders AS o
JOIN dbo.Customers AS c ON c.CustomerID = o.CustomerID
WHERE o.OrderDate >= '2026-01-01';
GO

SET SHOWPLAN_XML OFF;
GO
```

Khi `SHOWPLAN_XML` bật, SQL Server trả về XML execution plan thay vì thực thi các câu lệnh. Có thể dùng `SET SHOWPLAN_ALL ON` để nhận output dạng bảng.[^sqlserver-showplan]

Để kiểm chứng sau đó, bật **Include Actual Execution Plan** (`Ctrl+M`), chạy query trong môi trường an toàn, rồi so sánh `Estimated Number of Rows` và `Actual Number of Rows` ở các operator.

## 5. Cách sử dụng trên PostgreSQL

### 5.1. Lấy estimated plan

```sql
EXPLAIN
SELECT o.order_id, c.customer_name
FROM orders AS o
JOIN customers AS c ON c.customer_id = o.customer_id
WHERE o.order_date >= DATE '2026-01-01';
```

Để nhận kết quả thuận tiện cho công cụ xử lý, có thể yêu cầu JSON:

```sql
EXPLAIN (FORMAT JSON, VERBOSE)
SELECT *
FROM orders
WHERE customer_id = 42;
```

`EXPLAIN` không có `ANALYZE` chỉ lập kế hoạch; không chạy câu SQL. Trong output, kiểm tra loại node như `Seq Scan`, `Index Scan`, `Hash Join`, cùng `cost`, `rows` và `width`.[^pg-using]

### 5.2. Đối chiếu actual plan một cách an toàn

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 42;
```

`ANALYZE` khiến truy vấn thực sự chạy và thêm runtime statistics. Nếu đánh giá DML, bọc nó trong transaction rồi `ROLLBACK`:

```sql
BEGIN;
EXPLAIN ANALYZE
UPDATE orders
SET status = 'archived'
WHERE order_date < DATE '2020-01-01';
ROLLBACK;
```

PostgreSQL cần statistics để planner ước lượng; nên chạy `ANALYZE` khi statistics đã cũ hoặc dữ liệu vừa thay đổi đáng kể.[^pg-explain]

## 6. Cách sử dụng trên Oracle Database

### 6.1. Lấy estimated plan

```sql
EXPLAIN PLAN SET STATEMENT_ID = 'orders_plan' FOR
SELECT o.order_id, c.customer_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.order_date >= DATE '2026-01-01';

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, 'orders_plan', 'TYPICAL'));
```

`EXPLAIN PLAN` lưu kết quả vào `PLAN_TABLE`; `DBMS_XPLAN.DISPLAY` định dạng và hiển thị nó. Kiểm tra các cột `OPERATION`, `OPTIONS`, `OBJECT_NAME`, `ROWS`, `BYTES`, `COST` và phần `PREDICATE INFORMATION`. Nếu schema chưa có `PLAN_TABLE`, cần tạo theo script Oracle cung cấp.[^oracle-explain]

### 6.2. Xem statistics của plan đã chạy

Sau khi thực thi SQL, xem plan của cursor gần nhất bằng:

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

`ALLSTATS LAST` yêu cầu runtime statistics khả dụng và cho phép xem thống kê I/O, memory và timing của lần chạy cuối.[^oracle-xplan]

Oracle cảnh báo rằng `EXPLAIN PLAN` có thể khác plan thực tế do môi trường optimizer hoặc bind variable; với bind variables, output có thể không đại diện cho actual execution plan.[^oracle-tuning]

## 7. Quy trình phân tích đề xuất

1. Lấy estimated plan để xem optimizer dự kiến làm gì.
2. Tìm các estimate bất thường: `rows` quá cao/thấp, scan hoặc sort/hash lớn, join không phù hợp với kích thước đầu vào.
3. Kiểm tra statistics, độ chọn lọc của predicate, index và kiểu dữ liệu trước khi sửa query.
4. Trên môi trường an toàn, lấy actual plan và tìm operator đầu tiên có chênh lệch lớn giữa estimate và thực tế.
5. Thay đổi một yếu tố mỗi lần, đo lại plan lẫn thời gian/tài nguyên thực tế.

## 8. Nguồn tham khảo

[^sqlserver-display]: Microsoft Learn, [Display the Estimated Execution Plan](https://learn.microsoft.com/en-us/sql/relational-databases/performance/display-the-estimated-execution-plan?view=sql-server-2016).
[^sqlserver-architecture]: Microsoft Learn, [Query Processing Architecture Guide](https://learn.microsoft.com/en-us/sql/relational-databases/query-processing-architecture-guide?view=sql-server-ver17).
[^sqlserver-ce]: Microsoft Learn, [Cardinality Estimation (SQL Server)](https://learn.microsoft.com/en-us/sql/relational-databases/performance/cardinality-estimation-sql-server?view=sql-server-ver17).
[^sqlserver-showplan]: Microsoft Learn, [SET SHOWPLAN_ALL](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-showplan-all-transact-sql?view=sql-server-ver17).
[^pg-explain]: PostgreSQL Documentation, [EXPLAIN](https://www.postgresql.org/docs/17/sql-explain.html).
[^pg-using]: PostgreSQL Documentation, [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html).
[^oracle-explain]: Oracle Database SQL Language Reference, [EXPLAIN PLAN](https://docs.oracle.com/en/database/oracle/oracle-database/18/sqlrf/EXPLAIN-PLAN.html).
[^oracle-xplan]: Oracle Database PL/SQL Packages and Types Reference, [DBMS_XPLAN](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/arpls/DBMS_XPLAN.html).
[^oracle-tuning]: Oracle Database SQL Tuning Guide, [Using EXPLAIN PLAN](https://docs.oracle.com/cd/E24693_01/server.11203/e16638/ex_plan.htm).
[^redgate]: Redgate Simple Talk, [Execution Plans and Data Protection](https://www.red-gate.com/simple-talk/devops/data-privacy-and-protection/execution-plans-data-protection/).
[^brent]: Brent Ozar Unlimited, [Comparing Estimated and Actual Execution Plans in SQL Server](https://www.brentozar.com/archive/2014/07/comparing-estimated-actual-execution-plans-sql-server/).
