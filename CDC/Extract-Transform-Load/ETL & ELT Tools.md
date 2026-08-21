## Vấn đề đặt ra

“Công cụ [[ETL]]/[[ELT]]” thường bị hiểu như một loại sản phẩm duy nhất, nhưng một pipeline thực tế có ít nhất ba trách nhiệm: lấy dữ liệu, nạp dữ liệu, và biến đổi/công bố dữ liệu. Một công cụ có thể chỉ làm tốt một trách nhiệm. Chọn một nền tảng chỉ vì nó có nhãn ETL dễ tạo lỗ hổng: không có connector, thiếu transform có kiểm thử, hoặc không kiểm soát được chi phí vận hành.

Tài liệu khảo sát các lựa chọn managed và open source theo ba trách nhiệm: **AWS Glue** hoặc **Apache Spark** cho transform/ETL tùy biến; **Fivetran** hoặc **Airbyte OSS** cho extract-load; và **dbt Core** cho transform theo ELT trong data warehouse/lakehouse. Chúng có thể kết hợp, không phải các lựa chọn thay thế hoàn toàn.

## Phạm vi so sánh

| Công cụ | Vai trò chính | Phong cách phù hợp | Không thay thế cho |
| --- | --- | --- | --- |
| AWS Glue | Data integration và job ETL/ELT | Job serverless, Spark, AWS data lake | Một bộ connector SaaS managed chuyên sâu hoặc một lớp semantic model hoàn chỉnh |
| Fivetran | Extract + Load managed | Đồng bộ database/SaaS vào warehouse/lakehouse | Transform nghiệp vụ phức tạp và mô hình dữ liệu cuối |
| Airbyte OSS | Extract + Load tự quản | Connector mã nguồn mở, chạy trong hạ tầng của tổ chức | Transform nghiệp vụ và vận hành hạ tầng hoàn toàn tự động |
| dbt Core | Transform trong đích | SQL model, kiểm thử, lineage, CI/CD tự quản | Extract/load từ nguồn vào warehouse |
| Apache Spark | Engine transform phân tán | Xử lý tệp/dữ liệu lớn, SQL, Python/Scala, batch/stream | Connector SaaS managed, data catalog hay orchestrator hoàn chỉnh |

## AWS Glue

### Là gì?

AWS Glue là dịch vụ tích hợp dữ liệu serverless của AWS. Theo tài liệu AWS, nó hỗ trợ khám phá, chuẩn bị, di chuyển và tích hợp dữ liệu từ nhiều nguồn; có Data Catalog, giao diện tạo job và engine ETL serverless dựa trên Apache Spark. AWS cũng nói Glue hỗ trợ workload ETL, ELT và streaming. [AWS Glue overview](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html)

Trong thực tế, Glue thường làm **ETL**: đọc từ S3, database hoặc stream; xử lý bằng Spark/Python/SQL; rồi ghi Parquet, Iceberg, Redshift hay đích AWS khác. Nó cũng có thể làm ELT nếu chủ yếu nạp dữ liệu trước rồi để warehouse/lakehouse transform tiếp.

### Ưu điểm

- **Không quản lý cluster:** job serverless giảm việc cấp phát, vá và vận hành Spark cluster.
- **Phù hợp biến đổi phức tạp:** có thể dùng Spark để xử lý tệp lớn, dữ liệu bán cấu trúc, joins và logic không thuận tiện trong SQL thuần.
- **Tích hợp AWS mạnh:** Data Catalog, S3, IAM và các dịch vụ phân tích AWS cùng một hệ sinh thái; Glue Studio cung cấp giao diện trực quan để tạo, chạy và theo dõi job.
- **Một nơi cho nhiều bước kỹ thuật:** discovery, cataloging, cleansing và job execution có thể đặt trong cùng dịch vụ.

### Nhược điểm và đánh đổi

- **Gắn chặt AWS:** tích hợp là ưu điểm trong AWS, nhưng pipeline, IAM, catalog và vận hành dễ phụ thuộc vào hệ sinh thái này khi muốn chuyển nền tảng.
- **Cần năng lực Spark để tối ưu:** serverless không làm mất các vấn đề về partition, skew, schema, retry, bookmark và chi phí quét dữ liệu. Job phức tạp vẫn cần được kiểm thử và quan sát như mã sản xuất.
- **Không tự giải quyết mô hình nghiệp vụ:** crawler/catalog có thể suy ra schema kỹ thuật, nhưng không quyết định khóa nghiệp vụ, dữ liệu muộn, cách xử lý xóa hay định nghĩa doanh thu.
- **Chi phí phụ thuộc thiết kế và workload:** cần đo thời gian chạy, dung lượng quét, worker/compute và tần suất chạy; không nên chỉ so sánh giá niêm yết giữa các công cụ.

### Khi nên dùng

Chọn Glue khi dữ liệu và đích chủ yếu ở AWS, cần xử lý batch/stream tùy biến trên dữ liệu lớn, hoặc cần transform trước khi dữ liệu được công bố. Nó đặc biệt hợp cho tệp, log và dữ liệu bán cấu trúc cần Spark.

## Fivetran

### Là gì?

Fivetran là nền tảng đồng bộ dữ liệu được quản lý, thiên về **Extract + Load** cho ELT. Với database connector, tài liệu của Fivetran nêu rằng công cụ tự tạo schema/bảng ở đích, nạp initial sync, sau đó cập nhật theo batch bằng merge/upsert; lịch đồng bộ có thể từ 1 phút đến 24 giờ. Nó cũng tự phát hiện và áp dụng thay đổi schema; cách bắt thay đổi cụ thể phụ thuộc database, có thể là CDC native hoặc cơ chế riêng của Fivetran. [Fivetran database connectors](https://fivetran.com/docs/connectors/databases)

Nó thường được ghép với dbt hoặc SQL trong warehouse: Fivetran đưa dữ liệu nguồn vào lớp raw/staging, công cụ transform tạo các bảng phục vụ BI.

### Ưu điểm

- **Triển khai nhanh với nguồn phổ biến:** giảm mã connector, quản lý credential, incremental sync và nhiều công việc vận hành lặp lại.
- **Tự động hóa schema và đồng bộ incremental:** hữu ích khi cần đưa SaaS/database vào warehouse nhanh, với dữ liệu mới/sửa/xóa được đồng bộ theo khả năng connector.
- **Giảm gánh vận hành ingestion:** đội dữ liệu tập trung vào mô hình, kiểm thử và sản phẩm dữ liệu thay vì duy trì connector riêng.
- **Phân tách trách nhiệm rõ:** dữ liệu được nạp trước; transform nghiệp vụ có thể version hóa độc lập ở warehouse.

### Nhược điểm và đánh đổi

- **Không phải công cụ transform cuối cùng:** Fivetran không thay thế model nghiệp vụ, kiểm thử chất lượng hay mart cho BI; cần dbt/SQL hoặc công cụ khác sau bước load.
- **Khả năng và độ trễ phụ thuộc từng connector:** “CDC”, xử lý xóa, cột hỗ trợ, lịch sync và schema behavior phải kiểm tra ở tài liệu của nguồn cụ thể, không suy ra từ tên Fivetran.
- **Chi phí theo mức sử dụng:** Fivetran tính phí usage-based theo **Monthly Active Rows (MAR)**; insert, update và delete có thể tính vào MAR. Nguồn có nhiều bản ghi thay đổi hoặc có nhiều connection/destination cần được ước lượng trước. [Fivetran pricing](https://fivetran.com/docs/getting-started/pricing)
- **Ít chỗ cho logic extract đặc thù:** khi API nội bộ, quy tắc retry, format hoặc transform tiền nạp rất riêng, custom connector/pipeline có thể phù hợp hơn nhưng kéo theo trách nhiệm vận hành.

### Khi nên dùng

Chọn Fivetran khi cần đồng bộ nhanh các nguồn SaaS/database đã có connector vào warehouse/lakehouse, đội ngũ muốn giảm vận hành ingestion và chấp nhận mô hình chi phí theo mức thay đổi dữ liệu. Hãy thử với dữ liệu đại diện để đo MAR, latency và cách connector xử lý delete/schema change trước khi cam kết.

## Airbyte OSS

### Là gì?

Airbyte là nền tảng data replication mã nguồn mở, có bản **Self-Managed/OSS** để tổ chức tự chạy trong hạ tầng của mình. Tài liệu chính thức mô tả Airbyte có thể replicate dữ liệu từ hàng trăm nguồn vào warehouse, lake và database, đồng thời có catalog connector mã nguồn mở. [Airbyte documentation](https://docs.airbyte.com/)

Trong kiến trúc ELT, Airbyte OSS thường là lớp E+L thay cho Fivetran: tạo connection, chạy initial/incremental sync, đưa dữ liệu vào raw/staging; dbt Core hoặc SQL tiếp tục transform.

### Ưu điểm

- **Tự chủ hạ tầng và dữ liệu:** có thể triển khai trong môi trường của tổ chức, phù hợp khi yêu cầu mạng, residency hoặc chính sách không cho phép gửi dữ liệu qua dịch vụ managed bên thứ ba.
- **Connector mã nguồn mở và có thể tùy biến:** có thể đóng góp/sửa connector hoặc tạo connector riêng cho API nội bộ thay vì chờ nhà cung cấp hỗ trợ.
- **Không tính phí theo MAR của Fivetran:** chi phí chính chuyển thành compute, storage, vận hành và hỗ trợ; điều này có thể có lợi cho workload biến động lớn nhưng cần đo thực tế.

### Nhược điểm và đánh đổi

- **Tự chịu vận hành:** phải triển khai, nâng cấp, backup, giám sát, quản lý secret, scale worker và xử lý lỗi sync. Mã nguồn mở không đồng nghĩa không có chi phí vận hành.
- **Chất lượng connector không đồng đều:** cần kiểm tra riêng nguồn/đích, CDC, delete, lịch sync, schema evolution và độ ổn định của connector cần dùng.
- **Vẫn cần lớp transform:** Airbyte giải quyết replication; nó không thay thế model nghiệp vụ, data test hay mart.

### Khi nên dùng

Chọn Airbyte OSS khi cần lớp ingestion linh hoạt và tự quản, có năng lực platform/data engineering để vận hành nó, hoặc có nguồn nội bộ cần connector tùy biến. Nếu ưu tiên thời gian triển khai và giảm vận hành hơn quyền kiểm soát, đối chiếu với Fivetran bằng cùng nguồn dữ liệu là cách đánh giá thực tế nhất.

## dbt Core

### Là gì?

**dbt Core** là runtime CLI mã nguồn mở, tự quản của dbt cho phần **transform** trong ELT. Nhóm dữ liệu viết các câu `SELECT` SQL, dbt tạo model có thể bảo trì và chạy chúng trong cloud data platform. Tài liệu dbt mô tả project tạo context gồm lineage, tests, contracts, metrics và governance; dbt hoạt động cùng công cụ ingestion để transform dữ liệu trực tiếp trong nền tảng đích. [What is dbt?](https://docs.getdbt.com/docs/introduction)

Vì dbt không extract hoặc load nguồn, nó thường đứng sau Fivetran, Airbyte, CDC connector, Glue hoặc một pipeline tự viết.

### Ưu điểm

- **SQL gần dữ liệu:** nhà phân tích/kỹ sư quen SQL có thể xây transform mà không phải chuyển dữ liệu sang một runtime riêng.
- **Kỹ luật phần mềm cho dữ liệu:** model có phụ thuộc rõ, version control, test, documentation, CI/CD và lineage giúp review thay đổi dễ hơn.
- **Dễ chia lớp raw–staging–mart:** mỗi model là một đơn vị có thể tái sử dụng, kiểm tra và chạy lại; phù hợp quản trị nhiều sản phẩm dữ liệu.
- **Idempotent là mục tiêu thiết kế:** dbt khuyến khích transform có thể chạy lại để cho kết quả nhất quán, hữu ích khi retry và backfill. [dbt introduction](https://docs.getdbt.com/docs/introduction)

### Nhược điểm và đánh đổi

- **Không có extract/load:** phải có ingestion layer và cơ chế lưu trạng thái/offset ở trước dbt.
- **Phụ thuộc compute và SQL dialect của đích:** hiệu năng, `MERGE`, incremental model, transaction và chi phí do warehouse/lakehouse quyết định. SQL hợp lệ không đồng nghĩa truy vấn rẻ hoặc chạy nhanh.
- **Không thay thế xử lý stream/event phức tạp:** transform theo batch/incremental ở warehouse có thể không đáp ứng độ trễ rất thấp, xử lý stateful hoặc logic event-time khó; cần engine/stream processor phù hợp ở bước khác.
- **Cần quy ước mô hình tốt:** nếu không quản lý grain, khóa, freshness, test và ownership, số lượng model tăng sẽ tạo DAG khó hiểu thay vì dữ liệu đáng tin.

### Khi nên dùng

Chọn dbt Core khi dữ liệu đã được nạp vào warehouse/lakehouse, đa số transform biểu diễn tốt bằng SQL, và cần review, kiểm thử, lineage và tài liệu hóa model trong quy trình tự quản. Nó đặc biệt phù hợp làm lớp T sau Airbyte OSS, Fivetran hoặc bất kỳ ingestion pipeline nào.

## Apache Spark

### Là gì?

Apache Spark là engine mã nguồn mở cho xử lý dữ liệu phân tán. Spark SQL là module xử lý dữ liệu có cấu trúc, hỗ trợ SQL và Dataset/DataFrame API, đồng thời tối ưu thực thi dựa trên cấu trúc của dữ liệu và phép tính. [Spark SQL and DataFrames](https://spark.apache.org/docs/latest/sql-programming-guide)

Spark không phải một bộ ETL hoàn chỉnh: nó là runtime để viết transform bằng SQL, PySpark, Scala hoặc Java. AWS Glue dùng Spark ở phía dưới; dùng Spark trực tiếp cho phép triển khai trên hạ tầng tự chọn, nhưng đội ngũ phải tự lấp các phần platform mà Glue quản lý.

### Ưu điểm

- **Xử lý phân tán và linh hoạt:** phù hợp tệp rất lớn, dữ liệu bán cấu trúc, transform phức tạp, SQL và code trong cùng engine.
- **Không khóa vào một cloud cụ thể:** có thể chạy trên Kubernetes, YARN hoặc dịch vụ Spark managed; logic Spark có khả năng di chuyển tốt hơn so với job gắn chặt một dịch vụ.
- **Phù hợp ETL nặng:** hữu ích khi transform trước khi nạp/công bố, hoặc khi SQL warehouse không phù hợp với thuật toán hay định dạng dữ liệu.

### Nhược điểm và đánh đổi

- **Không có trải nghiệm managed mặc định:** cluster, dependency, scheduling, logging, monitoring, security, metadata catalog và recovery phải tự thiết kế hoặc ghép thêm công cụ.
- **Độ phức tạp vận hành/kỹ thuật cao:** partition, shuffle, skew, checkpoint và schema evolution có thể làm job chậm hoặc lỗi nếu không được tối ưu.
- **Không thay thế dbt Core cho mart SQL:** Spark có thể tạo bảng phân tích, nhưng dbt Core thường thuận tiện hơn cho lineage, test và quản trị model SQL trong warehouse. Nhiều đội dùng cả hai theo từng lớp.

### Khi nên dùng

Chọn Spark khi khối lượng hay độ phức tạp vượt khả năng/chi phí hợp lý của SQL warehouse, hoặc khi cần xử lý dữ liệu tệp/bán cấu trúc phân tán trên hạ tầng tự quản. Không chọn Spark chỉ vì dữ liệu “lớn”; thử trước transform bằng SQL/dbt Core nếu nó đủ đơn giản và dễ vận hành hơn.

## So sánh và kiến trúc kết hợp

| Tiêu chí             | AWS Glue                                      | Fivetran                                       | Airbyte OSS                            | dbt Core                                              | Apache Spark                                       |
| -------------------- | --------------------------------------------- | ---------------------------------------------- | -------------------------------------- | ----------------------------------------------------- | -------------------------------------------------- |
| Extract/load         | Có, tùy nguồn và job                          | Là trọng tâm                                   | Là trọng tâm                           | Không                                                 | Có qua code/connector, không phải thế mạnh managed |
| Transform            | Có, Spark/Python/SQL                          | Có khả năng bổ sung nhưng không phải trọng tâm | Không phải trọng tâm                   | Là trọng tâm, chủ yếu SQL trong đích                  | Là trọng tâm, SQL/Python/Scala/Java                |
| Hạ tầng vận hành     | Serverless AWS, vẫn cần cấu hình job/quan sát | Managed nhiều phần ingestion                   | Tự triển khai, nâng cấp và giám sát    | Tự quản project, runner và warehouse đích             | Tự quản cluster/platform hoặc dùng Spark managed   |
| Linh hoạt logic      | Cao, đổi lại cần kỹ năng Spark                | Cao trong phạm vi connector                    | Cao nếu có năng lực tùy biến connector | Cao cho model SQL, thấp hơn cho logic ngoài warehouse | Rất cao, đổi lại tăng độ phức tạp kỹ thuật         |
| Rủi ro chi phí chính | Compute/quét dữ liệu và thiết kế Spark        | MAR, số connection và mức thay đổi             | Compute, storage và nhân lực vận hành  | Compute warehouse/lakehouse do model tạo ra           | Cluster/compute, storage và nhân lực vận hành      |

Hai tổ hợp thường gặp:

1. **Fivetran + dbt Core:** connector managed nạp raw/staging -> dbt Core chuẩn hóa, kiểm thử và tạo mart. Phù hợp nguồn phổ biến, muốn triển khai nhanh nhưng vẫn tự quản code transform.
2. **Airbyte OSS + dbt Core:** Airbyte tự quản nạp raw/staging -> dbt Core tạo model. Đây là tổ hợp mã nguồn mở phổ biến, đổi chi phí license theo usage lấy quyền kiểm soát và trách nhiệm vận hành.
3. **Apache Spark + dbt Core:** Spark xử lý/chuẩn hóa nặng vào lake/warehouse -> dbt Core mô hình hóa phần phục vụ phân tích. Phù hợp tệp, log, dữ liệu bán cấu trúc hay transform không thuận tiện trong SQL warehouse.
4. **AWS Glue + dbt Core:** phiên bản AWS-managed của mô hình Spark + dbt Core, phù hợp khi ưu tiên giảm vận hành engine Spark hơn tính di động đa cloud.

Không có tổ hợp nào tự động bảo đảm chất lượng. Cả hai vẫn cần hợp đồng nguồn, quyền với raw/PII, alert freshness, đối soát số liệu, quy tắc dữ liệu muộn/xóa, và kiểm thử retry/backfill.

## Khuyến nghị chọn nhanh

- Cần xử lý dữ liệu lớn, format phức tạp hoặc logic trước khi công bố trong AWS: bắt đầu đánh giá **AWS Glue**.
- Cần đưa nhanh SaaS/database tiêu chuẩn vào kho dữ liệu và giảm vận hành connector: đánh giá **Fivetran**, đồng thời ước lượng MAR bằng dữ liệu thay đổi thực tế.
- Cần ingestion mã nguồn mở/tự quản: đánh giá **Airbyte OSS**, kèm kế hoạch vận hành, nâng cấp và giám sát connector.
- Dữ liệu đã nằm trong warehouse và mục tiêu là model SQL có test/lineage: dùng **dbt Core** cho lớp transform.
- Cần transform phân tán, tệp lớn hoặc logic phức tạp: đánh giá **Apache Spark**; nếu ở AWS và không muốn vận hành runtime Spark, đánh giá **AWS Glue**.
- Cần tổ hợp mã nguồn mở: thử **Airbyte OSS + dbt Core** trên một nguồn và một mart thật, đo latency, chi phí hạ tầng và tỷ lệ lỗi.

## Điểm cần kiểm chứng trong proof of concept

Các nhà cung cấp đều mô tả khả năng tổng quát, nhưng kết quả phụ thuộc nguồn và đích cụ thể. Một proof of concept nên kiểm tra:

- nguồn có hỗ trợ incremental/CDC hay bị full re-import; có bắt được `DELETE` không;
- schema change, bản ghi đến muộn, retry và backfill có làm trùng/mất dữ liệu không;
- quyền đọc nguồn, quyền raw/PII và audit có đáp ứng chính sách không;
- độ trễ end-to-end từ thay đổi nguồn đến mart;
- chi phí: DPU/compute/quét dữ liệu với Glue, MAR với Fivetran, compute và nhân lực vận hành với Airbyte/Spark, compute warehouse với dbt Core;
- chất lượng: uniqueness, referential integrity, freshness và đối soát tổng số/tổng tiền.

## Tài liệu tham khảo

1. Amazon Web Services, [What is AWS Glue?](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html), truy cập 21/08/2026.
2. Fivetran, [Database connectors](https://fivetran.com/docs/connectors/databases), truy cập 21/08/2026.
3. Fivetran, [Usage-Based Pricing](https://fivetran.com/docs/getting-started/pricing), truy cập 21/08/2026.
4. Airbyte, [Airbyte documentation](https://docs.airbyte.com/), truy cập 21/08/2026.
5. dbt Labs, [What is dbt?](https://docs.getdbt.com/docs/introduction), truy cập 21/08/2026.
6. Apache Spark, [Spark SQL and DataFrames](https://spark.apache.org/docs/latest/sql-programming-guide), truy cập 21/08/2026.
