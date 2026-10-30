# Kịch bản thuyết trình báo cáo DBS401

## Gợi ý mở đầu

Kính chào thầy và các bạn. Nhóm em xin trình bày đề tài xây dựng honeypot PostgreSQL kết hợp phân tích hành vi tấn công. Báo cáo tập trung vào ba vấn đề nối tiếp nhau: hệ thống ghi nhận được hành động nào, từ các truy vấn đã ghi có thể phân loại và giải thích hành vi ra sao, và các phương pháp đó còn hoạt động thế nào khi gặp kỹ thuật chưa được dùng trong quá trình phát triển.

Trong bài nói này, nhóm sẽ đi nhanh qua bối cảnh, dành phần chính cho câu hỏi và mục tiêu nghiên cứu, thiết kế phương pháp, rồi lần lượt giải thích kết quả thực nghiệm của RQ1, RQ2 và RQ3. Các số liệu đều đến từ môi trường mô phỏng; vì vậy đây là đánh giá có kiểm soát, không phải tỷ lệ phát hiện ngoài mạng thực tế.

## 1. Bối cảnh và vấn đề nghiên cứu

Honeypot cơ sở dữ liệu là dịch vụ mồi để quan sát cách người dùng trái phép dò tìm cấu hình, thử đăng nhập, truy vấn dữ liệu hoặc thay đổi hệ thống. Nhưng muốn phân tích hành vi thì trước tiên phải ghi được sự kiện cần thiết. Sau đó phải biết truy vấn nào thuộc cùng một phiên, truy vấn đó thể hiện giai đoạn nào, và bằng chứng nào hỗ trợ cho kết luận.

Một câu truy vấn đứng riêng đôi khi không đủ để hiểu mục đích. Ví dụ, truy vấn đọc danh mục hệ thống có thể là dò tìm cấu trúc, nhưng một chuỗi truy vấn liên tiếp có thể cho thấy kẻ tấn công đang chuẩn bị thu thập dữ liệu. Vì thế, đề tài không chỉ đo độ chính xác phân loại mà còn xem xét nhật ký, tái dựng phiên và bằng chứng cho nhãn.

Phạm vi là honeypot PostgreSQL chạy cục bộ trong Docker, chứa dữ liệu giả lập. Hệ thống không được đưa lên Internet. Nhãn hành vi là bộ nhãn do nhóm xác định, có tham chiếu các khái niệm MITRE ATT&CK; không phải bản sao nguyên văn taxonomy chính thức.

## 2. Câu hỏi và mục tiêu nghiên cứu

Câu hỏi tổng quát là: Honeypot PostgreSQL tương tác cao kết hợp phân tích hành vi có thể ghi nhận, nhận diện và giải thích các giai đoạn tấn công từ nhật ký đến mức nào, kể cả khi gặp kỹ thuật chưa từng dùng để xây dựng phương pháp?

Ba câu hỏi RQ tạo thành một chuỗi. RQ1 chọn cấu hình nhật ký. Nhật ký đó trở thành đầu vào cho RQ2. Sau khi cố định phương pháp ở RQ2, nhóm đem chính phương pháp ấy kiểm tra trên các họ kỹ thuật bị giữ lại ở RQ3.

### RQ1 — Ghi nhật ký và tái dựng phiên

**RQ1 hỏi:** Các cấu hình ghi nhật ký của honeypot PostgreSQL bỏ sót hoặc tái dựng sai loại hành động và phiên nào, mức độ bao nhiêu, và cấu hình nào đủ dùng cho phân tích hành vi tiếp theo?

- **RO1.1 — Đo độ bao phủ hành động.** Tính recall ở cấp hành động cho ba cấu hình C0, C1, C2 theo từng lớp hành vi và theo các thẻ mô tả khả năng hiển thị trong log, gọi là *gap tags*. Mục đích là xác định không chỉ tổng số bị bỏ sót mà còn nhóm hành động nào gây ra khoảng trống.
- **RO1.2 — Kiểm tra trường dữ liệu và tái dựng phiên.** Kiểm tra sáu thuộc tính: người dùng, IP nguồn, cơ sở dữ liệu, truy vấn, kết quả và tham số. Sau đó đo precision, recall và F1 theo cặp cho việc gom các bản ghi thành phiên, với ngưỡng không hoạt động 1, 5, 15 và 30 phút.
- **RO1.3 — Chọn cấu hình tối thiểu đạt yêu cầu của đề tài.** Nhóm đặt tiêu chí recall tối thiểu 95% trên toàn bộ hành động và ở từng giai đoạn, rồi cân nhắc độ bao phủ, mức đầy đủ trường dữ liệu và số lượng bản ghi log. Đây là ngưỡng nhóm tự chọn để làm tiêu chí thực nghiệm, không phải chuẩn ngành.

### RQ2 — Phân loại và bằng chứng

**RQ2 hỏi:** Giữa bộ phân loại dựa trên luật và mô hình ngôn ngữ lớn chạy cục bộ theo kiểu few-shot, phương pháp nào gán giai đoạn tấn công cho từng truy vấn chính xác hơn khi có truy vấn trước làm ngữ cảnh? Bằng chứng được chọn có khớp với bằng chứng chuẩn không?

- **RO2.1 — Đánh giá nhãn và độ tin cậy đầu ra.** Dùng macro-F1, F1 theo lớp và ma trận nhầm lẫn. Đồng thời ghi nhận đầu ra không hợp lệ, chẳng hạn JSON không phân tích được hoặc sai schema. Với Qwen, nhóm còn kiểm tra mã bằng chứng giả: ID không trỏ tới truy vấn nào trong cửa sổ đầu vào.
- **RO2.2 — Đánh giá bằng chứng.** So sánh danh sách truy vấn được mô hình viện dẫn với tập bằng chứng chuẩn sinh từ quan hệ phụ thuộc giữa các bước. Jaccard là thước đo chính; precision, recall và F1 của tập bằng chứng được báo cáo bổ sung. Nhóm rà soát thủ công 20% mẫu để có đánh giá sơ bộ về chất lượng dữ liệu chuẩn.
- **RO2.3 — Đánh giá tác động của ngữ cảnh.** So sánh cửa sổ gồm truy vấn hiện tại và 0, 2 hoặc 4 truy vấn đứng trước. Theo dõi hiệu năng phân loại, chất lượng bằng chứng khi có ngữ cảnh và thời gian suy luận.

### RQ3 — Khái quát hóa sang họ kỹ thuật chưa gặp

**RQ3 hỏi:** Hiệu năng phân loại và khớp bằng chứng của hai phương pháp giảm bao nhiêu trên các họ kỹ thuật bị loại khỏi giai đoạn phát triển, so với các biến thể của họ đã biết?

- **RO3.1 — Đo mức suy giảm.** Tính thay đổi macro-F1 từ tập họ đã biết sang từng tập họ chưa gặp; mô tả đồng thời thay đổi về mức đồng thuận bằng chứng.
- **RO3.2 — Phân tích kiểu lỗi.** Xem nhầm lẫn theo lớp, số cửa sổ bị gán Noise và mã bằng chứng giả trên dữ liệu chưa gặp.
- **RO3.3 — Khảo sát độ mới của kỹ thuật.** Mô tả định tính quan hệ giữa suy giảm hiệu năng với cấu trúc SQL mới lạ, hàm hoặc extension đặc thù PostgreSQL và độ phức tạp của phụ thuộc giữa các truy vấn.

## 3. Phương pháp nghiên cứu

### Thiết kế và quy trình chung

Đây là đánh giá định lượng trong môi trường có kiểm soát, gồm ba thí nghiệm tương ứng với ba RQ. Bộ sinh kịch bản tạo chuỗi truy vấn tấn công và truy vấn bình thường cùng nhãn chuẩn và quan hệ phụ thuộc. Các truy vấn chạy trên honeypot PostgreSQL theo cấu hình đã chọn. Bộ phân tích cú pháp chuẩn hóa log và ghép bản ghi với bước kịch bản qua mã `aid`. Với RQ1, các sự kiện được gom thành phiên. Với RQ2 và RQ3, truy vấn được xếp thành cửa sổ rồi phân loại và đối chiếu với dữ liệu chuẩn.

### Ba cấu hình ghi nhật ký

- **C0:** `log_statement = all`.
- **C1:** bật ghi nhật ký phiên bằng pgAudit.
- **C2:** C1 cộng với ghi nhận kết nối, ngắt kết nối và tham số (`log_connections`, `log_disconnections`, `pgaudit.log_parameter`; cấu hình cũng đặt rõ mức ghi câu lệnh lỗi).

Mỗi cấu hình chạy riêng trên cơ sở dữ liệu sạch với cùng bộ kịch bản Dev gồm 136 hành động. C2 dự kiến ghi nhận đầy đủ hơn do có log kết nối và tham số.

### Nhãn hành vi

RQ1 dùng bảy nhãn: **Discovery** (dò tìm), **Credential Access** (truy cập thông tin xác thực), **Privilege Escalation** (leo thang đặc quyền), **Collection** (thu thập), **Exfiltration** (đưa dữ liệu ra ngoài), **Impact** (tác động/thay đổi hệ thống) và **Noise** (hoạt động bình thường). Trong dữ liệu, Privilege Escalation chỉ xuất hiện dưới dạng sự kiện kết nối không có SQL, nên RQ2 và RQ3 phân loại sáu nhãn còn lại.

Mỗi bước kịch bản khai báo nhãn, ID bước, các bước trước mà nó phụ thuộc (`depends_on`) và gap tags. Nhờ vậy, nhãn và bằng chứng chuẩn được sinh từ kịch bản thay vì gán nhãn thủ công sau khi chạy. `aid` là mã ngẫu nhiên gắn vào câu lệnh để nối log với bước tạo ra nó; mã bị loại khỏi câu truy vấn trước khi chuyển dữ liệu cho mô hình.

### Bộ dữ liệu

| Tập | Vai trò | Quy mô |
|---|---|---:|
| Dev | Phát triển luật, prompt và chiến lược kết hợp | 17 kịch bản; 102 truy vấn (136 hành động dùng trong RQ1) |
| Holdout | Kiểm tra RQ2, chấm một lần | 22 kịch bản; 109 truy vấn |
| OOD-1 | Tập họ kỹ thuật chưa gặp thứ nhất, dùng cho RQ3 | 9 kịch bản; 57 truy vấn; 3 họ |
| OOD-2 | Tập họ kỹ thuật chưa gặp thứ hai, dùng cho RQ3 | 18 kịch bản; 145 truy vấn; 6 họ |

Dev được dùng để phát triển. Holdout và OOD được giữ khỏi quá trình điều chỉnh phương pháp, nhưng mức độc lập không hoàn toàn: Claude đã xem một phần nội dung Holdout khi rà lỗi; SQL của OOD-1 xuất hiện trong trao đổi sau khi phương pháp đã cố định; OOD-2 cũng lộ một phần chuỗi chương trình trong thông báo kiểm tra chạy. Vì vậy, cần thận trọng khi gọi đây là kiểm thử hoàn toàn mù.

### Hai phương pháp và cửa sổ ngữ cảnh

Phương pháp thứ nhất là bộ phân loại dựa trên luật, minh bạch và chạy gần như tức thời. Phương pháp thứ hai là Qwen3 8B chạy cục bộ qua Ollama, dùng prompt few-shot đã cố định ở phiên bản B+C+E. Mô hình nhận cửa sổ truy vấn, trả về JSON gồm nhãn và ID bằng chứng; định dạng được yêu cầu trong prompt và kiểm tra sau khi sinh, không bị ép cứng trong lúc giải mã.

Tham số `k` là số truy vấn trước được đưa thêm vào cửa sổ: `k=0` chỉ có truy vấn hiện tại; `k=2` hoặc `k=4` bổ sung lịch sử. H1 chuyển các trường hợp hai phương pháp đồng ý sang tự động chấp nhận và gửi trường hợp bất đồng đi rà soát. H3/H4 là các cách kết hợp khác; riêng H4 được hình thành sau khi đã xem OOD-1 nên kết quả H4 phải xem là thăm dò, không phải kiểm định đã đăng ký trước.

### Thước đo và cách đọc kết quả

Macro-F1 là trung bình F1 của các lớp, mỗi lớp có trọng số bằng nhau. Jaccard đo phần giao trên hợp giữa tập truy vấn bằng chứng dự đoán và tập chuẩn. Các khoảng tin cậy bootstrap 95% được lấy mẫu theo **kịch bản**, không theo từng truy vấn, để giữ cấu trúc phụ thuộc trong kịch bản.

Khi khoảng tin cậy của chênh lệch chứa 0, cách diễn giải là **chưa đủ bằng chứng kết luận có khác biệt**; điều đó không chứng minh hai phương pháp tương đương. Với RQ3, `ΔF1 = F1(Holdout) − F1(tập chưa gặp)`, nên số dương biểu thị điểm đã giảm. `D = ΔF1(Rule) − ΔF1(Qwen)` dương nghĩa là Qwen giảm ít hơn Rule; nó không tự động có nghĩa Qwen đạt điểm tuyệt đối cao hơn.

## 4. Kết quả thực nghiệm và diễn giải

### RQ1 — Nhật ký ghi được gì, và có ghép lại được phiên không?

#### RO1.1 — Độ bao phủ hành động

Trên 136 hành động của 17 kịch bản Dev, **C0 ghi nhận 116/136, recall 85,3%; C1 ghi 121/136, recall 89,0%; C2 ghi đủ 136/136, recall 100%**. Theo tiêu chí nhóm đặt ra, chỉ C2 đạt ít nhất 95% tổng thể và ở mọi giai đoạn.

Điểm đáng chú ý là hai cấu hình đơn giản không bỏ sót lỗi truy vấn hay đăng nhập thất bại như dự kiến ban đầu. Cả ba đều ghi nhận 11/11 truy vấn lỗi và 19/19 lần đăng nhập thất bại, vì mức log lỗi PostgreSQL mặc định đã ghi các sự kiện này. Khoảng trống thật tập trung ở hai loại khác:

- Với câu lệnh chuẩn bị sẵn, C0 ghi 6/11, tức 54,5%; C1 và C2 ghi 11/11.
- Với hành động dùng nhiều kết nối, C0 và C1 mỗi cấu hình chỉ ghi 7/19, tức 36,8%; C2 ghi 19/19 nhờ log kết nối/ngắt kết nối.
- Nhóm né tránh log không được kiểm thử, nên không thể kết luận cấu hình nào chống được kiểu né tránh đó.

Trong các tập chưa gặp, C2 ghi 62/63 hành động ở OOD-1 và 159/159 ở OOD-2. Cần lưu ý việc C2 ghi đủ sự kiện kết nối một phần là hệ quả trực tiếp của thiết kế cấu hình bật log kết nối, không phải bằng chứng rằng mọi dạng tấn công đều đã được bao phủ.

Đổi lại, C2 tạo **322 bản ghi**, so với 199 của C0 và 204 của C1. Như vậy C2 là cấu hình duy nhất đạt tiêu chí bao phủ trong thí nghiệm, đồng thời cũng có lượng bản ghi lớn nhất. Nhóm chỉ đo số bản ghi, chưa đo dung lượng byte, độ trễ truy vấn hay chi phí CPU.

#### RO1.2 — Trường dữ liệu và tái dựng phiên

Trường user, IP nguồn, database và kết quả có mặt trong toàn bộ bản ghi được ghép ở cả ba cấu hình. Nội dung câu lệnh có trong 83,6% bản ghi C0, 84,3% C1 và 75,0% C2. Tỷ lệ thấp hơn ở C2 một phần vì các sự kiện kết nối không có SQL. Tham số liên kết không có trong C0/C1; C2 chỉ có ở 5/136 bản ghi, tương đương 3,7%.

Kết quả quan trọng hơn là **ghi đủ hành động không đồng nghĩa tái dựng được phiên**. F1 theo cặp của gom phiên chỉ khoảng 0,11 ở cả ba cấu hình. Precision khoảng 0,06; recall khoảng 0,57–0,62. Đổi ngưỡng không hoạt động từ 1 lên 5, 15 hoặc 30 phút không thay đổi kết quả, vì tất cả kịch bản chạy từ cùng một IP nguồn. IP không giúp tách các phiên; gom theo user còn có thể chia một kịch bản khi kịch bản đổi user.

Đây là hạn chế của thiết kế mô phỏng một địa chỉ, không phải kết luận rằng gom phiên trong honeypot thực tế luôn kém. Kết quả cho thấy RQ2 chưa thể dựa vào phiên tự tái dựng một cách đáng tin cậy; vì vậy nhóm dùng ngữ cảnh lý tưởng lấy từ ID kịch bản và kiểm tra thêm độ nhạy với ngữ cảnh sai.

### RQ2 — Phân loại truy vấn và lựa chọn bằng chứng

#### RO2.1 — Macro-F1 trên Holdout

Ở thiết lập chính `k=2`, Rule đạt **0,823** (khoảng tin cậy 0,756–0,888), còn Qwen đạt **0,769** (0,701–0,827). Chênh lệch Qwen trừ Rule là **−0,053**, với khoảng tin cậy **−0,131 đến +0,016**. Vì khoảng này chứa 0, nhóm chưa đủ bằng chứng để kết luận một phương pháp tốt hơn phương pháp kia ở thiết lập chính. Điểm ước lượng của Rule cao hơn, nhưng không nên diễn giải thành chiến thắng có ý nghĩa thống kê; cũng không nên nói hai phương pháp tương đương.

Ở `k=0`, Rule đạt 0,826 còn Qwen 0,732; chênh lệch −0,094 với khoảng −0,151 đến −0,034, không chứa 0. Đây là thiết lập thăm dò. Ở `k=4`, Rule 0,823 và Qwen 0,769; khoảng của chênh lệch −0,053 là −0,133 đến +0,021, vẫn chứa 0.

Trong các lỗi cụ thể tại `k=2`, Rule thường nhầm Collection thành Discovery (4/15 trường hợp Collection), đồng thời nhầm một số Noise thành Discovery hoặc Collection. Qwen nhầm Impact thành Exfiltration (5/27) và Collection thành Impact (5/15). Vì vậy macro-F1 tổng hợp cần được đọc cùng ma trận nhầm lẫn: hai phương pháp có kiểu sai khác nhau.

#### RO2.2 — Bằng chứng có khớp không?

Trên Holdout tại `k=2`, Jaccard của Rule là **0,821**, Qwen là **0,754**; chênh lệch Qwen trừ Rule −0,067, khoảng tin cậy −0,129 đến −0,015. Ở `k=4`, hai điểm lần lượt là 0,777 và 0,767; chênh lệch −0,010 với khoảng −0,074 đến +0,044, chứa 0.

Do đó, tại `k=2`, tập truy vấn bằng chứng do Rule chọn khớp với tập chuẩn hơn theo Jaccard trong mẫu này. Nhưng tập chuẩn được tự động tạo từ `depends_on`; đây không phải đánh giá đầy đủ chất lượng giải thích trong mắt chuyên gia. Rà soát 20 cửa sổ Dev bởi một AI reviewer cho Jaccard 0,775: 12 cửa sổ khớp, 1 có truy vấn thừa, 4 thiếu truy vấn và 3 khác nhau. Kết quả này chỉ là kiểm tra sơ bộ giữa hai cách gán bằng chứng, không xác nhận ground truth là đúng.

Ở dữ liệu chưa gặp, Qwen có Jaccard cao hơn Rule trong một số so sánh: OOD-2 chênh +0,055 tại `k=2` và +0,092 tại `k=4`; OOD-1 chênh +0,095 ở `k=4`, còn ở `k=2` khoảng tin cậy vẫn chứa 0. Đây là mức đồng thuận với bộ bằng chứng chuẩn, không phải xếp hạng chất lượng lời giải thích tổng quát.

#### RO2.3 — Ngữ cảnh có giúp không?

Điểm Qwen quan sát được tăng từ 0,732 ở `k=0` lên 0,769 ở `k=2` và `k=4`. Tuy nhiên, chênh lệch ghép cặp `k=2` trừ `k=0` là +0,037 với khoảng −0,061 đến +0,113; `k=4` trừ `k=2` là +0,000 với khoảng −0,031 đến +0,038. Các khoảng đều chứa 0. Hơn nữa, `k=0` dùng ít ví dụ few-shot hơn (3 so với 10), nên so sánh `k=0` với hai thiết lập có ngữ cảnh không chỉ thay đổi độ dài lịch sử.

Trên Dev, nhóm cố ý làm nhiễu ngữ cảnh để kiểm tra độ nhạy. Ở `k=2`, Qwen giảm 0,058 khi dùng kết quả gom phiên và giảm 0,071 khi trộn hoàn toàn ngữ cảnh; ở `k=4`, mức giảm tương ứng là 0,089 và 0,048. Một số điều kiện có khoảng tin cậy chạm hoặc chứa 0. Rule gần như không đổi, mức biến động tối đa 0,016. Mức giảm không tăng đều theo tỷ lệ ngữ cảnh bị trộn, nên không xếp hạng các điều kiện này. Thí nghiệm chỉ chạy trên Dev.

Tóm lại, quan sát điểm số gợi ý Qwen có thể được lợi từ lịch sử truy vấn, nhưng dữ liệu chưa đủ chứng minh lợi ích đó. Đồng thời, ngữ cảnh bị sai có thể làm Qwen giảm điểm, nên chất lượng tái dựng phiên ở RQ1 là vấn đề thực tế cần giải quyết trước khi dựa vào phân loại theo ngữ cảnh.

#### Chiến lược kết hợp H1

Trên Holdout tại `k=2`, H1 tự chấp nhận 77,1% cửa sổ và độ chính xác trong nhóm được chấp nhận là 94,1%. Con số này không thể so trực tiếp với độ chính xác 81,7% của Rule trên toàn bộ cửa sổ, vì H1 chọn một tập con dễ hơn. Độ chính xác toàn quy trình có người rà soát chưa được đo.

Trên OOD-2, độ chính xác của các cửa sổ H1 tự chấp nhận giảm còn 43,2% ở `k=2`. Như vậy hai mô hình đồng ý không còn là dấu hiệu đáng tin rằng nhãn đúng khi gặp kỹ thuật lạ. Bộ lọc tự động dựa trên đồng thuận cần người kiểm tra trước khi sử dụng trong bối cảnh này.

### RQ3 — Hiệu năng trên các họ kỹ thuật chưa gặp

#### RO3.1 — Mức giảm macro-F1

Tất cả các phương pháp đều giảm rõ trên hai tập chưa gặp. So sánh trên các lớp chung, mức giảm của Rule ở OOD-1 khoảng **0,439–0,445**, Qwen **0,291–0,294**, H4 **0,354–0,388**. Ở OOD-2, Rule giảm **0,278–0,281**, Qwen **0,160–0,250**, H4 **0,150–0,226**. Các khoảng tin cậy của từng mức giảm đều không chứa 0.

`D` trả lời câu hỏi hẹp hơn: Qwen suy giảm ít hơn Rule bao nhiêu? Trên OOD-1, D khoảng +0,145 đến +0,151 ở cả ba giá trị k; sau điều chỉnh sáu phép so sánh, các khoảng vẫn không chứa 0. Trên OOD-2, tại `k=0`, D=+0,121 nhưng kết quả sát ngưỡng: cận dưới khoảng đồng thời chỉ +0,001 và p đã điều chỉnh là 0,0466. Tại `k=2` D=+0,028 và `k=4` D=+0,056; cả hai khoảng chứa 0.

Điểm này **không có nghĩa Qwen đạt điểm tuyệt đối cao hơn ở mọi tập**. Một phần D dương xuất hiện vì điểm Qwen trên Holdout ban đầu thấp hơn Rule. Trên OOD-2 ở `k=2`, macro-F1 Qwen là 0,520, vẫn thấp hơn Rule 0,545. Trên ba họ thực sự mới của OOD-2, chênh lệch Qwen trừ Rule âm và khoảng tin cậy chứa 0; lợi thế của Qwen tập trung ở hai họ tái sử dụng kỹ thuật đã có trong OOD-1. Phân tích nhóm này được thực hiện sau khi xem kết quả, số kịch bản ít, nên chỉ mang tính thăm dò.

#### RO3.2 — Nhầm lẫn và nhãn Noise

OOD-1 không có mẫu Noise thật. Vì vậy, bất kỳ cửa sổ nào bị gán Noise đều là lỗi. Rule gán 18–20 trong 57 cửa sổ thành Noise; Qwen gán 4–6; H4 gán 2–3.

OOD-2 có 27 cửa sổ Noise thật. Rule gán 83/145 cửa sổ thành Noise nhưng chỉ 12 trong số đó là Noise thật. Qwen gán 35–47 cửa sổ Noise, trong đó 7 là đúng; H4 gán 34–46, trong đó 7 là đúng. Riêng tại `k=2`, Rule gán nhầm 42 trong 52 cửa sổ Impact thành Noise; Qwen gán nhầm 31. Điều này cho thấy Rule có xu hướng mặc định Noise khi không khớp mẫu luật, còn mô hình ngôn ngữ có thể nhận ra một phần hành vi nhưng vẫn mắc lỗi nhầm giữa các giai đoạn.

Qwen không tạo ID bằng chứng giả và không có đầu ra JSON không hợp lệ trên hai tập chưa gặp. Đây là điểm tích cực về định dạng và tham chiếu ID trong thử nghiệm này, nhưng không đồng nghĩa nhãn phân loại luôn đúng.

#### RO3.3 — Độ mới của kỹ thuật

Nhóm chấm độ mới theo ba yếu tố: cấu trúc SQL, hàm/extension riêng PostgreSQL và chuỗi phụ thuộc. Mỗi yếu tố có mức 1–3. Với chỉ ba họ ở OOD-1 và sáu họ ở OOD-2, nhóm không tính tương quan thống kê hay khẳng định yếu tố nào gây suy giảm nhiều nhất.

Ví dụ, họ `definer_escalation` có điểm mới cao nhưng Rule lại đạt độ chính xác tốt nhất trong ba họ OOD-1; `ext_lateral` có độ chính xác thấp hơn. Ở OOD-2, `lockhog` có độ mới thấp nhất về chuỗi phụ thuộc nhưng lại là họ có độ chính xác thấp nhất. Các ví dụ này nhắc rằng điểm mới lạ đơn giản không giải thích hết độ khó; phân tích hiện tại chỉ mang tính mô tả.

## 5. Kết luận, giới hạn và thông điệp chính

Thứ nhất, C2 là cấu hình duy nhất ghi đủ 136 hành động và đạt tiêu chí recall nhóm đặt ra, nhưng cũng tạo nhiều log nhất. Độ bao phủ hành động cao chưa đủ để tái dựng phiên: trong mô phỏng một IP, F1 theo cặp chỉ khoảng 0,11.

Thứ hai, trên Holdout, Rule có điểm macro-F1 cao hơn Qwen ở thiết lập chính, 0,823 so với 0,769, nhưng khoảng tin cậy của chênh lệch chứa 0. Vì vậy, chưa đủ bằng chứng để tuyên bố phương pháp nào tốt hơn ở `k=2`. Rule có Jaccard bằng chứng cao hơn ở `k=2`; kết quả rà soát ground truth vẫn sơ bộ.

Thứ ba, cả Rule, Qwen và phương pháp lai đều giảm đáng kể trên các họ kỹ thuật chưa gặp. Qwen giảm ít hơn Rule ở một số so sánh, nhưng chưa có bằng chứng cho thấy Qwen vượt Rule trên ba họ thực sự mới. H4 có thể nâng một số điểm trên OOD-2 nhưng đánh đổi bằng nhận diện Noise kém hơn, nên chưa thể gọi là giải pháp tổng quát.

Các giới hạn cần nhấn mạnh khi thuyết trình: dữ liệu mô phỏng và mẫu nhỏ; kịch bản chạy từ một IP; ngữ cảnh phân loại phần lớn là ngữ cảnh lý tưởng; dữ liệu kiểm tra chưa hoàn toàn độc lập; khoảng tin cậy theo kịch bản không phản ánh độ biến thiên giữa các họ kỹ thuật; prompt được tinh chỉnh trên Dev; phần rà soát bằng chứng chỉ có một AI reviewer; và nhóm chưa đo CPU, bộ nhớ, độ trễ hay overhead ghi log.

**Thông điệp kết:** Trong môi trường mô phỏng này, ghi log đầy đủ là điều kiện cần nhưng chưa đủ. Phân loại truy vấn cần bằng chứng và ngữ cảnh đáng tin; còn hiệu năng trên kỹ thuật quen thuộc không đảm bảo khái quát hóa sang kỹ thuật mới. Muốn đưa kết quả gần với thực tế hơn, bước tiếp theo cần mô phỏng nhiều IP nguồn, tạo dữ liệu đa dạng hơn, đánh giá với người rà soát độc lập và đo chi phí vận hành.

## Câu hỏi thảo luận có thể gặp

**Vì sao C2 được chọn dù tạo nhiều log nhất?** Vì nó là cấu hình duy nhất đạt tiêu chí recall tối thiểu 95% tổng thể và từng giai đoạn trong workload đã thử. Nhóm chưa khẳng định C2 tối ưu về chi phí; chỉ số chi phí hiện có là số bản ghi.

**Qwen có tốt hơn Rule không?** Chưa thể kết luận ở thiết lập chính trên Holdout vì khoảng tin cậy của chênh lệch macro-F1 chứa 0. Qwen có một số kết quả ít suy giảm hơn trên tập chưa gặp, nhưng điều đó không đồng nghĩa điểm tuyệt đối cao hơn hay khái quát hóa tốt hơn trên các họ thực sự mới.

**Ngữ cảnh có giúp mô hình không?** Điểm Qwen quan sát được cao hơn khi có ngữ cảnh, nhưng chênh lệch ghép cặp chưa đủ bằng chứng. So sánh `k=0` còn đồng thời thay đổi số ví dụ few-shot.

**Vì sao kết quả tái dựng phiên yếu?** Mọi kịch bản cùng dùng một IP nguồn, nên IP không phân biệt được phiên. Điều này làm phép đo không đại diện tốt cho triển khai có nhiều nguồn tấn công.

**Có thể triển khai hệ thống này ngay không?** Báo cáo chưa chứng minh điều đó. Thử nghiệm chưa đo overhead, dữ liệu còn mô phỏng và đánh giá trên họ kỹ thuật mới còn hạn chế. Kết quả phù hợp để định hướng nghiên cứu tiếp theo hơn là làm tuyên bố sẵn sàng vận hành.
