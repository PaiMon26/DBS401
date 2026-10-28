# BÁO CÁO PROJECT DBS401

## Xây dựng honeypot cơ sở dữ liệu PostgreSQL kết hợp AI phân tích hành vi kẻ tấn công

**Nhóm 3**

| Thành viên | Mã sinh viên |
|---|---|
| Phan Thanh Thảo | SE193364 |
| Trần Thị Minh Anh | SE190902 |
| Vũ Trung Hiếu | SE192673 |

**Giảng viên:** Hồ Hải
**Môn học:** DBS401
**Địa điểm:** Thành phố Hồ Chí Minh
**Thời gian:** Tháng 10 năm 2026

---

## Tóm tắt

Các hệ quản trị cơ sở dữ liệu kết nối Internet hấp dẫn kẻ tấn công vì lưu trữ thông tin có giá trị và cung cấp giao diện truy vấn trực tiếp. Honeypot cơ sở dữ liệu tạo môi trường có kiểm soát để quan sát tương tác, nhưng giá trị của dữ liệu thu thập phụ thuộc vào cấu hình ghi log và khả năng tái dựng các truy vấn thành phiên tấn công mạch lạc. Project đề xuất một honeypot PostgreSQL chạy cục bộ, dùng dữ liệu tổng hợp trông như nhạy cảm và môi trường truy vấn tương tác có kiểm soát để nghiên cứu độ bao phủ của log và phân tích hành vi tự động. Các kịch bản tấn công và bình thường được tham số hóa để tạo chuỗi truy vấn có nhãn cùng quan hệ phụ thuộc; log từ bốn cấu hình được chuẩn hóa về một schema sự kiện chung. Nghiên cứu dự kiến so sánh độ bao phủ hành động, độ đầy đủ trường dữ liệu và khả năng tái dựng phiên theo các ngưỡng thời gian không hoạt động. Sau đó, bộ phân loại dựa trên luật được so sánh với mô hình ngôn ngữ lớn few-shot chạy cục bộ trên các cửa sổ truy vấn. Việc phân loại được đánh giá bằng macro-F1 và kết quả theo lớp; bằng chứng giải thích được so sánh với ground truth từ quan hệ phụ thuộc. Cuối cùng, hai phương pháp được đánh giá trên các họ kỹ thuật tấn công được giữ riêng khỏi quá trình phát triển. Thí nghiệm chưa được thực hiện; phần này mô tả thiết kế và phương pháp đánh giá dự kiến, không trình bày kết quả thực nghiệm.

**Từ khóa:** honeypot PostgreSQL; kiểm toán cơ sở dữ liệu; tái dựng phiên tấn công; phân tích hành vi; few-shot; phân loại dựa trên bằng chứng; khái quát hóa ngoài phân phối.

# Chương 1. Giới thiệu

## 1.1. Bối cảnh

Hệ quản trị cơ sở dữ liệu là một phần quan trọng trong lớp dữ liệu của ứng dụng hiện đại. Khi dịch vụ cơ sở dữ liệu được mở trên mạng công cộng, kẻ tấn công có thể dò tìm hệ thống và cấu hình, thử xác thực, truy vấn để khám phá hoặc thu thập dữ liệu, thay đổi trạng thái cơ sở dữ liệu, hoặc tận dụng khả năng của DBMS để tiếp cận tài nguyên khác. Những hoạt động này có thể tạo thành chuỗi nhiều bước, trong đó hành động sau phụ thuộc vào thông tin hoặc quyền truy cập thu được ở bước trước. Vì vậy, phân tích hành vi cần vừa thu thập sự kiện đáng tin cậy, vừa giữ được quan hệ giữa chúng.

Honeypot cơ sở dữ liệu cung cấp môi trường có kiểm soát để quan sát hoạt động hướng đến một dịch vụ mồi. Song, Smaragdakis và Griffioen triển khai 278 honeypot, trong đó có PostgreSQL; họ ghi nhận các nỗ lực brute-force kéo dài và các kiểu khai thác nhiều bước ở những triển khai có mức tương tác cao hơn [Song et al.]. Wegerer và Tjoa cũng trình bày khả năng xây dựng honeypot cơ sở dữ liệu bằng công nghệ MySQL mã nguồn mở [Wegerer & Tjoa]. Các công trình này cho thấy giá trị của deception tập trung vào DBMS và đặt ra yêu cầu đo lường cẩn thận khả năng quan sát của từng cách triển khai.

Log thô là nền tảng cho phân tích tiếp theo. Cấu hình chỉ ghi câu lệnh nhưng bỏ thông tin kết nối, lỗi hoặc tham số có thể khiến hồ sơ tương tác không đầy đủ. Ngược lại, ghi quá chi tiết có thể tăng dung lượng, chi phí xử lý và nguy cơ lộ dữ liệu nhạy cảm. Ngay cả khi sự kiện đã được ghi, vẫn cần xác định sự kiện nào thuộc cùng phiên logic và truy vấn nào là bằng chứng cho một giai đoạn tấn công. Do đó, thiết kế logging ảnh hưởng trực tiếp đến độ đầy đủ và độ tin cậy của phân tích hành vi.

Nghiên cứu gần đây dùng mô hình ngôn ngữ lớn để diễn giải cảnh báo an ninh và ánh xạ quan sát mức thấp sang hành vi tấn công mức cao. Schärmer và cộng sự cho thấy ví dụ few-shot được chọn cẩn thận cùng ngữ cảnh hệ thống có thể hỗ trợ phân loại cảnh báo IDS, nhưng chi phí, độ tin cậy và nguy cơ lộ dữ liệu vẫn là hạn chế [Schärmer et al.]. RHINO xem việc ánh xạ log mạng sang kỹ thuật đối phương là nhiệm vụ suy luận có hướng dẫn và kiểm tra bằng chứng để hạn chế ánh xạ thiếu căn cứ [Meng et al.]. Các kết quả này gợi ý cần kiểm tra phân loại theo ngữ cảnh và lựa chọn bằng chứng trên log truy vấn DB, thay vì mặc định nhãn do mô hình đưa ra tự nó đã là lời giải thích đáng tin cậy.

## 1.2. Phát biểu vấn đề

Trong bối cảnh PostgreSQL của project, có ba vấn đề thực tiễn. Thứ nhất, các cấu hình logging có thể khác nhau về khả năng ghi nhận hành động cần thiết để nhận diện bảy lớp hành vi do project quy định: Discovery, Credential Access, Privilege Escalation, Collection, Exfiltration, Impact và Noise. Project cần đo những khác biệt này và xác định bản ghi có đủ trường cho phân tích tiếp theo hay không. Thứ hai, các dòng log riêng lẻ không tự cho biết một phiên tấn công hoàn chỉnh. Gom phiên theo ngưỡng thời gian không hoạt động có thể ghép nhầm các hoạt động không liên quan hoặc tách rời một chuỗi nhiều bước; cần đối chiếu với ground truth của kịch bản. Thứ ba, phân loại truy vấn khó khi ý nghĩa phụ thuộc vào truy vấn trước. Luật có thể minh bạch nhưng cứng trước biến thể; LLM few-shot có thể tận dụng ngữ cảnh tốt hơn nhưng đưa ra nhãn hoặc mã bằng chứng thiếu căn cứ. Cần đo đồng thời chất lượng phân loại và chất lượng bằng chứng.

Một thách thức nữa là khả năng khái quát hóa. Phương pháp có thể hoạt động tốt với biến thể gần kịch bản phát triển nhưng thất bại trước họ kỹ thuật chưa dùng để viết luật hoặc ví dụ. Đánh giá cần chia dữ liệu theo họ kịch bản, cách ly họ giữ lại khỏi quá trình phát triển và đánh giá cả nhãn dự đoán lẫn truy vấn được dùng làm bằng chứng.

Vì vậy, project sẽ xây dựng pipeline đánh giá cục bộ, có thể tái lập, kết nối việc ghi log PostgreSQL, tái dựng phiên, phân loại truy vấn và đánh giá bằng chứng. Mục tiêu là đo mức phù hợp đầu-cuối của trace tương tác được thu thập, không khẳng định các kịch bản tổng hợp đại diện đầy đủ cho mọi cuộc tấn công thực tế.

## 1.3. Câu hỏi nghiên cứu và mục tiêu

Câu hỏi tổng quát: Honeypot PostgreSQL có mức tương tác cao kết hợp phân tích hành vi có thể ghi nhận, nhận diện và giải thích các giai đoạn tấn công từ log đến mức nào, kể cả khi đối phương dùng kỹ thuật chưa gặp?

### RQ1: Logging và tái dựng phiên

**RQ1.** Các cấu hình logging của honeypot PostgreSQL bỏ sót hoặc tái dựng sai loại hành động và phiên nào, ở mức độ nào, và cấu hình nào đủ cho phân tích hành vi tiếp theo?

- **RO1.1:** Đo action-level recall cho từng lớp hành vi tấn công và bình thường trên bốn cấu hình logging.
- **RO1.2:** Kiểm tra log có sáu thuộc tính phiên cần thiết hay không: user, IP nguồn, database, query, kết quả và tham số; đánh giá tái dựng phiên bằng precision, recall, F1 theo các ngưỡng không hoạt động 1, 5, 15 và 30 phút.
- **RO1.3:** Chọn cấu hình có chi phí thấp nhất nhưng đạt yêu cầu định trước về độ bao phủ, độ đầy đủ trường, dung lượng log và độ trễ truy vấn.

### RQ2: Phân loại và bằng chứng

**RQ2.** Giữa phương pháp dựa trên luật và LLM few-shot chạy cục bộ, phương pháp nào gán giai đoạn tấn công cho từng truy vấn chính xác hơn khi có ngữ cảnh truy vấn trước đó, và bằng chứng được chọn khớp ground truth đến đâu?

- **RO2.1:** So sánh macro-F1, F1 từng lớp, ma trận nhầm lẫn, tỷ lệ đầu ra sai schema và tỷ lệ mã bằng chứng giả.
- **RO2.2:** Đánh giá bằng chứng dự đoán so với ground truth sinh từ quan hệ phụ thuộc bằng precision, recall và F1; rà soát thủ công mẫu 20% đã chọn trước.
- **RO2.3:** So sánh cửa sổ gồm truy vấn hiện tại và 0, 2 hoặc 4 truy vấn trước đó; đo hiệu năng phân loại, chất lượng bằng chứng khi có ngữ cảnh và thời gian suy luận.

### RQ3: Khái quát hóa sang họ kỹ thuật chưa gặp

**RQ3.** Hiệu năng phân loại và khớp bằng chứng của hai phương pháp giảm bao nhiêu trên các họ kỹ thuật bị loại khỏi quá trình phát triển so với biến thể của họ đã biết?

- **RO3.1:** Đo thay đổi macro-F1 và evidence-F1 giữa tập họ đã biết và tập họ chưa gặp được cách ly.
- **RO3.2:** Phân tích nhầm lẫn theo lớp và thay đổi về bằng chứng không có căn cứ trên các họ chưa gặp.
- **RO3.3:** Xem xét mức độ mới về cấu trúc SQL, hàm/extension riêng PostgreSQL và độ phức tạp phụ thuộc giữa truy vấn liên hệ thế nào với mức suy giảm.

Nhãn tấn công là taxonomy của project, có liên hệ với các khái niệm MITRE ATT&CK nhưng không được trình bày như nguyên văn các tactic chính thức. Mỗi bước kịch bản có nhãn giai đoạn và liên kết phụ thuộc tường minh để tạo ground truth nhất quán.

## 1.4. Ý nghĩa của project

Về phương pháp, project đánh giá toàn bộ quá trình từ tương tác DB đến kết quả truy vấn có thể diễn giải. Project tách độ bao phủ log khỏi tái dựng phiên, đồng thời tách độ chính xác nhãn khỏi chất lượng bằng chứng. Nhờ đó có thể xác định lỗi bắt nguồn từ thiếu telemetry, gom phiên sai, phân loại giai đoạn hay chọn bằng chứng.

Về thực tiễn, phép so sánh giúp làm rõ đánh đổi giữa độ bao phủ, độ đầy đủ trường, dung lượng và độ trễ khi chọn cấu hình logging. Triển khai cục bộ với dữ liệu giả hỗ trợ thí nghiệm lặp lại mà không cần dịch vụ suy luận trả phí hoặc dữ liệu thật của tổ chức. Kết quả chỉ phản ánh môi trường thử nghiệm được kiểm soát, không phải tỷ lệ phát hiện trong mạng sản xuất.

## 1.5. Phạm vi và hạn chế

Project giới hạn ở honeypot PostgreSQL chạy cục bộ trong Docker, dùng dữ liệu giả và bảng mồi trông như chứa thông tin nhạy cảm. Môi trường không được thiết kế để mở dịch vụ không giới hạn ra Internet. Bộ sinh kịch bản Python có tham số sẽ thay đổi cấu trúc truy vấn, tên đối tượng, thứ tự, công cụ/client và độ lệch thời gian. Kịch bản lấy cảm hứng từ hành vi được tài liệu nghiên cứu mô tả; chuỗi truy vấn và quan hệ phụ thuộc cụ thể được ghi nhận như ground truth tổng hợp.

### Cấu hình logging

RQ1 so sánh bốn cấu hình, giá trị tham số chính thức sẽ được chốt trong phương pháp trước khi chạy kịch bản:

- **C0:** `log_statement = all`.
- **C1:** bật logging phiên của pgAudit.
- **C2:** mở rộng C1 với `log_connections`, `log_disconnections`, `log_min_error_statement` và `pgaudit.log_parameter`.
- **C3:** cấu hình proxy hoặc packet capture; chỉ báo cáo nếu triển khai kịp.

Log được chuẩn hóa về một schema chung. Sessionization dựa trên IP nguồn, user và ngưỡng thời gian không hoạt động. Các chỉ số gồm action recall, độ đầy đủ trường, chỉ số ghép phiên theo cặp, dung lượng log và độ trễ truy vấn.

### Taxonomy và gán nhãn

Bảy nhãn gồm sáu giai đoạn tấn công và một lớp bình thường là Noise. `COPY ... FROM PROGRAM` được gán Impact vì nó làm thay đổi trạng thái hệ thống bằng cách chạy lệnh hệ điều hành. Quy tắc cho các hàm truy cập file và large object như `pg_read_file` và `lo_export` sẽ được chốt trước khi sinh kịch bản; trường hợp mơ hồ được ghi lại trong tài liệu ground truth.

### Phân loại và bằng chứng

RQ2 so sánh bộ phân loại luật với LLM cục bộ khoảng 7–8 tỷ tham số qua Ollama. Các mô hình ứng viên gồm Qwen2.5-7B-Instruct, Llama-3.1-8B-Instruct và Mistral-7B-Instruct; lựa chọn cuối dựa trên thử tốc độ khoảng 20 cửa sổ. Cả hai phương pháp nhận cửa sổ truy vấn đã chuẩn hóa. LLM trả JSON gồm nhãn và danh sách mã bằng chứng. Schema giới hạn định dạng nhưng không chặn mã ngoài cửa sổ trong lúc giải mã, để có thể đo tỷ lệ mã giả. Tập test được giữ kín cho đến khi chốt phương pháp. Ground truth bằng chứng gồm truy vấn mục tiêu và các phụ thuộc được khai báo nằm trong cửa sổ.

### Hạn chế

Kịch bản hữu hạn và tổng hợp; chất lượng ground truth phụ thuộc thiết kế kịch bản; môi trường mô phỏng cục bộ có thể khác DB sản xuất. Một vài họ giữ lại không đại diện mọi tấn công chưa biết. Đầu ra mô hình có thể khác theo phần cứng, phiên bản và thiết lập suy luận. Rà soát thủ công chỉ thực hiện trên mẫu định trước.

## 1.6. Cấu trúc báo cáo

Báo cáo trình bày vấn đề, mục tiêu, ý nghĩa và phạm vi nghiên cứu; tổng quan tài liệu; phương pháp và kế hoạch đánh giá. Các phần kết quả, thảo luận và kết luận sẽ được hoàn thiện khi project thực hiện các giai đoạn tương ứng.

# Chương 2. Tổng quan tài liệu

## 2.1. Phạm vi tổng quan

Chương này xem xét các nghiên cứu về honeypot cơ sở dữ liệu và hành vi đối phương, audit logging và tái dựng tấn công, diễn giải dữ liệu an ninh bằng ML/LLM, khái quát hóa sang hành vi chưa gặp và đánh giá tính chân thực của honeypot. Các nghiên cứu khác nhau về nền tảng, mức tương tác, loại dữ liệu và điều kiện đánh giá; vì vậy, kết quả dùng để định hướng thiết kế chứ không dự đoán trực tiếp hiệu năng trên trace PostgreSQL tổng hợp của project.

## 2.2. Honeypot cơ sở dữ liệu và hành vi tấn công

Song, Smaragdakis và Griffioen triển khai 278 honeypot trong 20 ngày, trong đó 220 honeypot tương tác thấp mô phỏng MySQL, MSSQL, PostgreSQL và Redis; 58 honeypot tương tác trung bình/cao cho phép quan sát khai thác sâu hơn [Song et al.]. Nhóm nghiên cứu ghi nhận dò quét, brute-force kéo dài, thao tác dữ liệu, sao chép/xóa dữ liệu và việc dùng DBMS làm đường tiếp cận máy chủ nền. Họ cũng chuẩn hóa log nhiều định dạng về dạng chung để phân tích. Đây là nghiên cứu thực nghiệm gần đề tài nhất vì có PostgreSQL và hành vi nhiều bước. Tuy nhiên, triển khai Internet-facing khác với thí nghiệm cục bộ, nên tần suất quan sát không thể xem là kết quả kỳ vọng của project.

Rocha và Pereira dùng honeypot để khảo sát tấn công vào DBMS quan hệ, báo cáo MySQL là hệ thống bị nhắm nhiều nhất trong số các nền tảng được nghiên cứu [Rocha & Pereira]. Kết quả nhấn mạnh giá trị của quan sát chuyên biệt theo DBMS; mức độ phổ biến của mục tiêu chỉ đúng trong phạm vi triển khai của bài báo.

Wegerer và Tjoa trình bày khái niệm honeypot cơ sở dữ liệu, honeytoken và triển khai MySQL bằng công cụ mã nguồn mở [Wegerer & Tjoa]. Công trình cho thấy audit DB là yếu tố thiết kế quan trọng, nhưng plugin cụ thể cho MySQL không thể áp dụng nguyên trạng cho PostgreSQL.

Các nghiên cứu mới hơn hướng đến deception thích ứng. Mirage là môi trường mô phỏng/emulation để thử deception trước đối thủ tự động dùng tìm kiếm và lập kế hoạch [Kouremetis et al.]. D3O-IIoT dùng học tăng cường sâu để phối hợp honeypot, moving-target defense và telemetry giả trong IIoT [Wushishi et al.]. Đây là bối cảnh cho hướng phát triển tương lai, nhưng khác phạm vi honeypot truy vấn DB của project.

## 2.3. Audit logging và tái dựng kịch bản tấn công

Zimniak, Getta và Benn phân tích audit trail để khám phá mẫu tuần hoàn trong workload cơ sở dữ liệu [Zimniak et al.]. Dù mục tiêu là dự báo workload chứ không phát hiện tấn công, nghiên cứu cho thấy cần giữ thứ tự và thời điểm của sự kiện khi phân tích chuỗi thao tác.

Biswas và Niemi chỉ ra rằng log không đủ sẽ cản trở audit, còn ghi dư tạo chi phí và overhead [Biswas & Niemi]. Điều này hỗ trợ việc đo đồng thời độ bao phủ, độ đầy đủ trường, dung lượng và độ trễ ở RQ1. Zeng, Xiao và Chen cũng đánh giá overhead và khả năng điều chỉnh auditing trên Linux, củng cố việc xem chi phí instrumentation là một phần của đánh giá [Zeng et al.].

SLEUTH tái dựng kịch bản tấn công từ audit data bằng đồ thị phụ thuộc và phân tích nguồn/ảnh hưởng [Hossain et al.]. TraceBack khai thác provenance graph, quan hệ nhân quả và thứ tự để tái dựng kịch bản [Zhu et al.]. Các hệ thống này không tập trung vào log truy vấn DB, nhưng hỗ trợ ý tưởng xem sự kiện như các bước có phụ thuộc. Project dùng liên kết phụ thuộc trong kịch bản làm ground truth để đánh giá tái dựng phiên và lựa chọn bằng chứng.

Choi và Lee phân tích transaction log của SQL Server trong vùng chưa cấp phát của hệ thống file để phục vụ pháp chứng [Choi & Lee]. Kết quả của họ nhắc rằng quan hệ giữa câu truy vấn và bản ghi log không nhất thiết là một-một; parser không nên giả định mọi truy vấn sinh đúng một bản ghi. Tuy nhiên, nghiên cứu tập trung SQL Server, không trực tiếp mô tả logging PostgreSQL.

## 2.4. IDS, học máy và diễn giải bằng LLM

Liu và Lang tổng quan ML/DL cho IDS, gồm thuật toán, bộ dữ liệu và vấn đề như false alarm, độ chính xác và phát hiện tấn công mới [Liu & Lang]. Alharthi, Alaryani và Kaddoura so sánh các mô hình trên bộ dữ liệu nhị phân và đa lớp, cho thấy mô hình tốt nhất phụ thuộc dữ liệu và nhiệm vụ [Alharthi et al.]. Hai nghiên cứu hỗ trợ chọn macro-F1, kết quả theo lớp và baseline minh bạch; các điểm số từ dữ liệu mạng/ICS không thể chuyển thẳng sang log SQL.

Rehman và cộng sự chuyển flow mạng thành văn bản ngắn, thử zero-shot/few-shot, ràng buộc định dạng đầu ra và hiệu chỉnh ngưỡng [Rehman et al.]. Kết quả nhạy với prompt, đặc trưng và quy mô tập đánh giá; baseline dạng bảng ổn định hơn trong thí nghiệm. Mehavilla và cộng sự so sánh LLM với ML/DL trên Zeek logs, báo cáo XGBoost đạt 96,96% F1 và dùng ít tài nguyên CPU hơn các LLM được thử [Mehavilla et al.]. Điều này nhắc rằng LLM không mặc nhiên là lựa chọn tốt hơn cho phân loại dữ liệu có cấu trúc.

Schärmer và cộng sự đánh giá LLM diễn giải cảnh báo IDS, phân biệt cảnh báo tấn công với false positive và ánh xạ kỹ thuật tấn công [Schärmer et al.]. Họ xem xét tác động của log, cấu hình hệ thống và ví dụ few-shot. RHINO dùng suy luận có hướng dẫn để ánh xạ log mạng sang tactic/technique và nhấn mạnh kiểm chứng bằng chứng [Meng et al.]. CYBERTEAM chuẩn hóa threat hunting thành các tác vụ và mô-đun vận hành, báo cáo cấu trúc hóa quy trình giúp cải thiện so với một số kiểu suy luận mở [Meng et al.]. Những nghiên cứu này gợi ý kiểm tra xem ngữ cảnh có cải thiện phân loại truy vấn và liệu bằng chứng được chọn có thực sự được log hỗ trợ.

APTREC nhận diện tactic/technique từ đồ thị tấn công theo kiến trúc hai giai đoạn; bài báo báo cáo cải thiện F1 và recall so với các phương pháp đối chiếu [Yang et al.]. Wang và cộng sự khảo sát nhận diện đa nhãn zero-shot với khám phá bằng chứng ngữ nghĩa [Wang et al.]. Đây là các hướng liên quan đến việc nối nhãn với bằng chứng, nhưng đầu vào đồ thị khác cửa sổ SQL của project.

Neves và cộng sự so sánh LLM zero-shot với phương pháp dựa trên luật cho phân loại kiểu hình lâm sàng từ tóm tắt bệnh nhân tổng hợp [Neves et al.]. Nghiên cứu cho thấy hai phương pháp có thể có điểm mạnh bổ sung: luật có precision cao nhưng recall thấp, còn LLM đạt cân bằng khác. Đây là phép so sánh gần về thiết kế với RQ2, nhưng lĩnh vực lâm sàng khác an ninh DB.

Duan và cộng sự tổng quan few-shot learning cho phát hiện xâm nhập trong bối cảnh dữ liệu có nhãn hạn chế [Duan et al.]. Project dùng few-shot prompting với mô hình cố định, không đề xuất thuật toán few-shot learning mới; vì vậy tài liệu này là nền tảng khái niệm.

## 2.5. Khái quát hóa sang hành vi tấn công chưa gặp

Lust và Condurache tổng quan các hướng đánh giá khả năng khái quát hóa mạng nơ-ron, gồm bất định dự đoán, phát hiện ngoài phân phối và ví dụ đối kháng [Lust & Condurache]. Zhou và Wang đề xuất GROD để cải thiện khả năng khái quát hóa ngoài phân phối của transformer bằng outlier tổng hợp [Zhou & Wang]. GROD hướng đến huấn luyện mạng, không áp dụng trực tiếp cho LLM prompting, nhưng cho thấy cần đo riêng hiệu năng trên dữ liệu lệch phân phối.

Das và Nair công bố bộ dữ liệu xâm nhập nhà thông minh với bảy kịch bản tấn công nhiều giai đoạn [Das & Nair]. Dù thuộc dữ liệu mạng, nghiên cứu minh họa nhu cầu có kịch bản và nhãn nhiều bước. Project sẽ tự tạo các kịch bản truy vấn có quan hệ phụ thuộc và mô tả cách sinh để có ground truth tái lập.

## 2.6. Tính chân thực và đánh giá honeypot

Nunez Migliorisi đề xuất đo realism và hiệu quả deception qua thời gian đối phương lưu lại, độ sâu tương tác, độ đa dạng hành vi và độ phong phú malware; số lượng kết nối đơn thuần chưa đủ [Nunez Migliorisi]. Ferguson-Walter và Major đánh giá deception qua tiến triển của đối phương, tài nguyên tiêu tốn và thay đổi nhận thức [Ferguson-Walter & Major]. Các chỉ số này có thể bổ sung cho việc đánh giá honeypot, dù project tập trung chính vào khả năng quan sát log và chất lượng phân tích trong kịch bản kiểm soát.

PHANTOM tạo honeytoken theo ngữ cảnh tổ chức và đánh giá tính đáng tin qua hợp lệ cú pháp, nhất quán ngữ nghĩa, hợp lý thống kê và chấp nhận của người đánh giá [Weinberg]. Honeysweeper cho thấy honeytoken phổ biến có thể bị fingerprint mà không kích hoạt cảnh báo [Msaad et al.]. Các nghiên cứu này gợi ý cần kiểm tra tính hợp lý của dữ liệu mồi, nhưng project không tuyên bố dữ liệu giả không thể bị nhận diện.

Bridges và cộng sự tổng hợp nghiên cứu honeypot kết hợp LLM, gồm vector fingerprint, kiến trúc, phân tích log và tiêu chí đánh giá như tính đáng tin, fidelity, mức tương tác và chi phí bên phòng thủ [Bridges et al.]. Họ nhấn mạnh thiếu dữ liệu thực tế có nhãn và nguy cơ đối thủ thông minh nhận ra phản hồi LLM. Project dùng LLM để phân tích trace chứ không sinh phản hồi PostgreSQL, nên fidelity phản hồi không phải chỉ số chính; tuy vậy cần đánh giá đầu ra tường minh và không xem lời giải thích của LLM là bằng chứng tự xác thực.

## 2.7. Tổng hợp tài liệu

Các nghiên cứu honeypot DB cho thấy decoy chuyên biệt có thể quan sát cả nỗ lực xác thực kéo dài lẫn khai thác nhiều bước; mức tương tác và cơ chế ghi log quyết định loại hành vi có thể thu được. Nghiên cứu audit trail và tái dựng tấn công nhấn mạnh thứ tự, thời gian, phụ thuộc và độ đầy đủ trường, đồng thời lưu ý overhead của logging. Nghiên cứu IDS/LLM cung cấp baseline, cách dùng ngữ cảnh, đầu ra có cấu trúc và tiêu chí đánh giá; nhưng kết quả trên flow mạng, cảnh báo hoặc văn bản lâm sàng chưa chứng minh hiệu năng trên truy vấn SQL. Đánh giá honeypot cũng cần xem xét độ sâu tương tác và realism thay vì chỉ đếm kết nối.

Khoảng trống nằm ở giao điểm các hướng này: tài liệu hiện có chưa xác định cấu hình logging PostgreSQL nào đủ để tái dựng phiên truy vấn có nhãn trong honeypot kiểm soát, cũng chưa cho biết LLM few-shot cục bộ có cải thiện phân loại giai đoạn và lựa chọn bằng chứng so với luật khi kiểm tra trên các họ kỹ thuật bị giữ lại hay không. Project lấp khoảng trống này bằng một pipeline thống nhất đánh giá độ bao phủ, tái dựng phiên, phân loại, chất lượng bằng chứng và khái quát hóa. Kết luận chỉ áp dụng cho cấu hình và kịch bản được thử nghiệm.

## 2.8. Đóng góp của nghiên cứu

Các đóng góp dự kiến:

1. So sánh bốn cấu hình logging PostgreSQL theo action-level recall, độ đầy đủ trường, dung lượng log và độ trễ truy vấn.
2. Đánh giá tái dựng phiên theo nhiều ngưỡng không hoạt động, đối chiếu với kịch bản có nhãn và phụ thuộc tường minh.
3. So sánh bộ phân loại dựa trên luật với LLM few-shot cục bộ trên cùng cửa sổ truy vấn; báo cáo macro-F1, hiệu năng theo lớp, đầu ra không hợp lệ và precision/recall/F1 của bằng chứng.
4. Kiểm tra khái quát hóa trên các họ kỹ thuật tấn công được loại khỏi quá trình xây luật và prompt.

Đây là đóng góp dự kiến của thiết kế và triển khai nghiên cứu, chưa phải kết quả thực nghiệm. Mục tiêu là xác định các đánh đổi quan sát được trong môi trường kiểm soát và tạo nền tảng cho mở rộng sau này.

# Chương 3. Phương pháp

## 3.1. Thiết kế nghiên cứu

Nghiên cứu là đánh giá định lượng trong môi trường kiểm soát, gồm ba thí nghiệm tương ứng RQ1–RQ3. RQ1 đo khả năng ghi nhận hành động và tái dựng phiên của bốn cấu hình PostgreSQL. RQ2 so sánh luật với LLM few-shot cục bộ về phân loại giai đoạn truy vấn và chọn bằng chứng. RQ3 đo mức suy giảm hiệu năng trên các họ kỹ thuật bị loại khỏi phát triển.

Pipeline chung gồm sáu bước: (1) bộ sinh kịch bản tạo chuỗi truy vấn tấn công/bình thường cùng nhãn và phụ thuộc; (2) chạy kịch bản trên honeypot cục bộ theo cấu hình logging; (3) phân tích log vào schema sự kiện chung; (4) nhóm phiên cho RQ1; (5) tạo cửa sổ truy vấn và phân loại cho RQ2/RQ3; (6) so sánh dự đoán với ground truth bằng các metric đã định nghĩa.

## 3.2. Môi trường honeypot và cấu hình logging

Honeypot là PostgreSQL chạy trong Docker trên máy cục bộ, không mở ra mạng công cộng. Cơ sở dữ liệu chứa dữ liệu giả, gồm bảng mồi có tên và nội dung giống các bản ghi nhạy cảm như tài khoản và thanh toán. Không dùng dữ liệu cá nhân/tổ chức thật.

Bốn cấu hình logging được so sánh: C0 ghi mọi câu lệnh; C1 bật pgAudit session logging; C2 bổ sung log kết nối, ngắt kết nối, lỗi câu lệnh và tham số; C3 dùng proxy hoặc packet capture nếu kịp triển khai. Bản ghi được chuẩn hóa về timestamp, user, IP nguồn, database, câu lệnh, trạng thái kết quả và tham số. Trường không được cấu hình ghi nhận sẽ để thiếu, không suy diễn.

## 3.3. Kịch bản, dữ liệu và gán nhãn

Bộ sinh Python tạo phiên tấn công và phiên bình thường từ các template. Mỗi bước có nhãn, mã định danh và danh sách `depends_on` chỉ các bước trước có kết quả được sử dụng. Quan hệ này tạo ground truth bằng chứng. Bộ sinh thay đổi tên bảng/cột, thứ tự khi phù hợp, công cụ/client, giá trị literal và thời gian giữa truy vấn. ID bước chỉ lưu trong ground truth, không chèn vào SQL hay đầu vào mô hình.

Bảy nhãn gồm Discovery, Credential Access, Privilege Escalation, Collection, Exfiltration, Impact và Noise. Ví dụ: truy vấn catalog là Discovery; đọc bảng credential là Credential Access; `GRANT`/`ALTER USER` là Privilege Escalation; đọc dữ liệu nhạy cảm hàng loạt là Collection; `COPY ... TO` là Exfiltration; `DROP`, `UPDATE`, `DELETE` và `COPY ... FROM PROGRAM` là Impact; truy vấn ứng dụng bình thường là Noise. Quy tắc gán nhãn được chốt trước khi sinh dữ liệu; tình huống mơ hồ được ghi lại.

Các kịch bản chia thành họ theo kỹ thuật và mục tiêu. Họ đã biết dùng để phát triển luật và prompt. Hai hoặc ba họ giữ lại do thành viên không viết luật/prompt tạo, được cố định trước khi phát triển. Phiên Noise được trộn vào các thí nghiệm.

## 3.4. Thu thập dữ liệu

### Dữ liệu RQ1

Mỗi biến thể kịch bản chạy dưới cả bốn cấu hình để so sánh trực tiếp. Runner phía client ghi câu lệnh, thời điểm gửi, ID kết nối và ID bước làm ground truth đối chiếu. Parser chuyển log vào schema chung. Overhead đo bằng số byte log trên phiên và độ trễ mỗi câu lệnh, có/không bật từng cấu hình, cùng phần cứng. Mỗi phép đo lặp lại; báo cáo median và khoảng tứ phân vị.

### Dữ liệu RQ2/RQ3

Tập phân loại gồm 100 truy vấn có nhãn từ các họ đã biết và Noise, thu dưới cấu hình C2 để có đủ sáu thuộc tính. Nếu RQ1 chọn cấu hình nhẹ hơn, RQ2 được chạy lại như kiểm tra độ nhạy. Trước phân loại, literal được thay bằng placeholder kiểu dữ liệu như `<STR>` và `<NUM>`; tên bảng/cột được đổi riêng theo từng kịch bản để tránh mô hình học thuộc tên. SQL gốc giữ trong ground truth; mô hình nhận phiên bản chuẩn hóa.

## 3.5. Chia tập dữ liệu

Dữ liệu được chia bằng `StratifiedGroupKFold` của scikit-learn với 3 fold, shuffle bật và seed 42. Nhóm là phiên hoặc ID kịch bản để mọi truy vấn trong một phiên nằm cùng fold; nhãn được dùng để giữ phân bố lớp gần cân bằng. Một fold khoảng một phần ba làm test kín; phần còn lại dùng phát triển/huấn luyện và một grouped split khác phục vụ tinh chỉnh.

Với 100 truy vấn, test có khoảng 33 truy vấn cho bảy lớp, nên một số lớp sẽ có rất ít ví dụ. Tỷ lệ thực tế khoảng 67% phát triển và 33% test, thay vì 70/30 ban đầu; kết quả theo lớp cần được diễn giải thận trọng.

## 3.6. Kỹ thuật phân tích

### RQ1: độ bao phủ và tái dựng phiên

Action recall cho lớp `g` và profile `p` được tính bằng số hành động trong lớp được phát hiện chia cho tổng số hành động lớp đó. Hành động chỉ tính là phát hiện nếu bản ghi khớp câu lệnh đã chạy sau chuẩn hóa và có đủ thuộc tính bắt buộc. Ví dụ, đăng nhập thất bại cần user, IP nguồn và trạng thái kết quả.

Độ đầy đủ trường là tỷ lệ bản ghi có đủ sáu thuộc tính phiên. Sự kiện được nhóm theo IP nguồn, user và database; phiên mới bắt đầu khi khoảng cách giữa hai sự kiện vượt ngưỡng 1, 5, 15 hoặc 30 phút. Phiên khớp chính xác nếu chứa đúng tập hành động của phiên thật. Báo cáo session precision/recall/F1 và pairwise metrics để phản ánh lỗi gộp/tách phiên.

Profile chỉ đủ điều kiện nếu đạt recall ít nhất 95% cho mọi lớp và đủ sáu thuộc tính phục vụ RQ2. Trong các profile đủ điều kiện, chọn profile có tổng chi phí dung lượng log và độ trễ thấp nhất. Nếu không profile nào đạt, chọn phương án tốt nhất hiện có và báo cáo thiếu hụt.

### Bộ phân loại dựa trên luật

Luật gán nhãn từng truy vấn bằng điều kiện trên cửa sổ truy vấn, có thể tham chiếu truy vấn hiện tại và truy vấn trước. Luật chỉ được tinh chỉnh trên fold phát triển. Đầu ra gồm nhãn dự đoán và ID truy vấn thỏa điều kiện; các ID là tập bằng chứng của luật.

### Bộ phân loại LLM few-shot

Mô hình dự kiến là Llama 3.1 8B Instruct chạy cục bộ qua Ollama, không fine-tune; nhiệt độ bằng 0. Ghi lại phiên bản model/Ollama và phần cứng. Prompt gồm vai trò, tập nhãn hợp lệ, schema JSON, 2–3 ví dụ few-shot chỉ lấy từ fold phát triển và cửa sổ truy vấn hiện tại cộng `k` truy vấn trước (`k` bằng 0, 2 hoặc 4). Đầu ra sai JSON hoặc nhãn ngoài danh sách được tính là không hợp lệ và là dự đoán sai; không retry.

### Chỉ số đánh giá

Phân loại báo cáo macro-F1, precision/recall/F1 theo lớp và ma trận nhầm lẫn. Độ tin cậy kỹ thuật gồm tỷ lệ đầu ra sai và tỷ lệ ID bằng chứng giả. Chất lượng bằng chứng đối chiếu tập ID dự đoán với tập ground truth gồm truy vấn đích và phụ thuộc nằm trong cửa sổ; báo cáo Jaccard, precision, recall, F1. Mẫu 20% được rà soát thủ công để ước lượng chất lượng ground truth. Hiệu năng theo `k` và thời gian suy luận cũng được ghi nhận.

Khoảng tin cậy macro-F1/evidence-F1 được bootstrap theo phiên vì truy vấn trong cùng phiên phụ thuộc nhau. Hai phương pháp so sánh trên cùng test fold.

### RQ3: họ kỹ thuật chưa gặp

Mức suy giảm hiệu năng là hiệu năng trên họ đã biết trừ hiệu năng trên họ giữ lại, tính cho macro-F1 và evidence-F1. Báo cáo thêm F1 theo lớp, thay đổi tỷ lệ bằng chứng giả và mô tả độ mới cấu trúc SQL, mức chuyên biệt PostgreSQL và độ phức tạp phụ thuộc. Chỉ có 2–3 họ giữ lại nên quan hệ giữa các yếu tố mới và suy giảm chỉ mang tính mô tả.

## 3.7. Hạn chế của phương pháp

Tập dữ liệu nhỏ (100 truy vấn), khiến ước lượng theo lớp không ổn định. Hành vi tấn công tổng hợp phản ánh template của nhóm chứ không phải phân phối tấn công thực. Bảy nhãn không bao quát mọi hành vi PostgreSQL; ánh xạ trường hợp biên có thể ảnh hưởng kết quả. Hai hoặc ba họ giữ lại không đại diện mọi kỹ thuật chưa biết. LLM có thể không hoàn toàn tất định do phần cứng/thư viện; độ trễ và dung lượng phụ thuộc workload và máy chạy. Chấm novelty và rà soát bằng chứng có yếu tố chủ quan; báo cáo độ đồng thuận giữa người đánh giá.

## 3.8. Khả năng tái lập và cân nhắc đạo đức

Mã bộ sinh, parser, prompt, định nghĩa luật, seed ngẫu nhiên và chỉ số chia tập sẽ lưu trong repository có version control để tái tạo bảng/hình. Honeypot chỉ chứa dữ liệu giả, không mở ra mạng ngoài và không chạy malware thật. Bộ sinh chỉ tạo câu lệnh SQL; không sử dụng hạ tầng tấn công trực tiếp.

---

## Tài liệu tham khảo được dùng trong bản dịch

Các trích dẫn trong bản tiếng Việt là dạng rút gọn để dễ đọc; khóa và thông tin thư mục đầy đủ xem trong `ref.bib` và phần References của báo cáo LaTeX.
