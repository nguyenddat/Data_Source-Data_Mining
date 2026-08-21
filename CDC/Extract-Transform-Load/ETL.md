## Vấn đề đặt ra

Dữ liệu phục vụ vận hành thường nằm rải rác trong cơ sở dữ liệu giao dịch, API, tệp, log và thiết bị. Chúng khác nhau về lược đồ, đơn vị đo, múi giờ, mã định danh và mức độ tin cậy. Nếu sao chép trực tiếp sang hệ thống báo cáo, số liệu có thể bị trùng, thiếu, sai nghĩa hoặc thay đổi giữa các lần chạy. Đọc dữ liệu lớn trực tiếp từ hệ thống giao dịch còn có thể ảnh hưởng đến ứng dụng đang phục vụ người dùng.

ETL tạo một luồng có kiểm soát: 
- Lấy đúng phạm vi dữ liệu
- Áp quy tắc nghiệp vụ và chất lượng
- Nạp kết quả vào nơi phân tích. 

=> Không chỉ là di chuyển dữ liệu mà còn làm cho dữ liệu có thể truy vết, chạy lại và tin cậy cho báo cáo hay mô hình phân tích.

## ETL là gì?

**ETL** là viết tắt của **Extract – Transform – Load**:

Nguồn dữ liệu -> Extract -> vùng tạm/raw -> Transform -> Load -> kho phân tích/ứng dụng đích

- **Extract:** đọc dữ liệu hoặc phần dữ liệu đã thay đổi từ nguồn.
- **Transform:** làm sạch, chuẩn hóa, kiểm tra, kết hợp và tính toán theo mục đích sử dụng.
- **Load:** ghi dữ liệu đã được kiểm soát vào bảng, tệp, kho dữ liệu hoặc hệ thống đích.

Đây là ba trách nhiệm logic, không nhất thiết là ba chương trình hay ba máy riêng biệt. Pipeline có thể chạy theo lịch, theo sự kiện hoặc liên tục; vùng raw/staging thường được dùng để tách dữ liệu nguồn khỏi dữ liệu đã công bố.

## Extract: lấy đúng dữ liệu và giữ bối cảnh

Nguồn thường gặp gồm cơ sở dữ liệu quan hệ, SaaS/API, CSV/Excel, log, message broker và thiết bị. Trước khi lấy dữ liệu phải xác định đơn vị dữ liệu, khóa định danh, thời điểm có hiệu lực, lược đồ và quyền đọc.

### Snapshot, incremental và [CDC](../Change%20Data%20Capture.md)

| Tiêu chí                   | Snapshot / full load                                                        | Incremental load                                                                        | CDC (Change Data Capture)                                                                                                    |
| -------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Dữ liệu lấy                | Toàn bộ tập dữ liệu trong phạm vi đã chọn                                   | Bản ghi mới hoặc đã đổi từ lần chạy thành công trước                                    | Các sự kiện thay đổi `INSERT` / `UPDATE` / `DELETE`                                                                          |
| Cách xác định phần cần lấy | Quét hoặc xuất toàn bộ bảng/tệp                                             | Lọc bằng `updated_at`, khóa tăng dần, phân vùng, `LastModifiedDate` hoặc watermark      | Đọc change stream của nguồn, như MySQL binlog hoặc PostgreSQL logical replication                                            |
| Xử lý xóa                  | Chỉ thấy khi so sánh snapshot mới với trạng thái đích                       | Thường không thấy, trừ khi có cột soft-delete hoặc cơ chế theo dõi riêng                | Có thể phát hiện sự kiện xóa nếu nguồn và connector hỗ trợ                                                                   |
| Độ trễ điển hình           | Theo lịch chạy; cao với dữ liệu lớn                                         | Theo chu kỳ polling/batch                                                               | Thấp hơn, nhưng không mặc định là real-time                                                                                  |
| Tải lên nguồn              | Cao vì đọc lại toàn bộ                                                      | Thấp đến trung bình; phụ thuộc chỉ mục, điều kiện lọc và số tệp phải quét               | Thường thấp trên bảng ứng dụng, nhưng cần đọc log và cấu hình quyền/retention phù hợp                                        |
| Trạng thái cần lưu         | Mã lần chạy và phiên bản/phạm vi snapshot                                   | Watermark hoặc bảng điều khiển của lần chạy thành công                                  | Offset/LSN/SCN hoặc checkpoint của change stream                                                                             |
| Khởi tạo dữ liệu đích      | Phù hợp nhất để nạp lần đầu hoặc làm mới hoàn toàn                          | Cần có mốc khởi đầu hoặc một snapshot trước đó                                          | Thường bắt đầu bằng snapshot/full load rồi tiếp tục CDC; cũng có thể dùng CDC-only khi đích đã đồng bộ tại đúng điểm bắt đầu |
| Rủi ro chính               | Tốn tài nguyên; dữ liệu có thể không nhất quán nếu nguồn đổi trong lúc quét | Bỏ sót hoặc đọc lặp vì bản ghi đến muộn, cùng timestamp hay `updated_at` không đáng tin | Mất log/offset, sai điểm bắt đầu, thay đổi lược đồ, thứ tự sự kiện và bản ghi bị áp dụng lặp                                 |
| Khi nên dùng               | Tập nhỏ, nạp ban đầu, backfill hoặc khi nguồn không có dấu vết thay đổi     | Báo cáo theo giờ/ngày và nguồn có watermark đáng tin                                    | Đồng bộ gần thời gian thực, cần bắt xóa hoặc cần chuỗi thay đổi đầy đủ                                                       |

Incremental load không đồng nghĩa với CDC: 
- Incremental load suy ra thay đổi từ thuộc tính dữ liệu hoặc phạm vi đã xử lý.
- CDC thường đọc transaction log qua API riêng của từng database; Debezium, chẳng hạn, đọc MySQL binlog và PostgreSQL logical replication stream. [Debezium Architecture](https://debezium.io/documentation/reference/stable/architecture.html) [AWS DMS CDC](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Task.CDC.html).

Mẫu watermark lấy dữ liệu giữa watermark cũ và mới; với tệp, việc lọc theo `LastModifiedDate` vẫn có thể chậm nếu phải quét rất nhiều tệp. [Azure Data Factory](https://learn.microsoft.com/en-us/azure/data-factory/tutorial-incremental-copy-overview):
- Chỉ đáng tin khi quy tắc cập nhật và múi giờ được xác định rõ. Bản ghi đến muộn hoặc có cùng thời điểm có thể bị bỏ sót nếu chỉ lấy giá trị lớn hơn watermark. 
- Cách thận trọng là lưu trạng thái lần chạy, dùng khoảng thời gian chồng lấn rồi khử trùng lặp theo khóa và phiên bản/sự kiện mới nhất. 

Với CDC cần giữ log đủ lâu, xử lý xóa, thay đổi lược đồ, thứ tự sự kiện và giai đoạn snapshot ban đầu.

## Transform: biến dữ liệu thành dữ liệu có nghĩa

Transform chuyển dữ liệu từ dạng của hệ thống phát sinh sang mô hình dùng ở đích. Các thao tác thường có:

- Standardization: chuẩn hóa định dạng, đơn vị, mã quốc gia, múi giờ và tên cột;
- Constraints: kiểm tra khóa bắt buộc, miền giá trị, tính duy nhất, quan hệ tham chiếu và độ mới;
- Duplicate: khử trùng lặp; đưa bản ghi thiếu hoặc sai vào vùng **quarantine** thay vì im lặng bỏ qua;
- ghép dữ liệu tham chiếu, ánh xạ mã nguồn sang mã chuẩn, làm giàu thuộc tính;
- tổng hợp theo thời gian/đơn vị kinh doanh và áp dụng quy tắc nghiệp vụ có phiên bản.

Quy tắc chất lượng cần đo được, có ngưỡng và hành động khi thất bại: chặn công bố, cảnh báo hoặc cô lập bản ghi. AWS Glue Data Quality minh họa việc đánh giá quy tắc trên dữ liệu đang nạp và đã lưu, đồng thời xác định bản ghi làm giảm điểm chất lượng. [AWS Glue Data Quality](https://docs.aws.amazon.com/glue/latest/dg/glue-data-quality.html) Đây là tính năng của một sản phẩm; nguyên lý tổng quát là pipeline phải công bố được quy tắc và kết quả kiểm tra.

### Lược đồ và dữ liệu muộn

Phần này nói về hai tình huống dễ làm ETL cho ra số liệu sai: **cấu trúc dữ liệu nguồn thay đổi** và **dữ liệu đến sau thời điểm pipeline đã xử lý kỳ liên quan**.

**Schema evolution** là thay đổi tên cột, kiểu dữ liệu, thêm/bớt cột hoặc đổi ý nghĩa của trường. Ví dụ, bảng `orders` ban đầu có cột `amount` là số nguyên tính bằng VND. Một ngày, hệ thống nguồn đổi tên thành `total_amount`, cho phép số thập phân, hoặc chuyển ý nghĩa sang USD. Nếu pipeline vẫn dùng quy tắc cũ, nó có thể lỗi hoặc tính doanh thu sai.

Vì vậy, pipeline cần phát hiện thay đổi trước khi công bố dữ liệu. Thay đổi tương thích ngược, như thêm một cột chưa dùng, thường có thể chấp nhận. Thay đổi phá vỡ hợp đồng dữ liệu, như đổi kiểu hoặc đơn vị tiền tệ, nên bị chặn hoặc đưa vào vùng **quarantine** để kiểm tra và cập nhật quy tắc transform.

**Dữ liệu đến muộn** là sự kiện xảy ra trong quá khứ nhưng chỉ được pipeline nhận sau đó. Chẳng hạn, đơn hàng phát sinh lúc 10:55 nhưng do mạng chậm đến 11:10 mới được nạp. Báo cáo doanh thu 10:00–11:00 có thể đã được tính và công bố khi đơn hàng này đến.

Pipeline phải chọn trước một quy tắc xử lý:

- **Tính lại kỳ cũ:** cập nhật lại tổng doanh thu 10:00–11:00. Cách này chính xác nhất nhưng tốn tài nguyên và số liệu đã xem có thể thay đổi.
- **Chờ một khoảng trước khi chốt:** chỉ công bố doanh thu của giờ 10 sau, ví dụ, 15 phút. Cách này giảm khả năng phải tính lại nhưng tăng độ trễ.
- **Gắn cờ dữ liệu muộn:** vẫn lưu đơn hàng, nhưng điều chỉnh ở lần công bố sau hoặc hiển thị rõ rằng số liệu kỳ cũ đã thay đổi.

Dashboard vận hành thường ưu tiên dữ liệu nhanh nên có thể chấp nhận điều chỉnh sau. Báo cáo tài chính thường ưu tiên chính xác nên hay tính lại kỳ cũ hoặc chốt số liệu theo quy trình kiểm soát riêng.

## Load: công bố dữ liệu an toàn

Đích có thể là data warehouse, data lake, cơ sở dữ liệu phục vụ ứng dụng, công cụ tìm kiếm hoặc topic/tệp cho bước sau. Mẫu nạp phổ biến gồm:

- **append** cho sự kiện bất biến;
- **upsert/merge** theo khóa nghiệp vụ cho trạng thái hiện tại;
- **overwrite partition** cho tập được tính lại theo ngày/tháng;
- nạp vào **staging** rồi xác thực và hoán đổi/công bố theo cách nguyên tử.

Thiết kế load phải trả lời: chạy lại có tạo trùng không, dữ liệu mới có lộ khi phần liên quan chưa hoàn tất không, và khóa nào xác định bản ghi đích? Airflow khuyến nghị xem task như giao dịch: không tạo đầu ra dở dang, phải cho cùng kết quả khi chạy lại, dùng UPSERT thay INSERT dễ gây trùng và đọc/ghi phân vùng xác định. [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)

## Mô hình vận hành hiện nay

| Mô hình            | Cách chạy                        | Phù hợp                                     | Đánh đổi chính                                      |
| ------------------ | -------------------------------- | ------------------------------------------- | --------------------------------------------------- |
| Batch              | Theo giờ/ngày hoặc khi có tệp    | Báo cáo định kỳ, nguồn lớn                  | Dữ liệu có độ trễ theo lịch chạy                    |
| Micro-batch        | Gom sự kiện thành lô nhỏ lặp lại | Dashboard gần thời gian thực, có trạng thái | Cần checkpoint, xử lý dữ liệu muộn và backlog       |
| Streaming liên tục | Xử lý dòng sự kiện đang đến      | Độ trễ rất thấp, phản ứng sự kiện           | Vận hành phức tạp; bảo đảm phụ thuộc engine và sink |

Không nên gọi mọi pipeline có Kafka hay CDC là “real-time”. Độ trễ thực tế gồm thời gian lấy, hàng đợi, transform, ghi đích và thời gian dữ liệu được người dùng thấy. 

Spark Structured Streaming mặc định xử lý theo **micro-batch**; tài liệu Spark phân biệt điều này với Continuous Processing: micro-batch dùng checkpoint/write-ahead log cho bảo đảm exactly-once của engine, còn continuous mode ưu tiên độ trễ thấp và có bảo đảm at-least-once. [Spark Structured Streaming](https://spark.apache.org/docs/latest/streaming/index.html) Bảo đảm đầu-cuối vẫn phải được kiểm tra với nguồn, sink và thao tác nạp cụ thể.

## Điều phối, độ tin cậy và khả năng quan sát

ETL cần cơ chế điều phối để biểu diễn phụ thuộc, lịch chạy, trạng thái và retry. Apache Airflow mô hình hóa workflow thành **DAG**, trong đó task và quan hệ phụ thuộc xác định thứ tự thực hiện; nền tảng điều phối công việc nhưng không tự biến thao tác dữ liệu thành an toàn. [Airflow Architecture](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html)
Các kiểm soát nên có:

- **Idempotency và retry:** dùng khóa/lần chạy/phân vùng cố định; retry chỉ an toàn khi load không nhân đôi kết quả.
- **State và checkpoint:** lưu watermark, offset, phiên bản lược đồ, phạm vi đầu vào và trạng thái thành công. AWS Glue gọi trạng thái này là job bookmark, dùng để tránh xử lý lại dữ liệu cũ trong các nguồn được hỗ trợ. [AWS Glue job bookmarks](https://docs.aws.amazon.com/glue/latest/dg/monitor-continuations.html)
- **Backfill:** chạy lại một khoảng ngày xác định, tách khỏi luồng hiện tại và không ghi đè ngoài ý muốn.
- **Observability:** log có mã lần chạy; metric số bản ghi đọc/ghi/lỗi/độ trễ; cảnh báo khi dữ liệu trễ hoặc tỷ lệ lỗi tăng; lineage từ bảng đích về nguồn và quy tắc transform.
- **Bảo mật:** cấp quyền tối thiểu; quản lý secret ngoài mã; mã hóa khi truyền/lưu; che hoặc mã hóa dữ liệu nhạy cảm; lưu audit log cho truy cập và thay đổi quy tắc.

## Ví dụ xuyên suốt: dữ liệu bán lẻ

Một chuỗi bán lẻ cần doanh thu theo cửa hàng và sản phẩm theo giờ.

1. **Extract:** ban đầu lấy snapshot giao dịch, danh mục sản phẩm và cửa hàng; về sau dùng watermark hoặc CDC. Lưu raw event kèm mã giao dịch, thời điểm nhận và mã lần chạy.
2. **Transform:** kiểm tra mã cửa hàng/sản phẩm; chuẩn hóa tiền tệ và múi giờ; loại trùng theo mã giao dịch và phiên bản; ghép danh mục rồi tính doanh thu ròng. Bản ghi chưa có sản phẩm tham chiếu đi vào quarantine.
3. **Load:** upsert bảng giao dịch chuẩn theo khóa giao dịch và tính lại partition doanh thu giờ chịu ảnh hưởng. Chỉ công bố tổng hợp khi kiểm tra số dòng, tổng tiền và tỷ lệ quarantine đạt ngưỡng.

Nếu lần chạy thất bại sau khi đã ghi một phần, pipeline dùng cùng khóa/phân vùng và MERGE hoặc staging + publish để chạy lại mà không cộng doanh thu hai lần. Một sự kiện hủy giao dịch muộn cần quy tắc rõ ràng: điều chỉnh báo cáo giờ cũ hoặc ghi bút toán điều chỉnh riêng.

## Khi nào nên dùng ETL và các đánh đổi

ETL phù hợp khi cần chuẩn hóa và kiểm soát dữ liệu trước khi công bố cho phân tích, khi nguồn có chất lượng không đồng đều, hoặc khi cần tách tải phân tích khỏi hệ thống giao dịch. Nó đặc biệt hữu ích khi quy tắc nghiệp vụ, audit và chất lượng dữ liệu phải được quản trị tập trung.

Đổi lại, ETL tạo thêm độ trễ, hạ tầng, chi phí vận hành và trách nhiệm duy trì quy tắc/lược đồ. Pipeline càng gần thời gian thực càng cần đầu tư vào checkpoint, dữ liệu muộn, điều phối lỗi và quan sát. Mục tiêu thiết kế phải nêu rõ độ trễ, độ đầy đủ, độ chính xác và chi phí của từng sản phẩm dữ liệu.

## Điểm cần phân biệt khi đọc tài liệu

Các nguồn được đối chiếu thống nhất rằng ETL hiện đại cần trạng thái, kiểm tra và khả năng phục hồi. Tuy vậy, thuật ngữ không hoàn toàn đồng nghĩa:

- **CDC** chỉ là một cách extract thay đổi; nó không thay thế transform, kiểm tra chất lượng hay load idempotent.
- **Watermark/job bookmark/checkpoint** đều lưu tiến độ, nhưng phạm vi và điều kiện cập nhật do từng công cụ quy định.
- **Exactly-once** trong tài liệu engine thường nói về phạm vi engine hoặc sink được hỗ trợ; không phải cam kết tự động cho toàn bộ nguồn–đích.
- **Streaming** có thể là micro-batch hoặc xử lý liên tục, với độ trễ và bảo đảm khác nhau.

Vì vậy, khi triển khai cần kiểm chứng hợp đồng giao hàng, thứ tự, schema evolution, retry và thao tác ghi của đúng connector, nguồn và đích.

## Tài liệu tham khảo

1. Apache Airflow, [Architecture Overview](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html), truy cập 21/08/2026.
2. Apache Airflow, [Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html), truy cập 21/08/2026.
3. Apache Spark, [Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/streaming/index.html), truy cập 21/08/2026.
4. Debezium, [Architecture](https://debezium.io/documentation/reference/stable/architecture.html), truy cập 21/08/2026.
5. Amazon Web Services, [AWS Glue Data Quality](https://docs.aws.amazon.com/glue/latest/dg/glue-data-quality.html), truy cập 21/08/2026.
6. Amazon Web Services, [Tracking processed data using job bookmarks](https://docs.aws.amazon.com/glue/latest/dg/monitor-continuations.html), truy cập 21/08/2026.
7. Microsoft, [Incrementally load data from a source data store to a destination data store](https://learn.microsoft.com/en-us/azure/data-factory/tutorial-incremental-copy-overview), truy cập 21/08/2026.
8. Amazon Web Services, [Creating tasks for ongoing replication using AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Task.CDC.html), truy cập 21/08/2026.
