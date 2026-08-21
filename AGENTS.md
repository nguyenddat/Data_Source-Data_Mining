# Hướng dẫn viết tài liệu

## Phạm vi

Áp dụng các quy tắc dưới đây cho mọi tệp Markdown phân tích, nghiên cứu hoặc giải thích một công nghệ. Không áp dụng cho trang mục lục, nhật ký, ghi chú ngắn, mẫu biểu hay tệp cấu hình.

## Cách viết

1. Không thêm tiêu đề cấp một (`# ...`) chỉ để lặp lại tên tệp; Obsidian đã hiển thị tên ghi chú.
2. Bắt đầu tài liệu trực tiếp bằng tiêu đề `## 1. Vấn đề đặt ra`.
3. Ở phần này, nêu ngắn gọn bối cảnh, hạn chế hoặc nhu cầu thực tế khiến công nghệ cần thiết.
4. Chỉ sau đó mới giải thích công nghệ: nó là gì, hoạt động như thế nào, khi nào nên dùng và các giới hạn hoặc đánh đổi quan trọng khi phù hợp.
5. Viết súc tích, chính xác và đủ ý; ưu tiên ví dụ cụ thể khi giúp làm rõ nội dung.
6. Khi mô tả từ ba mục cùng loại trở lên theo các thuộc tính lặp lại (ví dụ: trường của event, mã thao tác, lựa chọn cấu hình), ưu tiên bảng Markdown nếu bảng làm việc đối chiếu dễ đọc hơn. Không ép dùng bảng cho luồng tuần tự, giải thích nhân quả hoặc nội dung ngắn vốn rõ hơn ở dạng văn xuôi/danh sách.
7. Đánh số tiêu đề theo phân cấp: tiêu đề cấp hai dùng `## 1. ...`, `## 2. ...`; tiêu đề cấp ba dùng `### 1.1. ...`, `### 1.2. ...`; các cấp sâu hơn tiếp tục dạng `#### 1.2.1. ...`. Mỗi cấp con bắt đầu lại từ `1` theo tiêu đề cha của nó.

## Liên kết phân cấp

1. Các tài liệu công nghệ được tổ chức theo phân cấp cha–con. Mỗi tài liệu con phải nhắc và liên kết đến tài liệu cha trực tiếp bằng wikilink Obsidian.
2. Lồng liên kết này tự nhiên trong phần `## 1. Vấn đề đặt ra` hoặc khi lần đầu giới thiệu công nghệ. Ví dụ: Debezium là một Log-Based CDC|log-based CDC.
3. Không dùng một dòng nhãn riêng như `Cha: ...`, và không dùng liên kết Markdown dạng `[văn bản](đường-dẫn)` cho mục đích biểu thị quan hệ cha–con.
4. Liên kết phải trỏ đến cấp gần nhất, không bỏ qua cấp trung gian: tài liệu về log-based CDC liên kết đến Change Data Capture|CDC; tài liệu về Debezium liên kết đến Log-Based CDC|log-based CDC.

## Nghiên cứu nguồn

Khi nghiên cứu một công nghệ, đọc và đối chiếu nhiều website độc lập để tăng tính hội tụ kiến thức; không kết luận chỉ từ một nguồn. Dùng Playwright để mở và đọc các trang web. Ưu tiên tài liệu chính thức, tiêu chuẩn kỹ thuật, bài viết từ nhà cung cấp hoặc nguồn chuyên môn đáng tin cậy; ghi nhận các điểm khác biệt hoặc chưa thống nhất giữa các nguồn trước khi tổng hợp kết luận.
