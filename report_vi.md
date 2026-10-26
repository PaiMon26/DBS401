# BÁO CÁO PROJECT MÔN DBS401

## Xây dựng Honeypot Cơ sở dữ liệu PostgreSQL kết hợp AI Phân tích Hành vi Kẻ tấn công

**Nhóm 3**

| Sinh viên | Mã sinh viên |
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

Các hệ quản trị cơ sở dữ liệu được kết nối Internet là mục tiêu hấp dẫn vì lưu trữ thông tin có giá trị và cung cấp giao diện truy vấn trực tiếp. Honeypot cơ sở dữ liệu tạo ra môi trường có kiểm soát để quan sát các tương tác này, nhưng giá trị của dữ liệu thu thập được phụ thuộc vào cấu hình ghi log và khả năng tái dựng các truy vấn thành phiên tấn công mạch lạc. Project này đề xuất xây dựng honeypot PostgreSQL chạy cục bộ, sử dụng dữ liệu giả có vẻ nhạy cảm và môi trường truy vấn tương tác có kiểm soát để nghiên cứu độ bao phủ của log và phân tích hành vi tự động. Các kịch bản tấn công và truy vấn bình thường được tham số hóa sẽ tạo ra chuỗi truy vấn có nhãn và quan hệ phụ thuộc tường minh; log từ bốn cấu hình được xác định trước sẽ được chuẩn hóa về cùng một schema sự kiện. Nghiên cứu trước hết so sánh độ bao phủ hành động, mức độ đầy đủ của trường dữ liệu và khả năng tái dựng phiên theo các ngưỡng thời gian không hoạt động. Tiếp theo, nghiên cứu so sánh bộ phân loại dựa trên luật với mô hình ngôn ngữ lớn few-shot chạy cục bộ trên các cửa sổ truy vấn, gồm truy vấn hiện tại và một số truy vấn trước đó. Phân loại được đánh giá bằng macro-F1 và kết quả theo từng lớp; phần giải thích được đánh giá bằng precision, recall và F1 so với bằng chứng sinh từ quan hệ phụ thuộc. Cuối cùng, hai phương pháp được đánh giá mà không chỉnh sửa trên các họ kỹ thuật tấn công được cách ly khỏi quá trình phát triển. Kết quả dự kiến là đánh giá thực nghiệm về đánh đổi giữa các cấu hình log, một pipeline đánh giá truy vấn có thể tái lập, và bằng chứng về độ vững của phân loại và giải thích trước các biến thể kỹ thuật. Vì thí nghiệm chưa được thực hiện, phần tóm tắt này mô tả thiết kế project và phương pháp đánh giá dự kiến, không trình bày kết quả thực nghiệm.

**Từ khóa:** honeypot PostgreSQL; kiểm toán cơ sở dữ liệu; tái dựng phiên tấn công; phân tích hành vi; học few-shot; phân loại dựa trên bằng chứng; khái quát hóa ngoài phân phối.

---

# Chương 1. Giới thiệu

## 1.1. Bối cảnh

Hệ quản trị cơ sở dữ liệu là một thành phần quan trọng trong lớp dữ liệu của các ứng dụng hiện đại. Khi dịch vụ cơ sở dữ liệu được phơi bày trên mạng công cộng, kẻ tấn công có thể dò tìm hệ thống và cấu hình, thử xác thực, gửi truy vấn để khám phá hoặc thu thập dữ liệu, thay đổi trạng thái cơ sở dữ liệu, hoặc tận dụng các khả năng của cơ sở dữ liệu để tiếp cận tài nguyên khác. Những hành động này không nhất thiết diễn ra độc lập; chúng có thể tạo thành một chuỗi trong đó các bước sau phụ thuộc vào thông tin hoặc quyền truy cập thu được từ các bước trước. Vì vậy, hiểu hành vi tấn công đòi hỏi vừa thu thập sự kiện đáng tin cậy, vừa phân tích được mối quan hệ giữa các sự kiện.

Honeypot cơ sở dữ liệu cung cấp một phương thức có kiểm soát để quan sát hoạt động hướng đến dịch vụ mồi. Các nghiên cứu trước đã xem xét việc sử dụng honeypot cơ sở dữ liệu để tìm hiểu các cuộc tấn công trong thực tế. Song, Smaragdakis và Griffioen triển khai honeypot mô phỏng nhiều hệ quản trị cơ sở dữ liệu, trong đó có PostgreSQL; nghiên cứu ghi nhận các nỗ lực đánh cắp thông tin xác thực kéo dài và những mô hình khai thác nhiều bước trong các triển khai có mức tương tác cao hơn [1]. Một nghiên cứu trước đó cũng minh họa tính khả thi của việc xây dựng honeypot cơ sở dữ liệu tương tác bằng công nghệ mã nguồn mở [2]. Những công trình này cho thấy giá trị của kỹ thuật đánh lừa tập trung vào cơ sở dữ liệu, đồng thời đặt ra yêu cầu đo lường cẩn thận khả năng quan sát của từng cách triển khai.

Dữ liệu log thô là nền tảng cho các bước phân tích tiếp theo. Cấu hình ghi log chỉ ghi câu lệnh nhưng bỏ qua thông tin kết nối, lỗi hoặc tham số cần thiết có thể khiến hồ sơ tương tác không đầy đủ. Ngược lại, ghi log quá chi tiết có thể làm tăng dung lượng lưu trữ và chi phí xử lý truy vấn, đồng thời có nguy cơ làm lộ giá trị nhạy cảm nếu sử dụng dữ liệu thật. Ngay cả khi từng sự kiện riêng lẻ đã được ghi nhận, người phân tích vẫn cần xác định sự kiện nào thuộc cùng một phiên logic và truy vấn nào là bằng chứng cho một giai đoạn tấn công cụ thể. Do đó, thiết kế log không chỉ là thiết lập vận hành; nó ảnh hưởng trực tiếp đến tính đầy đủ và độ tin cậy của phân tích hành vi ở các bước sau.

Các nghiên cứu gần đây cũng tìm hiểu việc sử dụng mô hình ngôn ngữ lớn để diễn giải cảnh báo an ninh và ánh xạ quan sát mức thấp sang hành vi tấn công mức cao. Schärmer và cộng sự nghiên cứu mô hình ngôn ngữ lớn trong diễn giải cảnh báo xâm nhập, đồng thời cho thấy ví dụ được lựa chọn cẩn thận và thông tin ngữ cảnh bổ sung có thể ảnh hưởng đến hiệu năng cũng như tính nhất quán của phân loại [3]. Framework RHINO cũng xem việc ánh xạ quan sát mạng sang kỹ thuật của đối phương là một nhiệm vụ suy luận, đồng thời đưa vào bước kiểm chứng dựa trên bằng chứng nhằm xử lý các ánh xạ thiếu căn cứ hoặc bị mô hình bịa ra [4]. Các kết quả này là cơ sở để kiểm tra khả năng phân loại theo ngữ cảnh và chọn bằng chứng trên log truy vấn cơ sở dữ liệu, thay vì mặc định rằng nhãn do mô hình đưa ra tự nó đã là một lời giải thích đáng tin cậy.

## 1.2. Phát biểu vấn đề

Mặc dù honeypot và phân tích an ninh có hỗ trợ mô hình ngôn ngữ đã được quan tâm, vẫn còn ba câu hỏi thực tiễn trong bối cảnh PostgreSQL của project này. Thứ nhất, các cấu hình log có thể khác nhau về khả năng ghi nhận hành động cần thiết để nhận diện bảy nhóm hành vi do project quy định: **Discovery, Credential Access, Privilege Escalation, Collection, Exfiltration, Impact/Persistence** và **Other/Noise**. Project cần định lượng sự khác biệt này và xác định bản ghi thu được có đủ các trường cần thiết cho phân tích tiếp theo hay không. Thứ hai, các dòng log thu được chưa chắc trực tiếp cho biết một phiên tấn công hoàn chỉnh. Gom phiên theo ngưỡng thời gian không hoạt động có thể ghép nhầm hoạt động không liên quan hoặc tách rời một chuỗi nhiều bước thực sự; do đó, độ chính xác cần được đo đối chiếu với ground truth của kịch bản. Thứ ba, phân loại ở cấp truy vấn trở nên khó khăn khi ý nghĩa của truy vấn phụ thuộc vào các sự kiện trước đó. Phương pháp dựa trên luật có thể minh bạch nhưng kém linh hoạt trước biến thể; mô hình ngôn ngữ few-shot có thể tận dụng ngữ cảnh tốt hơn nhưng cũng có thể đưa ra nhãn hoặc mã định danh bằng chứng không có căn cứ. Không nên xem phương pháp nào là đáng tin cậy nếu chưa đo đồng thời chất lượng phân loại và chất lượng bằng chứng.

Một thách thức khác là khả năng khái quát hóa. Một phương pháp có thể hoạt động tốt trên các biến thể gần với kịch bản dùng khi phát triển nhưng thất bại trước một họ kỹ thuật không xuất hiện trong quá trình viết luật hoặc ví dụ. Để phân biệt hai trường hợp này, việc đánh giá cần chia dữ liệu theo họ kịch bản và cách ly các họ được giữ lại khỏi quá trình phát triển phương pháp. Đánh giá cũng cần xem xét không chỉ nhãn dự đoán mà cả các truy vấn được dùng làm bằng chứng. Nếu thiếu cách chia tách này và đánh giá ở mức bằng chứng, hiệu năng có vẻ cao có thể đến từ rò rỉ dữ liệu hoặc mẫu quen thuộc thay vì khả năng phân tích hành vi chưa gặp.

Vì vậy, project sẽ xây dựng một pipeline đánh giá cục bộ, có thể tái lập, kết nối việc ghi log PostgreSQL, tái dựng phiên, phân loại truy vấn và đánh giá bằng chứng. Mục tiêu là đo lường mức độ phù hợp từ đầu đến cuối của log tương tác cho phân tích hành vi, chứ không khẳng định rằng các kịch bản tổng hợp đại diện đầy đủ cho mọi hành vi tấn công ngoài thực tế.

## 1.3. Câu hỏi nghiên cứu và mục tiêu

Câu hỏi tổng quát của project là: **Honeypot PostgreSQL tương tác cao kết hợp với phân tích hành vi có thể ghi nhận, nhận diện và giải thích các giai đoạn tấn công từ log tương tác đến mức nào, kể cả khi kẻ tấn công sử dụng kỹ thuật chưa từng thấy?**

### RQ1: Khả năng ghi log và tái dựng phiên

**RQ1.** Các cấu hình ghi log của honeypot PostgreSQL bỏ sót hoặc tái dựng sai những loại hành động và phiên tấn công nào, ở mức độ bao nhiêu, và cấu hình nào đủ để phục vụ phân tích hành vi ở bước tiếp theo?

- **RO1.1:** Đo recall hành động theo từng nhóm hành vi tấn công và bình thường trên bốn cấu hình log đã định nghĩa.
- **RO1.2:** Đánh giá log đã phân tích cú pháp có đủ sáu thuộc tính phiên cần thiết hay không: user, địa chỉ IP nguồn, cơ sở dữ liệu, query, kết quả và tham số. Đánh giá tái dựng phiên bằng pairwise precision, recall và F1 với các ngưỡng thời gian không hoạt động là 1, 5, 15 và 30 phút.
- **RO1.3:** Chọn cấu hình log có chi phí thấp nhất nhưng vẫn đáp ứng các yêu cầu đã xác định trước cho phân tích RQ2, xét đến độ bao phủ, mức đầy đủ của trường dữ liệu, dung lượng log và độ trễ truy vấn.

### RQ2: Phân loại và bằng chứng

**RQ2.** Giữa phương pháp dựa trên luật và mô hình ngôn ngữ lớn few-shot chạy cục bộ, phương pháp nào gán giai đoạn tấn công cho từng query chính xác hơn khi có xét các truy vấn trước đó, và tập query mà mỗi phương pháp chọn làm bằng chứng khớp với bằng chứng đã biết đến mức nào?

- **RO2.1:** So sánh hiệu năng phân loại và độ ổn định kỹ thuật của đầu ra bằng macro-F1, F1 theo lớp, confusion matrix, tỷ lệ đầu ra không hợp lệ và—riêng với mô hình ngôn ngữ—tỷ lệ mã định danh bằng chứng bị bịa.
- **RO2.2:** Đánh giá bằng chứng query được dự đoán so với ground truth sinh từ quan hệ phụ thuộc bằng precision, recall và F1; đồng thời kiểm tra thủ công một mẫu 20% đã xác định trước để đánh giá chất lượng ground truth.
- **RO2.3:** So sánh cửa sổ ngữ cảnh gồm query hiện tại và 0, 2 hoặc 4 query trước đó; đo hiệu năng phân loại, chất lượng bằng chứng khi có ngữ cảnh, và thời gian suy luận.

### RQ3: Khái quát hóa sang các họ kỹ thuật chưa biết

**RQ3.** Hiệu năng phân loại và độ khớp bằng chứng của hai phương pháp giảm bao nhiêu trên các họ kỹ thuật tấn công bị loại khỏi quá trình phát triển, so với các biến thể của những họ đã biết?

- **RO3.1:** Đo mức thay đổi macro-F1 và evidence F1 giữa tập đánh giá các họ đã biết và tập các họ mới được cách ly.
- **RO3.2:** Phân tích nhầm lẫn theo lớp và thay đổi về dự đoán bằng chứng không được hỗ trợ trên các họ mới.
- **RO3.3:** Khảo sát mối liên hệ giữa mức độ mới lạ của cấu trúc SQL, hàm hoặc extension đặc thù PostgreSQL, độ phức tạp của quan hệ phụ thuộc giữa các truy vấn và mức suy giảm hiệu năng.

Các nhãn tấn công là taxonomy do project xây dựng, có ánh xạ đến những khái niệm MITRE ATT&CK liên quan; đây không phải danh sách tactic chính thức được sao chép nguyên văn từ MITRE ATT&CK. Mỗi bước kịch bản sẽ có nhãn giai đoạn và liên kết phụ thuộc tường minh để sinh ground truth nhất quán cho hành động, phiên và bằng chứng.

## 1.4. Ý nghĩa của project

Project có ý nghĩa về phương pháp và thực tiễn. Về phương pháp, project đánh giá toàn bộ quy trình từ tương tác với cơ sở dữ liệu đến kết quả truy vấn có thể diễn giải. Nghiên cứu tách riêng độ bao phủ log khỏi khả năng tái dựng phiên, đồng thời tách độ chính xác nhãn khỏi chất lượng bằng chứng. Nhờ đó có thể xác định lỗi phân tích bắt nguồn từ telemetry bị thiếu, gom phiên sai, phân loại giai đoạn hay chọn bằng chứng. Việc nhóm theo họ kịch bản và cách ly các họ được giữ lại nhằm giảm rò rỉ ở cấp query, tạo phép kiểm tra khả năng khái quát hóa có ý nghĩa hơn.

Về thực tiễn, kết quả so sánh có thể hỗ trợ lựa chọn cấu hình log bằng cách làm rõ đánh đổi giữa độ bao phủ, mức đầy đủ của trường dữ liệu, dung lượng lưu trữ và độ trễ. Triển khai cục bộ cùng dữ liệu tổng hợp có vẻ nhạy cảm được thiết kế để hỗ trợ thí nghiệm có thể lặp lại mà không cần dịch vụ suy luận trả phí hoặc đưa dữ liệu thật ra ngoài. Parser dùng schema chung và bộ sinh kịch bản có thể tái lập cũng tạo nền tảng để mở rộng trong tương lai sang các hành vi PostgreSQL, bộ phân loại hoặc cấu hình log khác. Kết quả cần được hiểu là bằng chứng trong môi trường thử nghiệm có kiểm soát, không phải ước lượng trực tiếp tỷ lệ phát hiện trong mạng sản xuất.

## 1.5. Phạm vi và hạn chế

Project giới hạn trong honeypot PostgreSQL chạy cục bộ bằng Docker, được nạp dữ liệu giả và các bảng mồi có vẻ nhạy cảm. Môi trường hướng đến thí nghiệm có kiểm soát, không được thiết kế để phơi bày một dịch vụ không hạn chế ra Internet công cộng. Bộ sinh kịch bản Python có tham số sẽ thay đổi cấu trúc query, tên đối tượng, thứ tự, đặc điểm client/công cụ và độ lệch thời gian. Kịch bản được xây dựng dựa trên hành vi tấn công được ghi nhận trong tài liệu và báo cáo mối đe dọa đã xem xét; chuỗi query và quan hệ phụ thuộc cụ thể sẽ được ghi lại làm ground truth tổng hợp.

RQ1 sẽ so sánh bốn cấu hình log. Các thiết lập chính xác cần được xác định và cố định trong phần phương pháp trước khi chạy kịch bản; phần giới thiệu này không tự giả định chi tiết của một cấu hình chưa được chốt. Bản ghi sau khi parse sẽ được chuẩn hóa về schema dùng chung. Quá trình gom phiên dựa trên IP nguồn, user và các ngưỡng thời gian không hoạt động; kết quả sẽ được so sánh với cách nhóm phiên trong ground truth. Thí nghiệm ghi nhận recall hành động, độ đầy đủ trường, chỉ số pairwise cho phiên, dung lượng log và độ trễ truy vấn.

RQ2 sẽ so sánh bộ phân loại dựa trên luật với mô hình ngôn ngữ cỡ 7–8 tỷ tham số chạy cục bộ thông qua Ollama. Hai phương pháp nhận cửa sổ query đã được chuẩn hóa nhất quán; mô hình trả đầu ra có cấu trúc gồm nhãn và mã định danh bằng chứng. Tập phát triển và đánh giá được chia theo phiên hoặc họ kịch bản; tập test được giữ kín cho đến khi các phương pháp được cố định. Ground truth bằng chứng gồm query mục tiêu và các bước phụ thuộc được khai báo còn nằm trong cửa sổ ngữ cảnh. RQ3 dùng hai hoặc ba họ kỹ thuật tấn công được cách ly khỏi người viết luật và prompt, không cho phép tinh chỉnh phương pháp sau khi xem các họ này.

Các hạn chế chính gồm số lượng kịch bản hữu hạn và được tổng hợp, chất lượng ground truth phụ thuộc vào thiết kế kịch bản, và hành vi trong môi trường giả lập cục bộ có thể khác với tương tác trên cơ sở dữ liệu sản xuất. Một số lượng nhỏ họ kỹ thuật được giữ lại không thể đại diện cho mọi kiểu tấn công chưa biết. Đầu ra mô hình cũng có thể thay đổi theo phần cứng, phiên bản mô hình và thiết lập suy luận; các yếu tố này sẽ được ghi lại để tăng khả năng tái lập. Việc kiểm tra thủ công bằng chứng chỉ giới hạn trong mẫu đã định trước và không thay thế cho hoạt động gán nhãn độc lập trên quy mô lớn hơn.

## 1.6. Cấu trúc báo cáo

Báo cáo project này giới thiệu vấn đề nghiên cứu, mục tiêu, ý nghĩa và phạm vi. Các phần công việc tiếp theo được tổ chức xoay quanh ba câu hỏi nghiên cứu: RQ1 đánh giá độ bao phủ log và khả năng tái dựng phiên; RQ2 so sánh phân loại và lựa chọn bằng chứng giữa phương pháp dựa trên luật và mô hình ngôn ngữ few-shot; RQ3 kiểm tra khả năng khái quát hóa sang các họ kỹ thuật tấn công được cách ly. Các báo cáo project tiếp theo sẽ trình bày nghiên cứu liên quan, phương pháp và phần triển khai, kết quả thực nghiệm, thảo luận và kết luận khi các giai đoạn tương ứng hoàn tất.

---

## Tài liệu tham khảo

[1] Y. Song, G. Smaragdakis, and H. Griffioen, “Decoy Databases: Analyzing Attacks on Public Facing Databases,” in *Proc. 2025 ACM Internet Measurement Conf. (IMC ’25)*, Oct. 2025, pp. 709–726, doi: [10.1145/3730567.3764481](https://doi.org/10.1145/3730567.3764481).

[2] M. Wegerer and S. Tjoa, “Defeating the Database Adversary Using Deception—A MySQL Database Honeypot,” in *Proc. 2016 Int. Conf. on Software Security and Assurance (ICSSA)*, Aug. 2016, pp. 6–10, doi: [10.1109/ICSSA.2016.8](https://doi.org/10.1109/ICSSA.2016.8).

[3] A. Schärmer, M. Landauer, F. Skopik, M. Wurzenberger, and M. Squarcina, “Let the Alerts Speak: LLM-Based IDS Alert Interpretation for SOC Triage,” in *Availability, Reliability and Security. ARES 2026 International Workshops*, ser. *Lecture Notes in Computer Science*, vol. 16901, pp. 346–364. Cham, Switzerland: Springer, 2027, doi: [10.1007/978-3-032-35579-9_19](https://doi.org/10.1007/978-3-032-35579-9_19).

[4] F. Meng, J. Gui, Y. Li, and Y. Wu, “RHINO: Guided Reasoning for Mapping Network Logs to Adversarial Tactics and Techniques with Large Language Models,” *arXiv:2510.14233 [cs.CR]*, Oct. 2025. [Online]. Available: [https://arxiv.org/abs/2510.14233](https://arxiv.org/abs/2510.14233).
