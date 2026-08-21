## Vấn đề đặt ra

Khi tích hợp dữ liệu từ nhiều hệ thống, không thể biết hết các cách sử dụng tại thời điểm nạp. Nếu mọi dữ liệu đều phải được làm sạch và mô hình hóa trước khi vào kho phân tích, một quy tắc mới hoặc nhu cầu khám phá mới có thể buộc phải sửa pipeline trung gian rồi chạy lại từ đầu. Trong khi đó, data warehouse/lakehouse hiện đại có thể lưu trữ lớn và thực thi SQL song song.

ELT đưa dữ liệu vào đích trước để giữ một lớp gần nguồn và tận dụng compute của đích cho nhiều phép biến đổi sau đó. Điều này không loại bỏ yêu cầu về chất lượng, bảo mật, chi phí truy vấn và khả năng chạy lại; chúng chỉ chuyển phần lớn sang môi trường đích.

## ELT là gì?

**ELT** là viết tắt của **Extract – Load – Transform**:

Nguồn dữ liệu -> Extract -> Load vào raw/staging ở đích -> Transform trong đích -> bảng phục vụ phân tích

- **Extract:** lấy snapshot hoặc phần thay đổi từ database, API, tệp, log hay message broker.
- **Load:** ghi dữ liệu nguồn, hoặc dữ liệu chỉ được chuẩn hóa kỹ thuật tối thiểu, vào data lake, data warehouse hoặc lakehouse.
- **Transform:** chạy SQL, procedure, hoặc compute của nền tảng đích để làm sạch, kết hợp, kiểm tra và tạo bảng cho người dùng.

Điểm phân biệt là **vị trí và thời điểm transform nghiệp vụ**: trong ELT, transform diễn ra sau khi dữ liệu đã ở hệ thống đích. “Load trước” không có nghĩa nạp không kiểm soát: dữ liệu vẫn cần kiểm tra định dạng tối thiểu, mã hóa khi truyền, giới hạn quyền và xử lý bản ghi lỗi.

Google Cloud mô tả ELT là nạp raw vào data lake hoặc cloud data warehouse rồi mới transform; AWS xem đây là biến thể của ETL, phù hợp khi hệ thống đích đủ mạnh để xử lý biến đổi. [Google Cloud](https://cloud.google.com/discover/what-is-elt) [AWS Data Warehousing](https://docs.aws.amazon.com/whitepapers/latest/data-warehousing-on-aws/data-processing.html)

## Luồng dữ liệu điển hình

Một triển khai thường tách dữ liệu ở đích thành các lớp có trách nhiệm rõ ràng:

| Lớp                | Mục đích                                                       | Ví dụ                                                      | Người dùng trực tiếp                  |
| ------------------ | -------------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------- |
| **Raw / landing**  | Giữ dữ liệu gần với lúc nhận nhất, phục vụ truy vết và nạp lại | JSON API, bản sao bảng nguồn, event CDC kèm offset         | Kỹ sư dữ liệu/quyền đặc biệt          |
| **Staging**        | Chuẩn hóa kỹ thuật, ép kiểu, đổi tên, khử trùng lặp cơ bản     | `stg_orders` có một dòng hợp lệ cho mỗi phiên bản đơn hàng | Nhóm xây dựng dữ liệu                 |
| **Intermediate**   | Tái sử dụng logic ghép, ánh xạ và quy tắc nghiệp vụ            | đơn hàng đã gắn khách hàng, tỷ giá, trạng thái             | Nhóm xây dựng dữ liệu                 |
| **Mart / serving** | Công bố dữ liệu có nghĩa, ổn định cho BI và sản phẩm dữ liệu   | `fct_sales`, `dim_product`, doanh thu theo giờ             | BI, phân tích, ứng dụng được ủy quyền |

Tên các lớp không phải chuẩn bắt buộc. Điều quan trọng là biết bảng nào là bản sao nguồn, bảng nào đã được kiểm tra, và bảng nào là hợp đồng dữ liệu cho báo cáo. Dashboard không nên đọc trực tiếp raw vì lược đồ, bản ghi trùng, xóa và ý nghĩa trường có thể chưa được xử lý.

## Extract: lấy đầy đủ và có thể tiếp tục

ELT dùng cùng các chiến lược extract với ETL. Transform ở đích không tự làm cho việc lấy dữ liệu trở thành incremental hay gần thời gian thực.

| Cách lấy                             | Phù hợp                                                     | Trạng thái cần giữ                    | Rủi ro cần xử lý                                             |
| ------------------------------------ | ----------------------------------------------------------- | ------------------------------------- | ------------------------------------------------------------ |
| Snapshot/full load                   | Nạp ban đầu, tập nhỏ, backfill                              | Mã lần chạy, phạm vi snapshot         | Tốn tài nguyên; nguồn đổi trong lúc đọc                      |
| Incremental bằng watermark           | Nguồn có `updated_at`, khóa tăng dần hoặc phân vùng tin cậy | Watermark của lần chạy thành công     | Cùng timestamp, dữ liệu muộn, bản ghi bị xóa                 |
| [CDC](../Change%20Data%20Capture.md) | Cần bắt `INSERT`/`UPDATE`/`DELETE` với độ trễ thấp          | Offset/LSN/SCN và trạng thái snapshot | Retention log, thứ tự sự kiện, schema evolution, áp dụng lặp |

Với incremental load, chỉ lưu watermark sau khi ghi thành công; dùng cửa sổ chồng lấn và khử trùng lặp theo khóa cùng phiên bản/thời điểm sự kiện khi nguồn có thể gửi dữ liệu muộn. Với CDC, raw nên giữ khóa sự kiện, loại thao tác, thời điểm nhận và offset để tái dựng hoặc kiểm tra chuỗi thay đổi.

## Load: đưa dữ liệu vào đích mà không mất dấu

Load trong ELT thường ưu tiên **tính bảo toàn** hơn là mô hình hóa sớm. Một batch raw nên có metadata tối thiểu như `ingested_at`, `source_system`, `batch_id`, phiên bản lược đồ và, nếu có, offset CDC. Nhờ đó có thể trả lời dữ liệu đến từ đâu, lúc nào và lần chạy nào tạo ra nó.

- Ghi theo batch hoặc partition xác định để retry không tạo bản sao không kiểm soát. Với bảng, dùng khóa nguồn cùng version/event id; với tệp, dùng đường dẫn có ngày nhận và batch id.
- Tách dữ liệu chưa được tin cậy khỏi dữ liệu công bố. Bản ghi sai định dạng đi vào **quarantine** kèm lý do lỗi, không bị âm thầm bỏ qua.
- Giữ raw đủ lâu theo retention và nhu cầu audit/backfill, nhưng không lấy đó làm lý do để giữ dữ liệu cá nhân vô thời hạn.
- Cấp quyền theo lớp. Raw có thể chứa PII hoặc trường chưa được phân loại; quyền BI không mặc định bao gồm quyền đọc raw.

Mục tiêu không nhất thiết là bản sao byte-for-byte. Connector có thể thêm metadata hoặc chuyển JSON sang cột, nhưng phải bảo tồn thông tin nguồn cần thiết để giải thích và tái xử lý kết quả.

## Transform trong data warehouse/lakehouse

Transform tạo dữ liệu có nghĩa từ các lớp đã nạp. Công việc thường gồm:

- ép kiểu, chuẩn hóa múi giờ/đơn vị/mã, kiểm tra trường bắt buộc;
- khử trùng lặp theo khóa nghiệp vụ và phiên bản mới nhất;
- xử lý xóa, soft-delete hoặc event tombstone theo hợp đồng nguồn;
- ghép bảng tham chiếu, ánh xạ mã, tính chỉ số và tổng hợp;
- mô hình hóa thành fact/dimension, bảng rộng, view hoặc data product;
- kiểm tra `not null`, duy nhất, miền giá trị, quan hệ tham chiếu, số dòng, độ mới và đối soát tổng tiền.

SQL là lựa chọn phổ biến vì transform chạy gần dữ liệu, nhưng ELT không đồng nghĩa với “chỉ SQL”. Stored procedure, Python/Spark chạy trong lakehouse hay UDF cũng có thể là transform nếu dữ liệu đã nằm trong nền tảng đích. Ngược lại, pipeline dùng SQL ở máy trung gian trước khi nạp kho vẫn là ETL theo nghĩa logic.

### Mô hình hóa theo phụ thuộc

Nên khai báo transform thành các model nhỏ có đầu vào/đầu ra rõ thay vì một câu lệnh dài tạo trực tiếp bảng báo cáo. Quan hệ phụ thuộc tạo thành DAG:

`raw_orders -> stg_orders -> int_order_items -> fct_sales -> dashboard`

Mỗi model có thể có owner, mô tả, kiểm tra và lịch chạy. Công cụ transform như dbt thường đảm nhiệm phần này: biên dịch model thành SQL chạy trong warehouse; ingest vẫn do connector hoặc pipeline khác chịu trách nhiệm. Tài liệu dbt nhấn mạnh lineage, kiểm tra và freshness của nguồn trong vận hành dữ liệu. [dbt Developer Hub](https://docs.getdbt.com/) [dbt Project Evaluator: testing](https://dbt-labs.github.io/dbt-project-evaluator/main/rules/testing/)

### Incremental transform, dữ liệu muộn và chạy lại

Không phải transform nào cũng nên chạy lại toàn bộ lịch sử. Bảng lớn có thể chỉ tính phần thay đổi bằng `MERGE` theo khóa, ghi đè partition bị ảnh hưởng, hoặc append event bất biến:

- **Event bất biến:** append, đồng thời loại trùng theo event id khi nguồn có thể giao lại.
- **Trạng thái hiện tại:** `MERGE`/upsert theo khóa nghiệp vụ và phiên bản mới nhất.
- **Tổng hợp theo thời gian:** tính lại các partition chịu ảnh hưởng, gồm một khoảng lùi để hấp thụ dữ liệu đến muộn.

Ví dụ, doanh thu 10:00–11:00 đã công bố nhưng có đơn hàng đến lúc 11:10. Pipeline có thể tính lại partition giờ 10, chờ một khoảng trước khi chốt, hoặc ghi điều chỉnh ở kỳ sau. Phải công bố rõ chính sách; nếu không dashboard có thể đổi số mà không ai hiểu vì sao.

Mỗi model incremental cần có khóa, phạm vi đọc và hành vi full refresh rõ ràng. Chỉ lọc theo `MAX(updated_at)` dễ bỏ bản ghi cùng timestamp hoặc đến muộn; dùng điều kiện biên xác định, cửa sổ chồng lấn khi cần và khử trùng lặp/merge có tính idempotent.

## Chất lượng, quản trị và quan sát

Nạp raw sớm tăng khả năng tái sử dụng, nhưng dữ liệu chưa xử lý cũng xuất hiện sớm trong môi trường phân tích. ELT nên có các kiểm soát sau:

- **Data contract và schema evolution:** phát hiện cột mới/mất, đổi kiểu hoặc đổi ngữ nghĩa; thay đổi phá vỡ hợp đồng phải chặn công bố hoặc qua quy trình phê duyệt.
- **Kiểm tra trước và sau transform:** kiểm tra batch, uniqueness, referential integrity, phân bố bất thường và đối soát với nguồn khi phù hợp.
- **Lineage và version:** biết bảng/mart nào được tạo từ batch raw, model và phiên bản quy tắc nào.
- **Quan sát:** đo source freshness, độ trễ end-to-end, số dòng nạp/loại/quarantine, thời gian/bytes quét, tỷ lệ lỗi và backlog.
- **Bảo mật:** phân loại dữ liệu khi nạp; cấp quyền tối thiểu; masking/tokenization PII trước khi cấp cho đa số người dùng; mã hóa và audit truy cập.
- **Chi phí:** ELT chuyển phần lớn chi phí compute sang đích. Partition, clustering, định dạng cột, materialization hợp lý và giới hạn truy vấn ad-hoc giúp tránh quét raw quá lớn.

Kho dữ liệu có ACID không tự bảo đảm logic đúng. Tính nhất quán giữa raw, staging và mart vẫn phụ thuộc thứ tự chạy, transaction/hoán đổi khi publish, retry và thao tác ghi của engine cụ thể.

## ELT và ETL: chọn theo ràng buộc, không theo tên gọi

| Tiêu chí | ELT | ETL |
| --- | --- | --- |
| Nơi transform chính | Data warehouse, lakehouse hoặc data lake đích | Engine/staging ngoài đích hoặc trước lớp công bố |
| Trình tự logic | Extract -> Load -> Transform | Extract -> Transform -> Load |
| Dữ liệu raw tại đích | Thường có, hữu ích cho khám phá và tái xử lý | Có thể chỉ ở staging hoặc không nạp vào warehouse đích |
| Điểm mạnh | Linh hoạt, tận dụng compute đích, SQL gần dữ liệu, dễ tạo nhiều mart từ một raw source | Chặn/lọc dữ liệu trước đích, hợp khi đích bị giới hạn hoặc quy tắc phải áp trước khi lưu |
| Rủi ro nổi bật | PII/dữ liệu kém chất lượng xuất hiện sớm; transform kém tối ưu gây chi phí cao | Logic tiền xử lý thành nút thắt; khó đáp ứng nhu cầu mới nếu raw không được giữ |

Ranh giới không tuyệt đối. ELT vẫn có thể che trường nhạy cảm hoặc kiểm tra bắt buộc trước khi load; ETL vẫn có thể giữ raw ở data lake. Kiến trúc **hybrid** thường biến đổi kỹ thuật/kiểm soát bắt buộc trước, rồi transform nghiệp vụ trong warehouse. Google Cloud cũng nêu lựa chọn phụ thuộc hạ tầng, khối lượng và nhu cầu phân tích; có thể kết hợp cả hai. [Google Cloud](https://cloud.google.com/discover/what-is-elt)

## Ví dụ: dữ liệu bán lẻ

Một chuỗi bán lẻ cần báo cáo doanh thu theo cửa hàng và sản phẩm, đồng thời muốn nhà phân tích có thể nghiên cứu hành vi mua hàng mới.

1. **Extract và load:** nạp snapshot `orders`, `order_items`, `products`; sau đó dùng CDC cho đơn hàng. Raw lưu event, khóa giao dịch, thao tác CDC, offset, thời điểm nhận và batch id.
2. **Staging:** chuẩn hóa timestamp về UTC, ép kiểu tiền, giữ event mới nhất của mỗi phiên bản đơn hàng và đưa mã sản phẩm không hợp lệ vào quarantine.
3. **Transform:** ghép danh mục sản phẩm/cửa hàng, tính doanh thu ròng và tạo `fct_sales` cùng tổng hợp giờ. Tính lại các partition vài giờ gần nhất để xử lý hủy đơn hoặc đơn đến muộn.
4. **Công bố và theo dõi:** dashboard chỉ đọc mart; cảnh báo khi source freshness vượt ngưỡng, tổng tiền raw và mart chênh bất thường, hoặc số bản ghi quarantine tăng.

Nếu bộ phận kinh doanh cần chỉ số mới, họ tạo model từ staging/intermediate mà không phải yêu cầu nguồn gửi lại lịch sử. Nhưng nếu raw chứa email và địa chỉ, bảng đó vẫn phải giới hạn quyền; có raw không đồng nghĩa mọi người được đọc raw.

## Khi nào nên dùng ELT và các đánh đổi

ELT phù hợp khi đích là warehouse/lakehouse có compute co giãn, cần giữ dữ liệu gốc để tái sử dụng, có nhiều nhóm tiêu thụ với nhu cầu thay đổi, và tổ chức muốn quản trị transform bằng SQL/model versioned gần dữ liệu. Nó tự nhiên với dữ liệu bán cấu trúc, khối lượng lớn và phân tích khám phá.

ELT không phù hợp nếu chính sách cấm đưa dữ liệu chưa lọc vào đích, warehouse không đủ compute hoặc chi phí không kiểm soát được, hay transform cần công nghệ chỉ có ở môi trường trước đích. Khi đó ETL hoặc hybrid thường hợp lý hơn.

Đừng đồng nhất ELT với streaming hay CDC: ELT có thể chạy batch hằng ngày; CDC chỉ là một cách lấy dữ liệu thay đổi. Cũng không nên đồng nhất ELT với một công cụ: connector xử lý extract/load, công cụ model/orchestrator xử lý transform và điều phối, còn bảo đảm cuối-cuối phải kiểm chứng với nguồn, đích và thao tác ghi thực tế.

## Tài liệu tham khảo

1. Google Cloud, [What is ELT (extract, load, and transform)?](https://cloud.google.com/discover/what-is-elt), truy cập 21/08/2026.
2. Google Cloud, [Migrate data pipelines](https://docs.cloud.google.com/bigquery/docs/migration/pipelines), truy cập 21/08/2026.
3. Amazon Web Services, [Data processing — Data Warehousing on AWS](https://docs.aws.amazon.com/whitepapers/latest/data-warehousing-on-aws/data-processing.html), truy cập 21/08/2026.
4. Amazon Web Services, [Amazon RDS zero-ETL integrations](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/zero-etl.html), truy cập 21/08/2026.
5. dbt Labs, [dbt Developer Hub](https://docs.getdbt.com/), truy cập 21/08/2026.
6. dbt Labs, [Testing — dbt Project Evaluator](https://dbt-labs.github.io/dbt-project-evaluator/main/rules/testing/), truy cập 21/08/2026.
