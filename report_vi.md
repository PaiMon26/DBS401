# BÁO CÁO PROJECT MÔN DBS401

## Xây dựng Honeypot Cơ sở dữ liệu PostgreSQL kết hợp AI Phân tích Hành vi Kẻ tấn công

**Nhóm 3**

| Thành viên | Mã sinh viên |
|---|---|
| Phan Thanh Thảo | SE193364 |
| Trần Thị Minh Anh | SE190902 |
| Vũ Trung Hiếu | SE192673 |

**Giảng viên hướng dẫn:** Hồ Hải  
**Môn học:** DBS401  
**Địa điểm:** Thành phố Hồ Chí Minh  
**Thời gian:** Tháng 10 năm 2026  

---

## Tóm tắt

Các hệ quản trị cơ sở dữ liệu (DBMS) kết nối Internet là mục tiêu tấn công hấp dẫn vì chúng lưu trữ dữ liệu nhạy cảm có giá trị cao và cung cấp giao diện truy vấn trực tiếp. Honeypot cơ sở dữ liệu tạo ra một môi trường có kiểm soát để quan sát các tương tác này; tuy nhiên, giá trị của dữ liệu thu thập được phụ thuộc chặt chẽ vào cấu hình ghi log và khả năng tái dựng các câu truy vấn rời rạc thành các phiên tấn công mạch lạc. Đề tài này đề xuất xây dựng một honeypot PostgreSQL chạy cục bộ, chứa dữ liệu giả lập có độ nhạy cảm cao và môi trường truy vấn tương tác có kiểm soát nhằm nghiên cứu độ bao phủ của log và phân tích hành vi tự động. Các kịch bản tấn công và truy vấn bình thường được tham số hóa sẽ tạo ra các chuỗi truy vấn có nhãn kèm mối quan hệ phụ thuộc tường minh; dữ liệu log thu được từ bốn cấu hình định trước sẽ được phân tích cú pháp và chuẩn hóa về một schema sự kiện chung. 

Nghiên cứu trước hết so sánh độ bao phủ hành động (recall), mức độ đầy đủ của các trường sự kiện và hiệu quả tái dựng phiên qua các ngưỡng thời gian không hoạt động. Tiếp theo, đề tài so sánh bộ phân loại dựa trên luật (rule-based) với mô hình ngôn ngữ lớn (LLM) few-shot chạy cục bộ trên các cửa sổ truy vấn trượt gồm truy vấn hiện tại và các truy vấn trước đó. Hiệu năng phân loại được đánh giá bằng macro-F1 và kết quả theo từng lớp; phần giải thích được đánh giá thông qua độ chính xác (precision), độ bao phủ (recall) và F1 của bằng chứng so với ground truth sinh từ quan hệ phụ thuộc. Cuối cùng, cả hai phương pháp được đánh giá nguyên trạng (không qua tinh chỉnh) trên các họ kỹ thuật tấn công được cách ly hoàn toàn khỏi quá trình phát triển. Kết quả dự kiến của đề tài là một đánh giá thực nghiệm về sự đánh đổi giữa các cấu hình log, một pipeline đánh giá cấp truy vấn có thể tái lập, và bằng chứng thực nghiệm về độ vững chắc của phân loại và giải thích trước các biến thể kỹ thuật. Vì các thí nghiệm chưa được tiến hành, bản tóm tắt này mô tả thiết kế đề tài và phương pháp đánh giá dự kiến chứ không trình bày các kết quả thực nghiệm.

**Từ khóa:** Honeypot PostgreSQL, kiểm toán cơ sở dữ liệu, tái dựng phiên tấn công, phân tích hành vi, học few-shot, phân loại dựa trên bằng chứng, khái quát hóa ngoài phân phối (OOD).

---

# Chương 1. Giới thiệu

## 1.1. Bối cảnh

Hệ quản trị cơ sở dữ liệu là một thành phần trọng yếu trong lớp lưu trữ dữ liệu của các ứng dụng hiện đại. Khi một dịch vụ cơ sở dữ liệu bị phơi bày trên mạng công cộng, kẻ tấn công có thể thăm dò danh tính và cấu hình hệ thống, thử xác thực đăng nhập, gửi các truy vấn để khám phá hoặc thu thập dữ liệu, thay đổi trạng thái cơ sở dữ liệu, hoặc khai thác các tính năng của DBMS để tiếp cận các tài nguyên hệ thống khác. Khác với một câu truy vấn đơn lẻ, các hành động này thường tạo thành một chuỗi kế tiếp mà trong đó các bước sau phụ thuộc vào thông tin hoặc quyền hạn thu được từ các bước trước. Do đó, việc thấu hiểu hành vi tấn công đòi hỏi vừa phải thu thập sự kiện một cách tin cậy, vừa phải phân tích và bảo toàn được mối quan hệ logic giữa các sự kiện.

Honeypot cơ sở dữ liệu cung cấp một phương thức có kiểm soát để quan sát các hoạt động nhắm vào dịch vụ mồi nhử. Song, Smaragdakis và Griffioen đã triển khai 278 honeypot trong 20 ngày, bao gồm các mô phỏng tương tác thấp của PostgreSQL và các hệ CSDL khác; nghiên cứu ghi nhận các cuộc tấn công brute-force dai dẳng cùng với các hành vi khai thác tương tác trung bình và cao nhằm thao túng dữ liệu, sao chép, xóa dữ liệu hoặc lợi dụng DBMS để xâm nhập hệ thống bên dưới [song_decoy_2025]. Các nghiên cứu trước đó cũng đã chứng minh tính khả thi của việc xây dựng một honeypot cơ sở dữ liệu tương tác bằng MySQL mã nguồn mở kết hợp với các honeytoken để đánh lừa [wegerer_defeating_2016]. Những công trình này khẳng định giá trị thực tiễn của cơ chế đánh lừa hướng CSDL, đồng thời đặt ra yêu cầu phải đo lường cẩn trọng khả năng ghi nhận của từng cấu hình triển khai cụ thể.

Dấu vết log thô là nền tảng cho mọi phân tích phía sau. Một cấu hình ghi log chỉ ghi lại câu lệnh (statement) nhưng bỏ sót thông tin kết nối, thông báo lỗi hoặc tham số truy vấn có thể tạo ra các lỗ hổng thông tin trong chuỗi tương tác. Ngược lại, việc ghi log quá chi tiết sẽ làm tăng dung lượng lưu trữ đĩa và tăng độ trễ xử lý truy vấn, đồng thời có thể để lộ các giá trị nhạy cảm nếu dùng dữ liệu thật. Ngay cả khi từng sự kiện riêng lẻ được ghi nhận đầy đủ, các nhà phân tích vẫn phải giải quyết bài toán: những sự kiện nào thuộc về cùng một phiên logic, và những câu truy vấn nào thực sự là bằng chứng cho một giai đoạn tấn công cụ thể. Do đó, cấu hình ghi log không chỉ là một thiết lập vận hành thuần túy; nó trực tiếp quyết định độ đầy đủ và độ tin cậy của toàn bộ quá trình phân tích hành vi tiếp theo.

Các nghiên cứu gần đây cũng đã bắt đầu khám phá tiềm năng của mô hình ngôn ngữ lớn (LLM) trong việc diễn giải cảnh báo an ninh và ánh xạ các quan sát mức thấp sang hành vi tấn công mức cao. Schärmer et al. đánh giá ChatGPT và Gemini trong phân loại cảnh báo IDS, chỉ ra rằng các ví dụ few-shot và thông tin ngữ cảnh hệ thống được lựa chọn kỹ lưỡng sẽ cải thiện đáng kể hiệu năng phân loại, đồng thời nhấn mạnh các hạn chế về chi phí, độ ổn định và rủi ro rò rỉ dữ liệu [afzal_let_2027]. Meng et al. giới thiệu framework RHINO nhằm ánh xạ log mạng sang các chiến thuật và kỹ thuật đối kháng; họ chứng minh rằng việc phân loại trực tiếp một bước bằng LLM thường dẫn đến hiện tượng bịa đặt (hallucination) tên kỹ thuật và tạo ra các ánh xạ thiếu ngữ cảnh, từ đó đề xuất quy trình suy luận có hướng dẫn (guided reasoning) như một giải pháp thay thế [meng_rhino_2025]. Những phát hiện này là động lực thúc đẩy việc thử nghiệm phân loại theo ngữ cảnh và trích xuất bằng chứng trên chuỗi truy vấn cơ sở dữ liệu, thay vì mặc định thừa nhận nhãn do mô hình đưa ra tự thân nó đã là một lời giải thích đáng tin cậy.

## 1.2. Phát biểu vấn đề

Mặc dù honeypot và phân tích an ninh hỗ trợ bởi mô hình ngôn ngữ đã được quan tâm nghiên cứu, vẫn còn tồn tại ba câu hỏi thực tiễn lớn trong bối cảnh PostgreSQL của đề tài này:

1. **Khả năng bao phủ của cấu hình log:** Các cấu hình ghi log có thể khác biệt lớn về khả năng ghi nhận các hành động cần thiết để nhận diện 8 nhóm hành vi do đề tài quy định: *Discovery (Khám phá), Credential Access (Truy cập thông tin xác thực), Privilege Escalation (Leo thang đặc quyền), Execution (Thực thi mã), Collection (Thu thập dữ liệu), Exfiltration (Tuồn dữ liệu), Impact/Persistence (Tác động/Duy trì truy cập)* và *Other/Noise (Nhiễu/Hành vi khác)*. Đề tài cần định lượng chính xác sự khác biệt này và xác định xem bản ghi thu được có đủ các trường thuộc tính cần thiết cho bước phân tích tiếp theo hay không.
2. **Độ chính xác của việc tái dựng phiên:** Các dòng log thô thu được không trực tiếp chỉ ra một phiên tấn công hoàn chỉnh. Việc gom phiên (sessionization) dựa trên ngưỡng thời gian không hoạt động (inactivity threshold) có thể gộp nhầm các hoạt động không liên quan hoặc chia cắt một chuỗi nhiều bước thực tế. Do đó, độ chính xác của quá trình tái dựng phiên cần được đo lường đối chiếu với ground truth của kịch bản đã biết.
3. **Phân loại cấp truy vấn và chất lượng bằng chứng:** Việc phân loại ở cấp độ từng truy vấn trở nên rất phức tạp khi ngữ nghĩa của một câu truy vấn phụ thuộc vào các sự kiện diễn ra trước đó. Phương pháp dựa trên luật (rule-based) có tính minh bạch cao nhưng kém linh hoạt trước các biến thể; trong khi đó, mô hình ngôn ngữ few-shot có thể khai thác ngữ cảnh linh hoạt hơn nhưng lại có nguy cơ đưa ra nhãn không có căn cứ hoặc bịa đặt mã định danh bằng chứng. Không thể xem phương pháp nào là đáng tin cậy nếu chưa đo lường đồng thời cả chất lượng phân loại và chất lượng bằng chứng giải thích.

Một thách thức lớn khác là **khả năng khái quát hóa (generalization)**. Một phương pháp có thể hoạt động rất tốt trên các biến thể tương đồng với kịch bản dùng khi phát triển nhưng lại thất bại hoàn toàn trước một họ kỹ thuật mới chưa từng xuất hiện trong quá trình viết luật hoặc tạo prompt. Để phân biệt rõ hai trường hợp này, quá trình đánh giá đòi hỏi phải phân chia dữ liệu có kiểm soát theo họ kịch bản, và cách ly nghiêm ngặt các họ kiểm thử khỏi quá trình xây dựng phương pháp. Quá trình đánh giá cũng phải đối chiếu không chỉ nhãn dự đoán mà cả tập truy vấn được chọn làm bằng chứng. Nếu thiếu sự phân tách này và thiếu đánh giá ở cấp độ bằng chứng, hiệu năng cao bề ngoài có thể chỉ là hệ quả của rò rỉ dữ liệu (data leakage) hoặc việc khớp các mẫu quen thuộc, chứ không phản ánh năng lực phân tích hành vi mới.

Vì vậy, đề tài sẽ xây dựng một pipeline đánh giá cục bộ, có khả năng tái lập, kết nối chặt chẽ giữa việc ghi log PostgreSQL, tái dựng phiên, phân loại truy vấn và đánh giá bằng chứng. Mục tiêu là đo lường mức độ phù hợp từ đầu đến cuối của log tương tác phục vụ phân tích hành vi, chứ không khẳng định rằng các kịch bản tổng hợp đại diện đầy đủ cho mọi hành vi tấn công ngoài thực tế.

## 1.3. Câu hỏi nghiên cứu và mục tiêu

Câu hỏi tổng quát của đề tài là: **Honeypot PostgreSQL tương tác cao kết hợp với phân tích hành vi có thể ghi nhận, nhận diện và giải thích các giai đoạn tấn công từ log tương tác đến mức nào, kể cả khi kẻ tấn công sử dụng các kỹ thuật chưa từng thấy trong quá trình xây dựng phương pháp?**

### RQ1: Khả năng ghi log và tái dựng phiên

**RQ1:** Các cấu hình ghi log của honeypot PostgreSQL bỏ sót hoặc tái dựng sai những loại hành động và phiên tấn công nào, ở mức độ bao nhiêu, và cấu hình nào đủ để phục vụ phân tích hành vi ở bước tiếp theo?

* **RO1.1:** Đo độ bao phủ hành động (recall) theo từng nhóm hành vi tấn công và bình thường trên 4 cấu hình log định trước.
* **RO1.2:** Đánh giá log đã phân tích cú pháp có đủ 6 trường thuộc tính cần thiết hay không: user, địa chỉ IP nguồn, database, câu lệnh truy vấn, kết quả thực thi (mã lỗi/trạng thái) và tham số. Đánh giá độ chính xác tái dựng phiên bằng pairwise Precision, Recall và F1 với 4 ngưỡng thời gian không hoạt động: 1, 5, 15 và 30 phút.

### RQ2: Phân loại và bằng chứng

**RQ2:** Giữa phương pháp dựa trên luật và mô hình ngôn ngữ lớn few-shot chạy cục bộ, phương pháp nào gán giai đoạn tấn công cho từng truy vấn (có xét ngữ cảnh các truy vấn trước đó) chính xác hơn, và tập truy vấn mà mỗi phương pháp chỉ ra làm bằng chứng khớp với bằng chứng thực tế đến mức nào?

* **RO2.1:** Đánh giá hiệu năng phân loại giai đoạn tấn công của phương pháp dựa trên luật và LLM few-shot trên các cửa sổ ngữ cảnh $k \in \{0, 2, 4\}$ truy vấn trước đó, sử dụng macro-averaged F1, F1 từng lớp và ma trận nhầm lẫn (confusion matrix).
* **RO2.2:** Đo lường chất lượng bằng chứng giải thích thông qua precision, recall và F1 của tập bằng chứng so với ground truth sinh từ quan hệ phụ thuộc; đối với LLM, đo lường thêm tỉ lệ sinh đầu ra sai định dạng (malformed output) và tỉ lệ bịa đặt mã định danh bằng chứng (hallucinated evidence IDs).

### RQ3: Khái quát hóa sang các họ kỹ thuật chưa biết

**RQ3:** Độ chính xác phân loại và độ khớp bằng chứng của hai phương pháp giảm bao nhiêu khi gặp họ kỹ thuật tấn công không được dùng trong quá trình xây dựng phương pháp, so với khi gặp biến thể của kỹ thuật đã biết?

* **RO3.1:** Đo mức suy giảm $\Delta$ macro-F1 và $\Delta$ evidence F1 giữa tập biến thể đã biết và tập các họ kỹ thuật mới được cách ly.
* **RO3.2:** Báo cáo F1 theo từng lớp trên tập họ kỹ thuật mới, đo lường sự thay đổi trong tỉ lệ bịa bằng chứng của LLM, và ước lượng khoảng tin cậy bootstrap theo phiên (session-level bootstrap confidence interval) cho hiệu năng của từng phương pháp.

## 1.4. Ý nghĩa của đề tài

Đề tài kết nối trực tiếp cấu hình ghi log của PostgreSQL với yêu cầu dữ liệu của các thuật toán phân tích hành vi. Về mặt vận hành, kết quả so sánh các cấu hình log sẽ giúp người quản trị xác định rõ những khoảng trống thông tin và đánh đổi hiệu năng (dung lượng log, độ trễ query) khi lựa chọn cấu hình ghi log cho cơ sở dữ liệu. Về mặt phân tích an ninh, việc đánh giá song song giữa hệ luật minh bạch và mô hình LLM few-shot cục bộ sẽ làm rõ liệu việc bổ sung ngữ cảnh chuỗi và năng lực suy luận ngôn ngữ có thực sự cải thiện khả năng phát hiện và giải thích giai đoạn tấn công hay không. Về mặt phương pháp luận, việc cách ly các họ kỹ thuật mới và đánh giá ở cấp độ bằng chứng sẽ cung cấp một thước đo thực tế về độ bền vững của mô hình AI khi đối mặt với các kỹ thuật tấn công ngoài phân phối (out-of-distribution).

## 1.5. Phạm vi và hạn chế

Đề tài sử dụng một honeypot PostgreSQL tương tác cao triển khai trong môi trường Docker cục bộ, kết hợp dữ liệu giả lập có bảng nhạy cảm và không phơi bày dịch vụ ra mạng Internet công cộng. Một bộ sinh kịch bản Python có tham số sẽ tự động biến đổi cấu trúc truy vấn, tên đối tượng, thứ tự câu lệnh, đặc trưng công cụ/client và độ lệch thời gian (timing jitter). Các kịch bản được xây dựng dựa trên hành vi tấn công ghi nhận trong y văn thực tế, bao gồm kỹ thuật dò quét thông tin xác thực, truy cập dữ liệu sau xác thực, và thực thi chương trình ngoài hệ thống do Song et al. báo cáo [song_decoy_2025], cùng kỹ thuật khám phá thông tin xác thực sau xâm nhập do Gómez-López et al. mô tả [gomez-lopez_multi-agent_2026]. Chuỗi truy vấn và liên kết phụ thuộc cụ thể được lập tài liệu làm ground truth tổng hợp chứ không sao chép nguyên trạng một chiến dịch tấn công cụ thể.

### Cấu hình ghi log
RQ1 so sánh 4 cấu hình ghi log, các tham số chính xác được cố định trước khi thực nghiệm:
* **C0:** Cấu hình log cơ sở của PostgreSQL với `log_statement = all`.
* **C1:** Kích hoạt kiểm toán phiên của tiện ích mở rộng pgAudit (`pgaudit.log = 'read, write, function, role, ddl'`).
* **C2:** Mở rộng từ C1, bổ sung thêm log kết nối và lỗi: `log_connections = on`, `log_disconnections = on`, `log_min_error_statement = error`, và `pgaudit.log_parameter = on`.
* **C3:** Bổ sung proxy hoặc cơ chế bắt gói tin mạng (tùy chọn theo quỹ thời gian thực hiện).

Tất cả các bản ghi log sau khi parse được đưa về một schema sự kiện chung. Thuật toán gộp phiên logic sử dụng IP nguồn, user và các ngưỡng thời gian không hoạt động (1, 5, 15, 30 phút). Các chỉ số đo lường gồm: recall hành động theo nhóm, độ đầy đủ trường, pairwise precision/recall/F1 của phiên, dung lượng log trên mỗi phiên và độ trễ truy vấn tăng thêm.

### Hệ thống phân loại và gán nhãn
Hệ thống nhãn gồm 8 lớp: Discovery, Credential Access, Privilege Escalation, Execution, Collection, Exfiltration, Impact/Persistence, và Other/Noise. Nhãn *Execution* được bổ sung tường minh vì các câu lệnh gọi tính năng thực thi chương trình hệ thống như `COPY ... FROM PROGRAM` là mẫu tấn công thực tế rất phổ biến trên PostgreSQL [song_decoy_2025] và không thể xếp lẫn vào các nhóm khác. Các quy tắc gán nhãn cho các hàm truy cập tệp và đối tượng lớn (Large Object) như `pg_read_file` và `lo_export` được viết và chốt trước khi sinh kịch bản; mọi trường hợp lưỡng nghĩa đều được ghi lại trong tài liệu ground truth.

### Phân loại và trích xuất bằng chứng
RQ2 so sánh bộ phân loại dựa trên luật với một mô hình ngôn ngữ lớn cục bộ khoảng 7–8 tỷ tham số phục vụ qua Ollama. Các mô hình ứng viên gồm Qwen2.5-7B-Instruct, Llama-3.1-8B-Instruct và Mistral-7B-Instruct; lựa chọn cuối cùng dựa trên bài đo tốc độ suy luận trên khoảng 20 cửa sổ truy vấn mẫu. Cả hai phương pháp đều nhận đầu vào là các cửa sổ truy vấn đã được chuẩn hóa (thay thế literal bằng placeholder, đổi tên bảng ngẫu nhiên). LLM bắt buộc sinh đầu ra JSON theo schema gồm nhãn và danh sách ID bằng chứng (`{label, evidence_ids}`). Schema ràng buộc cú pháp nhưng không cưỡng ép ID phải thuộc cửa sổ lúc giải mã, giúp đo lường được hiện tượng bịa đặt ID bằng chứng thay vì ép về 0. 

Dữ liệu train/dev/test được chia theo phiên hoặc họ kịch bản, và tập test được niêm phong cho đến khi luật và prompt hoàn thiện. Ground truth bằng chứng được định nghĩa gồm: `{truy vấn mục tiêu} ∪ {các truy vấn nó phụ thuộc còn nằm trong cửa sổ}`. Một mẫu kiểm tra thủ công 20% bằng chứng được thực hiện độc lập sau khi luật và prompt đã chốt để đánh giá độ tin cậy của ground truth tự động.

Đối với RQ3, 2–3 họ kỹ thuật mới (như thao tác Large Object, extension dblink, hàm `SECURITY DEFINER`) được xây dựng bởi người độc lập và cách ly hoàn toàn khỏi quá trình viết luật và prompt. Hạn chế chính của đề tài là tính chất tổng hợp của kịch bản, sự phụ thuộc của ground truth vào thiết kế kịch bản, và khả năng hành vi trong môi trường giả lập khác với CSDL sản xuất. Tính chân thực của dữ liệu mồi nhử cũng là một giới hạn: các nghiên cứu trước chỉ ra rằng kẻ tấn công nâng cao có thể nhận diện hầu hết các honeytoken mã nguồn mở phổ biến mà không kích hoạt cảnh báo [reiser_honeysweeper_2022], do đó các bản ghi giả lập được kiểm tra tính hợp lý nhưng không mặc định là không thể bị phát hiện.

## 1.6. Cấu trúc báo cáo

Báo cáo project môn học này bao gồm: Chương 1 giới thiệu bài toán nghiên cứu, mục tiêu, ý nghĩa và phạm vi; Chương 2 tổng quan toàn diện các công trình nghiên cứu liên quan và xác định rõ đóng góp khoa học của đề tài. Các nội dung triển khai chi tiết, kết quả thực nghiệm, thảo luận và kết luận sẽ được hoàn thiện trong các giai đoạn tiếp theo của đồ án.

---

# Chương 2. Tổng quan tài liệu

## 2.1. Phạm vi của tổng quan

Chương này phân tích các nghiên cứu học thuật hỗ trợ trực tiếp cho các giai đoạn của đề tài: triển khai honeypot cơ sở dữ liệu và hành vi kẻ tấn công, ghi log kiểm toán và tái dựng tấn công, giải thích dữ liệu an ninh dựa trên mô hình học máy và LLM, khả năng khái quát hóa sang hành vi tấn công chưa biết, và đánh giá tính chân thực của honeypot. Các công trình được tổng quan khác biệt về nền tảng, mức độ tương tác, loại dữ liệu và môi trường thực nghiệm. Do đó, kết quả của chúng đóng vai trò định hướng thiết kế và phương pháp luận cho đề tài, chứ không thể dùng để suy đoán trước hiệu năng trên chuỗi truy vấn PostgreSQL có kiểm soát.

## 2.2. Honeypot cơ sở dữ liệu và hành vi kẻ tấn công

Song, Smaragdakis và Griffioen đã tiến hành triển khai 278 honeypot trong vòng 20 ngày vào tháng 3–4 năm 2024 [song_decoy_2025]. Trong đó, 220 honeypot tương tác thấp mô phỏng MySQL, MSSQL, PostgreSQL và Redis ghi nhận hoạt động quét tương đối thấp nhưng các nỗ lực dò quét mật khẩu (brute-force) diễn ra dai dẳng. 58 honeypot tương tác trung bình và cao bộc lộ ba dạng khai thác rõ rệt: (1) thao túng trực tiếp hệ quản trị CSDL, (2) tấn công đòi tiền chuộc thông qua việc sao chép và xóa dữ liệu mục tiêu, và (3) lợi dụng CSDL làm bàn đạp để chiếm quyền điều khiển hệ thống bên dưới. Đây là công trình thực nghiệm liên quan trực tiếp nhất đến đề tài vì khảo sát trực tiếp PostgreSQL và các chuỗi hành vi nhiều bước. Do nghiên cứu của Song et al. triển khai trực diện trên Internet, lưu lượng và tần suất quan sát được sẽ khác với môi trường thực nghiệm cục bộ có kiểm soát của đề tài này.

Rocha và Pereira sử dụng các công cụ honeypot để đặc trưng hóa các cuộc tấn công vào hệ quản trị CSDL quan hệ và báo cáo rằng MySQL là hệ thống bị nhắm mục tiêu nhiều nhất trong số các hệ thống được khảo sát [rocha_what_2025]. Nghiên cứu kiểm thử các mồi nhử MySQL, PostgreSQL và Oracle, khẳng định tầm quan trọng của việc đo lường an ninh chuyên biệt cho từng loại DBMS.

Wegerer và Tjoa khảo sát các honeypot CSDL cùng kỹ thuật honeytoken, đồng thời mô tả một honeypot MySQL xây dựng từ các thành phần mã nguồn mở kết hợp plugin kiểm toán để ghi nhận truy vấn và các lượt đăng nhập [wegerer_defeating_2016]. Công trình này khẳng định cơ chế kiểm toán CSDL là trọng tâm thiết kế cốt lõi của một honeypot CSDL, mặc dù giải pháp plugin kiểm toán của MySQL không thể áp dụng nguyên trạng cho PostgreSQL.

Các nghiên cứu chuyên sâu về honeypot PostgreSQL và MS SQL xác định các tương tác sau xác thực (post-authentication) là đối tượng thu thập trọng tâm. Al Hafidz et al. nghiên cứu nâng cao khả năng phát hiện trong honeypot shell PostgreSQL thông qua cơ chế phản hồi truy vấn tương tác [al_hafidz_enhancing_2025]. Karo Karo et al. nghiên cứu thu thập hành vi sau đăng nhập trên honeypot MS SQL bằng relay TDS nhận biết giao thức [karo_karo_capturing_2025]. Cả hai công trình đều chứng minh rằng những gì diễn ra sau khi kẻ tấn công đăng nhập thành công là bài toán thu thập trung tâm, trực tiếp dẫn dắt câu hỏi nghiên cứu RQ1.

Gómez-López et al. giới thiệu SQLiAgent, một framework đa tác tử kiểm thử SQL Injection có chế độ hậu khai thác (post-exploitation) hỗ trợ bởi LLM nhằm giải thích schema CSDL, xác định các bảng chứa thông tin xác thực và sinh các câu lệnh SQL leo thang đặc quyền [gomez-lopez_multi-agent_2026]. Mặc dù mang tính tấn công, công trình này tài liệu hóa chi tiết các hành vi thực tế mà kịch bản Credential Access và Privilege Escalation của đề tài cần mô phỏng. Nghiên cứu cũng chứng minh rằng các công cụ hỗ trợ bởi LLM hoàn toàn có thể tự động sinh các truy vấn khai thác phức tạp từ cấu trúc schema suy luận được.

## 2.3. Ghi log kiểm toán, phân tích phiên và tái dựng tấn công

Dấu vết kiểm toán (audit trails) cơ sở dữ liệu cung cấp các bản ghi có cấu trúc hỗ trợ phân tích vượt trội so với việc kiểm tra từng truy vấn đơn lẻ. Zimniak, Getta và Benn chuyển đổi dữ liệu audit-trail CSDL để khám phá các mẫu định kỳ trong tải công việc (workload) [zimniak_predicting_2015]. Công trình này tập trung vào dự báo hiệu năng thay vì phát hiện an ninh, nhưng đã chứng minh rõ giá trị của việc bảo toàn thứ tự thời gian và cấu trúc khi phân tích chuỗi thao tác CSDL.

Biswas và Niemi xem xét các yêu cầu kiểm toán quyết định những loại log CSDL nào bắt buộc phải thu thập [biswas_transforming_2011]. Họ chỉ ra sự đánh đổi thực tiễn: ghi log không đủ sẽ cản trở việc kiểm toán tin cậy, trong khi ghi log dư thừa làm tăng chi phí lưu trữ và gây suy giảm hiệu năng CSDL. Luận điểm này là cơ sở trực tiếp cho việc so sánh 4 cấu hình log trong đề tài qua độ bao phủ hành động, độ đầy đủ trường, dung lượng log và độ trễ truy vấn. Zeng, Xiao và Chen nghiên cứu chi phí kiểm toán và khả năng thích ứng trong framework kiểm toán Linux, đề xuất cơ chế log thích ứng thay đổi kiểu sự kiện và tần suất theo các chuẩn bảo mật (CAPP, EAL, LSPP) [zeng_auditing_2015]. Công trình chỉ ra rằng chi phí kiểm toán không cố định mà phụ thuộc trực tiếp vào việc lựa chọn sự kiện ghi log.

Choi và Lee phân tích pháp y kỹ thuật số trên transaction log của SQL Server nằm trong vùng chưa phân bổ của hệ thống tệp [choi_forensic_2023]. Kết quả cho thấy log mức câu lệnh và transaction ghi nhận chi tiết các bước truy vấn và thay đổi trạng thái, đóng vai trò sống còn trong điều tra số. Goseva-Popstojanova et al. phân tích đặc trưng các phiên web độc hại từ honeypot tương tác cao và chứng minh rằng thời lượng phiên và số lượng request tuân theo phân phối đuôi nặng (Pareto) [goseva-popstojanova_characterization_2014]. Điều này hàm ý rằng việc gom phiên logic dựa trên ngưỡng thời gian không hoạt động sẽ phải đối mặt với sự phân tán lớn giữa các phiên ngắn và các phiên kéo dài.

Hossain et al. phát triển hệ thống SLEUTH nhằm tái dựng kịch bản tấn công thời gian thực từ log kiểm toán hệ thống COTS dựa trên đồ thị phụ thuộc lưu trong bộ nhớ [hossain_sleuth_nodate]. Bằng cách áp dụng lan truyền nhãn (tag propagation) theo hai chiều phân tích ngược (backward) và phân tích xuôi (forward), SLEUTH trích xuất chính xác nguồn gốc và tác động của cuộc tấn công. Phương pháp này cung cấp nền tảng toán học trực tiếp cho định nghĩa ground truth bằng chứng của đề tài: lần ngược chuỗi phụ thuộc nhân quả giữa các câu truy vấn. Zhu et al. giới thiệu TraceBack giải quyết bài toán bùng nổ phụ thuộc (dependency explosion) khi tái dựng tấn công APT từ đồ thị provenance [zhu_traceback_nodate], kết hợp lọc nhân quả trên POI và mô hình chuỗi để loại bỏ các nhánh nhiễu. Công trình này chứng minh tại sao cần phải đo lường và kiểm soát hiện tượng LLM bịa đặt mã bằng chứng không có quan hệ phụ thuộc thực tế.

## 2.4. Diễn giải và ánh xạ kỹ thuật dựa trên LLM

Schärmer et al. đánh giá ChatGPT và Gemini trong bài toán phân loại cảnh báo IDS và nhận thấy việc cung cấp ví dụ few-shot cùng mô tả ngữ cảnh hệ thống giúp nâng cao hiệu quả phân loại, đồng thời cảnh báo về chi phí và tính ổn định [afzal_let_2027]. Meng et al. đề xuất framework RHINO ánh xạ log mạng sang các kỹ thuật MITRE ATT&CK [meng_rhino_2025], chứng minh rằng phân loại trực tiếp một bước bằng LLM thường suy diễn sai hoặc mất ngữ cảnh, do đó cần quy trình suy luận có hướng dẫn dựa trên bằng chứng.

Wang et al. nghiên cứu nhận diện đa nhãn kỹ thuật tấn công dạng zero-shot bằng LLM thông qua khám phá bằng chứng ngữ nghĩa [wang_toward_2026]. Yang et al. đề xuất APTREC nhận diện chiến thuật và kỹ thuật APT dựa trên mô hình ngôn ngữ lớn [yang_aptrec_2026]. Meng et al. xây dựng benchmark CYBERTEAM chuẩn hóa các tác vụ săn lùng mối đe dọa (threat hunting) của Blue Team [meng_benchmarking_2026], chỉ ra rằng việc chuẩn hóa quy trình phân tích thành các module có cấu trúc giúp LLM hoạt động tin cậy hơn so với suy luận tự do. Landauer et al. phát triển bộ dữ liệu CAM-LDS ánh xạ các biểu hiện tấn công trong log hệ thống theo MITRE ATT&CK [landauer_cam-lds_2026], nhấn mạnh rằng nhiều hành vi tấn công riêng lẻ (như khám phá hệ thống) có biểu hiện y hệt hành vi quản trị thông thường nếu không được đặt trong ngữ cảnh toàn chuỗi.

## 2.5. Máy học cổ điển và LLM trên dữ liệu an ninh có cấu trúc

Liu và Lang khảo cứu toàn diện các phương pháp máy học và học sâu cho hệ thống phát hiện xâm nhập (IDS) [liu_machine_2019]. Alharthi, Alaryani và Kaddoura so sánh thực nghiệm các mô hình ML và DL trong phân loại nhị phân và đa lớp trên tập dữ liệu UNSW-NB15, thảo luận kỹ về sự mất cân bằng giữa các lớp tấn công và nguyên tắc phân chia dữ liệu huấn luyện [alharthi_comparative_2025].

Rehman et al. đánh giá LLM zero-shot và few-shot trên dữ liệu luồng mạng bằng cách chuyển đổi bản ghi thành văn bản (flows-to-words), bổ sung cờ logic, áp dụng ngữ pháp GBNF ràng buộc đầu ra JSON và hiệu chuẩn ngưỡng quyết định trên tập dev [rehman_flows_2025]. Kết quả cho thấy mô hình 7B có cờ logic đạt macro-F1 xấp xỉ 0.78, nhưng chất lượng quyết định suy giảm khi kích thước tập kiểm thử tăng lên. Mô hình truyền thống ổn định và nhanh hơn, nhưng LLM thích ứng linh hoạt mà không cần huấn luyện lại trọng số. Mehavilla et al. tiến hành benchmark hệ thống các LLM độc lập (LLaMA-3.2-1B, GPT-2, GPT-Neo-125M) trên dữ liệu luồng mạng so sánh với Random Forest, XGBoost và MLP [mehavilla_evaluating_2026]. Nghiên cứu chỉ ra chi phí suy luận khổng lồ của LLM và độ nhạy cảm cao trước mất cân bằng lớp. Neves et al. so sánh LLM zero-shot với phương pháp dựa trên luật trong trích xuất kiểu hình lâm sàng [neves_zero-shot_2025], cho thấy hệ luật vượt trội hơn hẳn LLM không qua tinh chỉnh khi xử lý các định dạng miền hẹp.

## 2.6. Khái quát hóa sang hành vi tấn công chưa biết

Duan et al. khảo cứu các phương pháp Few-Shot Learning (FSL) trong phát hiện xâm nhập mạng [duan_survey_2021], phân tích các hướng tiếp cận theo mô hình, độ đo và tối ưu hóa nhằm thích ứng với các cuộc tấn công mới lạ khi có rất ít mẫu huấn luyện. Lust et al. khảo cứu không gian khái quát hóa của mạng nơ-ron trước các mẫu ngoài phân phối (out-of-distribution - OOD) và mẫu đối kháng [lust_survey_2026]. 

Zhou et al. đề xuất giải thuật GROD nhằm nâng cao khả năng khái quát hóa của Transformer thông qua phát hiện OOD [zhou_grod_2024], tích hợp cơ chế sinh dữ liệu ngoại lai tổng hợp để phạt các phân loại sai ngoài phân phối. Guo, Wildberger và Schölkopf nghiên cứu bài toán khái quát hóa ngoài biến số (out-of-variable generalization) khi tập thuộc tính quan sát trong kiểm thử thay đổi so với khi huấn luyện [guo_out--variable_2024]. Những nghiên cứu này là cơ sở lý thuyết để đề tài thiết kế bài kiểm tra RQ3: đo lường mức độ suy giảm hiệu năng khi mô hình đối mặt với các họ kỹ thuật bị cách ly hoàn toàn khỏi quá trình xây dựng prompt và luật.

## 2.7. Tính chân thực của Honeypot, Dữ liệu mồi nhử tổng hợp và Đánh giá

Nunez Migliorisi đề xuất khung chỉ số thực nghiệm dựa trên viễn trắc (dwell time, interaction depth) để đánh giá tính chân thực và hiệu quả đánh lừa của honeypot [noelia_nunez_migliorisi_empirical_2026]. Ferguson-Walter et al. công bố nghiên cứu thực nghiệm Tularosa trên hơn 130 red teamer, chứng minh rằng sự hiện diện của decoy làm chậm đáng kể tiến trình tấn công và khiến đối thủ tiêu tốn tài nguyên gấp nhiều lần [ferguson-walter_examining_nodate]. Shahin et al. mô hình hóa chiến lược đánh lừa dưới góc độ lý thuyết trò chơi và rò rỉ thông tin [shahin_evaluating_2026]. Zambianco et al. xây dựng mô hình lựa chọn decoy chủ động dựa trên MITRE ATT&CK [zambianco_proactive_2025]. Das và Nair công bố bộ dữ liệu tấn công đa giai đoạn sử dụng MITRE Caldera [das_novel_2026].

Về dữ liệu mồi nhử, Weinberg giới thiệu mô hình PHANTOM sinh honeytoken đa hình thích ứng với ngữ cảnh tổ chức [weinberg_phantom_2026], phân rã điểm đánh giá theo cú pháp, thống kê và ngữ nghĩa. Cơ chế này định hướng cho việc kiểm tra tính tự nhiên của các bảng nhạy cảm giả lập trong đề tài. Ngược lại, Msaad et al. chỉ ra rằng kẻ tấn công nâng cao có thể nhận diện hầu hết các honeytoken mã nguồn mở hiện nay mà không kích hoạt cảnh báo [reiser_honeysweeper_2022], một hạn chế mà đề tài cần thừa nhận.

Về xu hướng tích hợp AI vào đánh lừa mạng, Kouremetis et al. giới thiệu Mirage thử nghiệm đánh lừa chống lại các tác nhân tấn công tự động [kouremetis_mirage_2024]. Wushishi et al. đề xuất D3O-IIoT điều phối bẫy động bằng học tăng cường sâu [wushishi_d3o-iiot_2025]. Safargalieva et al. phát triển honeypot đa giao thức OHRA điều khiển bởi LLM [safargalieva_ohra_nodate]. Cuối cùng, Bridges et al. hệ thống hóa toàn diện mối quan hệ giữa Honeypot và LLM (SoK) [bridges_sok_2026], cung cấp phân loại 4 giai đoạn tiến hóa của phân tích log honeypot, chỉ ra kiến trúc chuẩn tắc và nhấn mạnh sự khan hiếm trầm trọng của dữ liệu gán nhãn thực tế.

## 2.8. Tóm tắt tổng quan tài liệu

Các nghiên cứu honeypot CSDL cho thấy mồi nhử chuyên biệt cho DBMS có thể thu hút các nỗ lực xác thực kéo dài và bộc lộ hành vi khai thác sâu nhiều bước, trong đó mức tương tác và cơ chế thu thập quyết định trực tiếp dữ liệu ghi nhận được [song_decoy_2025, al_hafidz_enhancing_2025, karo_karo_capturing_2025]. Các nghiên cứu audit trail và tái dựng tấn công khẳng định thứ tự sự kiện, thời gian, quan hệ phụ thuộc và độ phủ trường thông tin là tối quan trọng, đồng thời xác định chi phí ghi log là yếu tố đánh đổi phải đo lường [biswas_transforming_2011, zeng_auditing_2015, choi_forensic_2023]. Nghiên cứu ứng dụng LLM trong an ninh cung cấp các chiến lược prompt theo ngữ cảnh, thiết kế định hướng bằng chứng và quy trình đánh giá chuẩn mực, nhưng các kết quả trên luồng mạng hay cảnh báo IDS chưa thể áp dụng trực tiếp cho chuỗi truy vấn SQL [afzal_let_2027, meng_rhino_2025, neves_zero-shot_2025, rehman_flows_2025]. Các nghiên cứu về khái quát hóa thúc đẩy việc cách ly các họ kỹ thuật mới thay vì chia ngẫu nhiên [lust_survey_2026], và nghiên cứu đánh giá honeypot khẳng định mức độ tương tác đòi hỏi nhiều hơn là việc chỉ đếm số kết nối thô [noelia_nunez_migliorisi_empirical_2026, ferguson-walter_examining_nodate].

Tuy nhiên, vẫn còn một khoảng trống nghiên cứu rõ rệt tại giao điểm của các lĩnh vực trên: Các tài liệu hiện có chưa làm rõ **cấu hình ghi log PostgreSQL nào là đủ để tái dựng các phiên truy vấn có nhãn trong một honeypot có kiểm soát**, và liệu **mô hình ngôn ngữ lớn few-shot chạy cục bộ có thực sự cải thiện độ chính xác phân loại giai đoạn truy vấn và chất lượng bằng chứng so với phương pháp dựa trên luật khi đánh giá trên các họ kỹ thuật bị cách ly hay không**. Đề tài này giải quyết trực tiếp khoảng trống đó bằng một pipeline thực nghiệm nhất quán và có thể tái lập.

## 2.9. Đóng góp nghiên cứu của đề tài

Đóng góp dự kiến của đề tài là một đánh giá thực nghiệm toàn diện và có thể tái lập trên honeypot PostgreSQL cục bộ cùng pipeline phân tích hành vi tích hợp AI:

* **C1:** So sánh định lượng 4 cấu hình ghi log định trước (C0–C3) thông qua recall hành động theo nhóm, độ đầy đủ của các trường sự kiện bắt buộc, dung lượng log sinh ra và độ trễ truy vấn tăng thêm.
* **C2:** Đo lường độ chính xác của việc tái dựng phiên truy vấn qua các ngưỡng thời gian không hoạt động (1, 5, 15, 30 phút) so với kịch bản ground truth có quan hệ phụ thuộc tường minh.
* **C3:** So sánh thực nghiệm bộ phân loại dựa trên luật với mô hình LLM few-shot chạy cục bộ trên cùng các cửa sổ ngữ cảnh truy vấn, báo cáo macro-F1, F1 từng lớp, tỉ lệ output sai định dạng, tỉ lệ ID bằng chứng bịa, cùng precision, recall và F1 của bằng chứng.
* **C4:** Đánh giá độ bền vững và khả năng khái quát hóa của hai phương pháp trên các họ kỹ thuật tấn công mới được cách ly hoàn toàn khỏi quá trình xây dựng luật và prompt.

---

## Tài liệu tham khảo

[song_decoy_2025] Y. Song, G. Smaragdakis, and H. Griffioen, "Decoy Databases: Analyzing Attacks on Public Facing Databases," in *Proc. 2025 ACM Internet Measurement Conf. (IMC '25)*, Oct. 2025, pp. 709–726, doi: 10.1145/3730567.3764481.

[wegerer_defeating_2016] M. Wegerer and S. Tjoa, "Defeating the Database Adversary Using Deception—A MySQL Database Honeypot," in *Proc. 2016 Int. Conf. on Software Security and Assurance (ICSSA)*, Aug. 2016, pp. 6–10, doi: 10.1109/ICSSA.2016.8.

[afzal_let_2027] A. Schärmer, M. Landauer, F. Skopik, M. Wurzenberger, and M. Squarcina, "Let the Alerts Speak: LLM-Based IDS Alert Interpretation for SOC Triage," in *Availability, Reliability and Security. ARES 2026 International Workshops*, ser. LNCS, vol. 16901, Cham: Springer, 2027, pp. 346–364, doi: 10.1007/978-3-032-35579-9_19.

[meng_rhino_2025] F. Meng, J. Gui, Y. Li, and Y. Wu, "RHINO: Guided Reasoning for Mapping Network Logs to Adversarial Tactics and Techniques with Large Language Models," *arXiv:2510.14233 [cs.CR]*, Oct. 2025.

[rocha_what_2025] A. Rocha and F. S. F. Pereira, "What is the most targeted database? Using Honeypots to Characterize Attacks on Relational Databases," *Procedia Computer Science*, vol. 256, pp. 114–121, 2025, doi: 10.1016/j.procs.2025.01.077.

[al_hafidz_enhancing_2025] R. F. Al Hafidz, A. H. Muhammad, and F. J. P. S. D. Putra, "Enhancing Detection Postgresql Shell Honeypot with Interactive Query Response," in *2025 8th Int. Conf. on Vocational Education and Electrical Engineering (ICVEE)*, 2025, pp. 1–6.

[karo_karo_capturing_2025] H. Karo Karo, A. H. Muhammad, and F. J. P. S. D. Putra, "Capturing Post-Login Behaviors in MS SQL Honeypots: Profiling Attackers via a Protocol-Aware TDS Relay," in *2025 6th Int. Conf. on Cybernetics and Computational Intelligence (CyberneticsCom)*, 2025, pp. 1–6.

[gomez-lopez_multi-agent_2026] A. Gómez-López, M. J. Escalona, and J. A. García-García, "A Multi-Agent Framework for SQL Injection Auditing with LLM-Driven Post-Exploitation and Privilege Escalation," *Computers*, vol. 15, no. 1, p. 658, 2026, doi: 10.3390/computers15100658.

[zimniak_predicting_2015] M. Zimniak, J. R. Getta, and W. Benn, "Predicting database workloads through mining periodic patterns in database audit trails," *Distributed and Parallel Databases*, vol. 33, no. 4, pp. 583–614, 2015, doi: 10.1007/s10619-014-7164-8.

[biswas_transforming_2011] D. Biswas and V. Niemi, "Transforming Privacy Policies to Auditing Specifications," in *2011 IEEE 10th Int. Conf. on Trust, Security and Privacy in Computing and Communications*, 2011, pp. 627–634, doi: 10.1109/TrustCom.2011.83.

[zeng_auditing_2015] L. Zeng, Y. Xiao, and H. Chen, "Auditing overhead, auditing adaptation, and benchmark evaluation in Linux," *Security and Communication Networks*, vol. 8, no. 18, pp. 3523–3534, 2015, doi: 10.1002/sec.1277.

[choi_forensic_2023] H. Choi and S. Lee, "Forensic analysis of SQL server transaction log in unallocated area of file system," *Forensic Science International: Digital Investigation*, vol. 46, p. 301605, 2023, doi: 10.1016/j.fsidi.2023.301605.

[goseva-popstojanova_characterization_2014] K. Goseva-Popstojanova, C. Veeramachaneni, and R. Singh, "Characterization and classification of malicious Web traffic," *Computers & Security*, vol. 42, pp. 92–115, 2014, doi: 10.1016/j.cose.2014.01.006.

[hossain_sleuth_nodate] M. N. Hossain, S. Milajerdi, J. Wang, B. Eshete, R. Gjomemo, R. Sekar, S. Stoller, and V. Venkatakrishnan, "SLEUTH: Real-time Attack Scenario Reconstruction from COTS Audit Data," in *Proc. 26th USENIX Security Symp.*, Aug. 2017, pp. 487–504.

[zhu_traceback_nodate] T. Zhu, M. Li, Z. Wang, W. Cheng, Q. Chen, J. Ren, and Z. Tian, "TraceBack: Sequence-Enhanced and Causality-Driven Attack Scenario Reconstruction from Provenance Graphs," *SSRN Electronic Journal*, 2025, doi: 10.2139/ssrn.5351687.

[wang_toward_2026] J. Wang, S. Zhang, Y. Xiao, Y. Tan, and H. Wang, "Toward Zero-Shot Multi-Label Identification of Attack Techniques: Enhancing LLM Reasoning With Semantic Evidence Discovery," *IEEE Trans. on Information Forensics and Security*, vol. 38, pp. 1120–1134, 2026.

[yang_aptrec_2026] K. Yang, S. Chen, H. Zhang, and X. Liu, "APTREC: APT tactic/technique recognition based on large language model," *Computers & Security*, vol. 280, p. 104112, 2026, doi: 10.1016/j.cose.2026.104112.

[meng_benchmarking_2026] Y. Meng, L. Tang, F. Yu, X. Li, G. Yan, P. Yang, and Z. Xi, "Benchmarking LLM-Assisted Blue Teaming via Standardized Threat Hunting," *arXiv:2509.23571 [cs.CR]*, Sep. 2025.

[landauer_cam-lds_2026] M. Landauer, M. Wurzenberger, F. Skopik, et al., "CAM-LDS: Cyber Attack Manifestations for Automatic Interpretation of System Logs and Security Alerts," *arXiv:2603.04186 [cs.CR]*, Mar. 2026.

[liu_machine_2019] H. Liu and B. Lang, "Machine Learning and Deep Learning Methods for Intrusion Detection Systems: A Survey," *Applied Sciences*, vol. 9, no. 20, p. 4396, 2019, doi: 10.3390/app9204396.

[alharthi_comparative_2025] A. Alharthi, M. Alaryani, and S. Kaddoura, "A comparative study of machine learning and deep learning models in binary and multiclass classification for intrusion detection systems," *Array*, vol. 26, p. 100406, 2025, doi: 10.1016/j.array.2025.100406.

[rehman_flows_2025] M. A. Rehman, S. I. A. Shah, A. Anwar, and N. Islam, "From Flows to Words: Can Zero-/Few-Shot LLMs Detect Network Intrusions? A Grammar-Constrained, Calibrated Evaluation on UNSW-NB15," *arXiv:2510.17883 [cs.CR]*, Oct. 2025.

[mehavilla_evaluating_2026] M. Mehavilla, R. A. Sukamto, and B. Hendradjaya, "Evaluating large language models effectiveness for flow-based intrusion detection: a comparative study with ML and DL baselines," *Artificial Intelligence Review*, vol. 59, p. 50, 2026, doi: 10.1007/s10462-025-11432-2.

[neves_zero-shot_2025] M. Neves, A. Sousa, and P. Oliveira, "Zero-shot learning for clinical phenotyping: Comparing LLMs and rule-based methods," *Journal of Biomedical Informatics*, vol. 192, p. 104882, 2025, doi: 10.1016/j.jbi.2025.104882.

[duan_survey_2021] R. Duan, D. Li, Q. Tong, T. Yang, X. Liu, and X. Liu, "A Survey of Few-Shot Learning: An Effective Method for Intrusion Detection," *Security and Communication Networks*, vol. 2021, p. 4259629, 2021, doi: 10.1155/2021/4259629.

[lust_survey_2026] F. Lust, M. Schneider, and C. Reuter, "A survey on assessing the generalization area of neural networks: Uncertainty, out-of-distribution and adversarial samples," *Neurocomputing*, vol. 706, p. 129480, 2026, doi: 10.1016/j.neucom.2026.129480.

[zhou_grod_2024] C. Zhou, Y. Chen, and L. Huang, "GROD: Enhancing Generalization of Transformer with Out-of-Distribution Detection," *arXiv:2408.01234 [cs.LG]*, 2024.

[guo_out--variable_2024] S. Guo, J. Wildberger, and B. Schölkopf, "Out-of-Variable Generalization for Discriminative Models," *Transactions on Machine Learning Research*, 2024.

[noelia_nunez_migliorisi_empirical_2026] T. N. Nunez Migliorisi, "Empirical Telemetry-Based Metrics for Evaluating Honeypot Realism and Deception Effectiveness," *Int. Journal of Network Security & Its Applications (IJNSA)*, vol. 18, no. 1, pp. 1–18, 2026, doi: 10.5121/ijnsa.2026.18101.

[ferguson-walter_examining_nodate] K. J. Ferguson-Walter, M. M. Major, C. K. Johnson, and D. H. Muhleman, "Examining the Efficacy of Decoy-based and Psychological Cyber Deception," in *Proc. 30th USENIX Security Symp.*, Aug. 2021, pp. 3767–3784.

[shahin_evaluating_2026] M. Shahin, A. Al-Fuqaha, and M. Guizani, "Evaluating Synthetic Cyber Deception Strategies Under Uncertainty via Game Theory Approach: Linking Information Leakage and Game Outcomes in Cyber Deception," *Sensors*, vol. 26, no. 5, p. 1748, 2026, doi: 10.3390/s26051748.

[zambianco_proactive_2025] M. Zambianco, P. Bellavista, and A. Corradi, "A Proactive Decoy Selection Scheme for Cyber Deception using MITRE ATT&CK," *Computers & Security*, vol. 148, p. 104144, 2025, doi: 10.1016/j.cose.2024.104144.

[das_novel_2026] V. Das and B. B. Nair, "A novel multi-stage attack dataset for smart home intrusion detection," *Data in Brief*, vol. 66, p. 112770, 2026, doi: 10.1016/j.dib.2026.112770.

[weinberg_phantom_2026] A. I. Weinberg, "PHANTOM: Polymorphic Honeytoken Adaptation with Narrative-Tailored Organisational Mimicry," *arXiv:2605.02992 [cs.CR]*, May 2026.

[reiser_honeysweeper_2022] M. Msaad, S. Srinivasa, M. M. Andersen, D. Audran, C. U. Orji, and E. Vasilomanolakis, "Honeysweeper: Towards stealthy Honeytoken fingerprinting techniques," in *Nordic Conf. on Secure IT Systems (NordSec)*, ser. LNCS, vol. 13700, Springer, 2022, pp. 87–104, doi: 10.1007/978-3-031-22295-5_6.

[kouremetis_mirage_2024] M. Kouremetis et al., "Mirage: cyber deception against autonomous cyber attacks in emulation and simulation," *Cybersecurity*, vol. 7, no. 1, p. 79, 2024, doi: 10.1186/s42400-024-00272-4.

[wushishi_d3o-iiot_2025] A. Wushishi et al., "D3O-IIoT: deep reinforcement learning-driven dynamic deception orchestration for industrial IoT security," *Scientific Reports*, vol. 16, p. 1205, 2025, doi: 10.1038/s41598-025-33426-4.

[safargalieva_ohra_nodate] A. Safargalieva, A. Rüffer, and E. Vasilomanolakis, "OHRA: dynamic multi-protocol LLM-based cyber deception," in *Proc. Nordic Conf. on Secure IT Systems (NordSec)*, 2025.

[bridges_sok_2026] R. A. Bridges, T. R. Mitchell, M. Muñoz, and T. Henriksson, "SoK: Honeypots & LLMs, More Than the Sum of Their Parts?," *arXiv:2510.25939 [cs.CR]*, Oct. 2025.
