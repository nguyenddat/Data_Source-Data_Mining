## 1. Vấn đề đặt ra

Estimated plan cho biết optimizer dự định làm gì, nhưng không cho biết query *đã* làm gì khi gặp dữ liệu, tham số, bộ nhớ và trạng thái cache thực tế. Khi truy vấn chậm, khác biệt giữa dự đoán và runtime mới là bằng chứng để khoanh vùng nguyên nhân.

**Actual Execution Plan**  là plan được thu thập sau khi câu SQL chạy. Các khái niệm chung về execution plan, operator, `cost`, cardinality và cách đọc scan/join đã được trình bày tại [[Estimated Execution Plan]]. Ghi chú này chỉ tập trung vào thông tin runtime, cách lấy và các giới hạn riêng của actual plan.

## 2. Actual Execution Plan là gì?

Actual Execution Plan là plan mà DBMS thực sự dùng trong lần chạy vừa quan sát, kèm số liệu đo được. Những số liệu quan trọng nhất là: truy vấn thực tế trả bao nhiêu dòng, mỗi bước chạy bao nhiêu lần, mất bao lâu và có phải đọc/ghi nhiều dữ liệu hay không.

Điểm cốt lõi không phải là so `cost` với thời gian. Hãy so sánh **số hàng ước lượng** và **số hàng thực tế** tại từng operator. Sai lệch cardinality ở một node đầu vào sẽ lan truyền: DBMS có thể chọn sai thứ tự/loại join, cấp phát memory không phù hợp hoặc chọn chiến lược truy cập không hiệu quả.[^postgres-using]

| Khía cạnh | Actual plan cho biết thêm so với estimated plan |
|---|---|
| Cardinality | Số hàng thực tế trả ra ở operator (`Actual Rows`, `A-Rows`, `rows`) |
| Khối lượng lặp | Số lần operator chạy (`loops`, `Starts`) |
| Thời gian | Elapsed time toàn query và/hoặc từng operator |
| Tài nguyên | Buffer/I/O, memory, spill, cảnh báo; tên trường tùy DBMS |
| Tính đại diện | Chỉ phản ánh lần chạy với tham số, dữ liệu, cache, tải hệ thống và cấu hình hiện tại |

### 2.1. Cách hiểu tối thiểu

Khi mới đọc actual plan, chỉ cần trả lời lần lượt ba câu hỏi:

1. **Bước nào trả ra số dòng thực tế khác xa dự đoán?** Đây thường là manh mối đầu tiên.
2. **Bước nào tốn nhiều thời gian hoặc đọc nhiều dữ liệu?** Đây là nơi đang chậm.
3. **Vì sao DBMS dự đoán sai ở bước đó?** Thường liên quan đến statistics, điều kiện lọc hoặc tham số.

Các từ ngắn gặp thường xuyên:

| Thuật ngữ | Hiểu đơn giản |
|---|---|
| Operator / node | Một bước nhỏ trong plan, ví dụ đọc bảng, lọc, join hoặc sắp xếp |
| Actual rows | Số dòng thật đi ra từ bước đó |
| Estimated rows | Số dòng DBMS đã đoán trước khi chạy |
| Loops / Starts | Bước đó bị lặp bao nhiêu lần |
| Buffer / I/O | Lượng dữ liệu DBMS lấy từ cache hoặc phải đọc/ghi |
| Spill | Dữ liệu tạm không vừa bộ nhớ, phải dùng vùng lưu trữ trên disk |

## 3. Khi nào cần dùng?

- Truy vấn đã chậm hoặc hồi quy hiệu năng và cần bằng chứng runtime thay vì giả thuyết từ estimated plan.
- So sánh một plan tốt và một plan xấu khi cùng câu SQL có hiệu năng thất thường.
- Xác nhận một thay đổi index, statistics hoặc cách viết query có cải thiện thật sự.
- Tìm operator đầu tiên có `actual rows` lệch mạnh so với estimate; đây thường là vị trí điều tra hiệu quả hơn node gốc của plan.

Actual plan không phải công cụ đo benchmark duy nhất. Nó có thể thêm overhead thu thập thống kê, còn kết quả phụ thuộc warm/cold cache, tham số và concurrency lúc thử nghiệm. Vì vậy, không suy ra hiệu năng production chỉ từ một lần chạy trên dữ liệu nhỏ hoặc môi trường khác biệt.[^postgres-caveats]

## 4. Cách đọc và chẩn đoán

### 4.1. Tìm sai lệch số dòng trước

Đọc plan từ các bước đọc dữ liệu ở dưới lên kết quả ở trên, rồi so sánh `Estimated Rows` với `Actual Rows`. Với PostgreSQL, `actual rows` và `actual time` là **trung bình của mỗi lần chạy**; cần nhân với `loops` để biết tổng phần việc của bước đó.[^postgres-using]

| Dấu hiệu nhìn thấy | Nên kiểm tra gì tiếp theo |
|---|---|
| Actual Rows lớn hơn Estimated Rows rất nhiều | Statistics có cũ không; dữ liệu có tập trung vào một vài giá trị; điều kiện lọc có phức tạp không |
| Actual Rows thấp hơn Estimated Rows rất nhiều | DBMS đã nghĩ điều kiện lọc kém hiệu quả hơn thực tế; kiểm tra statistics và parameter |
| Sort hoặc Hash phải dùng disk tạm | Bộ nhớ cho bước này có thể thiếu, đầu vào lớn hơn dự đoán, hoặc index chưa hỗ trợ thứ tự cần thiết |
| Một bước có nhiều buffer reads/I/O | Bước này đọc nhiều dữ liệu; kiểm tra điều kiện lọc, index và cache |
| `loops`/`Starts` rất cao trong Nested Loop | Bước bên trong bị gọi lặp quá nhiều; kiểm tra số dòng đầu vào và index cho điều kiện join |

Đừng thêm index chỉ vì thấy `Table Scan`. Hãy tìm bước đầu tiên có số dòng dự đoán sai, rồi mới xác định nguyên nhân ở dữ liệu hoặc câu query.

### 4.2. Quy trình an toàn

1. Chụp estimated plan để ghi nhận dự đoán ban đầu.
2. Chạy actual plan chỉ trên môi trường/khối lượng dữ liệu được phép; chuẩn bị transaction + `ROLLBACK` cho DML khi DBMS hỗ trợ.
3. Ghi lại parameter, thời điểm chạy, cache state và các thiết lập liên quan để kết quả có thể tái lập.
4. So sánh số dòng thực tế/dự đoán ở từng bước, từ dưới lên trên.
5. Sửa đúng nguyên nhân—ví dụ statistics, index, điều kiện lọc hoặc kiểu dữ liệu—rồi chạy lại trong cùng điều kiện.

## 5. SQL Server

Trong SSMS, chọn **Query → Include Actual Execution Plan** hoặc nhấn `Ctrl+M`, rồi thực thi query. SQL Server tạo actual plan sau khi batch hoàn tất; plan có runtime metrics và runtime warnings nếu có. Tính năng này yêu cầu quyền chạy query và quyền `SHOWPLAN` trên các database được tham chiếu.[^sqlserver-actual]

```sql
SET STATISTICS XML ON;
GO

SELECT o.OrderID, c.CustomerName
FROM dbo.Orders AS o
JOIN dbo.Customers AS c ON c.CustomerID = o.CustomerID
WHERE o.OrderDate >= '2026-01-01';
GO

SET STATISTICS XML OFF;
GO
```

`SET STATISTICS XML ON` trả XML plan sau khi query thực thi; SSMS có thể mở nó dưới dạng graphical plan.[^sqlserver-actual] Với từng operator, ưu tiên so `Actual Number of Rows` với `Estimated Number of Rows`, sau đó kiểm tra warnings, số lần thực thi, CPU/elapsed time khi phiên bản và loại plan cung cấp chúng.

Vì query phải chạy xong mới có actual plan, không dùng cách này để “xem trước” một lệnh rủi ro; xem [[Estimated Execution Plan]] trong trường hợp đó.

## 6. PostgreSQL

### 6.1. Thu thập runtime plan

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS, WAL, FORMAT JSON)
SELECT o.order_id, c.customer_name
FROM orders AS o
JOIN customers AS c ON c.customer_id = o.customer_id
WHERE o.order_date >= DATE '2026-01-01';
```

- `ANALYZE` thực thi câu SQL và thêm actual time, actual rows, `loops`.
- `BUFFERS` hiển thị số buffer `hit`, `read`, `dirtied`, `written`; đây là số lần truy cập buffer, không phải số page riêng biệt.[^postgres-using]
- `SETTINGS` giúp ghi lại các tham số cấu hình ảnh hưởng đến plan; `WAL` hữu ích cho lệnh ghi; `FORMAT JSON` thuận tiện để lưu và so sánh bằng công cụ.

### 6.2. Giới hạn quan trọng

`EXPLAIN ANALYZE` thực thi cả DML; kết quả trả về của `SELECT` bị bỏ đi nhưng tác dụng phụ vẫn xảy ra. Khi chỉ phân tích DML, dùng transaction:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS)
UPDATE orders
SET status = 'archived'
WHERE order_date < DATE '2020-01-01';
ROLLBACK;
```

Runtime của `EXPLAIN ANALYZE` có thể dài hơn chạy bình thường do overhead profiling. Ngoài ra, do không gửi rows về client, nó không bao gồm chi phí truyền mạng và chuyển đổi output; vì thế không dùng `Execution Time` của nó như end-to-end latency.[^postgres-caveats]

## 7. Oracle Database

### 7.1. Thu thập statistics cho lần chạy

Thêm hint `GATHER_PLAN_STATISTICS`, chạy query đến khi hoàn tất, rồi hiển thị cursor plan:

```sql
SELECT /*+ GATHER_PLAN_STATISTICS */
       o.order_id, c.customer_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.order_date >= DATE '2026-01-01';

SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

Hoặc bật thu thập cho session trước khi chạy SQL:

```sql
ALTER SESSION SET statistics_level = ALL;
```

`DISPLAY_CURSOR` đọc plan của cursor trong cursor cache. Với `ALLSTATS LAST`, output có thể hiện `E-Rows` (ước lượng), `A-Rows` (thực tế), `Starts`, `A-Time`, `Buffers`, `Reads`; `LAST` tránh cộng dồn mọi lần thực thi của cursor.[^oracle-blog]

### 7.2. Lưu ý vận hành

- `DISPLAY_CURSOR(NULL, NULL, ...)` mặc định lấy statement cuối trong session; nếu có lệnh khác xen vào, truyền rõ `sql_id` và `child_number`.
- Cursor phải còn trong cache. Để lấy lại plan cũ từ Automatic Workload Repository, có thể dùng `DISPLAY_AWR` khi môi trường và quyền/licensing cho phép.[^oracle-blog]
- `ALLSTATS` cần runtime statistics được thu thập bằng hint hoặc `statistics_level = ALL`; người gọi cũng cần quyền đọc các fixed view liên quan như `V$SQL`, `V$SQL_PLAN` và `V$SQL_PLAN_STATISTICS_ALL`.[^oracle-xplan]

## 8. Nguồn tham khảo

[^sqlserver-actual]: Microsoft Learn, [Display an Actual Execution Plan](https://learn.microsoft.com/en-us/sql/relational-databases/performance/display-an-actual-execution-plan?view=sql-server-ver17).
[^postgres-using]: PostgreSQL Documentation, [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html).
[^postgres-caveats]: PostgreSQL Documentation, [EXPLAIN](https://www.postgresql.org/docs/16/sql-explain.html).
[^oracle-xplan]: Oracle Database PL/SQL Packages and Types Reference, [DBMS_XPLAN](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/arpls/DBMS_XPLAN.html).
[^oracle-blog]: Oracle SQL Blog, [How to Create an Execution Plan](https://blogs.oracle.com/sql/how-to-create-an-execution-plan).
[^pganalyze]: pganalyze, [EXPLAIN plan comparison](https://pganalyze.com/docs/explain/plan-comparison).
[^brent]: Brent Ozar Unlimited, [Comparing Estimated and Actual Execution Plans in SQL Server](https://www.brentozar.com/archive/2014/07/comparing-estimated-actual-execution-plans-sql-server/).
