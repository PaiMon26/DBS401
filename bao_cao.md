# Xây dựng honeypot cơ sở dữ liệu PostgreSQL với phân tích hành vi kẻ tấn công dựa trên AI

**BÁO CÁO ĐỒ ÁN MÔN HỌC DBS401 — Trường Đại học FPT**

Nhóm 3: Phan Thanh Thảo (SE193364), Trần Thị Minh Anh (SE190902), Vũ Trung Hiếu (SE192673)
Giảng viên: Hồ Hải — Môn học: DBS401 — TP. Hồ Chí Minh, tháng 10/2026

> Đây là bản tiếng Việt của báo cáo `report.tex` (bản gốc tiếng Anh theo mẫu IAP491, áp dụng cho báo cáo môn học). Số liệu trong các bảng được sinh tự động từ `final_results_tables.csv` và các tệp kết quả đi kèm. Quy ước: KTC = khoảng tin cậy bootstrap 95% theo kịch bản; ΔF1 dương nghĩa là điểm giảm.

---

## Tóm tắt

Honeypot cơ sở dữ liệu có thể cho thấy kẻ tấn công sử dụng cơ sở dữ liệu như thế nào, nhưng giá trị của nó phụ thuộc vào việc nhật ký ghi lại những gì và việc phân tích sau đó gán nhãn các truy vấn đã ghi một cách tin cậy đến đâu. Đồ án môn học này nghiên cứu một honeypot PostgreSQL chạy cục bộ chứa dữ liệu giả, qua ba câu hỏi: ba cấu hình ghi log ghi lại đầy đủ đến mức nào các hành động tấn công mô phỏng và cho phép dựng lại phiên ra sao; một bộ phân loại dựa trên luật hay một mô hình ngôn ngữ chạy cục bộ với vài ví dụ gán giai đoạn tấn công cho từng truy vấn chính xác hơn và chỉ ra đúng các truy vấn làm bằng chứng; và cả hai phương pháp mất bao nhiêu điểm trên các họ kỹ thuật tấn công bị giữ lại khỏi quá trình phát triển. Toàn bộ dữ liệu đến từ một trình mô phỏng có nhãn biết trước. Việc đánh giá dùng tập phát triển, một tập holdout độc lập gồm 22 kịch bản và hai tập họ kỹ thuật chưa thấy, với khoảng tin cậy bootstrap tính theo kịch bản. Cấu hình ghi log chi tiết nhất ghi đủ 136 hành động mô phỏng, trong khi hai cấu hình kia ghi 116 và 121; tuy vậy việc dựng lại phiên rất yếu (F1 theo cặp khoảng 0,11) vì trình mô phỏng dùng một địa chỉ nguồn duy nhất. Với kỹ thuật quen thuộc, phương pháp dựa trên luật có ước lượng điểm cao hơn; ở thiết lập chính với hai truy vấn đứng trước, chênh lệch khoảng 0,05 macro-F1 không phân biệt được với 0. Mọi phương pháp mất từ khoảng 0,15 đến 0,45 macro-F1 trên kỹ thuật chưa thấy. Mô hình ngôn ngữ mất ít hơn phương pháp luật trên tập chưa thấy thứ nhất và ở một thiết lập của tập thứ hai, nhưng không có bằng chứng rằng nó vượt phương pháp luật trên các kỹ thuật thật sự mới. Một phương pháp lai chuyển cho mô hình ngôn ngữ khi luật không nhận ra gì có cải thiện nhẹ, đổi lại nhiều cảnh báo sai hơn. Dữ liệu nhỏ và mô phỏng, nên kết quả cho thấy xu hướng chứ không chứng minh sự vượt trội tổng quát.

**Từ khóa:** honeypot PostgreSQL, cấu hình ghi log, dựng lại phiên, phân loại giai đoạn tấn công, mô hình ngôn ngữ few-shot, chọn bằng chứng, tổng quát hóa ngoài phân phối, khoảng tin cậy bootstrap.

## Lời cảm ơn

Nhóm cảm ơn thầy Hồ Hải đã hướng dẫn đồ án môn học DBS401 và góp ý cho đề cương.

Nhóm cũng ghi lại các công cụ và agent đã tham gia vì điều này ảnh hưởng đến mức độc lập của dữ liệu. Trợ lý AI Claude (Anthropic) hỗ trợ viết trình mô phỏng, bộ phân tích log, mã đánh giá và bộ kiểm thử; viết các kịch bản của tập họ kỹ thuật chưa thấy thứ nhất (OOD-1) trước khi đóng băng các phương pháp; và là người duy nhất rà soát bằng chứng thủ công. Một agent AI khác viết các kịch bản Holdout và tập chưa thấy thứ hai (OOD-2). Hệ quả đối với tính độc lập của dữ liệu được nêu ở mục 3.2.4 và 3.4.

## Từ viết tắt

| Ký hiệu | Nghĩa |
|---|---|
| `aid` | Mã hành động mờ, ghi dưới dạng comment SQL `/*aid=<id>*/` để đối chiếu bản ghi log với bước kịch bản đã sinh ra nó |
| C0, C1, C2 | Ba cấu hình ghi log so sánh trong RQ1 (mục 3.2.1) |
| KTC | Khoảng tin cậy bootstrap 95%, lấy mẫu lại theo kịch bản |
| D | Hiệu của hai mức giảm điểm, D = ΔF1(Rule) − ΔF1(Qwen) |
| DBMS | Hệ quản trị cơ sở dữ liệu |
| ΔF1 | Mức giảm macro-F1 từ Holdout sang tập họ chưa thấy; dương nghĩa là điểm giảm |
| H1, H3, H4 | Các chiến lược kết hợp đầu ra của luật và mô hình ngôn ngữ (mục 3.2.6) |
| Holdout | Tập kiểm tra độc lập gồm kịch bản họ đã biết, chấm một lần |
| k | Số truy vấn đứng trước trong cửa sổ phân loại |
| LLM | Mô hình ngôn ngữ lớn |
| macro-F1 | Trung bình không trọng số của F1 từng lớp |
| MITRE ATT&CK | Cơ sở tri thức về chiến thuật, kỹ thuật tấn công; nhãn của đồ án bám theo các khái niệm này |
| OOD-1, OOD-2 | Tập thứ nhất và thứ hai gồm các họ kỹ thuật tấn công bị giữ lại khỏi quá trình phát triển phương pháp |
| pgAudit | Tiện ích mở rộng ghi log kiểm toán của PostgreSQL |
| Qwen | Mô hình Qwen3 8B với prompt đóng băng B+C+E, chạy cục bộ qua Ollama |
| RQ, RO | Câu hỏi nghiên cứu, mục tiêu nghiên cứu |
| Rule | Bộ phân loại dựa trên luật |

---

# Chương 1. Giới thiệu

## 1.1 Bối cảnh

Hệ quản trị cơ sở dữ liệu là một phần của tầng dữ liệu trọng yếu trong các ứng dụng hiện đại. Khi dịch vụ cơ sở dữ liệu bị lộ ra mạng công cộng, kẻ tấn công có thể dò danh tính và cấu hình, thử xác thực, gửi truy vấn để khám phá hoặc thu thập dữ liệu, và cố thay đổi trạng thái cơ sở dữ liệu hay dùng khả năng của nó để tiếp cận tài nguyên khác. Khác với một truy vấn đơn lẻ, các hành động này có thể tạo thành chuỗi trong đó bước sau phụ thuộc vào thông tin hoặc quyền truy cập thu được ở bước trước. Hiểu hành vi như vậy đòi hỏi cả việc ghi nhận sự kiện đáng tin cậy lẫn phân tích giữ được quan hệ giữa các sự kiện.

Honeypot cơ sở dữ liệu là cách có kiểm soát để quan sát hoạt động nhắm vào một dịch vụ mồi. Song, Smaragdakis và Griffioen triển khai 278 honeypot trong 20 ngày, gồm các bản mô phỏng tương tác thấp của PostgreSQL và các DBMS khác, và ghi nhận các nỗ lực dò mật khẩu dai dẳng cùng việc khai thác ở mức tương tác trung bình/cao: thao tác dữ liệu, sao chép và xóa dữ liệu, hoặc dùng DBMS để tiếp cận hệ thống bên dưới [1]. Công trình trước đó cho thấy cách xây honeypot cơ sở dữ liệu trên MySQL mã nguồn mở và dùng cùng honeytoken để đánh lừa [2]. Các nghiên cứu này khẳng định giá trị của đánh lừa tập trung vào cơ sở dữ liệu, đồng thời đặt ra yêu cầu đo cẩn thận những gì một triển khai cụ thể quan sát được.

Dấu vết thô là nền tảng cho phân tích phía sau. Một cấu hình ghi log chỉ lấy câu lệnh mà bỏ qua thông tin kết nối, lỗi hoặc tham số có thể để lại khoảng trống trong mô tả một tương tác. Ngược lại, ghi log quá chi tiết làm tăng dung lượng lưu trữ và chi phí xử lý truy vấn, và có thể làm lộ giá trị nhạy cảm nếu dùng dữ liệu thật. Ngay cả khi từng sự kiện đã được ghi, người phân tích vẫn phải xác định sự kiện nào thuộc cùng một phiên logic và truy vấn nào là bằng chứng cho một giai đoạn tấn công.

Gần đây các mô hình ngôn ngữ lớn cũng được dùng để diễn giải cảnh báo bảo mật và ánh xạ quan sát mức thấp sang hành vi tấn công mức cao. Schärmer và cộng sự đánh giá ChatGPT và Gemini trong phân loại cảnh báo IDS, báo cáo rằng ví dụ few-shot được chọn kỹ và ngữ cảnh hệ thống giúp phân loại hiệu quả hơn, đồng thời nêu hạn chế về chi phí, độ tin cậy và lộ dữ liệu nhạy cảm [3]. Meng và cộng sự giới thiệu RHINO, ánh xạ log mạng sang chiến thuật và kỹ thuật đối kháng; họ lập luận rằng phân loại LLM một bước thường bịa tên kỹ thuật và cho ánh xạ mất ngữ cảnh, và đề xuất suy luận có hướng dẫn [4]. Những kết quả này thúc đẩy việc thử phân loại có ngữ cảnh và chọn bằng chứng trên dấu vết truy vấn cơ sở dữ liệu, thay vì cho rằng chỉ riêng nhãn của mô hình là lời giải thích đáng tin.

## 1.2 Phát biểu vấn đề

Dù honeypot và phân tích bảo mật có hỗ trợ của mô hình ngôn ngữ đã có liên quan, ba câu hỏi thực tiễn vẫn còn đối với honeypot PostgreSQL. Thứ nhất, các cấu hình ghi log có thể khác nhau về khả năng ghi các hành động cần thiết để nhận diện bảy lớp hành vi do đồ án định nghĩa: Discovery, Credential Access, Privilege Escalation, Collection, Exfiltration, Impact và Noise. Sáu lớp mô tả giai đoạn tấn công và một lớp mô tả hoạt động lành tính. Đồ án phải định lượng các khác biệt này và xác định các bản ghi có chứa các trường cần cho phân tích sau hay không. Thứ hai, các dòng log có thể không trực tiếp xác định một phiên tấn công hoàn chỉnh. Việc nhóm phiên theo ngưỡng không hoạt động có thể gộp hoạt động không liên quan hoặc tách một chuỗi nhiều bước thật, và độ chính xác của nó phải được đo so với ground truth của kịch bản đã biết. Thứ ba, phân loại ở mức truy vấn khó khi ý nghĩa của truy vấn phụ thuộc vào sự kiện trước đó. Phương pháp dựa trên luật minh bạch nhưng giòn trước biến thể; mô hình ngôn ngữ few-shot có thể dùng ngữ cảnh linh hoạt hơn nhưng có thể sinh nhãn hoặc mã bằng chứng không có cơ sở. Không phương pháp nào được coi là tin cậy nếu chưa đo cả chất lượng phân loại lẫn chất lượng bằng chứng.

Một thách thức nữa là tổng quát hóa. Một phương pháp có thể chạy tốt trên các biến thể giống kịch bản phát triển nhưng thất bại với họ kỹ thuật không được dùng để viết luật hay ví dụ. Để đánh giá sự khác biệt này, việc đánh giá cần một phép chia có kiểm soát theo họ kịch bản, với các họ giữ lại được cách ly khỏi quá trình phát triển, và so sánh cả bằng chứng truy vấn dùng để biện minh cho nhãn chứ không chỉ nhãn. Không có sự tách biệt này, hiệu năng biểu kiến có thể phản ánh rò rỉ hoặc mẫu quen thuộc thay vì khả năng phân tích hành vi chưa thấy.

Do đó đồ án xây dựng một quy trình đánh giá cục bộ, tái lập được, nối ghi log PostgreSQL, dựng lại phiên, phân loại truy vấn và đánh giá bằng chứng, rồi dùng nó trả lời các câu hỏi nghiên cứu bên dưới. Đồ án đo mức độ phù hợp của dấu vết tương tác thu được cho phân tích hành vi; nó không khẳng định rằng các kịch bản tổng hợp đại diện cho toàn bộ sự đa dạng của kẻ tấn công thật.

## 1.3 Mục tiêu nghiên cứu

Câu hỏi bao trùm của đồ án: *Một honeypot PostgreSQL tương tác cao kết hợp phân tích hành vi có thể ghi nhận, nhận diện và giải thích các giai đoạn tấn công từ log tương tác đến mức nào, kể cả khi kẻ tấn công dùng kỹ thuật chưa từng thấy?* Ba câu hỏi nghiên cứu tạo thành một chuỗi: RQ1 chọn cấu hình ghi log cung cấp log cho RQ2, và các phương pháp được đóng băng trong RQ2 là những phương pháp được thử trên kỹ thuật chưa thấy trong RQ3.

### RQ1: Ghi log và dựng lại phiên

**RQ1.** Cấu hình ghi log nào của honeypot PostgreSQL bỏ sót hoặc dựng sai loại hành động và phiên tấn công nào, ở mức độ nào, và cấu hình nào đủ cho phân tích hành vi tiếp theo?

- **RO1.1:** Đo recall mức hành động cho từng lớp hành vi tấn công và lành tính qua ba cấu hình ghi log (C0, C1, C2), và tách theo giai đoạn và theo các thẻ hiển thị log ("gap tag") gắn với mỗi hành động.
- **RO1.2:** Đánh giá sáu thuộc tính phiên — người dùng, địa chỉ IP nguồn, cơ sở dữ liệu, truy vấn, kết quả và tham số — có mặt trong các bản ghi log đã phân tích hay không, báo cáo theo từng trường, và đánh giá việc dựng lại phiên bằng precision, recall và F1 theo cặp ở các ngưỡng không hoạt động 1, 5, 15 và 30 phút.
- **RO1.3:** Chọn cấu hình tối thiểu thỏa tiêu chí áp dụng trong đồ án (recall ít nhất 95% tổng thể và ở từng giai đoạn; ngưỡng do nhóm chọn, không phải chuẩn), xét độ phủ ghi nhận, độ đầy đủ trường và dung lượng log đo bằng số bản ghi.

### RQ2: Phân loại và bằng chứng

**RQ2.** Giữa phương pháp dựa trên luật và một mô hình ngôn ngữ few-shot chạy cục bộ, phương pháp nào gán giai đoạn tấn công cho từng truy vấn chính xác hơn khi có ngữ cảnh truy vấn đứng trước, và bằng chứng mà mỗi phương pháp chọn khớp với bằng chứng đã biết đến đâu?

- **RO2.1:** So sánh hiệu năng phân loại và độ tin cậy kỹ thuật của đầu ra bằng macro-F1, F1 từng lớp, ma trận nhầm lẫn, tỉ lệ đầu ra không hợp lệ (không phân tích được hoặc vi phạm schema JSON, đếm riêng) và, với mô hình ngôn ngữ, tỉ lệ mã bằng chứng bịa (mã không ứng với truy vấn nào trong cửa sổ).
- **RO2.2:** Đánh giá bằng chứng dự đoán so với ground truth suy ra từ phụ thuộc. Chỉ số Jaccard là thước đo chính; precision, recall và F1 của tập bằng chứng được báo cáo thêm. Rà soát thủ công mẫu 20% để đánh giá chất lượng ground truth.
- **RO2.3:** So sánh cửa sổ ngữ cảnh gồm truy vấn hiện tại và 0, 2 hoặc 4 truy vấn đứng trước, đo phân loại, chất lượng bằng chứng khi có ngữ cảnh và thời gian suy luận.

### RQ3: Tổng quát hóa sang họ kỹ thuật chưa thấy

**RQ3.** Hiệu năng phân loại và khớp bằng chứng của hai phương pháp giảm bao nhiêu trên các họ kỹ thuật tấn công bị loại khỏi quá trình phát triển, so với các biến thể của họ đã biết?

- **RO3.1:** Đo thay đổi macro-F1 giữa tập đánh giá họ đã biết và từng tập họ chưa thấy được cách ly, và mô tả thay đổi tương ứng của mức khớp bằng chứng.
- **RO3.2:** Phân tích nhầm lẫn từng lớp, số cửa sổ bị gán Noise và mã bằng chứng bịa trên họ chưa thấy.
- **RO3.3:** Xem xét cách độ lạ về cấu trúc SQL, hàm/extension riêng của PostgreSQL và độ phức tạp của phụ thuộc giữa các truy vấn liên hệ với mức suy giảm hiệu năng, ở dạng mô tả định tính.

Các nhãn tấn công là phân loại riêng của đồ án, căn theo các khái niệm MITRE ATT&CK liên quan; không được trình bày như bộ chiến thuật chính thức nguyên văn. Mỗi bước kịch bản mang nhãn giai đoạn tường minh và liên kết phụ thuộc để ground truth hành động, phiên và bằng chứng được sinh nhất quán. Chỉ RQ1 dùng cả bảy nhãn; RQ2 và RQ3 phân loại sáu nhãn (mục 3.2.2).

## 1.4 Ý nghĩa của nghiên cứu

Đồ án có ý nghĩa về phương pháp và thực tiễn. Về phương pháp, nó đánh giá toàn bộ đường đi từ tương tác cơ sở dữ liệu đến kết quả mức truy vấn có thể diễn giải. Nó tách độ phủ ghi log khỏi dựng lại phiên, và tách độ chính xác nhãn khỏi chất lượng bằng chứng. Nhờ vậy có thể xác định lỗi phân tích bắt nguồn từ thiếu dữ liệu đo, nhóm phiên sai, phân loại giai đoạn hay chọn bằng chứng. Việc dùng nhóm theo họ kịch bản và các họ giữ lại cách ly, cùng khoảng tin cậy tính theo kịch bản, nhằm hạn chế rò rỉ mức truy vấn và làm lộ độ bất định của mẫu nhỏ.

Về thực tiễn, nghiên cứu chỉ ra các giới hạn quan trọng với ai xây hệ thống tương tự: log đầy đủ ở mức hành động chưa đủ để dựng lại phiên trong bối cảnh này; phương pháp luật và mô hình chạy cục bộ không tách biệt được trên kỹ thuật quen thuộc ở thiết lập chính; và cả hai đều giảm độ chính xác trên kỹ thuật lạ. Triển khai cục bộ với dữ liệu giả cho phép thí nghiệm lặp lại được mà không cần dịch vụ suy luận trả phí hay lộ dữ liệu thật của tổ chức. Kết quả là bằng chứng về môi trường thử nghiệm có kiểm soát, không phải ước lượng tỉ lệ phát hiện trong mạng thực.

## 1.5 Phạm vi và hạn chế

Đồ án giới hạn ở một honeypot PostgreSQL chạy cục bộ trong Docker, chứa dữ liệu giả và các bảng mồi trông như nhạy cảm. Môi trường không được mở ra Internet công cộng. Một bộ sinh kịch bản Python tham số hóa thay đổi cấu trúc truy vấn, tên đối tượng, thứ tự, đặc điểm client/công cụ và độ trễ thời gian. Các kịch bản dựa trên hành vi tấn công được báo cáo trong tài liệu đã rà soát, gồm dò mật khẩu, truy cập dữ liệu sau xác thực và các tính năng thực thi chương trình do Song và cộng sự nêu [1], và khám phá thông tin xác thực sau khai thác do Gómez-López và cộng sự mô tả [5]. Chuỗi truy vấn và liên kết phụ thuộc chính xác của chúng là ground truth tổng hợp, không phải bản sao các chiến dịch đã quan sát.

**Các cấu hình ghi log.** RQ1 so sánh ba cấu hình: C0 đặt `log_statement = all`; C1 bật ghi phiên pgAudit; C2 mở rộng C1 với `log_connections`, `log_disconnections` và `pgaudit.log_parameter`. Cấu hình thứ tư dựa trên proxy hoặc bắt gói tin được cân nhắc lúc đầu và nằm ngoài phạm vi. Các bản ghi được chuẩn hóa về schema chung và phiên được nhóm theo IP nguồn, người dùng và ngưỡng không hoạt động.

**Phân loại và bằng chứng.** RQ2 so sánh bộ phân loại dựa trên luật với mô hình ngôn ngữ chạy cục bộ cỡ khoảng tám tỉ tham số (Qwen3 8B) phục vụ qua Ollama. Cả hai nhận các cửa sổ truy vấn được chuẩn hóa nhất quán. Mô hình trả về JSON gồm nhãn và danh sách mã bằng chứng; schema được yêu cầu trong prompt và kiểm tra sau đó, không ép khi giải mã, để đo được mã bịa thay vì loại trừ chúng từ đầu. Ground truth bằng chứng gồm truy vấn đích và các phụ thuộc khai báo còn nằm trong cửa sổ.

**Các hạn chế áp dụng cho mọi kết quả.**

- Toàn bộ dữ liệu là mô phỏng, kể cả lưu lượng Noise lành tính. Kịch bản dựa trên Song và cộng sự và một báo cáo của hãng; chúng không phải ghi lại kẻ tấn công thật.
- Mẫu nhỏ: 102 truy vấn phát triển, 109 truy vấn Holdout, và hai tập họ chưa thấy với 57 và 145 truy vấn từ chỉ ba và sáu họ. Khoảng tin cậy rộng, và chênh lệch nhỏ hơn khoảng tin cậy của nó không hỗ trợ kết luận nào.
- Trình mô phỏng chạy từ một địa chỉ nguồn duy nhất, nên việc nhóm phiên không thể thử thực tế. Ở tập phát triển và Holdout, ngữ cảnh của mỗi cửa sổ lấy theo mã kịch bản thật (ngữ cảnh lý tưởng).
- Việc cách ly dữ liệu kiểm tra chưa hoàn toàn: Claude đã đọc một phần nội dung Holdout khi kiểm tra lỗi, và SQL của tập chưa thấy thứ nhất (OOD-1), do Claude viết trước khi đóng băng phương pháp, xuất hiện trong cuộc trò chuyện làm việc sau khi đóng băng (không dùng để điều chỉnh gì). Ảnh hưởng của hai sự kiện này không được đo; nhóm chấp nhận rủi ro, đó là một quyết định chứ không phải một phép đo.
- Prompt của mô hình ngôn ngữ được tinh chỉnh trên tập phát triển, nên điểm phát triển lạc quan.
- Một số bước kịch bản thất bại vì lý do môi trường (ví dụ lệnh `COPY ... PROGRAM` chạm ra ngoài container, một extension không được nạp, hoặc `LOCK` ngoài giao dịch); câu lệnh vẫn được ghi log và khớp.
- Rà soát bằng chứng thủ công chỉ có một người, là AI (Claude), không phải thành viên nhóm.
- Chưa làm: đo CPU và bộ nhớ khi suy luận, đo overhead bằng pgbench, và nhóm kịch bản "né log". Ngoài phạm vi: cấu hình proxy/bắt gói và thí nghiệm nhiều địa chỉ nguồn.
- Một số ngưỡng là lựa chọn riêng của nhóm và chưa được xác nhận là chuẩn: định nghĩa cấu hình "nhẹ", tiêu chí recall 95% và việc áp dụng cho tổng hay từng giai đoạn, và nghĩa của "đủ thuộc tính".

## 1.6 Cấu trúc báo cáo

Báo cáo môn học này theo cấu trúc sáu phần của hướng dẫn môn. Chương 1 giới thiệu vấn đề, mục tiêu, ý nghĩa và phạm vi. Chương 2 tổng quan các nghiên cứu liên quan. Chương 3 mô tả phương pháp: thiết kế, dữ liệu, kỹ thuật phân tích và hạn chế. Chương 4 trình bày và diễn giải kết quả thực nghiệm cho RQ1–RQ3. Chương 5 thảo luận phát hiện và giới hạn của chúng; Chương 6 kết luận và đề xuất hướng phát triển. Các phụ lục chứa bảng bổ sung, các phát hiện kiểm toán, bảng kiểm tuyên bố, đầu ra gốc của các script thống kê và lệnh tái lập.

---

# Chương 2. Tổng quan tài liệu

## 2.1 Các nghiên cứu trước

### 2.1.1 Phạm vi tổng quan

Phần này xem xét các nghiên cứu hỗ trợ những giai đoạn chính của đồ án: triển khai honeypot cơ sở dữ liệu và hành vi kẻ tấn công, ghi log kiểm toán và dựng lại tấn công, diễn giải dữ liệu bảo mật dựa trên học máy, tổng quát hóa sang hành vi tấn công chưa thấy, và đánh giá độ thật của honeypot. Các nghiên cứu khác nhau về nền tảng, mức tương tác, loại dữ liệu và bối cảnh đánh giá. Vì vậy kết quả của chúng chỉ định hướng thiết kế đồ án chứ không dự đoán hiệu năng trên dấu vết truy vấn PostgreSQL có kiểm soát. Khi một nguồn chỉ có tiêu đề hoặc tóm tắt ngắn, mô tả bên dưới chỉ giới hạn trong điều nguồn đó nêu.

### 2.1.2 Honeypot cơ sở dữ liệu và hành vi kẻ tấn công

Song, Smaragdakis và Griffioen triển khai 278 honeypot trong 20 ngày vào tháng 3–4/2024 [1]. 220 honeypot tương tác thấp mô phỏng MySQL, MSSQL, PostgreSQL và Redis cho thấy hoạt động quét tương đối thấp nhưng các nỗ lực dò mật khẩu dai dẳng. 58 honeypot tương tác trung bình và cao cho thấy ba kiểu khai thác: thao tác trực tiếp cơ sở dữ liệu, sao chép và xóa dữ liệu nhằm tống tiền, và dùng DBMS làm đường chiếm hệ thống bên dưới. Đây là nghiên cứu thực nghiệm liên quan trực tiếp nhất với đồ án vì bao gồm PostgreSQL và hành vi nhiều bước. Triển khai của họ nối Internet và khác thí nghiệm cục bộ có kiểm soát ở đây, nên khối lượng và tần suất quan sát được không được xem là kết quả kỳ vọng của đồ án.

Rocha và Pereira dùng công cụ honeypot để mô tả các cuộc tấn công vào DBMS quan hệ và báo cáo MySQL là hệ bị nhắm nhiều nhất trong số được phân tích [6]. Nghiên cứu xem xét mồi MySQL, PostgreSQL và Oracle, nhưng kết quả mức độ bị nhắm là riêng của triển khai của họ và không chuyển sang PostgreSQL trong bối cảnh này. Giá trị với đồ án là động cơ cho việc đo riêng theo DBMS.

Wegerer và Tjoa khảo sát honeypot cơ sở dữ liệu và honeytoken, và mô tả một honeypot MySQL xây bằng thành phần mã nguồn mở, gồm plugin kiểm toán ghi truy vấn và lần đăng nhập [2]. Công trình cho thấy cơ chế kiểm toán cơ sở dữ liệu là mối quan tâm thiết kế trung tâm của honeypot cơ sở dữ liệu, dù cài đặt riêng cho MySQL và plugin kiểm toán không thể coi là áp dụng nguyên vẹn cho PostgreSQL.

Các công trình tập trung vào honeypot PostgreSQL và MS SQL coi tương tác sau xác thực là đối tượng cần ghi. Al Hafidz và cộng sự nghiên cứu nâng cao phát hiện trong honeypot shell PostgreSQL bằng phản hồi truy vấn tương tác [7]. Karo Karo và cộng sự nghiên cứu ghi hành vi sau đăng nhập ở honeypot MS SQL qua relay TDS nhận biết giao thức [8]. Cả hai tiêu đề cho thấy điều xảy ra sau xác thực là bài toán ghi nhận trung tâm, đúng mối quan tâm của RQ1. Kết quả đánh giá đầy đủ của họ không có sẵn cho tổng quan này nên không rút ra so sánh định lượng.

Gómez-López và cộng sự trình bày SQLiAgent, khung đa agent kiểm toán SQL injection, trong đó chế độ hậu khai thác có LLM hỗ trợ diễn giải schema cơ sở dữ liệu, xác định bảng chứa thông tin xác thực và sinh câu lệnh SQL leo thang đặc quyền [5]. Công trình mang tính tấn công và được kiểm chứng trên môi trường OWASP Broken Web Applications, nhưng nó ghi lại các hành vi mà kịch bản Credential Access và Privilege Escalation của đồ án phải biểu diễn. Nó cũng cho thấy công cụ có LLM hỗ trợ có thể sinh các câu lệnh đó từ cấu trúc schema suy ra, điều liên quan đến mô hình mối đe dọa.

### 2.1.3 Ghi log kiểm toán, phân tích phiên và dựng lại tấn công

Vết kiểm toán cơ sở dữ liệu cung cấp bản ghi có cấu trúc hỗ trợ phân tích vượt ra ngoài việc xem từng truy vấn. Zimniak, Getta và Benn biến đổi thông tin vết kiểm toán để phát hiện mẫu chu kỳ trong khối lượng công việc [9]. Công trình nói về dự báo tải chứ không phải phát hiện bảo mật, nhưng cho thấy giá trị của việc giữ thứ tự và thời gian khi phân tích chuỗi thao tác cơ sở dữ liệu.

Biswas và Niemi xem xét cách yêu cầu kiểm toán quyết định log nào phải thu thập [10]. Họ nêu một đánh đổi thực tế: ghi log không đủ thì không kiểm toán tin cậy được, còn ghi thừa thì tăng chi phí và overhead hiệu năng cơ sở dữ liệu. Điều này thúc đẩy việc so sánh các cấu hình ghi log của đồ án theo độ phủ hành động, độ đầy đủ trường và dung lượng log. Zeng, Xiao và Chen cũng nghiên cứu overhead kiểm toán và thích ứng trong khung kiểm toán Linux, đề xuất ghi log thích ứng thay đổi loại và tần suất sự kiện được thu [11]. Dù không riêng PostgreSQL, công trình cho thấy overhead đo đạc là chi phí đáng kể của ghi log chi tiết; đồ án chỉ đo dung lượng log và ghi việc đo overhead là hạn chế còn mở.

Choi và Lee phân tích cấu trúc nội bộ của log giao dịch Microsoft SQL Server trong vùng đĩa chưa cấp phát [12]. Phân tích cho thấy INSERT, UPDATE, DELETE tương ứng một-một với bản ghi log, trong khi CREATE và DROP TABLE được biểu diễn bằng tổ hợp nhiều bản ghi trong một giao dịch. Với đồ án, phát hiện này là lời cảnh báo cho thiết kế bộ phân tích: một truy vấn có thể sinh nhiều bản ghi và có truy vấn không sinh bản ghi nào, nên bộ phân tích không được giả định tương ứng cố định giữa truy vấn và bản ghi.

Goseva-Popstojanova và cộng sự mô tả lưu lượng web độc hại bắt được bởi honeypot tương tác cao chạy hệ thống web ba tầng đầy đủ chức năng [13]. Họ mô tả phiên độc hại bằng 43 thuộc tính phiên, gồm thời lượng, số yêu cầu và số byte, và cho thấy bộ học có giám sát tách phiên tấn công khỏi phiên quét lỗ hổng với tỉ lệ phát hiện cao và báo động sai thấp. Việc dùng thuộc tính mức phiên thúc đẩy các đặc trưng phiên và đánh giá mức phiên trong RQ1, dù miền của họ là HTTP chứ không phải truy vấn cơ sở dữ liệu.

Landauer và cộng sự giới thiệu CAM-LDS, tập log có nhãn gồm bảy kịch bản tấn công bao phủ 81 kỹ thuật thuộc 13 chiến thuật, thu trong môi trường mã nguồn mở và tái lập được [14]. Nghiên cứu minh họa của họ dùng LLM diễn giải log và báo cáo rằng kỹ thuật tấn công đúng được dự đoán hoàn hảo cho khoảng một phần ba bước tấn công và vừa đủ cho một phần ba nữa. Kết quả gợi ý diễn giải log bằng LLM thành công một phần và một tỉ lệ đáng kể bước vẫn khó, điều này ủng hộ việc báo cáo kết quả từng lớp chứ không chỉ điểm tổng hợp.

SLEUTH dựng lại kịch bản tấn công theo thời gian thực từ dữ liệu kiểm toán thương mại phổ thông bằng đồ thị phụ thuộc trong bộ nhớ và các kỹ thuật gắn thẻ để xác định nguồn và phân tích tác động [15]. TraceBack cũng dựng lại kịch bản tấn công từ đồ thị nguồn gốc, dùng phân tích nhân quả và mô hình chuỗi để giảm bùng nổ phụ thuộc và báo động sai [16]. Các hệ thống này làm việc trên dữ liệu máy chủ và nguồn gốc chứ không phải vết truy vấn cơ sở dữ liệu, nhưng ủng hộ việc mô hình hóa sự kiện thành các bước phụ thuộc nhau. Zambianco và cộng sự cho thấy dùng tiền điều kiện và hệ quả thực thi của các kỹ thuật ATT&CK để mô hình hóa đường tấn công cho phép chọn mồi tính đến các phụ thuộc logic giữa các bước [17]. Đây là cơ sở khái niệm cho các chú thích `depends_on` dùng để sinh ground truth bằng chứng.

### 2.1.4 Diễn giải bằng LLM và ánh xạ kỹ thuật

Schärmer và cộng sự đánh giá LLM trong diễn giải cảnh báo IDS, gồm phân biệt cảnh báo tấn công với báo động sai và gắn cảnh báo với kỹ thuật tấn công [3]. Họ xem xét log bổ sung, dữ liệu cấu hình và ví dụ few-shot ảnh hưởng thế nào đến hiệu năng và tính nhất quán, và kết luận rằng ví dụ chọn kỹ và ngữ cảnh hệ thống hỗ trợ phân loại hiệu quả. Đầu vào của họ là cảnh báo mạng và máy chủ chứ không phải câu lệnh SQL, nên phát hiện phải được kiểm chứng trên dữ liệu truy vấn trước khi giả định áp dụng ở đây.

Meng và cộng sự giới thiệu RHINO, ánh xạ log mạng sang chiến thuật và kỹ thuật đối kháng bằng suy luận có hướng dẫn [4]. Tóm tắt của họ báo cáo các phương pháp LLM hiện có thường bịa tên kỹ thuật hoặc cho ánh xạ mất ngữ cảnh vì phân loại một bước, và các thất bại này thúc đẩy cách tiếp cận dựa trên suy luận. Meng và cộng sự cũng giới thiệu CYBERTEAM, benchmark tổ chức săn mối đe dọa thành quy trình chuẩn gồm 30 tác vụ và 9 mô-đun vận hành, và báo cáo cấu trúc này cải thiện kết quả so với các chiến lược suy luận mở như in-context learning, chain-of-thought và tree-of-thought [18]. Cả hai thúc đẩy việc thử xem ngữ cảnh có cấu trúc có cải thiện phân loại mức truy vấn không và bằng chứng được chọn có thật sự được các sự kiện đã ghi hỗ trợ không.

Yang và cộng sự đề xuất APTREC, dùng LLM nhận diện chiến thuật và kỹ thuật APT từ các đồ thị con tấn công qua kiến trúc hai giai đoạn: mô-đun nhận diện kỹ thuật và mô-đun suy luận chiến thuật [19]. Họ báo cáo cải thiện 13% F1 và tăng 9% recall so với các phương pháp hiện có tiêu biểu, kèm khả năng giải thích. Thiết kế hai giai đoạn là đối chiếu hữu ích với prompt một bước dùng trong RQ2, dù đầu vào đồ thị con khác với các cửa sổ truy vấn ở đây.

Wang và cộng sự nghiên cứu nhận diện đa nhãn zero-shot các kỹ thuật tấn công, trong đó suy luận của LLM được tăng cường bằng khám phá bằng chứng ngữ nghĩa [20]. Liên quan đến RQ2 ở chỗ liên kết tường minh giữa nhãn dự đoán và bằng chứng hỗ trợ. Tóm tắt không có sẵn nên mô tả giới hạn ở tiêu đề.

Neves và cộng sự so sánh LLM zero-shot với phương pháp dựa trên luật trong xác định kiểu hình lâm sàng từ tóm tắt bệnh nhân tổng hợp [21]. GPT-4o đạt recall 0,97 và macro-F1 0,92, trong khi phương pháp luật rất chính xác (0,92) nhưng recall chỉ 0,36. Các tác giả cũng chỉ ra kết hợp luật và LLM có thể hướng việc rà soát thủ công vào các ca bất đồng. Đây là tương tự đã công bố gần nhất với thiết kế RQ2: nó so sánh hai cách tiếp cận trên một tác vụ phân loại có cấu trúc, và kết quả luật và LLM thất bại theo cách bổ sung cho nhau gợi ý nên phân tích chung đầu ra của hai phương pháp thay vì chỉ xếp hạng.

Duan và cộng sự khảo sát học few-shot cho phát hiện xâm nhập và thảo luận tiềm năng khi ví dụ có nhãn khan hiếm cùng các thách thức mở [22]. Đồ án dùng prompt few-shot với mô hình cố định chạy cục bộ thay vì huấn luyện phương pháp học few-shot mới, nên khảo sát đóng vai nền tảng cho phân loại bảo mật ít ví dụ chứ không phải phương pháp để tái tạo.

### 2.1.5 Học máy cổ điển và LLM trên dữ liệu bảo mật có cấu trúc

Liu và Lang khảo sát các phương pháp học máy và học sâu cho phát hiện xâm nhập, gồm phân loại, thuật toán, tập dữ liệu và các thách thức về độ chính xác, tỉ lệ báo động sai và tấn công chưa biết [23]. Alharthi và cộng sự so sánh các mô hình học máy cổ điển và học sâu trên tập xâm nhập nhị phân và đa lớp, và thấy mô hình mạnh nhất phụ thuộc cả tập dữ liệu và tác vụ [24]. Các nghiên cứu này ủng hộ việc báo cáo macro-F1 và kết quả từng lớp và đưa vào các baseline minh bạch. Không gian đặc trưng của chúng khác log SQL, nên số liệu không chuyển được sang tác vụ của đồ án.

Rehman và cộng sự tuần tự hóa luồng mạng thành văn bản gọn và nghiên cứu prompt zero-shot, có hướng dẫn và few-shot với đầu ra ràng buộc ngữ pháp và ngưỡng quyết định hiệu chỉnh [25]. Họ thấy prompt không hướng dẫn không tin cậy và hướng dẫn cộng cờ boolean cải thiện đáng kể phát hiện, trong khi chất lượng quyết định giảm khi tập đánh giá tăng từ 200 lên 2000 luồng. Baseline dạng bảng ổn định và nhanh hơn. Mehavilla và cộng sự so LLM với mô hình học máy và học sâu trên log Zeek và báo cáo LLM đạt F1 trên 95% nhưng không vượt mô hình cổ điển tốt nhất, XGBoost, đạt 96,96% chỉ dùng 4% CPU [26]. Cùng nhau, các nghiên cứu này cảnh báo không nên giả định LLM luôn ưu việt cho phân loại có cấu trúc, và thúc đẩy so sánh trực tiếp với cùng đầu vào và cùng phép chia.

### 2.1.6 Tổng quát hóa sang hành vi tấn công chưa thấy

Tổng quát hóa là mối quan tâm trung tâm của RQ3. Lust và Condurache khảo sát các phương pháp đánh giá liệu đầu ra của mạng nơ-ron sâu có đáng tin cho một đầu vào, gom độ bất định dự đoán, phát hiện ngoài phân phối và phát hiện mẫu đối kháng vào câu hỏi rộng hơn về hiệu năng tổng quát hóa [27]. Khung này ủng hộ việc phân biệt biến thể của họ kỹ thuật đã biết với họ chưa thấy, và gợi ý nên đo hành vi ngoài phân phối thay vì giả định. Zhou và Wang báo cáo mạng transformer khó tổng quát hóa sang dữ liệu ngoài phân phối và đề xuất GROD, dùng điểm ngoại lai tổng hợp và phạt phân loại sai ngoài phân phối khi huấn luyện [28]. Phương pháp dành cho huấn luyện nơ-ron chứ không phải LLM dùng prompt, nên không áp dụng trực tiếp, nhưng khẳng định tổng quát hóa của transformer sang phân phối chưa thấy là vấn đề đã được thừa nhận. Guo và cộng sự đề cập tổng quát hóa ngoài biến cho các mô hình phân biệt [29]; chỉ có tiêu đề của công trình này cho tổng quan, nên không rút ra phát hiện nào.

Sự khan hiếm dữ liệu nhiều giai đoạn có nhãn là một trở ngại liên quan. Das và Nair công bố tập dữ liệu lưu lượng nhà thông minh bình thường và bảy kịch bản tấn công nhiều giai đoạn, xuất phát từ nhận xét rằng tấn công nhiều giai đoạn khó phát hiện hơn [30]. Miền của họ là luồng mạng, nhưng bài báo minh họa vì sao đồ án phải tự sinh các kịch bản truy vấn có nhãn, có chú thích phụ thuộc và ghi chi tiết quy trình sinh.

### 2.1.7 Độ thật của honeypot, dữ liệu mồi tổng hợp và đánh giá

Giá trị của honeypot phụ thuộc vào chiều sâu và chất lượng các tương tác nó duy trì. Nunez Migliorisi đề xuất các thước đo độ thật và hiệu quả đánh lừa dựa trên telemetry, gồm thời gian lưu lại của kẻ tấn công, chiều sâu tương tác, đa dạng hành vi và độ phong phú họ mã độc, và lập luận rằng chỉ đếm kết nối là không đủ [31]. Ferguson-Walter và cộng sự phân tích một thí nghiệm có kiểm soát với hơn 130 red teamer chuyên nghiệp và so sánh các thước đo tiến triển qua bốn điều kiện xác định bởi có mồi hay không và kẻ tấn công có được báo về đánh lừa hay không. Họ báo cáo tổ hợp mồi và thông tin về đánh lừa có tác động lớn nhất đến hành vi kẻ tấn công so với đối chứng [32]. Các công trình này ủng hộ việc dùng nhiều chỉ báo mức độ tương tác, dù đồ án hiện chủ yếu đánh giá khả năng quan sát và chất lượng phân tích trong kịch bản có kiểm soát.

Giá trị mồi tổng hợp cũng cần kiểm chứng. Weinberg đề xuất PHANTOM, sinh honeytoken có điều kiện theo quy ước đặt tên và phân phối giá trị riêng của tổ chức và đánh giá bằng điểm đáng tin kết hợp tính hợp lệ cú pháp, mạch lạc ngữ nghĩa, hợp lý thống kê và chấp nhận của con người [33]. Cách phân rã đó có thể định hướng việc kiểm tra tính hợp lý của các bảng nhạy cảm tổng hợp trong đồ án, dù bản ghi của đồ án không phải honeytoken và điểm không được tái tạo đầy đủ. Msaad và cộng sự cho thấy kẻ tấn công tiên tiến có thể nhận dạng vân tay phần lớn honeytoken mã nguồn mở phổ biến mà không kích hoạt cảnh báo [34], một hạn chế đồ án phải thừa nhận.

Đánh lừa thích ứng và dựa trên LLM cho thấy hướng đi của các công trình gần đây. Kouremetis và cộng sự giới thiệu Mirage, hệ thống mô phỏng và giả lập để thử các đánh lừa trước đối thủ dùng tìm kiếm và lập kế hoạch AI [35]. Wushishi và cộng sự đề xuất D3O-IIoT, phương pháp học tăng cường phối hợp triển khai honeypot, phòng thủ mục tiêu di động và telemetry giả trong mạng IoT công nghiệp [36]. Safargalieva và cộng sự giới thiệu OHRA, honeypot điều khiển bằng LLM hỗ trợ nhiều giao thức, trong đó SSH, Telnet và HTTP tương tác đầy đủ còn FTP, SMTP, IPP và SNMP cài một phần [37]. Bridges và cộng sự hệ thống hóa nghiên cứu về honeypot và LLM, đưa ra phân loại các vector phát hiện, kiến trúc chuẩn cho honeypot chạy bằng LLM, khung đánh giá và thảo luận phân tích log như đường dẫn đến tình báo tự động [38]. Tổng quan của họ nêu bật sự thiếu hụt dữ liệu có nhãn thực tế và rủi ro đối thủ thông minh phát hiện hành vi do LLM điều khiển. Đồ án dùng LLM để phân tích vết truy vấn đã ghi thay vì sinh phản hồi, nên độ trung thực của phản hồi không phải kết quả đo, nhưng tổng quan ủng hộ các tiêu chí đánh giá tường minh và cảnh báo không coi đầu ra LLM là bằng chứng tự xác nhận.

## 2.2 Tóm tắt tổng quan tài liệu

Các nghiên cứu honeypot cơ sở dữ liệu đã xem xét cho thấy mồi riêng theo DBMS có thể phơi bày các nỗ lực xác thực dai dẳng và hành vi nhiều bước sâu hơn, và mức tương tác cùng cơ chế ghi quyết định điều gì quan sát được [1, 7, 8]. Các nghiên cứu về vết kiểm toán và dựng lại cho thấy vì sao thứ tự sự kiện, thời gian, phụ thuộc và độ phủ trường quan trọng, đồng thời xác định overhead ghi log là chi phí đáng kể [10, 11, 12]. Nghiên cứu về LLM cho dữ liệu bảo mật cung cấp baseline hữu ích, chiến lược prompt có ngữ cảnh, thiết kế hướng bằng chứng và thực hành đánh giá, nhưng kết quả trên luồng mạng, cảnh báo và văn bản lâm sàng không xác lập hiệu năng trên vết truy vấn SQL [3, 4, 21, 25]. Các nghiên cứu tổng quát hóa thúc đẩy việc cô lập họ chưa thấy thay vì dựa vào chia ngẫu nhiên [27], và các nghiên cứu đánh giá honeypot cho thấy mức tương tác và độ thật đòi hỏi nhiều hơn việc đếm kết nối [31, 32].

Vẫn còn một khoảng trống ở giao điểm các lĩnh vực này. Các nguồn đã xem xét không xác lập cấu hình ghi log PostgreSQL nào đủ để dựng lại các phiên truy vấn có nhãn trong honeypot có kiểm soát, cũng không xác lập liệu một LLM few-shot cục bộ có cải thiện phân loại giai đoạn truy vấn và chọn bằng chứng so với phương pháp luật khi đánh giá trên các họ kỹ thuật bị giữ lại khỏi phát triển hay không. Đồ án giải quyết khoảng trống này bằng cách đánh giá độ phủ ghi nhận, dựng lại phiên, phân loại, chất lượng bằng chứng và tổng quát hóa trong một quy trình tái lập duy nhất. Các kết luận của báo cáo áp dụng cho cấu hình và kịch bản đã thử, không áp dụng trực tiếp cho mọi cơ sở dữ liệu sản xuất hay kẻ tấn công thật.

## 2.3 Đóng góp của nghiên cứu

Đồ án đưa ra một đánh giá tích hợp, tái lập được về honeypot PostgreSQL cục bộ và quy trình phân tích hành vi của nó. Cụ thể:

- **C1:** so sánh ba cấu hình ghi log theo recall ghi nhận mức hành động, độ đầy đủ theo từng trường của các thuộc tính cần thiết và dung lượng log theo số bản ghi;
- **C2:** đo việc dựng lại phiên truy vấn qua các ngưỡng không hoạt động so với kịch bản có nhãn và phụ thuộc tường minh giữa các bước tấn công, và cho thấy mô phỏng một địa chỉ làm phép đo này không mang thông tin;
- **C3:** so sánh bộ phân loại dựa trên luật với mô hình ngôn ngữ few-shot đặt cục bộ trên cùng các cửa sổ truy vấn, báo cáo macro-F1, hiệu năng từng lớp, đầu ra không hợp lệ, mã bằng chứng bịa và mức khớp bằng chứng, kèm độ bất định theo kịch bản;
- **C4:** thử cả hai phương pháp, giữ nguyên, trên hai tập họ kỹ thuật bị loại khỏi phát triển, với thước đo hiệu của các mức giảm, hiệu chỉnh đa so sánh và một bản kê rõ phần nào của thiết kế được chọn sau khi thấy kết quả và dữ liệu kiểm tra được cách ly đến đâu.

Đây là các phát hiện về một môi trường mô phỏng có kiểm soát. Nghiên cứu dùng chúng để mô tả các đánh đổi của ghi log và phân tích, và làm cơ sở cho các mở rộng tương lai nêu ở Chương 6.

---

# Chương 3. Phương pháp nghiên cứu

## 3.1 Thiết kế nghiên cứu

Nghiên cứu là một đánh giá định lượng trong môi trường có kiểm soát, gồm ba thí nghiệm, mỗi thí nghiệm trả lời một câu hỏi nghiên cứu. RQ1 đo ba cấu hình ghi log PostgreSQL ghi nhận hành động tấn công và dựng lại phiên tốt đến đâu. RQ2 so sánh bộ phân loại dựa trên luật với mô hình ngôn ngữ few-shot đặt cục bộ trên nhãn giai đoạn tấn công mức truy vấn và việc chọn bằng chứng. RQ3 đo mức giảm hiệu năng của cả hai phương pháp trên các họ kỹ thuật tấn công bị loại khỏi quá trình phát triển. Ba thí nghiệm tạo thành một chuỗi: RQ1 chọn cấu hình (C2) có log là đầu vào của RQ2; các phương pháp đóng băng trong RQ2 sau đó được áp dụng không đổi trong RQ3; và việc nhóm phiên yếu phát hiện ở RQ1 là lý do phải thử độ nhạy của ngữ cảnh trong RQ2.

Cả ba thí nghiệm dùng chung một quy trình (Hình 3.1) để khác biệt giữa các kết quả quy được về biến đang nghiên cứu chứ không phải do xử lý dữ liệu không nhất quán.

```
Trình mô phỏng kịch bản (nhãn, depends_on)
  → Honeypot PostgreSQL (C0 / C1 / C2)
  → Bộ phân tích log + đối chiếu aid
       ├→ Nhóm phiên (RQ1)
       └→ Cửa sổ (k = 0, 2, 4) → Rule, Qwen, H1, H3, H4 → Đánh giá so với ground truth
```
*Hình 3.1. Quy trình đánh giá dùng chung cho ba thí nghiệm.*

Một bộ sinh kịch bản tham số hóa tạo ra các chuỗi truy vấn tấn công và lành tính cùng ground truth nhãn và phụ thuộc; các chuỗi được thực thi trên honeypot PostgreSQL cục bộ theo cấu hình ghi log đã chọn; log được phân tích thành schema sự kiện chung và mỗi bản ghi được đối chiếu với bước kịch bản qua thẻ `aid`; với RQ1, các sự kiện đã phân tích được nhóm thành phiên; với RQ2 và RQ3, các sự kiện được sắp thành cửa sổ truy vấn và phân loại; và dự đoán được so với ground truth bằng các thước đo ở mục 3.3.

Bốn tập dữ liệu có vai trò khác nhau (Bảng 3.3). Tập Dev dùng để chọn luật, prompt và chiến lược kết hợp, và chạy tự do. Tập Holdout kiểm tra RQ2 và chỉ chấm một lần. Hai tập họ kỹ thuật chưa thấy (OOD-1 và OOD-2) kiểm tra RQ3 và mỗi tập chấm một lần. Các phương pháp được đóng băng trước mọi lần chấm kiểm tra: bộ phân loại dựa trên luật, và mô hình Qwen3 8B với prompt B+C+E, seed 42, temperature 0. Các chiến lược kết hợp H1 và H3 được xác định trên Dev trước khi chấm Holdout. H4 được nghĩ ra sau khi OOD-1 đã được chấm, và định nghĩa của nó được cố định trước khi chấm OOD-2. Vì vậy đồ án không được đăng ký trước toàn bộ, và chỉ OOD-2 là phép thử sạch cho H4.

## 3.2 Phương pháp thu thập dữ liệu

### 3.2.1 Môi trường honeypot và các cấu hình ghi log

Honeypot là máy chủ PostgreSQL 15 (image Docker chính thức có thêm gói pgAudit) chạy trong Docker trên máy cục bộ, ghi log dạng JSON. Nó không bị lộ ra mạng công cộng. Các cơ sở dữ liệu chứa dữ liệu giả, gồm các bảng mồi có tên và nội dung giống bản ghi nhạy cảm như tài khoản người dùng và thông tin thanh toán. Không dùng dữ liệu cá nhân hay tổ chức thật. Mỗi cấu hình được chạy riêng từ cơ sở dữ liệu sạch, với cùng khối lượng công việc: tập kịch bản Dev (136 hành động).

RQ1 so sánh ba cấu hình ghi log, tóm tắt ở Bảng 3.1. Các khoảng trống kỳ vọng là giả thuyết nêu trước khi chạy. Các lần chạy bác bỏ một phần: cả ba cấu hình đều ghi câu lệnh lỗi và đăng nhập thất bại, vì `log_min_error_statement` mặc định là `error`. Các khoảng trống thực sự tìm thấy được báo cáo ở mục 4.2.1. Ở cả ba cấu hình, pgAudit (khi được nạp) ghi mọi lớp câu lệnh (`pgaudit.log = all`, gồm truy vấn catalog), C0 và C1 để tắt ghi kết nối, và `log_min_error_statement = error`, vốn là mặc định của PostgreSQL, được đặt giống nhau ở cả ba (C2 chỉ nêu tường minh). Đăng nhập thất bại được máy chủ ghi như lỗi ở mọi cấu hình.

*Bảng 3.1. Các cấu hình ghi log của RQ1 và các khoảng trống giả thuyết trước khi chạy.*

| Cấu hình | Thiết lập | Khoảng trống giả thuyết | Kết quả (mục 4.2.1) |
|---|---|---|---|
| C0 | `log_statement = all` | Câu lệnh không parse được và đăng nhập thất bại không được ghi; tham số prepared statement không được ghi | Câu lệnh lỗi và đăng nhập thất bại đều được ghi; prepared statement được ghi 54,5% và hành động nhiều kết nối 36,8% |
| C1 | Ghi phiên pgAudit | Sự kiện kết nối/ngắt kết nối và tham số prepared statement không được ghi | Cùng khoảng trống như C0 với hành động nhiều kết nối (36,8%); prepared statement ghi đầy đủ |
| C2 | C1 cộng `log_connections`, `log_disconnections` và `pgaudit.log_parameter` | Kỳ vọng ghi đủ sáu thuộc tính phiên | Ghi đủ 136 hành động; tham số chỉ có trong 3,7% bản ghi |

Sáu thuộc tính phiên là người dùng, IP nguồn, cơ sở dữ liệu, truy vấn, kết quả và tham số. RQ1 kiểm tra, với từng cấu hình và từng trường, thuộc tính xuất hiện bao nhiêu lần trong các bản ghi đã phân tích khớp một hành động kịch bản. Kiểm tra thuộc tính được báo cáo theo từng trường; không tính cờ tổng hợp "đủ thuộc tính".

### 3.2.2 Các nhãn hành vi

Mỗi hành động được gán một trong bảy nhãn, sáu nhãn mô tả giai đoạn tấn công và một nhãn mô tả hoạt động lành tính. Các nhãn là phân loại riêng của đồ án, căn theo khái niệm của khung MITRE ATT&CK; không phải bản sao nguyên văn các chiến thuật của nó. Bảng 3.2 định nghĩa từng nhãn. Các hành động duy trì (persistence) như tạo role cửa hậu được gán nhãn Impact.

*Bảng 3.2. Các nhãn hành vi và câu lệnh tiêu biểu.*

| Nhãn | Định nghĩa | Câu lệnh tiêu biểu |
|---|---|---|
| Discovery | Dò cấu trúc cơ sở dữ liệu và máy chủ | Truy vấn `information_schema`, `pg_catalog`, `version()` |
| Credential Access | Lấy hoặc đoán thông tin xác thực | Đọc bảng chứa thông tin xác thực; đăng nhập thất bại lặp lại |
| Privilege Escalation | Giành đặc quyền cơ sở dữ liệu cao hơn | Trong dữ liệu chỉ xuất hiện dưới dạng sự kiện kết nối không có SQL (chỉ RQ1) |
| Collection | Đọc hàng loạt dữ liệu nhạy cảm | `SELECT` trên bảng nhạy cảm, tập kết quả lớn |
| Exfiltration | Đưa dữ liệu ra ngoài cơ sở dữ liệu | `COPY ... TO PROGRAM`, `lo_export`, `dblink` tới máy chủ khác |
| Impact | Thay đổi hoặc phá dữ liệu/cấu trúc, hoặc chạy lệnh hệ điều hành | `DROP TABLE`, `UPDATE`, `DELETE`, `COPY ... FROM PROGRAM`, tạo role cửa hậu |
| Noise | Truy vấn thông thường không thuộc tấn công | Đọc/ghi kiểu ứng dụng thông thường |

Các sự kiện Privilege Escalation trong dữ liệu thu được là sự kiện kết nối và không có câu lệnh SQL để phân loại. Vì vậy RQ1 dùng bảy nhãn, còn RQ2 và RQ3 phân loại sáu nhãn có câu lệnh SQL; bốn hành động Privilege Escalation của OOD-1 nằm ngoài phạm vi RQ2 và RQ3. Các câu lệnh gọi tính năng thực thi chương trình được gán theo quy tắc cố định trước khi sinh kịch bản: `COPY ... FROM PROGRAM` là Impact vì thay đổi trạng thái hệ thống bằng chạy lệnh hệ điều hành, một mẫu tấn công PostgreSQL có ghi nhận [1], còn `COPY ... TO PROGRAM` là Exfiltration. Câu lệnh nào các quy tắc gán nhãn không xác định được dứt khoát được ghi trong tài liệu ground truth và không do bộ phân loại giải quyết.

### 3.2.3 Bộ sinh kịch bản, ground truth và thẻ `aid`

Một bộ sinh Python tạo các phiên tấn công và lành tính từ mẫu. Mỗi mẫu là chuỗi bước, mỗi bước khai báo nhãn giai đoạn, mã bước duy nhất, danh sách `depends_on` nêu các bước trước mà kết quả của nó được dùng, và không hoặc nhiều gap tag. Bảy gap tag mô tả hành động xuất hiện trong log ra sao và độc lập với giai đoạn: `error_query`, `auth_failure`, `nested_dynamic`, `prepared_statement`, `pg_specific`, `log_evasion` và `multi_connection`. Giai đoạn trả lời kẻ tấn công muốn gì; thẻ trả lời log nhìn thấy nó thế nào. Vì các trường này được khai báo trong mẫu, ground truth được sinh tự động chứ không gán nhãn sau. Bộ sinh thay đổi tên bảng và cột, thứ tự bước khi cho phép, công cụ client, giá trị literal và độ trễ thời gian.

Để nối bản ghi log với ground truth, trình mô phỏng gắn một comment mờ `/*aid=<id>*/` vào mọi câu lệnh. Thẻ là chuỗi hex ngẫu nhiên 12 ký tự và không mã hóa giai đoạn hay thứ tự. Bộ phân tích trích nó ra cột riêng và xóa khỏi câu lệnh trước khi dữ liệu tới RQ2. Câu lệnh chạy trong khối `DO` hoặc `EXECUTE` được pgAudit ghi mà không có thẻ của lệnh cha, nên bộ phân tích cho chúng kế thừa `aid` của cha; sau khi phân tích, các bản ghi mang cùng `aid` được gộp về bản đầu tiên, nên mỗi hành động cho một bản ghi. Các sự kiện không có SQL, như đăng nhập thất bại, được đối chiếu theo thời gian, IP nguồn và người dùng. Dòng log không có `aid` (health check, tiến trình nền) được đánh dấu không khớp và loại khỏi RQ2, không coi là Noise. Các phiên Noise lành tính được sinh từ mẫu truy vấn thường ngày và gồm cả câu lệnh giống Discovery (ví dụ client bình thường chạy `\d`) và lỗi vô hại, để bộ phân loại phải tách tấn công khỏi hoạt động bình thường.

### 3.2.4 Tập dữ liệu, tác giả và mức độ bị lộ

Các kịch bản được nhóm thành họ, mỗi họ là tập kịch bản có cùng kỹ thuật chính và mục tiêu. Tập Dev có sáu họ: `recon_sweep`, `cred_probe`, `data_heist`, `exfil_impact` và `full_chain` (mỗi họ ba biến thể) và họ lành tính `noise_app` (hai biến thể). Bảng 3.3 liệt kê bốn tập dữ liệu. Mô tả các họ chưa thấy chỉ giới hạn ở kỹ thuật, và báo cáo này không chứa SQL nào từ Holdout hay các tập chưa thấy, vì các tập này được niêm phong.

*Bảng 3.3. Các tập dữ liệu, tác giả và mức độ bị lộ.*

| Tập | Vai trò | Kích thước | Số lớp | Tác giả | Mức lộ |
|---|---|---|---|---|---|
| Dev | Chọn luật, prompt, chiến lược; chạy tự do | 17 kịch bản; 102 truy vấn (136 hành động trong RQ1) | 6 | Nhóm và Claude | Điểm lạc quan. Gồm fold kiểm tra cũ (6 kịch bản) đã bị lộ khi gỡ lỗi mô hình ngôn ngữ |
| Holdout | Kiểm tra RQ2, chấm một lần | 22 kịch bản; 109 truy vấn (138 bước) | 6 | Agent riêng | Không hoàn toàn mù: Claude đọc một phần nội dung và đầu ra kiểm thử từng in ra mảnh SQL. Ví dụ few-shot lấy từ Dev, có test chặn SQL trùng |
| OOD-1 | RQ3, thăm dò | 9 kịch bản, 3 họ; 57 truy vấn (63 hành động) | 5 (không Noise; một truy vấn Credential Access) | Claude, trước khi đóng băng | SQL xuất hiện trong cuộc trò chuyện làm việc sau khi đóng băng. Không phải phép thử độc lập |
| OOD-2 | RQ3, thử sau | 18 kịch bản, 6 họ; 145 truy vấn (159 hành động) | 6 (Discovery 21, Collection 27, Exfiltration 12, Impact 52, Credential Access 6, Noise 27) | Agent riêng, sau kết quả OOD-1 | Bị lộ một phần: thông báo lỗi của bộ kiểm tra lần chạy từng in ra chuỗi chương trình của các bước `COPY ... PROGRAM` |

OOD-1 gồm ba họ: `lo_exfil` (nhập/xuất large object), `definer_escalation` (leo thang đặc quyền qua chuỗi hàm `SECURITY DEFINER`) và `ext_lateral` (các extension như `dblink`, `file_fdw`, `pg_stat_statements`). OOD-2 gồm sáu họ: `timing_oracle`, `xml_veil` và `lockhog` là họ kỹ thuật mới; `lo_stash` và `definer_backdoor` là biến thể mới của hai kỹ thuật đã dùng ở OOD-1 (gọi là *dùng lại*); và `benign_ops` là hoạt động lành tính. Chỉ ba họ thật sự mới, và các nhãn mới, dùng lại, lành tính được gán theo tên họ và tác giả xác nhận.

**Cách chia dữ liệu.** Thiết kế ban đầu áp dụng `StratifiedGroupKFold` của scikit-learn [39] (ba fold, xáo trộn, `random_state = 42`, nhóm theo kịch bản, phân tầng theo giai đoạn) cho 17 kịch bản, cho 11 kịch bản huấn luyện và 6 kịch bản kiểm tra (65 và 37 truy vấn). Fold kiểm tra đã bị lộ khi gỡ lỗi mô hình ngôn ngữ nên được gộp vào Dev, và tập kiểm tra trở thành Holdout độc lập do agent khác viết. Phép chia kết quả (Dev 102 truy vấn, 48%; Holdout 109 truy vấn, 52%) không phải chia ngẫu nhiên từ một nguồn, không có tỉ lệ 70/30 và không có tập validation tách khỏi Dev. Điều được giữ: không kịch bản nào xuất hiện ở hai tập (có test kiểm mã và SQL), seed cố định, Holdout chấm một lần và mã được đóng băng trước khi chấm.

### 3.2.5 Tiền xử lý và cửa sổ phân loại

Trước khi phân loại, mọi truy vấn được chuẩn hóa bởi `preprocessing.py`: literal chuỗi thành `'?'`, literal số thành `?`, và comment `aid` bị xóa. Tên bảng được đổi thành `t<n>_<h>`, trong đó n đếm tên theo thứ tự xuất hiện đầu tiên và h là bốn ký tự của băm MD5 của seed và tên. Một ánh xạ được xây cho toàn bộ câu lệnh của một lần chạy, mang tính tất định (seed 42), và đối tượng hệ thống (`pg_*`, `information_schema`, `public`) được giữ. Do đó ánh xạ không phải đổi tên ngẫu nhiên mới cho mỗi lần chạy hay mỗi phiên. Câu lệnh gốc được giữ trong tệp ground truth và văn bản chuẩn hóa là đầu vào duy nhất của bộ phân loại.

Một cửa sổ gồm truy vấn đích và tối đa k truy vấn trước đó của cùng kịch bản (k ∈ {0, 2, 4}). Cửa sổ lấy ngữ cảnh theo mã kịch bản thật vì bộ nhóm phiên không thể nhóm khối lượng chỉ có một địa chỉ (mục 4.2.1); do đó ngữ cảnh là lý tưởng và có lẽ thuận lợi hơn so với dùng thực tế. Một thí nghiệm riêng trên Dev làm hỏng ngữ cảnh theo ba cách để cho thấy mỗi phương pháp nhạy đến đâu: ngữ cảnh lấy từ cách nhóm của chính bộ nhóm phiên (ngưỡng 5 phút), và các phép trộn trong đó mỗi ô ngữ cảnh bị thay, với xác suất 0,5 hoặc 1,0, bằng một truy vấn của kịch bản khác. Thí nghiệm áp dụng Rule và Qwen đã đóng băng ở k = 2 và k = 4.

### 3.2.6 Các phương pháp được thử

**Bộ phân loại dựa trên luật.** Bộ phân loại là danh sách có thứ tự các luật biểu thức chính quy áp dụng cho truy vấn đích (SQL trong `PREPARE` được bóc ra trước), với thứ tự ưu tiên Exfiltration, Impact, Credential Access, Collection, Discovery, Noise. Ví dụ, nhắc tới `pg_shadow` hoặc `pg_authid` cho Credential Access; `DROP TABLE`, đổi role, `COPY ... FROM` và `CREATE EXTENSION` cho Impact; `COPY ... TO` hoặc `lo_export` cho Exfiltration. Ngữ cảnh cửa sổ chỉ đổi nhãn trong một trường hợp: khối `DO` có nội dung ẩn lấy giai đoạn (Discovery hoặc Collection) của các truy vấn trước; còn lại ngữ cảnh chỉ quyết định truy vấn trước nào được liệt kê làm bằng chứng (các truy vấn thỏa điều kiện riêng của giai đoạn, cộng truy vấn đích). Khi không luật nào khớp, kết quả là Noise. Luật chỉ được viết và tinh chỉnh trên Dev. Vì nhãn của nó hầu như không phụ thuộc ngữ cảnh, điểm của Rule gần như không phụ thuộc k theo cấu tạo.

**Mô hình ngôn ngữ.** Mô hình là Qwen3 8B phục vụ qua Ollama [40], không tinh chỉnh, seed 42, temperature 0, tắt chế độ suy nghĩ. Ollama phiên bản 0.40.11 chạy cục bộ trên Intel Core i7-1165G7 (4 nhân, 8 luồng) với 15,8 GB bộ nhớ và chỉ có đồ họa tích hợp Intel Iris Xe, nên suy luận chạy trên CPU. Tác giả xác nhận đây là máy thí nghiệm. Mã không lưu phiên bản phần mềm trong tệp kết quả. Các mô hình ứng viên được cân nhắc lúc đầu (Qwen2.5-7B, Llama-3.1-8B, Mistral-7B); Llama 3.1 8B được dùng trước rồi thay bằng Qwen3 8B, hai mô hình còn lại không thử.

Prompt là prompt few-shot thông thường, không phải kiểu có hướng dẫn nhiều bước; tài liệu [4, 18, 25] gợi ý các ràng buộc đầu ra tường minh và sự cẩn trọng với ngữ cảnh. Nó được phát triển trên Dev qua ablation các thành phần A đến F (mục 4.2.2 và Phụ lục B). Phiên bản đóng băng B+C+E gồm phát biểu vai trò và nhiệm vụ, sáu nhãn, định dạng đầu ra, danh sách ví dụ few-shot cố định (3 ví dụ khi k = 0 và 10 khi k ≥ 2, không truy hồi), các dấu `CONTEXT` và `TARGET` tường minh kèm chỉ dẫn chỉ phân loại truy vấn đích, và định nghĩa nhãn chặt hơn. Bằng chứng luôn gồm truy vấn đích: mã tự thêm mã đích nếu mô hình bỏ sót. Vì các ví dụ có ngữ cảnh bị bỏ ở k = 0, chuyển từ k = 0 sang k ≥ 2 đổi cả ngữ cảnh lẫn số ví dụ (3 so với 10); chỉ so sánh k = 2 với k = 4, cùng dùng 10 ví dụ, cô lập được độ dài lịch sử. Nguyên văn prompt cho cửa sổ k = 2 nằm ở Phụ lục G.

Đối tượng `{...}` đầu tiên trong phản hồi được phân tích bằng `json.loads` rồi kiểm tra schema (nhãn ngoài sáu nhãn cho phép, hoặc bằng chứng không phải danh sách, là lỗi schema). Lỗi phân tích và lỗi schema được đếm riêng và không thử lại; sinh bị giới hạn 128 token. Nhãn thay thế cho đầu ra không hợp lệ là Noise, có thể trùng nhãn đúng; trong thí nghiệm không có đầu ra không hợp lệ nên lựa chọn này không ảnh hưởng số liệu nào. Mã bằng chứng là bịa nếu không ứng với truy vấn nào trong cửa sổ; mã bịa được đếm rồi bị loại khỏi tập bằng chứng, và tỉ lệ bằng chứng bịa là số của chúng chia cho mọi mã trả về (gồm cả mã đích do code thêm).

**Các chiến lược kết hợp.** Ba chiến lược kết hợp nhãn của Rule và Qwen. H1 chấp nhận nhãn tự động khi hai phương pháp đồng ý và gửi cửa sổ cho người rà soát khi bất đồng. H3 theo Rule trừ khi Qwen nói Credential Access: nó chuyển cho mô hình ngôn ngữ một lớp mà luật hay bỏ sót, cụ thể là lớp duy nhất mà mô hình thắng rõ rệt luật trên Dev. H4 giữ nhãn của Rule và chỉ dùng nhãn của Qwen khi Rule trả Noise, tức chuyển cho mô hình ngôn ngữ các cửa sổ mà luật không nhận ra. H3 được chọn trên Dev; H4 được nghĩ ra sau khi biết kết quả OOD-1. H4 được tính sau từ nhãn Rule và Qwen đã lưu bằng một hàm tất định chứ không chạy lại mô hình, nên được đánh dấu † trong các bảng.

## 3.3 Kỹ thuật lấy mẫu và phân tích dữ liệu

### 3.3.1 RQ1: độ phủ và dựng lại phiên

Recall mức hành động tính cho từng nhóm nhãn g (giai đoạn hoặc gap tag) và từng cấu hình p:

Recall(p, g) = (số hành động thuộc g có bản ghi log khớp theo p) / (số hành động thuộc g).

Đơn vị là hành động: 136 hành động gồm 102 câu lệnh SQL, 19 đăng nhập thất bại và 15 sự kiện kết nối. Việc đối chiếu dùng thẻ `aid`, và theo thời gian, IP và người dùng với sự kiện không có SQL. Độ đầy đủ thuộc tính là tỉ lệ bản ghi khớp có chứa từng thuộc tính. Phiên được nhóm theo IP nguồn và người dùng, và phiên mới bắt đầu khi khoảng cách giữa hai sự kiện liên tiếp vượt ngưỡng (1, 5, 15 hoặc 30 phút); cơ sở dữ liệu không nằm trong khóa. Chất lượng đo bằng precision, recall và F1 theo cặp trên các cặp sự kiện, trong đó cặp được lập giữa các bản ghi đã khớp và hai bản ghi được coi là thật sự cùng phiên nếu thuộc cùng kịch bản. Không tính khớp chính xác toàn bộ phiên.

Quy tắc chọn: một cấu hình đủ điều kiện nếu recall của nó ít nhất 95% tổng thể và ở mọi giai đoạn; trong các cấu hình đủ điều kiện, chọn cấu hình có ít bản ghi log nhất. Quy tắc không có cổng về sáu thuộc tính, vì không cấu hình nào có đủ sáu theo nghĩa chặt. Không báo cáo khoảng tin cậy cho RQ1, vì 136 hành động đến từ 17 kịch bản và không độc lập.

### 3.3.2 RQ2: phân loại, bằng chứng và ngữ cảnh

Hiệu năng phân loại báo cáo bằng macro-F1, trung bình không trọng số của F1 từng lớp, cùng kết quả từng lớp và ma trận nhầm lẫn. Thước đo chính đã đăng ký là macro-F1 ở k = 2; k = 0 và k = 4 mang tính thăm dò và không chọn "k tốt nhất". Accuracy là thước đo khác và chỉ dùng cho H1. Độ tin cậy kỹ thuật là tỉ lệ đầu ra không hợp lệ và tỉ lệ bằng chứng bịa.

Với truy vấn đích t, tập bằng chứng ground truth là E_gt(t) = {t} ∪ {d ∈ depends_on(t) : d nằm trong cửa sổ}. Thước đo bằng chứng chính là chỉ số Jaccard |E_pred ∩ E_gt| / |E_pred ∪ E_gt| lấy trung bình trên các cửa sổ, trong đó hai tập rỗng cho 1; precision, recall và F1 của tập bằng chứng là các thước đo bổ sung lấy từ `results.json` của mỗi lần chạy. Bằng chứng chỉ được đánh giá khi k ≥ 2, vì ở k = 0 cửa sổ chỉ chứa truy vấn đích.

Chất lượng ground truth được kiểm tra sau khi luật và prompt đã đóng băng. Một mẫu ngẫu nhiên có seed gồm 20% các cửa sổ Dev k = 4 có ngữ cảnh (20 cửa sổ, seed 42, 12 kịch bản, không phân tầng) được một người rà soát, là Claude, một AI chứ không phải thành viên nhóm, đánh giá mù đối với bằng chứng tự động. Tiêu chí của người rà soát (các truy vấn cần để giải thích nhãn) chặt hơn `depends_on`. Chỉ số Jaccard giữa tập tự động và tập đã rà soát ước lượng mức đồng thuận giữa hai cách gán bằng chứng và không chứng minh ground truth đúng.

Ảnh hưởng của ngữ cảnh được so bằng chênh lệch cặp macro-F1 giữa k ∈ {0, 2, 4} trong từng phương pháp. H1 được mô tả bằng bốn con số: độ phủ (tỉ lệ cửa sổ được chấp nhận), accuracy trên cửa sổ được chấp nhận, accuracy của Rule trên mọi cửa sổ, và accuracy của từng phương pháp trên các cửa sổ H1 gửi đi xét lại. Hiệu giữa accuracy trên cửa sổ được chấp nhận và accuracy của Rule trên mọi cửa sổ so sánh hai tập cửa sổ khác nhau (hiệu ứng chọn lọc) và không phải accuracy của cả quy trình, vốn không được đo.

### 3.3.3 RQ3: mức giảm hiệu năng trên họ chưa thấy

Với mỗi phương pháp, mức giảm là ΔF1 = F1(Holdout) − F1(chưa thấy), nên giá trị dương nghĩa là giảm. Nó được tính trên các lớp mà hai tập chung: OOD-1 có năm lớp (không có Noise), nên điểm Holdout được tính lại trên cùng năm lớp và khác điểm sáu lớp của RQ2; OOD-2 có cùng sáu lớp với Holdout. Hiệu giữa hai mức giảm là D = ΔF1(A) − ΔF1(B); D > 0 nghĩa là B giảm ít hơn. Ở đây D = ΔF1(Rule) − ΔF1(Qwen), nên D > 0 nghĩa là Qwen giảm ít hơn. Giảm ít hơn không có nghĩa là điểm cao hơn trên tập chưa thấy. Vì các tập khác nhau về tác giả, kích thước và tỉ lệ lớp, mức giảm không chỉ do độ lạ gây ra.

Độ lạ của mỗi họ được chấm 1 đến 3 trên ba yếu tố (Bảng 3.4) bởi người dựng họ, một lần, trước khi các tập chưa thấy được chấm. Không có người chấm thứ hai và không có độ đồng thuận giữa người chấm. Điểm của `ext_lateral` trong OOD-1 là ước tính từ mô tả họ; không có bản chấm gốc, tác giả quyết định giữ, và báo cáo đánh dấu đó là ước tính. Với chỉ ba và sáu họ, điểm chỉ được dùng ở mức mô tả.

*Bảng 3.4. Thang điểm đặc trưng độ lạ của một họ. Độ dài phụ thuộc đếm chuỗi `depends_on` và không liên quan đến k.*

| Yếu tố | 1 | 2 | 3 |
|---|---|---|---|
| Cấu trúc SQL | Chỉ khác tên, tham số hay điều kiện | Câu lệnh lồng hoặc tổ hợp điều kiện lạ | Cấu trúc nhiều tầng hoặc bậc hai, hàm xử lý chuỗi |
| Hàm riêng của PostgreSQL | SQL chuẩn | Một số hàm thông tin hệ thống | Hàm chuyên sâu hoặc extension nâng cao |
| Chuỗi phụ thuộc | Bước độc lập hoặc chuỗi dài 1 | Chuỗi dài 2–3 | Chuỗi đứt quãng, nhiều bước xa nhau |

### 3.3.4 Độ bất định và đa so sánh

Khoảng tin cậy là khoảng bootstrap phân vị lấy mẫu lại theo kịch bản (lấy mẫu lại kịch bản, không phải truy vấn, vì các truy vấn của một kịch bản phụ thuộc nhau) với 1000 lần lấy mẫu và seed 42, ghép cặp giữa các phương pháp để cả hai được chấm trên cùng một mẫu. Holdout và các tập chưa thấy được lấy mẫu lại độc lập khi tính một mức giảm. Một script bổ sung (`rq_extras`) dùng 10.000 lần lấy mẫu cho H1, chênh lệch tuyệt đối và chênh lệch bằng chứng. Khoảng tin cậy loại trừ 0 được đọc là bằng chứng về một khác biệt; khoảng chứa 0 nghĩa là bằng chứng chưa đủ, không có nghĩa hai phương pháp tương đương. Trừ khi nêu khác, khoảng là danh nghĩa và chưa hiệu chỉnh đa so sánh.

*Tập lớp cố định.* Macro-F1 mặc định chỉ lấy trung bình các lớp có mặt trong một lần lấy mẫu. Một lớp hiếm (Credential Access trong OOD-1: một truy vấn trong một trên chín kịch bản) vắng mặt trong 35,1% lần lấy mẫu, và vài lớp khác trong tối đa 5,3%. Vì vậy các khoảng chính cho ΔF1 và D dùng bộ ước lượng với tập lớp cố định, trong đó lớp vắng mặt được tính F1 = 0. Lựa chọn này được đưa ra sau khi phát hiện vấn đề và không đăng ký trước. Bộ ước lượng mặc định được giữ làm phân tích độ nhạy (Phụ lục A). Ước lượng điểm của hai bộ ước lượng giống hệt nhau.

*Họ sáu phép so sánh.* Sáu hiệu D(Rule − Qwen) (hai tập chưa thấy nhân k ∈ {0, 2, 4}) được hiệu chỉnh cùng nhau bằng thủ tục max-|t| một bước theo kiểu Westfall và Young. Trong mỗi lần trong 10.000 lần lấy mẫu (seed 42), Holdout được rút một lần cho cả sáu ước lượng và mỗi tập chưa thấy rút một lần cho ba giá trị k. Với se_j là độ lệch chuẩn của D*_j lấy mẫu lại và t*_j = (D*_j − D_j)/se_j, giá trị tới hạn c là phân vị 95 của max_j |t*_j| (ở đây c = 2,592, so với 1,96 cho một phép so sánh), khoảng đồng thời là D_j ± c·se_j, và p đã hiệu chỉnh là (1 + #{max_k |t*_k| ≥ |D_j/se_j|})/(B + 1). Giao thức đã đăng ký loại phép so sánh (khoảng tin cậy của hiệu hai mức giảm) nhưng không định nghĩa họ sáu phép này hay thủ tục đa so sánh nào; họ được định nghĩa ở giai đoạn cuối, trước khi tính hiệu chỉnh. Mọi so sánh khác (H1, H3, H4, nhóm con, bằng chứng và ba giá trị k trong RQ2) là danh nghĩa hoặc thăm dò.

## 3.4 Hạn chế của phương pháp

Các hạn chế của toàn đồ án được liệt kê ở mục 1.5; những điều sau liên quan riêng đến phương pháp.

- **Dữ liệu nhỏ, mô phỏng.** Ground truth phản ánh thiết kế mẫu, và các kịch bản dựa trên tài liệu [1, 5, 14] nhưng không phải ghi lại các cuộc tấn công thật. Ước lượng từng lớp dựa trên vài truy vấn; một số lớp chỉ có một đến sáu truy vấn trong một tập.
- **Ngữ cảnh lý tưởng.** Cửa sổ dùng mã kịch bản thật. Mô phỏng một địa chỉ khiến việc nhóm phiên thực tế không dùng được (F1 theo cặp khoảng 0,11), và thí nghiệm làm hỏng ngữ cảnh chỉ chạy trên Dev, không chạy trên Holdout.
- **Prompt tinh chỉnh trên Dev.** Định nghĩa E được viết từ lỗi trên Dev nên điểm Dev phóng đại hiệu năng. Việc điểm giảm 0,018 ở cả hai phương pháp từ Dev sang Holdout cho thấy không có mức giảm lớn nhưng không phải bằng chứng vắng rò rỉ, và độ lạc quan không đo được.
- **k bị trộn với số ví dụ.** Ở k = 0 prompt có 3 ví dụ và ở k ≥ 2 có 10.
- **Khoảng tin cậy không bao phủ độ bất định giữa các họ.** Bootstrap coi kịch bản là độc lập, nhưng mỗi họ chưa thấy chỉ có ba kịch bản và chỉ ba họ thật sự mới. Khoảng có thể hẹp hơn độ bất định thật về các họ kỹ thuật khác.
- **Khác biệt giữa các tập dữ liệu.** ΔF1 so sánh các tập khác nhau về tác giả, kích thước, tỉ lệ lớp (Impact chiếm 36% OOD-2) và độ khó. Credential Access chỉ có một truy vấn trong OOD-1.
- **Lựa chọn sau khi thấy kết quả.** Bộ ước lượng tập lớp cố định, các nhóm con mới/dùng lại/lành tính, các so sánh liên quan H4 và họ sáu phép so sánh được định nghĩa sau khi thấy kết quả.
- **Một người rà soát và các ngưỡng.** Rà soát bằng chứng dùng một AI; tiêu chí 95% và khái niệm đủ thuộc tính là lựa chọn riêng của nhóm; điểm độ lạ do tác giả họ chấm một lần (và một điểm là ước tính).
- **Bằng chứng đóng băng.** Repo chỉ có hai commit, nên thứ tự chỉnh sửa được ghi bằng thời điểm sửa tệp và nhật ký tiến độ: `rule_based.py` (26/10, 14:37), `llm_local.py` (26/10, 21:46, prompt B+C+E), `hybrid.py` (27/10, 08:03, thêm H4 sau OOD-1) và `preprocessing.py` (27/10, 22:16, thêm điều kiện về `session_logical`; suy ra từ code là không ảnh hưởng cửa sổ Holdout và chưa thấy, chưa kiểm trên dữ liệu niêm phong).
- **Thủ tục thống kê không có trích dẫn.** Bootstrap theo cụm, hiệu chỉnh max-|t|, F1 theo cặp và các ngưỡng nghỉ phiên là thủ tục chuẩn, nhưng báo cáo không trích dẫn nguồn đã xác minh nào cho chúng, và không bịa nguồn.
- **Tính không tất định.** Temperature 0 và seed cố định cho macro-F1 Holdout giống hệt khi chạy lại, nhưng kết quả có thể khác trên phần cứng hoặc phiên bản phần mềm khác.
- **Chưa đo.** CPU và bộ nhớ khi suy luận, overhead ghi log (pgbench), độ trễ truy vấn và dung lượng log theo byte (dung lượng đếm theo bản ghi).

**Khả năng tái lập và đạo đức.** Toàn bộ mã sinh, mã phân tích log, prompt, định nghĩa luật, seed và danh sách chia nằm trong repo có quản lý phiên bản, và các lệnh sinh lại từng bảng được liệt kê ở Phụ lục F. Honeypot chỉ chứa dữ liệu giả, không mở ra mạng ngoài và không chạy mã độc thật. Bộ sinh kịch bản chỉ tạo câu lệnh SQL, không dùng hạ tầng tấn công thật.

---

# Chương 4. Thực nghiệm và kết quả

## 4.1 Giới thiệu

Chương này báo cáo kết quả thực nghiệm theo thứ tự các câu hỏi nghiên cứu: ghi log và dựng lại phiên (RQ1, mục 4.2.1), phân loại và bằng chứng trên họ đã biết (RQ2, mục 4.2.2) và hiệu năng trên họ chưa thấy (RQ3, mục 4.2.3). Mục 4.3 phân tích các khoảng tin cậy, mục 4.4 nêu kết quả hỗ trợ và không hỗ trợ điều gì, và hai mục cuối so sánh kết quả với tài liệu và nêu hàm ý. Mọi con số lấy từ bảng kết quả cuối của đồ án (`final_results_tables.csv`), đầu ra đã lưu của các script thống kê hoặc nhật ký phát triển, và các bảng nêu rõ tập dữ liệu, thước đo, cỡ mẫu và loại khoảng tin cậy.

Ba quy ước áp dụng xuyên suốt. (1) ΔF1 là điểm trên Holdout trừ điểm trên tập chưa thấy, nên giá trị dương là giảm, và D = ΔF1(Rule) − ΔF1(Qwen) dương khi Qwen giảm ít hơn. (2) Khoảng tin cậy là bootstrap 95% theo kịch bản; khoảng chứa 0 nghĩa là bằng chứng chưa đủ chứ không phải tương đương. (3) Khoảng là danh nghĩa và chưa hiệu chỉnh trừ khi bảng ghi "đồng thời". OOD-1 và OOD-2 là hai tập họ chưa thấy, "Qwen" là Qwen3 8B với prompt đóng băng, và dấu † đánh dấu giá trị H4 được tính sau. Mọi dữ liệu đều là mô phỏng.

## 4.2 Trình bày dữ liệu

Bốn tập dữ liệu được mô tả ở Bảng 3.3. RQ1 chạy trên Dev (136 hành động của 17 kịch bản), RQ2 trên Holdout (22 kịch bản, 109 truy vấn) và RQ3 trên OOD-1 (9 kịch bản, 57 truy vấn) và OOD-2 (18 kịch bản, 145 truy vấn), so với Holdout.

### 4.2.1 RQ1: độ phủ ghi log và dựng lại phiên

**RO1.1: hành động bị bỏ sót.** Bảng 4.1 cho recall theo giai đoạn. Recall tổng thể là 85,3% với C0 (116 trên 136 hành động), 89,0% với C1 (121 trên 136) và 100% với C2 (136 trên 136). Recall thấp nhất theo giai đoạn là Privilege Escalation (53,8% với C0 và C1), vốn gồm các sự kiện kết nối không có SQL, tiếp theo là Credential Access (69,2% với C0, 76,9% với C1). C2 ghi mọi hành động ở mọi giai đoạn. Trên các họ chưa thấy, C2 ghi 62 trên 63 hành động ở OOD-1 và 159 trên 159 ở OOD-2. Vì C2 bật ghi kết nối theo thiết kế, việc nó phủ đầy đủ các sự kiện kết nối là hệ quả của cấu hình.

*Bảng 4.1. RQ1: recall mức hành động theo cấu hình và giai đoạn (Dev, 136 hành động, 17 kịch bản; không có khoảng tin cậy vì các hành động không độc lập). Hàng cuối là số bản ghi log mỗi cấu hình ghi.*

| Giai đoạn | n | C0 | C1 | C2 |
|---|---|---|---|---|
| Tất cả hành động | 136 | 85,3% (116/136) | 89,0% (121/136) | 100,0% (136/136) |
| Discovery | 35 | 91,4% (32/35) | 91,4% (32/35) | 100,0% (35/35) |
| Credential Access | 26 | 69,2% (18/26) | 76,9% (20/26) | 100,0% (26/26) |
| Privilege Escalation | 13 | 53,8% (7/13) | 53,8% (7/13) | 100,0% (13/13) |
| Collection | 18 | 83,3% (15/18) | 100,0% (18/18) | 100,0% (18/18) |
| Exfiltration | 9 | 100,0% (9/9) | 100,0% (9/9) | 100,0% (9/9) |
| Impact | 21 | 100,0% (21/21) | 100,0% (21/21) | 100,0% (21/21) |
| Noise | 14 | 100,0% (14/14) | 100,0% (14/14) | 100,0% (14/14) |
| Số bản ghi log | -- | 199 | 204 | 322 |

Bảng 4.2 chia recall theo gap tag. C0 chỉ ghi 54,5% hành động prepared statement (6 trên 11), và cả C0 lẫn C1 chỉ ghi 36,8% hành động nhiều kết nối (7 trên 19). C2 đạt 100% với mọi thẻ đã thử. Nhóm "né log" chưa được thử. Kỳ vọng trước đó rằng C0 và C1 sẽ không ghi câu lệnh lỗi và đăng nhập thất bại không đúng: cả ba cấu hình đều ghi chúng (19 trên 19 đăng nhập thất bại và 11 trên 11 truy vấn lỗi), vì `log_min_error_statement` mặc định là `error`.

*Bảng 4.2. RQ1: recall theo gap tag (Dev; một hành động có thể mang nhiều thẻ; số đếm là hành động).*

| Gap tag | n | C0 | C1 | C2 |
|---|---|---|---|---|
| `error_query` | 11 | 100,0% (11/11) | 100,0% (11/11) | 100,0% (11/11) |
| `auth_failure` | 19 | 100,0% (19/19) | 100,0% (19/19) | 100,0% (19/19) |
| `nested_dynamic` | 11 | 100,0% (11/11) | 100,0% (11/11) | 100,0% (11/11) |
| `prepared_statement` | 11 | 54,5% (6/11) | 100,0% (11/11) | 100,0% (11/11) |
| `pg_specific` | 16 | 100,0% (16/16) | 100,0% (16/16) | 100,0% (16/16) |
| `log_evasion` | 0 | chưa thử | chưa thử | chưa thử |
| `multi_connection` | 19 | 36,8% (7/19) | 36,8% (7/19) | 100,0% (19/19) |

**RO1.2: thuộc tính và phiên.** Bảng 4.3 cho sự hiện diện của từng thuộc tính trong các bản ghi khớp một hành động kịch bản. Người dùng, IP nguồn, cơ sở dữ liệu và kết quả có mặt trong mọi bản ghi khớp ở cả ba cấu hình. Văn bản câu lệnh có trong 83,6% (C0), 84,3% (C1) và 75,0% (C2) số bản ghi, vắng ở các sự kiện đăng nhập thất bại (19 ở mọi cấu hình) và, ở C2, 15 sự kiện kết nối, vốn không có SQL, và tham số ràng buộc không có trong bản ghi C0 và C1 nào và có trong 3,7% (5 trên 136) bản ghi C2. Định nghĩa "đủ thuộc tính" chưa được xác nhận, nên không báo cáo số tổng hợp như "năm trên sáu".

*Bảng 4.3. RQ1: tỉ lệ bản ghi log khớp có chứa từng thuộc tính phiên (Dev).*

| Thuộc tính | C0 (116 bản ghi) | C1 (121) | C2 (136) |
|---|---|---|---|
| người dùng | 100,0% (116/116) | 100,0% (121/121) | 100,0% (136/136) |
| source IP (`remote_host`) | 100,0% (116/116) | 100,0% (121/121) | 100,0% (136/136) |
| cơ sở dữ liệu | 100,0% (116/116) | 100,0% (121/121) | 100,0% (136/136) |
| văn bản câu lệnh | 83,6% (97/116) | 84,3% (102/121) | 75,0% (102/136) |
| result (`error_severity`) | 100,0% (116/116) | 100,0% (121/121) | 100,0% (136/136) |
| tham số ràng buộc | 0,0% (0/116) | 0,0% (0/121) | 3,7% (5/136) |

Việc nhóm phiên yếu (Bảng 4.4). Các cặp được lập giữa các bản ghi đã khớp, và hai bản ghi được coi là thật sự cùng nhau nếu thuộc cùng kịch bản. Precision theo cặp khoảng 0,06, recall 0,57 đến 0,62 và F1 khoảng 0,11 ở cả ba cấu hình; recall dưới 1 vì khóa nhóm gồm cả người dùng, nên kịch bản đổi người dùng bị tách. Giá trị không đổi khi thay ngưỡng không hoạt động 1, 5, 15 và 30 phút, vì mọi kịch bản chạy từ một địa chỉ nguồn duy nhất nên IP và người dùng không tách được phiên. Đây là giới hạn của thiết kế thí nghiệm chứ không phải bằng chứng về nhóm phiên trong honeypot nhiều địa chỉ nguồn; recall hành động 100% không có nghĩa là dựng lại được phiên.

*Bảng 4.4. RQ1: dựng lại phiên theo cặp (Dev). Giá trị giống hệt ở ngưỡng 1, 5, 15 và 30 phút (một IP mô phỏng).*

| Cấu hình | Precision theo cặp | Recall theo cặp | F1 theo cặp |
|---|---|---|---|
| C0 | 0,0585 | 0,6197 | 0,1069 |
| C1 | 0,0601 | 0,5986 | 0,1092 |
| C2 | 0,0617 | 0,5683 | 0,1113 |

**RO1.3: cấu hình được chọn.** Chỉ C2 đạt tiêu chí (recall ít nhất 95% tổng thể và ở mọi giai đoạn), nên C2 được chọn. C2 là cấu hình tối thiểu đạt tiêu chí; nó không phải cấu hình nhẹ nhất, vì nó ghi 322 bản ghi log so với 199 của C0 và 204 của C1 (gấp khoảng 1,6 lần; đơn vị là bản ghi, không phải byte). Độ trễ truy vấn và overhead không được đo.

### 4.2.2 RQ2: phân loại và bằng chứng trên họ đã biết

**Phát triển prompt trên Dev.** Với prompt đầu tiên, Qwen đạt macro-F1 0,515 ở k = 2 trên Dev, so với 0,841 của phương pháp luật, chủ yếu vì Noise bị gán Collection (12 trên 14 truy vấn Noise) và Impact bị gán Exfiltration (13 trên 21). Thêm ví dụ cân bằng (B) nâng điểm lên 0,639, và đánh dấu ngữ cảnh và đích cùng yêu cầu chỉ gán nhãn cho đích (C) nâng lên 0,641 (+0,126 [+0,036, +0,240] so với prompt đầu); yêu cầu nêu lý do trước nhãn (D) không giúp (0,513). Kết hợp B và C cho 0,706 (k = 2) và 0,726 (k = 4), và thêm định nghĩa nhãn chặt hơn (E) cho 0,787 [0,722, 0,834] ở k = 2. Một phiên bản có thêm ví dụ (BCEF) đạt 0,811 nhưng không phân biệt được với BCE (+0,024 [−0,018, +0,081]), nên B+C+E đơn giản hơn được đóng băng. Các định nghĩa trong E được viết từ lỗi trên Dev nên các điểm này lạc quan. Bảng ablation ở Phụ lục B.

**RO2.1: độ chính xác trên Holdout.** Bảng 4.5 cho macro-F1 trên Holdout, chấm một lần. Ở k = 2, thiết lập chính đã đăng ký, Rule đạt 0,823 và Qwen 0,769. Từ Dev sang Holdout, điểm giảm 0,018 ở cả hai phương pháp (Rule 0,841 xuống 0,823; Qwen 0,787 xuống 0,769). Qwen không sinh lỗi phân tích JSON, lỗi schema hay mã bằng chứng bịa nào ở mọi k trên cả ba tập đánh giá. Suy luận mất trung bình 4,6, 7,8 và 6,8 giây mỗi cửa sổ ở k = 0, 2, 4 với Qwen trên CPU (Holdout); Rule gần như tức thì. CPU và bộ nhớ không được đo. Chạy lại Holdout cho giá trị macro-F1 giống hệt.

*Bảng 4.5. RQ2: macro-F1 trên Holdout (22 kịch bản, 109 truy vấn, 6 lớp; KTC 95% danh nghĩa; hiệu ghép cặp).*

| k | Rule | Qwen | Qwen − Rule |
|---|---|---|---|
| 0 (thăm dò) | 0,826 [0,762, 0,904] | 0,732 [0,646, 0,852] | −0,094 [−0,151, −0,034] |
| 2 (chính) | 0,823 [0,756, 0,888] | 0,769 [0,701, 0,827] | −0,053 [−0,131, +0,016] |
| 4 (thăm dò) | 0,823 [0,756, 0,888] | 0,769 [0,692, 0,826] | −0,053 [−0,133, +0,021] |

Kết quả từng lớp và ma trận nhầm lẫn ở Phụ lục C. Ở k = 2 các lỗi của Rule là Collection bị gán Discovery (4 trên 15) và Noise bị gán Discovery hoặc Collection (mỗi loại 3 trên 21); lỗi của Qwen là Impact bị gán Exfiltration (5 trên 27) và Collection bị gán Impact (5 trên 15) hoặc Discovery (2). Kết hợp hai phương pháp không làm thay đổi bức tranh trên họ đã biết: H3 thấp hơn Rule 0,004 đến 0,006 với khoảng chứa 0, và H4 trừ Rule là −0,041 [−0,096, +0,023], +0,012 [−0,029, +0,061] và +0,016 [−0,033, +0,077] ở k = 0, 2, 4 (Bảng A.4 ở phụ lục).

**RO2.2: bằng chứng.** Bảng 4.6 cho chỉ số Jaccard trung bình giữa bằng chứng dự đoán và ground truth. Trên Holdout ở k = 2, Rule đạt 0,821 và Qwen 0,754, hiệu ghép cặp −0,067 [−0,129, −0,015]; ở k = 4 hiệu là −0,010 [−0,074, +0,044]. Rà soát thủ công 20 cửa sổ Dev ở k = 4 (12 kịch bản, một người rà soát AI) cho Jaccard 0,775 [0,667, 0,883] giữa bằng chứng tự động và đã rà soát: 12 cửa sổ khớp, 1 có truy vấn thừa, 4 thiếu truy vấn và 3 khác. Đây là thước đo sơ bộ về mức đồng thuận giữa hai cách gán bằng chứng trên Dev; nó không cho thấy ground truth đúng.

*Bảng 4.6. Mức khớp bằng chứng: chỉ số Jaccard trung bình với ground truth suy ra từ `depends_on` (bằng chứng chỉ khi k ≥ 2; KTC 95% theo kịch bản, danh nghĩa; Holdout 22, OOD-1 9, OOD-2 18 kịch bản).*

| Tập | k | Rule | Qwen | Qwen − Rule |
|---|---|---|---|---|
| Holdout | 2 | 0,821 [0,761, 0,881] | 0,754 [0,697, 0,811] | −0,067 [−0,129, −0,015] |
|  | 4 | 0,777 [0,708, 0,845] | 0,767 [0,720, 0,819] | −0,010 [−0,074, +0,044] |
| OOD-1 | 2 | 0,696 [0,622, 0,756] | 0,751 [0,675, 0,824] | +0,056 [−0,014, +0,136] |
|  | 4 | 0,612 [0,550, 0,664] | 0,708 [0,645, 0,774] | +0,095 [+0,021, +0,193] |
| OOD-2 | 2 | 0,662 [0,607, 0,725] | 0,717 [0,660, 0,778] | +0,055 [+0,026, +0,085] |
|  | 4 | 0,645 [0,591, 0,707] | 0,737 [0,680, 0,794] | +0,092 [+0,052, +0,141] |

**RO2.3: ảnh hưởng của k và của ngữ cảnh bị làm hỏng.** Điểm của Rule hầu như không đổi theo k (0,826, 0,823, 0,823), đúng như kỳ vọng vì nhãn của nó chỉ phụ thuộc ngữ cảnh ở các khối `DO` ẩn. Điểm quan sát của Qwen cao hơn ở k = 2 và k = 4 (0,769) so với k = 0 (0,732), nhưng mọi chênh lệch ghép cặp đều chứa 0 (Bảng 4.7). So sánh k = 0 với k ≥ 2 cũng đổi số ví dụ (3 so với 10), và chỉ k = 4 trừ k = 2 cô lập độ dài lịch sử; hiệu đó là +0,000 [−0,031, +0,038]. Trên Dev, Qwen mất điểm khi ngữ cảnh bị làm hỏng: ở k = 2 hiệu so với ngữ cảnh lý tưởng là −0,058 [−0,106, −0,000] với cách nhóm của bộ nhóm phiên và −0,071 [−0,132, −0,004] với ngữ cảnh trộn hoàn toàn, và ở k = 4 là −0,089 [−0,144, −0,012] và −0,048 [−0,105, +0,016]; phép trộn 50% cho −0,013 và −0,026 (cả hai khoảng chứa 0). Rule thay đổi tối đa 0,016. Mức mất không theo tỉ lệ ngữ cảnh bị trộn, nên không xếp hạng các điều kiện. Thí nghiệm này chỉ chạy trên Dev.

*Bảng 4.7. RQ2: chênh lệch ghép cặp macro-F1 giữa các giá trị k trên Holdout (22 kịch bản; KTC 95% danh nghĩa).*

| Phương pháp | So sánh (sau − trước) | Chênh lệch | KTC 95% |
|---|---|---|---|
| Rule | k=2 minus $k{=}0$ | −0,003 | [−0,021, +0,004] |
|  | $k{=}4$ minus k=2 | ±0,000 | [±0,000, ±0,000] |
|  | $k{=}4$ minus $k{=}0$ | −0,003 | [−0,021, +0,004] |
| Qwen | k=2 minus $k{=}0$ | +0,037 | [−0,061, +0,113] |
|  | $k{=}4$ minus k=2 | ±0,000 | [−0,031, +0,038] |
|  | $k{=}4$ minus $k{=}0$ | +0,037 | [−0,049, +0,099] |

**H1: đồng thuận làm bộ lọc.** Bảng 4.8 cho bốn con số mô tả H1. Trên Holdout ở k = 2, H1 chấp nhận tự động 77,1% cửa sổ và đúng 94,0% trong số đó, trong khi Rule một mình đúng khoảng 82% mọi cửa sổ. Đây là hai tập cửa sổ khác nhau (các cửa sổ được chấp nhận là những cửa sổ dễ hơn), nên hiệu +0,124 là hiệu ứng chọn lọc chứ không phải accuracy của cả quy trình, vốn không được đo. Trên các cửa sổ H1 gửi đi xét lại, Rule đúng 40,0% và Qwen 20,0%: các cửa sổ này khó với cả hai phương pháp.

*Bảng 4.8. H1 (chấp nhận khi Rule và Qwen đồng ý): độ phủ, accuracy trên cửa sổ được chấp nhận, accuracy của Rule trên mọi cửa sổ và accuracy của từng phương pháp trên cửa sổ H1 gửi xét lại. Accuracy không phải macro-F1. Khoảng tin cậy từng giá trị có trong bảng kết quả của đồ án; ở đây chỉ hiện khoảng cho hiệu.*

| Tập | k | Độ phủ | Acc. chấp nhận | Rule (mọi cửa sổ) | H1 chấp nhận − Rule [KTC] | Rule (xét lại) | Qwen (xét lại) |
|---|---|---|---|---|---|---|---|
| Holdout | 0 | 0,817 | 0,854 | 0,817 | +0,037 [−0,015, +0,098] | 0,650 | 0,200 |
|  | 2 | 0,771 | 0,941 | 0,817 | +0,124 [+0,068, +0,189] | 0,400 | 0,200 |
|  | 4 | 0,761 | 0,940 | 0,817 | +0,123 [+0,067, +0,189] | 0,423 | 0,192 |
| OOD-1 | 0 | 0,597 | 0,706 | 0,456 | +0,250 [+0,190, +0,327] | 0,087 | 0,435 |
|  | 2 | 0,544 | 0,677 | 0,456 | +0,221 [+0,134, +0,324] | 0,192 | 0,423 |
|  | 4 | 0,544 | 0,710 | 0,456 | +0,254 [+0,175, +0,348] | 0,154 | 0,423 |
| OOD-2 | 0 | 0,600 | 0,483 | 0,379 | +0,103 [+0,006, +0,234] | 0,224 | 0,500 |
|  | 2 | 0,655 | 0,432 | 0,379 | +0,052 [−0,022, +0,155] | 0,280 | 0,460 |
|  | 4 | 0,579 | 0,500 | 0,379 | +0,121 [+0,021, +0,273] | 0,213 | 0,475 |

### 4.2.3 RQ3: hiệu năng trên họ chưa thấy

**RO3.1: điểm tuyệt đối và mức giảm.** Bảng 4.9 cho macro-F1 trên từng tập với tập lớp cố định. Trên Holdout giới hạn ở năm lớp của OOD-1, Rule đạt 0,872, 0,857 và 0,857 ở k = 0, 2, 4 so với 0,427, 0,418 và 0,419 trên OOD-1; hiệu là mức giảm ở Bảng 4.10 và không so sánh được với điểm sáu lớp của RQ2. Cả ba phương pháp đều giảm trên cả hai tập chưa thấy, và cả 18 khoảng của ΔF1 đều loại trừ 0 (mọi giá trị vượt ngưỡng báo cáo 0,1, vốn không phải tiêu chí quyết định). Trên OOD-1, Rule giảm 0,439 đến 0,445, Qwen 0,291 đến 0,294 và H4† 0,354 đến 0,388; trên OOD-2, Rule giảm 0,278 đến 0,281, Qwen 0,160 đến 0,250 và H4† 0,150 đến 0,226.

*Bảng 4.9. Macro-F1 theo tập dữ liệu với tập lớp cố định (KTC 95% theo kịch bản, danh nghĩa). H4† được tính sau từ nhãn đã lưu. OOD-1 không có lớp Noise nên Holdout cũng được chấm trên năm lớp chung.*

| Tập (số kịch bản) | k | Rule | Qwen | H4† |
|---|---|---|---|---|
| Holdout (6 lớp) | 0 | 0,826 [0,749, 0,866] | 0,732 [0,643, 0,789] | 0,785 [0,700, 0,837] |
|  | 2 | 0,823 [0,736, 0,866] | 0,769 [0,657, 0,823] | 0,834 [0,732, 0,880] |
|  | 4 | 0,823 [0,736, 0,866] | 0,769 [0,666, 0,823] | 0,838 [0,740, 0,887] |
| Holdout trên 5 lớp chung với OOD-1 | 0 | 0,872 [0,794, 0,923] | 0,794 [0,683, 0,874] | 0,857 [0,748, 0,932] |
|  | 2 | 0,857 [0,780, 0,910] | 0,763 [0,683, 0,820] | 0,848 [0,763, 0,903] |
|  | 4 | 0,857 [0,780, 0,910] | 0,775 [0,684, 0,839] | 0,861 [0,767, 0,922] |
| OOD-1 (5 lớp) | 0 | 0,427 [0,275, 0,574] | 0,500 [0,347, 0,665] | 0,503 [0,336, 0,671] |
|  | 2 | 0,418 [0,267, 0,568] | 0,469 [0,326, 0,612] | 0,483 [0,321, 0,636] |
|  | 4 | 0,419 [0,265, 0,569] | 0,483 [0,341, 0,637] | 0,472 [0,309, 0,632] |
| OOD-2 (6 lớp) | 0 | 0,545 [0,475, 0,601] | 0,572 [0,473, 0,653] | 0,635 [0,572, 0,687] |
|  | 2 | 0,545 [0,475, 0,601] | 0,520 [0,404, 0,610] | 0,609 [0,551, 0,662] |
|  | 4 | 0,545 [0,475, 0,601] | 0,548 [0,444, 0,626] | 0,626 [0,570, 0,682] |

*Bảng 4.10. Mức giảm ΔF1 = F1(Holdout) − F1(chưa thấy) trên các lớp chung; dương nghĩa là giảm. Bộ ước lượng tập lớp cố định (chính), 1000 lần lấy mẫu, KTC 95% danh nghĩa. Phiên bản bộ ước lượng mặc định ở Phụ lục A.*

| Tập chưa thấy | k | Rule | Qwen | H4† |
|---|---|---|---|---|
| OOD-1 | 0 | +0,445 [+0,277, +0,605] | +0,294 [+0,104, +0,457] | +0,354 [+0,159, +0,526] |
|  | 2 | +0,439 [+0,273, +0,599] | +0,294 [+0,131, +0,443] | +0,365 [+0,195, +0,529] |
|  | 4 | +0,439 [+0,273, +0,600] | +0,291 [+0,115, +0,443] | +0,388 [+0,212, +0,559] |
| OOD-2 | 0 | +0,281 [+0,182, +0,354] | +0,160 [+0,040, +0,272] | +0,150 [+0,055, +0,228] |
|  | 2 | +0,278 [+0,172, +0,352] | +0,250 [+0,099, +0,362] | +0,226 [+0,110, +0,295] |
|  | 4 | +0,278 [+0,172, +0,352] | +0,222 [+0,087, +0,330] | +0,213 [+0,100, +0,284] |

Bảng 4.11 cho hiệu giữa mức giảm của Rule và Qwen. Trên OOD-1, Qwen giảm ít hơn Rule ở cả ba giá trị k (D = +0,151, +0,145, +0,147), và các khoảng đồng thời đã hiệu chỉnh cho họ sáu phép so sánh vẫn loại trừ 0. Trên OOD-2 kết quả phụ thuộc k: D = +0,121 ở k = 0 (khoảng đã hiệu chỉnh loại trừ 0 với biên 0,001, p đã hiệu chỉnh 0,0466), nhưng +0,028 ở k = 2 và +0,056 ở k = 4, với khoảng chứa 0.

*Bảng 4.11. D = ΔF1(Rule) − ΔF1(Qwen); D > 0 nghĩa là Qwen giảm ít hơn. KTC 95% danh nghĩa và KTC đồng thời max-|t| cho họ sáu phép so sánh (10.000 lần lấy mẫu, seed 42, c = 2,592); giá trị p chưa/đã hiệu chỉnh. Bộ ước lượng tập lớp cố định. So sánh với H4 mang tính thăm dò và ở Phụ lục A.*

| Tập chưa thấy | k | D | KTC danh nghĩa | KTC đồng thời | p chưa / đã hiệu chỉnh | Đọc KTC đồng thời |
|---|---|---|---|---|---|---|
| OOD-1 | 0 | +0,151 | [+0,090, +0,230] | [+0,058, +0,245] | 0,0004 / 0,0008 | loại trừ 0 |
|  | 2 | +0,145 | [+0,066, +0,232] | [+0,037, +0,252] | 0,0015 / 0,0059 | loại trừ 0 |
|  | 4 | +0,147 | [+0,059, +0,251] | [+0,022, +0,272] | 0,0052 / 0,0176 | loại trừ 0 |
| OOD-2 | 0 | +0,121 | [+0,027, +0,207] | [+0,001, +0,240] | 0,0096 / 0,0466 | loại trừ 0 (sát biên) |
|  | 2 | +0,028 | [−0,093, +0,139] | [−0,129, +0,185] | 0,6502 / 0,9919 | chứa 0 |
|  | 4 | +0,056 | [−0,073, +0,175] | [−0,107, +0,220] | 0,3707 / 0,8568 | chứa 0 |

Chênh lệch tuyệt đối giữa các phương pháp được tính riêng (Phụ lục A, Bảng A.4). Trên OOD-1, Qwen trừ Rule là +0,073 [+0,031, +0,131], +0,051 [+0,016, +0,090] và +0,065 [+0,016, +0,121] ở k = 0, 2, 4. Trên toàn OOD-2 là +0,027 [−0,046, +0,089], −0,025 [−0,128, +0,054] và +0,003 [−0,100, +0,094], tất cả chứa 0, và H4† trừ Rule là +0,090, +0,064 và +0,081, tất cả loại trừ 0. Bảng 4.12 tách OOD-2 theo loại kỹ thuật trong một phân tích nhóm con post hoc với chỉ 9, 6 và 3 kịch bản. Trên ba họ thật sự mới, Qwen trừ Rule âm với khoảng chứa 0 (khoảng −0,08 đến −0,02), còn trên hai họ dùng lại kỹ thuật OOD-1 là +0,136 đến +0,169 với khoảng loại trừ 0. H4† trừ Rule dương trên cả họ mới (+0,049 đến +0,073) lẫn họ dùng lại, nhưng âm (−0,204 [−0,351, +0,000]) trên họ lành tính, nơi macro-F1 là recall của Noise: Rule nhận ra 12 trên 27 cửa sổ Noise (44%) và H4† 7 trên 27 (26%).

*Bảng 4.12. OOD-2 tách theo loại kỹ thuật: chênh lệch macro-F1 tuyệt đối so với Rule (tập lớp cố định; KTC 95% danh nghĩa). Nhóm con post hoc, thăm dò, ít kịch bản.*

| Loại kỹ thuật (kịch bản, lớp) | k | Qwen − Rule | H4† − Rule |
|---|---|---|---|
| kỹ thuật mới (9 kịch bản, 4 lớp) | 0 | −0,075 [−0,198, +0,114] | +0,073 [+0,025, +0,125] |
|  | 2 | −0,039 [−0,164, +0,122] | +0,049 [+0,002, +0,122] |
|  | 4 | −0,023 [−0,143, +0,131] | +0,059 [+0,016, +0,145] |
| kỹ thuật dùng lại từ OOD-1 (6, 5 lớp) | 0 | +0,149 [+0,121, +0,192] | +0,210 [+0,145, +0,341] |
|  | 2 | +0,136 [+0,105, +0,179] | +0,214 [+0,153, +0,319] |
|  | 4 | +0,169 [+0,104, +0,308] | +0,214 [+0,153, +0,319] |
| lành tính (3, 1 lớp) | 0 | −0,204 [−0,351, ±0,000] | −0,204 [−0,351, ±0,000] |
|  | 2 | −0,204 [−0,351, ±0,000] | −0,204 [−0,351, ±0,000] |
|  | 4 | −0,204 [−0,351, ±0,000] | −0,204 [−0,351, ±0,000] |

**Bằng chứng trên họ chưa thấy.** Bảng 4.6 cũng cho giá trị Jaccard của các tập chưa thấy. Jaccard trung bình của Rule ở k = 2 giảm từ 0,821 trên Holdout xuống 0,696 trên OOD-1 và 0,662 trên OOD-2, trong khi của Qwen đi từ 0,754 xuống 0,752 và 0,717 (hiệu của các trung bình, không có khoảng cho thay đổi). Qwen trừ Rule là +0,055 [+0,026, +0,085] ở k = 2 và +0,092 [+0,052, +0,141] ở k = 4 trên OOD-2 và +0,095 [+0,021, +0,193] ở k = 4 trên OOD-1, còn trên OOD-1 ở k = 2 là +0,056 [−0,014, +0,136]. Trên Holdout ở k = 2 dấu ngược lại.

**RO3.2: nhầm lẫn và Noise.** Bảng 4.13 đếm các cửa sổ bị gán Noise. OOD-1 không có Noise thật, nên mọi cửa sổ bị gán Noise là lỗi: Rule gán Noise 18 đến 20 trên 57 cửa sổ, Qwen 4 đến 6 và H4† 2 đến 3. Trên OOD-2 có 27 cửa sổ Noise thật; Rule gán Noise 83 trên 145 (12 đúng), Qwen 35 đến 47 và H4† 34 đến 46. Ở OOD-2, 42 trên 52 cửa sổ Impact bị Rule gán Noise ở k = 2, và 31 bị Qwen gán. Rule mặc định về Noise khi không mẫu nào khớp, và Qwen gán Discovery thành Impact trên OOD-1 (6 trên 20 ở k = 2). Qwen không sinh mã bằng chứng bịa hay đầu ra không hợp lệ nào trên cả hai tập. Ma trận nhầm lẫn ở k = 2 nằm ở Phụ lục C.

*Bảng 4.13. Số cửa sổ bị mỗi phương pháp gán Noise, kèm số thật sự là Noise trong ngoặc (số cửa sổ).*

| Tập | k | Số cửa sổ | Noise thật | Rule | Qwen | H4† |
|---|---|---|---|---|---|---|
| Holdout | 0 | 109 | 21 | 26 (14) | 12 (7) | 12 (7) |
|  | 2 | 109 | 21 | 22 (14) | 14 (14) | 13 (13) |
|  | 4 | 109 | 21 | 22 (14) | 14 (13) | 12 (12) |
| OOD-1 | 0 | 57 | 0 | 20 (0) | 6 (0) | 3 (0) |
|  | 2 | 57 | 0 | 18 (0) | 5 (0) | 2 (0) |
|  | 4 | 57 | 0 | 18 (0) | 4 (0) | 2 (0) |
| OOD-2 | 0 | 145 | 27 | 83 (12) | 36 (7) | 35 (7) |
|  | 2 | 145 | 27 | 83 (12) | 47 (7) | 46 (7) |
|  | 4 | 145 | 27 | 83 (12) | 35 (7) | 34 (7) |

**RO3.3: điểm độ lạ.** Bảng 4.14 liệt kê điểm độ lạ 1 đến 3 bên cạnh accuracy ở k = 2. Điểm do người dựng từng họ chấm một lần trước khi chấm, trừ `ext_lateral`‡, có điểm là ước tính từ mô tả họ, không có bản chấm gốc; câu "chấm trước khi chạy bởi người dựng" đúng với tám trong chín họ. Ở OOD-1, điểm độ lạ cao nhất không xếp hạng họ khó nhất: `definer_escalation` (3, 3, 3) có accuracy Rule cao nhất (0,688) trong ba họ, trong khi `ext_lateral` (2, 3, 2) thấp nhất (0,208 với Rule và 0,333 với Qwen). Ở OOD-2, họ có độ lạ thấp nhất về yếu tố phụ thuộc, `lockhog`, có accuracy thấp nhất (0,111 với Rule, 0,185 với Qwen). Với ba và sáu họ, không tính tương quan nào và bảng mang tính mô tả; không thể gọi tên yếu tố nào là ảnh hưởng nhất.

*Bảng 4.14. Điểm độ lạ (cấu trúc SQL, hàm riêng PostgreSQL, chuỗi phụ thuộc; 1 đến 3) và accuracy ở k = 2 theo họ. ‡ ước tính từ mô tả họ, không do người dựng chấm. n là số cửa sổ; một người chấm cho mỗi họ.*

| Tập | Họ | SQL | PG | Dep. | Kỹ thuật | n | Rule | Qwen | H4† |
|---|---|---|---|---|---|---|---|---|---|
| OOD-1 | `lo_exfil` | 2 | 3 | 2 | -- | 17 | 0,588 | 0,765 | 0,765 |
| OOD-1 | `definer_escalation` | 3 | 3 | 3 | -- | 16 | 0,688 | 0,688 | 0,812 |
| OOD-1 | `ext_lateral`‡ | 2 | 3 | 2 | -- | 24 | 0,208 | 0,333 | 0,333 |
| OOD-2 | `timing_oracle` | 2 | 2 | 3 | mới | 22 | 0,455 | 0,318 | 0,500 |
| OOD-2 | `xml_veil` | 3 | 3 | 2 | mới | 27 | 0,333 | 0,519 | 0,519 |
| OOD-2 | `lockhog` | 2 | 2 | 1 | mới | 27 | 0,111 | 0,185 | 0,185 |
| OOD-2 | `lo_stash` | 2 | 3 | 3 | dùng lại từ OOD-1 | 21 | 0,286 | 0,571 | 0,714 |
| OOD-2 | `definer_backdoor` | 2 | 3 | 3 | dùng lại từ OOD-1 | 21 | 0,714 | 0,905 | 0,857 |
| OOD-2 | `benign_ops` | 1 | 2 | 1 | lành tính | 27 | 0,444 | 0,259 | 0,259 |

**H1 trên họ chưa thấy.** Bảng 4.8 cho thấy accuracy của các cửa sổ H1 chấp nhận giảm từ khoảng 0,94 trên Holdout xuống 0,68 đến 0,71 trên OOD-1 và 0,43 đến 0,50 trên OOD-2, nên sự đồng thuận giữa hai phương pháp không còn là dấu hiệu nhãn đúng. Trên các cửa sổ H1 gửi xét lại, Qwen đúng nhiều hơn Rule ở cả hai tập chưa thấy (0,42 đến 0,50 so với 0,09 đến 0,28).

## 4.3 Phân tích kết quả

**Đọc các khoảng tin cậy.** Các so sánh dưới đây theo một quy tắc: chỉ khoảng loại trừ 0 mới được coi là bằng chứng. Trên Holdout ở k = 0, hiệu Qwen − Rule là −0,094 với khoảng loại trừ 0, nên Rule đạt điểm cao hơn; đây là thiết lập thăm dò và ba giá trị k không được hiệu chỉnh đa so sánh. Ở thiết lập chính k = 2, hiệu là −0,053 [−0,131, +0,016], và ở k = 4 là −0,053 [−0,133, +0,021]: Rule có ước lượng điểm cao hơn, nhưng bằng chứng chưa đủ để khẳng định có khác biệt, và nó không cho thấy hai phương pháp bằng nhau. Các chênh lệch ghép cặp giữa các giá trị k đều chứa 0, với độ rộng khoảng ±0,1 với Qwen, nên dữ liệu không thể nói ngữ cảnh có giúp Qwen hay không, và cũng không thể nói là không.

**Mức giảm trên họ chưa thấy.** Mọi phương pháp và tập đều có mức giảm (Bảng 4.10), và phân tích độ nhạy với bộ ước lượng mặc định cho cùng ước lượng điểm, với cận dưới nhỏ hơn ở OOD-1 (ví dụ Rule ở k = 0: [+0,196, +0,594] so với [+0,277, +0,605]) và cùng kết luận (Phụ lục A). Hiệu của các mức giảm D chọn lọc hơn. Sau khi hiệu chỉnh cho họ sáu phép so sánh, các khoảng đồng thời loại trừ 0 với OOD-1 ở mọi k và với OOD-2 ở k = 0, và chứa 0 với OOD-2 ở k = 2 và k = 4. Ô OOD-2 ở k = 0 mong manh: cận dưới của nó là +0,001 và p đã hiệu chỉnh là 0,0466, nên seed bootstrap khác hay cách chọn họ khác có thể đổi kết quả. Nó được báo cáo là sát biên và không dùng làm bằng chứng chính. OOD-1 có vấn đề độc lập nên ba ô của nó cũng chỉ là bằng chứng thăm dò. Loại trừ 0 chỉ nói rằng mức giảm của Qwen nhỏ hơn trên các kịch bản được lấy mẫu; nó không xác lập tổng quát hóa sang họ kỹ thuật khác.

**Vì sao D > 0 không phải điểm cao hơn.** Một phần D đến từ việc Qwen thấp hơn Rule trên Holdout. Trên năm lớp chung ở k = 0, điểm Holdout là 0,872 với Rule và 0,794 với Qwen, trong khi trên OOD-1 là 0,427 và 0,500. Trên OOD-2 ở k = 2, điểm tuyệt đối của Qwen (0,520) thấp hơn của Rule (0,545), dù D dương ở đó. Lợi thế của Qwen trên OOD-1 hiện ra ở dạng tuyệt đối (+0,051 đến +0,073), nhưng không hiện ra trên toàn OOD-2.

**Loại kỹ thuật nào.** Kết quả gộp OOD-2 che giấu hai bức tranh khác nhau (Bảng 4.12). Trên ba họ thật sự mới không có bằng chứng Qwen vượt Rule. Lợi thế của Qwen tập trung ở hai họ dùng lại kỹ thuật của OOD-1. Các kỹ thuật đó cũng vắng ở Dev, luật và prompt; vì sao Qwen làm tốt hơn ở đó thì chưa biết, và quan sát này được đưa ra sau khi thấy kết quả và không phải giả thuyết đã đăng ký. Sai số chuẩn phản ánh kịch bản chứ không phải họ, nên khoảng của phép tách này có lẽ quá hẹp.

**H4 và H1.** H4† có macro-F1 cao hơn Rule trên toàn OOD-2 (+0,064 đến +0,090) và trên các họ mới (+0,049 đến +0,073, với cận dưới ở k = 2 là +0,002), không đổi kết quả trên Holdout (khoảng chứa 0), và phải trả bằng khả năng nhận ra hoạt động lành tính (recall Noise 44% xuống 26%). Bằng chứng của nó là thăm dò: H4 được nghĩ ra sau OOD-1, và nhóm con có 9 kịch bản. H1 lọc bỏ các cửa sổ mà hai phương pháp bất đồng, nhưng accuracy của những gì nó chấp nhận giảm còn 0,43 đến 0,50 trên OOD-2, và accuracy của cả quy trình có người xét lại không được đo.

## 4.4 Diễn giải kết quả

Bảng 4.15 liệt kê các tuyên bố chính mà kết quả hỗ trợ và không hỗ trợ. Giới hạn quan trọng nhất đối với diễn giải là giới hạn nêu trong mọi hàng: dữ liệu là mô phỏng, nhỏ và chưa được cách ly hoàn toàn.

*Bảng 4.15. Các tuyên bố chính, bằng chứng và cách diễn đạt mà bằng chứng cho phép. Bảng kiểm đầy đủ ở Phụ lục D.*

| Tuyên bố | Bằng chứng | Cách đọc được phép |
|---|---|---|
| C2 ghi mọi hành động | 136/136; OOD-1 62/63; OOD-2 159/159 | Đúng cho khối lượng mô phỏng đã thử; sự kiện kết nối được ghi theo thiết kế; một địa chỉ nguồn; né log và overhead chưa thử. |
| C2 là cấu hình nhẹ nhất | 322 bản ghi so với 199 và 204 | Sai. C2 là cấu hình duy nhất đạt tiêu chí và cũng nhiều bản ghi nhất. |
| Nhóm phiên đủ tốt | F1 theo cặp khoảng 0,11 | Không được hỗ trợ: yếu, là giới hạn của thiết kế một địa chỉ. |
| Rule tốt hơn Qwen trên họ đã biết | −0,094 [−0,151, −0,034] ở k = 0; −0,053 [−0,131, +0,016] ở k = 2 | Điểm Rule cao hơn ở k = 0 (thăm dò); bằng chứng chưa đủ ở k = 2 và k = 4; không phải tuyên bố tương đương. |
| Ngữ cảnh giúp Qwen | k = 2 trừ k = 0: +0,037 [−0,061, +0,113] | Không được hỗ trợ; k = 0 cũng khác số ví dụ. |
| Qwen giảm ít hơn Rule trên kỹ thuật chưa thấy | D ở Bảng 4.11 | Đúng trên OOD-1 (mọi k) và OOD-2 ở k = 0 (sát biên); chưa đủ ở k = 2, 4; OOD-1 không độc lập; D > 0 không phải điểm cao hơn. |
| Qwen tổng quát hóa tốt hơn sang kỹ thuật mới | Họ mới: Qwen − Rule −0,075 [−0,198, +0,114] ở k = 0 | Không được hỗ trợ. Lợi thế của nó nằm ở họ dùng lại kỹ thuật OOD-1 (post hoc). |
| H4 là giải pháp tổng quát | OOD-2: +0,064 đến +0,090; Holdout: khoảng chứa 0; recall lành tính 44% xuống 26% | Thăm dò: cao hơn Rule trên OOD-2 và họ mới (danh nghĩa), đổi lại nhiều cảnh báo sai. |
| H1 làm hệ thống chính xác hơn | Accuracy cửa sổ chấp nhận 0,94 so với Rule 0,82 trên Holdout | Không được hỗ trợ như tuyên bố về cả hệ thống: tập cửa sổ khác nhau; accuracy của người xét lại không đo. |
| Ground truth bằng chứng đã được kiểm chứng | Jaccard 0,775 [0,667, 0,883] trên 20 cửa sổ Dev | Đồng thuận sơ bộ với một người rà soát AI; không phải kiểm chứng của con người. |
| ΔF1 do độ lạ gây ra | Mức giảm dương trên cả hai tập | Không được hỗ trợ: các tập khác nhau về tác giả, kích thước và tỉ lệ lớp. |
| Không có rò rỉ từ Dev sang Holdout | Cả hai điểm giảm 0,018 | Không được hỗ trợ: chỉ là "không thấy mức giảm lớn". |

## 4.5 So sánh với tài liệu

Các nguồn chỉ được mô tả trong báo cáo này trong chừng mực tóm tắt của chúng cho phép, và so sánh dưới đây mang tính định tính; các nguồn khác nhau về miền và dữ liệu, và không có con số nào được chuyển từ chúng sang.

Meng và cộng sự lập luận rằng phân loại log bảo mật bằng LLM một bước bịa tên kỹ thuật và mất ngữ cảnh [4]. Trong nghiên cứu này, không gian nhãn đóng và mã bằng chứng được kiểm với cửa sổ; Qwen không sinh đầu ra không hợp lệ hay mã bịa nào. Điều này cho thấy thất bại cụ thể đó không xảy ra trong các cửa sổ nhỏ này, và không kiểm tra việc bịa tên kỹ thuật mà RHINO mô tả. Độ nhạy của Qwen với prompt (macro-F1 0,515 với prompt đầu và 0,787 sau khi phát triển trên Dev) phù hợp với báo cáo rằng prompt không hướng dẫn là không tin cậy và chỉ dẫn cải thiện kết quả [3, 25]. Điểm giảm khi ngữ cảnh bị làm hỏng nhất quán với mối quan ngại của Meng và cộng sự về ánh xạ mất ngữ cảnh, nhưng thí nghiệm của ta không cô lập một cơ chế.

Neves và cộng sự thấy phương pháp luật chính xác nhưng recall thấp trong xác định kiểu hình lâm sàng, LLM zero-shot có recall cao hơn, và kết hợp hai bên có thể hướng việc rà soát thủ công vào các ca bất đồng [21]. Mô hình trên kỹ thuật chưa thấy ở đây tương tự về hướng: Rule mặc định về Noise khi không khớp gì nên bỏ sót cửa sổ tấn công (83 cửa sổ bị gán Noise so với 27 thật ở OOD-2), và LLM thu hồi được một phần; H1 và H4 là hai cách dùng sự bất đồng. Ngược lại, trên họ đã biết Rule có ước lượng điểm cao hơn, trong khi ở nghiên cứu lâm sàng recall của LLM cao hơn nhiều, nên các nghiên cứu không đồng ý về phương pháp nào tốt hơn nói chung. Mehavilla và cộng sự báo cáo LLM không vượt mô hình cổ điển tốt nhất trên log mạng [26], và Landauer và cộng sự báo cáo LLM diễn giải đúng khoảng một phần ba bước tấn công và đạt yêu cầu một phần ba nữa [14]; không kết quả nào mâu thuẫn với phát hiện ở đây rằng Qwen không vượt rõ ràng một baseline minh bạch, nhưng chúng liên quan đến dữ liệu khác.

Biswas và Niemi mô tả đánh đổi giữa ghi log không đủ và chi phí ghi log thừa [10]. Kết quả chỉ C2 đạt tiêu chí recall và cũng ghi nhiều bản ghi nhất nhất quán với đánh đổi này về dung lượng log, nhưng overhead lên xử lý truy vấn không được đo ở đây. Bridges và cộng sự nhấn mạnh sự thiếu hụt dữ liệu có nhãn thực tế cho nghiên cứu honeypot [38], khớp với việc nghiên cứu này phụ thuộc vào dữ liệu mô phỏng. Không so sánh được các ngưỡng không hoạt động 1 đến 30 phút dùng ở đây với khoảng nghỉ nào do Song và cộng sự báo cáo [1], vì nội dung nguồn đó về điểm này chưa được xác minh cho báo cáo.

## 4.6 Hàm ý của kết quả

Với khối lượng mô phỏng kiểu đã dùng, C2 là cấu hình duy nhất ghi mọi hành động và đạt tiêu chí, và cũng là cấu hình ghi nhiều bản ghi nhất; lựa chọn C2 nên được đọc là mức tối thiểu đạt tiêu chí do nhóm chọn, không phải cấu hình phù hợp mọi môi trường. Ghi mọi hành động chưa đủ để dựng lại phiên trong bối cảnh một địa chỉ nguồn, nên một triển khai thực cần nhiều địa chỉ nguồn (hoặc khóa phiên khác) trước khi tin vào các cửa sổ có ngữ cảnh thật.

Giữa hai bộ phân loại, với hành vi quen thuộc, phương pháp luật có ước lượng điểm cao hơn ở mọi k và chạy tức thì, trong khi Qwen cần 4 đến 8 giây mỗi cửa sổ và không cho thấy lợi thế rõ rệt; vì vậy phương pháp luật là lựa chọn mặc định hợp lý, với Qwen là ý kiến thứ hai tùy chọn. Với kỹ thuật lạ, nghiên cứu cho thấy một rủi ro. Sự đồng thuận giữa hai phương pháp là dấu hiệu đúng yếu ở đó (accuracy của cửa sổ H1 chấp nhận giảm từ 0,94 xuống 0,43 đến 0,50), nên cổng tự động dựa trên đồng thuận là không an toàn nếu không có người xét lại. H4 là đề xuất có điều kiện: nó nâng điểm trên OOD-2 nhưng tăng cảnh báo sai với hoạt động lành tính, và bằng chứng của nó là thăm dò. Không tuyên bố nào trong số này áp dụng ngoài môi trường mô phỏng.

---

# Chương 5. Thảo luận

## 5.1 Nhắc lại vấn đề và mục tiêu nghiên cứu

Đồ án hỏi một honeypot PostgreSQL tương tác cao kết hợp phân tích hành vi có thể ghi nhận, nhận diện và giải thích các giai đoạn tấn công từ log tương tác đến mức nào, kể cả khi kẻ tấn công dùng kỹ thuật không được dùng để xây dựng phân tích. RQ1 hỏi cấu hình ghi log nào bỏ sót hành động nào và cấu hình nào đủ; RQ2 hỏi phương pháp luật hay mô hình ngôn ngữ few-shot chạy cục bộ gán giai đoạn tấn công chính xác hơn và chọn bằng chứng tốt hơn; và RQ3 hỏi cả hai mất bao nhiêu trên các họ kỹ thuật bị giữ lại khỏi phát triển. Ba câu hỏi tạo thành một chuỗi: cấu hình chọn ở RQ1 cung cấp log cho RQ2, và các phương pháp đóng băng ở RQ2 được áp dụng không đổi ở RQ3. Mọi câu trả lời áp dụng cho dữ liệu mô phỏng.

## 5.2 Tóm tắt các phát hiện chính

1. **F1. Log hành động đầy đủ, nhóm phiên yếu.** C2 ghi 136 trên 136 hành động mô phỏng, so với 116 và 121 của C0 và C1 (khoảng trống là prepared statement và hành động nhiều kết nối, không phải câu lệnh lỗi hay đăng nhập thất bại). C2 là cấu hình duy nhất đạt tiêu chí và là cấu hình ghi nhiều bản ghi nhất. Không dựng lại được phiên (F1 theo cặp khoảng 0,11), vì trình mô phỏng dùng một địa chỉ nguồn.
2. **F2. Họ đã biết: không có khác biệt phân biệt được ở thiết lập chính.** Phương pháp luật có ước lượng điểm macro-F1 cao hơn (0,823 so với 0,769 ở k = 2). Ở k = 0 khoảng của hiệu loại trừ 0, với Rule cao hơn (thăm dò); ở k = 2 và k = 4 nó chứa 0, nên bằng chứng chưa đủ, không phải bằng chứng của sự bằng nhau.
3. **F3. Không có bằng chứng ngữ cảnh giúp Qwen.** Điểm quan sát của Qwen cao hơn khi có ngữ cảnh, nhưng mọi chênh lệch ghép cặp giữa các giá trị k chứa 0, và k = 0 cũng có ít ví dụ hơn. Trên Dev, ngữ cảnh bị làm hỏng làm Qwen giảm điểm tới 0,09 trong khi Rule hầu như không đổi.
4. **F4. Cả hai phương pháp mất rất nhiều trên kỹ thuật chưa thấy.** Mức giảm macro-F1 khoảng từ 0,15 đến 0,45 với Rule, Qwen và H4†, và mọi khoảng loại trừ 0.
5. **F5. Qwen giảm ít hơn nhưng không vượt rõ Rule trên kỹ thuật mới.** Mức giảm của Qwen nhỏ hơn của Rule trên OOD-1 ở mọi k và trên OOD-2 ở k = 0 (sau cùng sát biên sau hiệu chỉnh), không phải ở k = 2 và k = 4 trên OOD-2. Đó là mức giảm nhỏ hơn chứ không phải điểm cao hơn, và trên ba họ thật sự mới, Qwen trừ Rule âm với khoảng chứa 0. Lợi thế của nó nằm ở các họ dùng lại kỹ thuật OOD-1, một quan sát post hoc.
6. **F6. Chiến lược lai đổi cảnh báo sai lấy recall.** H4†, chuyển cho mô hình ngôn ngữ các cửa sổ mà luật gán Noise, nâng macro-F1 trên OOD-2 và các họ mới của nó (danh nghĩa) và không đổi họ đã biết, nhưng hạ recall Noise trên hoạt động lành tính từ 44% xuống 26%. H1 không còn là bộ lọc tin cậy trên kỹ thuật chưa thấy (accuracy cửa sổ chấp nhận 0,94, rồi 0,68 đến 0,71, rồi 0,43 đến 0,50).

Các phát hiện này cũng có giới hạn mà người đọc cần ghi nhớ. Dữ liệu là mô phỏng và nhỏ; ngữ cảnh là lý tưởng; dữ liệu kiểm tra chưa được cách ly hoàn toàn (Claude đọc một phần Holdout, SQL của OOD-1 hiện ra sau khi đóng băng, và OOD-2 bị lộ một phần qua thông báo của bộ kiểm tra lần chạy), và ảnh hưởng của điều này không được đo; prompt được tinh chỉnh trên Dev; các khoảng bao phủ kịch bản chứ không phải họ, và chỉ ba họ thật sự mới; ΔF1 so sánh các tập khác nhau về tác giả, kích thước và tỉ lệ lớp; một số phân tích được chọn sau khi thấy kết quả; và bằng chứng được một người rà soát AI kiểm tra. Do đó các phát hiện cho thấy xu hướng trong một bối cảnh có kiểm soát và không thể hỗ trợ tuyên bố rằng phương pháp nào tổng quát hóa tốt hơn.

---

# Chương 6. Kết luận và hướng phát triển

## 6.1 Kết luận

Trên khối lượng PostgreSQL mô phỏng, cấu hình ghi log có pgAudit cùng ghi kết nối, lỗi và tham số (C2) ghi đủ cả 136 hành động tấn công và lành tính, điều mà không cấu hình nào khác đã thử làm được, nhưng nó cũng ghi nhiều bản ghi log nhất, và việc ghi đầy đủ ở mức hành động không cho phép dựng lại phiên: nhóm theo địa chỉ và người dùng chỉ đạt F1 theo cặp khoảng 0,11 vì trình mô phỏng dùng một địa chỉ. Điểm yếu này là tính chất của thiết kế thí nghiệm và cũng là lý do phải thử độ nhạy của ngữ cảnh đưa cho bộ phân loại.

Trên các họ kỹ thuật đã biết, phương pháp luật đạt điểm cao hơn mô hình Qwen3 8B chạy cục bộ (macro-F1 0,823 so với 0,769 ở thiết lập chính hai truy vấn đứng trước), nhưng chênh lệch không phân biệt được với 0 ở thiết lập đó, và dữ liệu không cho biết ngữ cảnh có giúp mô hình hay không. Luật, mô hình và các chiến lược lai đều mất từ khoảng 0,15 đến 0,45 macro-F1 trên các họ kỹ thuật bị giữ lại khỏi phát triển. Mô hình mất ít hơn luật trên tập chưa thấy thứ nhất và ở một thiết lập của tập thứ hai, nhưng trên các họ thật sự mới không có bằng chứng nó vượt luật, và một phương pháp lai chuyển cho mô hình khi luật không nhận ra gì có thu thêm điểm, đổi lại nhiều cảnh báo sai hơn với hoạt động lành tính.

Các kết luận này dựa trên dữ liệu mô phỏng nhỏ, ngữ cảnh lý tưởng và dữ liệu kiểm tra chưa được cách ly hoàn toàn khỏi những người và công cụ đã xây dựng các phương pháp. Chúng cho thấy xu hướng và không xác lập rằng phương pháp nào tổng quát hóa tốt hơn.

## 6.2 Hướng phát triển

- Dùng trình mô phỏng nhiều container, mỗi container có địa chỉ nguồn riêng, để đánh giá dựng lại phiên một cách thực tế, và đo ảnh hưởng của phiên không hoàn hảo lên phân loại (thí nghiệm làm hỏng ngữ cảnh chỉ chạy trên Dev và nên lặp lại trên dữ liệu kiểm tra).
- Xây một tập kiểm tra mới, lớn hơn (ít nhất 40 kịch bản, nhiều cửa sổ Noise) do người không tham gia phát triển phương pháp viết, với sự cách ly được thực thi.
- Thêm tập họ chưa thấy thứ ba với nhiều họ hơn, vì độ bất định nằm ở số họ chứ không phải số kịch bản mỗi họ.
- Cố định số ví dụ ở mọi k để tách độ dài ngữ cảnh khỏi số ví dụ.
- Thêm người rà soát bằng chứng thứ hai là con người; đăng ký trước họ các phép so sánh và thủ tục đa so sánh; và xác minh nguồn cho các thủ tục thống kê mà báo cáo này mô tả không kèm trích dẫn.
- Hoàn thành những gì chưa đo: né ghi log, overhead bằng pgbench và CPU/bộ nhớ khi suy luận; xem xét lại cấu hình proxy hoặc bắt gói tin đã nằm ngoài phạm vi.

---

# Tài liệu tham khảo

1. Song, Yuqian; Smaragdakis, Georgios; Griffioen, Harm, "Decoy Databases: Analyzing Attacks on Public Facing Databases", Proceedings of the 2025 ACM Internet Measurement Conference.
2. Wegerer, Mathias; Tjoa, Simon, "Defeating the Database Adversary Using Deception - A MySQL Database Honeypot", 2016 International Conference on Software Security and Assurance (ICSSA).
3. Schärmer, Alina; Landauer, Max; Skopik, Florian; Wurzenberger, Markus; Squarcina, Marco, "Let the Alerts Speak: LLM-Based IDS Alert Interpretation for SOC Triage", Availability, Reliability and Security. ARES 2026 International Workshops.
4. Meng, Fanchao; Gui, Jiaping; Li, Yunbo; Wu, Yue, "RHINO: Guided Reasoning for Mapping Network Logs to Adversarial Tactics and Techniques with Large Language Models", arXiv.
5. Gómez-López, Julio; Penco-Hernández, Daniel; Gómez-López, Óscar David; Martínez-López, Francisco Javier et al., "A Multi-Agent Framework for SQL Injection Auditing with LLM-Driven Post-Exploitation and Privilege Escalation".
6. Rocha, Andressa; Pereira, Fabiola S.F., "What is the most targeted database? Using Honeypots to Characterize Attacks on Relational Databases".
7. Al Hafidz, Jundi; Lim, Charles; Silaen, Kalpin Erlangga, "Enhancing Detection Postgresql Shell Honeypot with Interactive Query Response", 2025 IEEE 2nd International Conference on Cryptography, Informatics, and Cybersecurity (ICoCICs).
8. Karo Karo, Ardhi Atmaja; Lim, Charles; Silaen, Kalpin Erlangga, "Capturing Post-Login Behaviors in MS SQL Honeypots: Profiling Attackers via a Protocol-Aware TDS Relay", 2025 15th International Conference on Information \&amp; Communication Technology and System (ICTS).
9. Zimniak, Marcin; Getta, Janusz R.; Benn, Wolfgang, "Predicting database workloads through mining periodic patterns in database audit trails".
10. Biswas, Debmalya; Niemi, Valtteri, "Transforming Privacy Policies to Auditing Specifications", 2011 IEEE 13th International Symposium on High-Assurance Systems Engineering.
11. Zeng, Lei; Xiao, Yang; Chen, Hui, "Auditing overhead, auditing adaptation, and benchmark evaluation in Linux".
12. Choi, Hoyong; Lee, Sangjin, "Forensic analysis of SQL server transaction log in unallocated area of file system".
13. Goseva-Popstojanova, Katerina; Anastasovski, Goce; Dimitrijevikj, Ana; Pantev, Risto; Miller, Brandon, "Characterization and classification of malicious Web traffic".
14. Landauer, Max; Hotwagner, Wolfgang; Boenke, Thorina; Skopik, Florian; Wurzenberger, Markus, "CAM-LDS: Cyber Attack Manifestations for Automatic Interpretation of System Logs and Security Alerts", arXiv.
15. Hossain, Nahid, "SLEUTH: Real-time Attack Scenario Reconstruction from COTS Audit Data".
16. ZhuⰠ, Tiantian; LiⰠ, Mingda; WangⰠ整, Zijun, "TraceBack: Sequence-Enhanced and Causality-Driven Attack Scenario Reconstruction from Provenance Graphs".
17. Zambianco, Marco; Facchinetti, Claudio; Siracusa, Domenico, "A Proactive Decoy Selection Scheme for Cyber Deception using MITRE ATT\&CK".
18. Meng, Yuqiao; Tang, Luoxi; Yu, Feiyang; Li, Xi; Yan, Guanhua; Yang, Ping; Xi, Zhaohan, "Benchmarking LLM-Assisted Blue Teaming via Standardized Threat Hunting", arXiv.
19. Yang, Longjing; Ye, Ayong; Liu, Yuanhuang; Lu, Wenting; Huang, Chuang, "APTREC: APT tactic/technique recognition based on large language model".
20. Wang, Ziyu; Fan, Zhanhao; Li, Yanjun; Zhang, Xing; Dai, Peng; Tian, Zhihong, "Toward Zero-Shot Multi-Label Identification of Attack Techniques: Enhancing LLM Reasoning With Semantic Evidence Discovery".
21. Neves, Bernardo; Moreira, José Maria; Gonçalves, Simão; Cerejo, Jorge; Da Silva, Nuno A.; Leite, Francisca et al., "Zero-shot learning for clinical phenotyping: Comparing LLMs and rule-based methods".
22. Duan, Ruixue; Li, Dan; Tong, Qiang; Yang, Tao; Liu, Xiaotong; Liu, Xiulei, "A Survey of Few-Shot Learning: An Effective Method for Intrusion Detection".
23. Liu, Hongyu; Lang, Bo, "Machine Learning and Deep Learning Methods for Intrusion Detection Systems: A Survey".
24. Alharthi, Ayesha; Alaryani, Meera; Kaddoura, Sanaa, "A comparative study of machine learning and deep learning models in binary and multiclass classification for intrusion detection systems".
25. Rehman, Mohammad Abdul; Shah, Syed Imad Ali; Anwar, Abbas; Islam, Noor, "From Flows to Words: Can Zero-/Few-Shot LLMs Detect Network Intrusions? A Grammar-Constrained, Calibrated Evaluation on UNSW-NB15", arXiv.
26. Mehavilla, Lorena; Rodríguez, María; García, José; Alesanco, Álvaro, "Evaluating large language models effectiveness for flow-based intrusion detection: a comparative study with ML and DL baselines".
27. Lust, Julia; Condurache, Alexandru P., "A survey on assessing the generalization area of neural networks: Uncertainty, out-of-distribution and adversarial samples".
28. Zhou, Yijin; Wang, Yuguang, "GROD: Enhancing Generalization of Transformer with Out-of-Distribution Detection".
29. Guo, Siyuan; Wildberger, Jonas; Schölkopf, Bernhard, "OUT-OF-VARIABLE GENERALIZATION FOR DISCRIMINATIVE MODELS".
30. Das, Vipin; Nair, Binoy B, "A novel multi-stage attack dataset for smart home intrusion detection".
31. Noelia Nunez Migliorisi, Teresita, "EMPIRICAL TELEMETRY-BASED METRICS FOR EVALUATING HONEYPOT REALISM AND DECEPTION EFFECTIVENESS".
32. Ferguson-Walter, Kimberly J; Major, Maxine M, "Examining the Efficacy of Decoy-based and Psychological Cyber Deception".
33. Weinberg, Abraham Itzhak, "PHANTOM: Polymorphic Honeytoken Adaptation with Narrative-Tailored Organisational Mimicry", arXiv.
34. Msaad, Mohamed; Srinivasa, Shreyas; Andersen, Mikkel M.; Audran, David H.; Orji, Charity U.; Vasilomanolakis, Emmanouil, "Honeysweeper: Towards Stealthy Honeytoken Fingerprinting Techniques", Secure IT Systems.
35. Kouremetis, Michael; Lawrence, Dean; Alford, Ron; Cheuvront, Zoe; Davila, David; Geyer, Benjamin; Haigh, Trevor et al., "Mirage: cyber deception against autonomous cyber attacks in emulation and simulation".
36. Wushishi, Usman; Hussain, Altaf; Khalid, Muhammad Imran; Hussain, Nasir; Jamjoom, Mona; Ullah, Zahid, "D3O-IIoT: deep reinforcement learning-driven dynamic deception orchestration for industrial IoT security".
37. Safargalieva, Anastasia; Rüffer, Artur; Vasilomanolakis, Emmanouil, "OHRA: dynamic multi-protocol LLM-based cyber deception".
38. Bridges, Robert A.; Mitchell, Thomas R.; Muñoz, Mauricio; Henriksson, Ted, "SoK: Honeypots \& LLMs, More Than the Sum of Their Parts?", arXiv.
39. scikit-learn developers, "StratifiedGroupKFold".
40. Ollama, "Ollama Documentation".

---

# Phụ lục A. Các bảng kết quả bổ sung

Mọi bảng được sinh từ `final_results_tables.csv`. Khoảng tin cậy là bootstrap 95% theo kịch bản, danh nghĩa, trừ khi nêu khác.

*Bảng A.1. Độ nhạy: ΔF1 với bộ ước lượng mặc định (macro-F1 lấy trung bình trên các lớp có mặt trong mỗi lần lấy mẫu). Ước lượng điểm bằng Bảng 4.10.*

| Tập chưa thấy | k | Rule | Qwen | H4† |
|---|---|---|---|---|
| OOD-1 | 0 | +0,445 [+0,196, +0,594] | +0,294 [+0,016, +0,448] | +0,354 [+0,070, +0,516] |
|  | 2 | +0,439 [+0,184, +0,588] | +0,294 [+0,023, +0,434] | +0,365 [+0,083, +0,513] |
|  | 4 | +0,439 [+0,184, +0,589] | +0,291 [+0,026, +0,426] | +0,388 [+0,102, +0,544] |
| OOD-2 | 0 | +0,281 [+0,168, +0,376] | +0,160 [+0,025, +0,296] | +0,150 [+0,025, +0,281] |
|  | 2 | +0,278 [+0,166, +0,364] | +0,250 [+0,111, +0,370] | +0,226 [+0,097, +0,301] |
|  | 4 | +0,278 [+0,166, +0,364] | +0,222 [+0,089, +0,335] | +0,213 [+0,087, +0,297] |

*Bảng A.2. Hiệu của các mức giảm, mọi cặp, bộ ước lượng tập lớp cố định (KTC danh nghĩa). Rule − Qwen là phép so sánh đã đăng ký; các phép có H4† mang tính thăm dò.*

| Tập chưa thấy | Cặp | k | D [KTC] |
|---|---|---|---|
| OOD-1 | Rule − Qwen | 0 | +0,151 [+0,090, +0,230] |
|  |  | 2 | +0,145 [+0,066, +0,232] |
|  |  | 4 | +0,147 [+0,059, +0,251] |
|  | Rule − H4 | 0 | +0,091 [+0,024, +0,174] |
|  |  | 2 | +0,073 [+0,024, +0,132] |
|  |  | 4 | +0,050 [−0,002, +0,115] |
|  | H4 − Qwen | 0 | +0,060 [+0,024, +0,094] |
|  |  | 2 | +0,071 [+0,002, +0,148] |
|  |  | 4 | +0,097 [+0,027, +0,176] |
| OOD-2 | Rule − Qwen | 0 | +0,121 [+0,027, +0,207] |
|  |  | 2 | +0,028 [−0,093, +0,139] |
|  |  | 4 | +0,056 [−0,073, +0,175] |
|  | Rule − H4 | 0 | +0,131 [+0,051, +0,214] |
|  |  | 2 | +0,052 [−0,011, +0,120] |
|  |  | 4 | +0,065 [−0,013, +0,143] |
|  | H4 − Qwen | 0 | −0,010 [−0,076, +0,047] |
|  |  | 2 | −0,024 [−0,119, +0,066] |
|  |  | 4 | −0,009 [−0,099, +0,071] |

*Bảng A.3. Độ nhạy: hiệu của các mức giảm với bộ ước lượng mặc định.*

| Tập chưa thấy | Cặp | k | D [KTC] |
|---|---|---|---|
| OOD-1 | Rule − Qwen | 0 | +0,151 [+0,092, +0,243] |
|  |  | 2 | +0,145 [+0,066, +0,238] |
|  |  | 4 | +0,147 [+0,060, +0,260] |
|  | Rule − H4 | 0 | +0,091 [+0,029, +0,187] |
|  |  | 2 | +0,073 [+0,025, +0,149] |
|  |  | 4 | +0,050 [−0,002, +0,128] |
|  | H4 − Qwen | 0 | +0,060 [+0,023, +0,096] |
|  |  | 2 | +0,071 [±0,000, +0,148] |
|  |  | 4 | +0,097 [+0,022, +0,178] |
| OOD-2 | Rule − Qwen | 0 | +0,121 [+0,027, +0,207] |
|  |  | 2 | +0,028 [−0,093, +0,141] |
|  |  | 4 | +0,056 [−0,073, +0,176] |
|  | Rule − H4 | 0 | +0,131 [+0,051, +0,215] |
|  |  | 2 | +0,052 [−0,011, +0,122] |
|  |  | 4 | +0,065 [−0,013, +0,143] |
|  | H4 − Qwen | 0 | −0,010 [−0,076, +0,049] |
|  |  | 2 | −0,024 [−0,121, +0,068] |
|  |  | 4 | −0,009 [−0,102, +0,072] |

*Bảng A.4. Chênh lệch macro-F1 tuyệt đối so với Rule, bộ ước lượng tập lớp cố định (KTC danh nghĩa). H4† được tính sau.*

| Tập | k | Qwen − Rule | H4† − Rule |
|---|---|---|---|
| Holdout (22 kịch bản) | 0 | −0,094 [−0,148, −0,031] | −0,041 [−0,096, +0,023] |
|  | 2 | −0,053 [−0,125, +0,019] | +0,011 [−0,029, +0,061] |
|  | 4 | −0,053 [−0,129, +0,023] | +0,016 [−0,033, +0,077] |
| OOD-1 (9 kịch bản) | 0 | +0,073 [+0,031, +0,131] | +0,076 [+0,026, +0,140] |
|  | 2 | +0,051 [+0,016, +0,089] | +0,065 [+0,023, +0,103] |
|  | 4 | +0,065 [+0,016, +0,121] | +0,054 [+0,014, +0,094] |
| OOD-2 toàn bộ (18 kịch bản) | 0 | +0,027 [−0,045, +0,089] | +0,090 [+0,044, +0,145] |
|  | 2 | −0,025 [−0,128, +0,054] | +0,064 [+0,018, +0,120] |
|  | 4 | +0,003 [−0,100, +0,093] | +0,081 [+0,034, +0,139] |

# Phụ lục B. Kết quả trên tập phát triển

Các kết quả này lấy từ tập Dev (102 truy vấn, 17 kịch bản), trên đó prompt được tinh chỉnh; điểm lạc quan. Giá trị chép từ nhật ký dự án.

*Bảng B.1. Ablation prompt cho Qwen3 8B trên Dev (macro-F1). A: cùng ví dụ ở mọi k; B: thêm ví dụ cân bằng; C: dấu CONTEXT/TARGET và chỉ dẫn chỉ gán đích; D: nêu lý do trước nhãn; E: định nghĩa nhãn chặt hơn; F: thêm ví dụ Collection. Prompt đầu tiên là điểm xuất phát; phương pháp luật là tham chiếu.*

| Prompt | k=0 | k=2 | k=4 | Ghi chú |
|---|---|---|---|---|
| Prompt đầu tiên | 0,698 | 0,515 | 0,472 | Noise bị gán Collection; Impact bị gán Exfiltration |
| A | 0,631 | – | – | kém prompt đầu ở k=0 |
| B | – | 0,639 | – | không phân biệt được mức tăng khi dùng riêng |
| C | 0,670 | 0,641 | – | +0,126 [+0,036, +0,240] ở k=2 |
| D | 0,622 | 0,513 | – | không có tác dụng |
| B+C | – | 0,706 | 0,726 | |
| B+C+E (đóng băng) | – | 0,787 | – | 0,787 [0,722, 0,834] |
| B+C+F | – | 0,743 | – | |
| B+C+E+F | – | 0,811 | 0,822 | không phân biệt được với B+C+E |
| Rule (tham chiếu) | 0,800 | 0,841 | 0,841 | |

*Bảng B.2. Độ nhạy của Qwen với ngữ cảnh bị làm hỏng trên Dev (macro-F1 và chênh lệch so với ngữ cảnh lý tưởng; KTC 95%). Điều kiện "sessionizer" chỉ có hai phiên bị trộn.*

| Điều kiện | Điểm | Chênh lệch | Khoảng |
|---|---|---|---|
| k=2 lý tưởng | 0,787 | – | – |
| k=2 sessionizer | 0,729 | −0,058 | [−0,106, −0,000] |
| k=2 trộn 50% | 0,774 | −0,013 | chứa 0 |
| k=2 trộn 100% | 0,716 | −0,071 | [−0,132, −0,004] |
| k=4 lý tưởng | 0,782 | – | – |
| k=4 sessionizer | 0,693 | −0,089 | [−0,144, −0,012] |
| k=4 trộn 50% | 0,756 | −0,026 | chứa 0 |
| k=4 trộn 100% | 0,735 | −0,048 | [−0,105, +0,016] |

# Phụ lục C. Ma trận nhầm lẫn ở k = 2

Hàng là nhãn thật, cột là nhãn dự đoán; "other" là mọi nhãn ngoài sáu lớp (không xảy ra). Ma trận cho k = 0 và k = 4 nằm trong `report_extra_tables.md`. Disc. = Discovery, Cred. = Credential Access, Coll. = Collection, Exfil. = Exfiltration.

*Ma trận nhầm lẫn, Holdout, Rule, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 18 | 0 | 0 | 0 | 0 | 3 | 0 |
| Credential Access | 0 | 11 | 0 | 0 | 0 | 2 | 0 |
| Collection | 4 | 0 | 9 | 0 | 0 | 2 | 0 |
| Exfiltration | 0 | 0 | 0 | 12 | 0 | 0 | 0 |
| Impact | 0 | 0 | 1 | 0 | 25 | 1 | 0 |
| Noise | 3 | 0 | 3 | 0 | 1 | 14 | 0 |

*Ma trận nhầm lẫn, Holdout, Qwen, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 18 | 1 | 0 | 0 | 2 | 0 | 0 |
| Credential Access | 0 | 11 | 0 | 0 | 2 | 0 | 0 |
| Collection | 2 | 1 | 7 | 0 | 5 | 0 | 0 |
| Exfiltration | 0 | 0 | 0 | 12 | 0 | 0 | 0 |
| Impact | 0 | 0 | 0 | 5 | 22 | 0 | 0 |
| Noise | 4 | 0 | 1 | 0 | 2 | 14 | 0 |

*Ma trận nhầm lẫn, Holdout, H4†, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 19 | 1 | 0 | 0 | 1 | 0 | 0 |
| Credential Access | 0 | 11 | 0 | 0 | 2 | 0 | 0 |
| Collection | 4 | 1 | 10 | 0 | 0 | 0 | 0 |
| Exfiltration | 0 | 0 | 0 | 12 | 0 | 0 | 0 |
| Impact | 0 | 0 | 1 | 0 | 26 | 0 | 0 |
| Noise | 3 | 0 | 3 | 0 | 2 | 13 | 0 |

*Ma trận nhầm lẫn, OOD-1, Rule, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 10 | 0 | 0 | 0 | 5 | 5 | 0 |
| Credential Access | 0 | 0 | 1 | 0 | 0 | 0 | 0 |
| Collection | 0 | 0 | 9 | 0 | 0 | 6 | 0 |
| Exfiltration | 0 | 0 | 3 | 5 | 1 | 2 | 0 |
| Impact | 1 | 0 | 2 | 0 | 2 | 5 | 0 |
| Noise | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

*Ma trận nhầm lẫn, OOD-1, Qwen, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 12 | 1 | 1 | 0 | 6 | 0 | 0 |
| Credential Access | 0 | 0 | 0 | 0 | 0 | 1 | 0 |
| Collection | 4 | 0 | 7 | 0 | 1 | 3 | 0 |
| Exfiltration | 2 | 0 | 3 | 4 | 2 | 0 | 0 |
| Impact | 0 | 0 | 0 | 0 | 9 | 1 | 0 |
| Noise | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

*Ma trận nhầm lẫn, OOD-1, H4†, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 13 | 0 | 1 | 0 | 6 | 0 | 0 |
| Credential Access | 0 | 0 | 1 | 0 | 0 | 0 | 0 |
| Collection | 3 | 0 | 10 | 0 | 1 | 1 | 0 |
| Exfiltration | 1 | 0 | 4 | 5 | 1 | 0 | 0 |
| Impact | 1 | 0 | 2 | 0 | 6 | 1 | 0 |
| Noise | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

*Ma trận nhầm lẫn, OOD-2, Rule, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 12 | 0 | 0 | 0 | 0 | 9 | 0 |
| Credential Access | 0 | 6 | 0 | 0 | 0 | 0 | 0 |
| Collection | 0 | 0 | 10 | 0 | 0 | 17 | 0 |
| Exfiltration | 0 | 0 | 0 | 9 | 0 | 3 | 0 |
| Impact | 1 | 0 | 3 | 0 | 6 | 42 | 0 |
| Noise | 14 | 0 | 1 | 0 | 0 | 12 | 0 |

*Ma trận nhầm lẫn, OOD-2, Qwen, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 17 | 0 | 0 | 0 | 0 | 4 | 0 |
| Credential Access | 0 | 6 | 0 | 0 | 0 | 0 | 0 |
| Collection | 5 | 2 | 9 | 2 | 6 | 3 | 0 |
| Exfiltration | 1 | 0 | 2 | 6 | 1 | 2 | 0 |
| Impact | 1 | 0 | 1 | 0 | 19 | 31 | 0 |
| Noise | 20 | 0 | 0 | 0 | 0 | 7 | 0 |

*Ma trận nhầm lẫn, OOD-2, H4†, k = 2.*

| Thật \ dự đoán | Disc. | Cred. | Coll. | Exfil. | Impact | Noise | other |
|---|---|---|---|---|---|---|---|
| Discovery | 18 | 0 | 0 | 0 | 0 | 3 | 0 |
| Credential Access | 0 | 6 | 0 | 0 | 0 | 0 | 0 |
| Collection | 4 | 0 | 14 | 0 | 6 | 3 | 0 |
| Exfiltration | 1 | 0 | 0 | 9 | 0 | 2 | 0 |
| Impact | 1 | 0 | 4 | 0 | 16 | 31 | 0 |
| Noise | 19 | 0 | 1 | 0 | 0 | 7 | 0 |

# Phụ lục D. Các kiểm tra tuyên bố bổ sung

Các hàng khác của bảng kiểm tuyên bố, bổ sung cho Bảng 4.15.

| Tuyên bố | Bằng chứng | Cách đọc được phép |
|---|---|---|
| Qwen bị hại khi ngữ cảnh sai | Dev: −0,058 đến −0,089 ở các điều kiện nặng; phép trộn 50% chứa 0 | Thăm dò, chỉ trên Dev; Rule đổi tối đa 0,016; không xếp hạng các điều kiện. |
| Rule có bằng chứng tốt hơn | Jaccard Holdout, Qwen − Rule: −0,067 [−0,129, −0,015] ở k=2; −0,010 [−0,074, +0,044] ở k=4 | Chỉ đúng ở k=2, so với ground truth tự sinh; danh nghĩa. |
| Qwen có bằng chứng tốt hơn trên kỹ thuật chưa thấy | OOD-2: +0,055 và +0,092 (loại trừ 0); OOD-1 k=4: +0,095 | Jaccard cao hơn trên OOD-2; không phải xếp hạng chất lượng giải thích; OOD-1 không độc lập. |
| OOD-1 là kiểm chứng độc lập | Do Claude viết; SQL hiện ra sau khi đóng băng | Không được hỗ trợ; kết quả thăm dò. |
| Chiến lược lai H3 có ích | −0,004 đến −0,006 so với Rule trên Holdout, khoảng chứa 0 | Không có bằng chứng cải thiện; điều này không chứng minh H3 vô dụng. |

# Phụ lục E. Đầu ra gốc của các script thống kê

Các đầu ra chưa chỉnh sửa được giữ trong repo của dự án và được sao chép, không tính lại, cho báo cáo này: `rq_checks_output.txt` (lần chạy 20261028_025528: ảnh hưởng của k, mức giảm, hiệu của các mức giảm, độ nhạy tập lớp cố định, tỉ lệ lần lấy mẫu thiếu lớp, số Noise), `rq_extras_output.txt` (lần chạy 20261028_061053: H1, chênh lệch tuyệt đối, Jaccard bằng chứng, 10.000 lần lấy mẫu, seed 42) và `rq3_multiplicity.json` (max-|t|, giá trị tới hạn 2,5922, sáu hàng). Kết quả hiệu chỉnh đa so sánh có ở Bảng 4.11. Tỉ lệ lần lấy mẫu thiếu lớp trong `rq_checks_output.txt`: Credential Access vắng trong 35,1% lần lấy mẫu OOD-1 và tối đa 5,3% với mọi lớp và tập khác.

# Phụ lục F. Lệnh tái lập và seed

Seed: 42 cho phép chia, mô hình ngôn ngữ, bootstrap và thủ tục đa so sánh; 2026 cho bộ sinh Holdout. Mô hình ngôn ngữ: Qwen3 8B (tag `qwen3:8b`), temperature 0, tắt thinking, máy chủ Ollama 0.40.11 với client Python `ollama==0.6.3` (cũng được ghim trong `requirements.txt`: psycopg 3.2.13, pandas 2.3.3, scikit-learn 1.9.1, pytest 8.4.2). Các lệnh (tác giả chạy ở nơi dữ liệu được niêm phong):

- `python -m src.evaluation.verify_rq1` (tính lại độc lập RQ1; không có sai khác)
- `python -m src.evaluation.rq_checks` (ảnh hưởng của k, ΔF1, D, độ nhạy tập lớp cố định, số Noise)
- `python -m src.evaluation.rq_extras` (H1, chênh lệch tuyệt đối, chênh lệch bằng chứng)
- `python -m src.evaluation.rq_multiplicity` (họ sáu phép so sánh)
- `python -m src.evaluation.final_tables` (dựng bảng kết quả 366 hàng)
- `python -m src.evaluation.report_tables` (ma trận nhầm lẫn, số Noise, điểm độ lạ cạnh accuracy)
- `pytest` (mọi test; một test cũ của tệp kịch bản OOD-2 bị lỗi và không sửa vì tệp được niêm phong)

# Phụ lục G. Prompt nguyên văn

Prompt B+C+E được `src/models/llm_local.py` dựng và gửi dưới dạng một tin nhắn người dùng. Văn bản dưới đây là đầu ra chính xác của bộ dựng prompt cho cửa sổ k = 2, trong đó cửa sổ được thay bằng placeholder; các ví dụ đến từ bộ hạt giống chung và từ tập Dev. Ở k = 0, sáu ví dụ hạt giống có ngữ cảnh và cả bốn ví dụ cân bằng bị bỏ, còn lại ba ví dụ; ở k ≥ 2 hiện cả mười. Sinh dùng seed 42, temperature 0, tắt thinking và giới hạn 128 token mới.

```text
You are a PostgreSQL security analyst. Given a target SQL query and optional context (previous queries in the same session), classify the target query into one of these attack stages:

- Discovery: reconnaissance: listing tables, columns, roles, settings, version or server info
- Credential Access: ONLY PostgreSQL's own authentication data (pg_shadow, pg_authid, role password hashes) and login attempts. Credentials stored in ordinary application tables are NOT Credential Access
- Collection: reading data from ordinary tables that hold sensitive information (SELECT * dumps, password, secret, token, key or personal-data columns), also through PREPARE/EXECUTE, and reading server files (pg_read_file)
- Exfiltration: exporting data out (COPY TO, lo_export)
- Impact: destructive actions, privilege changes (DROP, ALTER ROLE, CREATE ROLE)
- Noise: normal/benign database activity

A DO block whose EXECUTE content is hidden ('?') cannot be read: judge it by the context, as the same kind of work as the latest relevant query before it, and call it Impact only if destructive statements are visible.

Respond ONLY with a JSON object: {"label": "<stage>", "evidence_ids": [<ids>]}
where evidence_ids lists the query IDs (q0, q1, ...) that support your classification. Always include the target query's ID in evidence_ids. Output ONLY valid JSON, no other text.

Classify ONLY the TARGET query. The context is there to disambiguate, not to carry its stage over: a harmless query stays Noise even when earlier queries look malicious.

### Examples

Queries:
TARGET (classify only this query):
q0: SELECT version()
Answer: {"label": "Discovery", "evidence_ids": [0]}

Queries:
CONTEXT (earlier queries, reference only, do NOT classify these):
q0: SELECT table_name FROM information_schema.tables WHERE table_schema = '?'
TARGET (classify only this query):
q1: SELECT * FROM t1
Answer: {"label": "Collection", "evidence_ids": [0, 1]}

Queries:
CONTEXT (earlier queries, reference only, do NOT classify these):
q0: SELECT version()
q1: SELECT rolname, rolsuper FROM pg_shadow
TARGET (classify only this query):
q2: ALTER ROLE postgres PASSWORD '?'
Answer: {"label": "Impact", "evidence_ids": [1, 2]}

Queries:
CONTEXT (earlier queries, reference only, do NOT classify these):
q0: SELECT table_name FROM information_schema.tables WHERE table_schema = '?'
q1: SELECT * FROM t1
TARGET (classify only this query):
q2: COPY (SELECT * FROM t1) TO '?'
Answer: {"label": "Exfiltration", "evidence_ids": [0, 1, 2]}

Queries:
TARGET (classify only this query):
q0: SELECT now()
Answer: {"label": "Noise", "evidence_ids": [0]}

Queries:
TARGET (classify only this query):
q0: SELECT rolname, rolsuper, rolcanlogin FROM pg_shadow WHERE rolcanlogin = true
Answer: {"label": "Credential Access", "evidence_ids": [0]}

Queries:
CONTEXT (earlier queries, reference only, do NOT classify these):
q0: SELECT id, username, email FROM t1 WHERE username = '?'
q1: SELECT count(*) FROM t2 WHERE username = '?' AND read_at IS NULL
TARGET (classify only this query):
q2: SELECT t.title, t.status, t.priority FROM t3 t WHERE t.project_id = ? AND t.status = '?'
Answer: {"label": "Noise", "evidence_ids": [2]}

Queries:
CONTEXT (earlier queries, reference only, do NOT classify these):
q0: SELECT table_name FROM information_schema.tables WHERE table_schema = '?'
TARGET (classify only this query):
q1: SELECT id, username FROM t1 ORDER BY created_at DESC LIMIT ?
Answer: {"label": "Noise", "evidence_ids": [1]}

Queries:
CONTEXT (earlier queries, reference only, do NOT classify these):
q0: SELECT rolname, rolsuper, rolcanlogin FROM pg_shadow WHERE rolcanlogin = true
TARGET (classify only this query):
q1: SELECT now()
Answer: {"label": "Noise", "evidence_ids": [1]}

Queries:
CONTEXT (earlier queries, reference only, do NOT classify these):
q0: SELECT * FROM t1
TARGET (classify only this query):
q1: DROP TABLE t2
Answer: {"label": "Impact", "evidence_ids": [1]}

### Now classify this:

Queries:
CONTEXT (earlier queries, reference only, do NOT classify these):
q0: <context query 0>
q1: <context query 1>
TARGET (classify only this query):
q2: <target SQL>

Answer:
```

# Phụ lục H. Các phát hiện kiểm toán

Kết quả được kiểm toán trước khi viết báo cáo này. 39 phát hiện được tóm tắt; bảng gốc nằm trong `rq_results_review.md`.

| # | Loại | Phát hiện và cách xử lý |
|---|---|---|
| 1 | Báo cáo | Dấu của ΔF1 không nhất quán; thống nhất là đã biết trừ chưa thấy, dương = giảm. |
| 2 | Thống kê | ΔF1 so sánh các tập lớp khác nhau; tính lại trên các lớp chung. |
| 3 | Thống kê | k=0 khác k≥2 ở số ví dụ (3 so với 10); nêu trong văn bản. |
| 4 | Báo cáo | "Ngữ cảnh giúp ở k=2" thiếu khoảng ghép cặp; mọi khoảng đều chứa 0. |
| 5 | Báo cáo | Mức mất do ngữ cảnh hỏng đo trên Dev, không phải Holdout; báo cáo theo từng điều kiện kèm khoảng. |
| 6 | Lỗi xác nhận | Tuyên bố lợi thế nằm ở kỹ thuật có dùng trong Dev là sai: các họ dùng lại lấy từ kỹ thuật OOD-1. |
| 7 | Lỗi xác nhận | H4 không được đăng ký trước từ đầu; ý tưởng post hoc, định nghĩa cố định trước OOD-2. |
| 8 | Lỗi xác nhận | "Một người rà soát" che việc người đó là AI; nêu tường minh. |
| 9 | Báo cáo | C2 từng bị mô tả là nhẹ nhất; thực tế nó nhiều bản ghi nhất. |
| 10 | Báo cáo | "Năm trên sáu thuộc tính" mâu thuẫn với bảng; chỉ báo cáo tỉ lệ từng trường. |
| 11 | Báo cáo | "LLM bền hơn, có ý nghĩa" sửa theo kết quả họ sáu phép so sánh. |
| 12 | Báo cáo | Hai số Noise của Qwen trên OOD-1 ứng với k khác nhau (6, 5, 4). |
| 13 | Báo cáo | Thêm khoảng còn thiếu cho Qwen và H4 trên OOD-2. |
| 14 | Thống kê | Thêm khoảng ghép cặp cho bằng chứng. |
| 15 | Thống kê | H1 so sánh hai tập cửa sổ khác nhau; định nghĩa lại thành bốn con số. |
| 16 | Thống kê | Hiệu Holdout so với OOD không chỉ do độ lạ. |
| 17 | Thống kê | Bootstrap theo kịch bản không bao phủ độ bất định giữa các họ. |
| 18 | Báo cáo | Giảm giọng nhân quả trong phần ablation. |
| 19 | Báo cáo | Nhãn dashboard đổi thành "khoảng loại trừ 0 / bằng chứng chưa đủ". |
| 20 | Thống kê | Điểm giảm từ Dev sang Holdout không phải bằng chứng vắng rò rỉ. |
| 21 | Thống kê | Tập lớp thay đổi giữa các lần lấy mẫu; thêm bộ ước lượng tập lớp cố định. |
| 22 | Lỗi xác nhận | Script in khối không đầy đủ; sửa và chạy lại. |
| 23 | Báo cáo | Số RQ1 gõ tay trong dashboard thay bằng giá trị tính ra. |
| 24 | Báo cáo | Dashboard hiện ΔF1 bộ ước lượng mặc định; bảng chính nay dùng tập lớp cố định. |
| 25 | Báo cáo | Số gõ tay trong văn bản dashboard thay bằng số tính ra. |
| 26 | Lỗi xác nhận | Vạch trục biểu đồ forest cắt khoảng; sửa. |
| 27 | Lỗi xác nhận | Tiêu đề trang bị thanh điều hướng che; sửa. |
| 28 | Báo cáo | Biểu đồ khó đọc trên màn hình hẹp; sửa. |
| 29 | Báo cáo | Kết quả Dev từng nổi bật hơn Holdout; sắp xếp lại. |
| 30 | Báo cáo | Thiếu H4 cho Holdout; thêm bằng `rq_extras`. |
| 31 | Thống kê | Chênh lệch tuyệt đối dùng bộ ước lượng mặc định; thêm bản tập lớp cố định. |
| 32 | Lỗi xác nhận | Test cũ của tệp kịch bản OOD-2 lỗi; không sửa (tệp niêm phong). |
| 33 | Lỗi xác nhận | Nhãn kết luận cho bằng chứng và H1 sai trong dashboard; sửa. |
| 34 | Báo cáo | Hiệu của H1 bị đọc thành accuracy hệ thống; định nghĩa lại. |
| 35 | Báo cáo | Dashboard dùng khoảng danh nghĩa nhưng nói đã hiệu chỉnh; sửa. |
| 36 | Báo cáo | Thiếu H4 cho OOD-1; thêm sau và đánh dấu †. |
| 37 | Lỗi xác nhận | Định nghĩa ΔF1 bị xóa nhầm trong dashboard; khôi phục. |
| 38 | Báo cáo | Khoảng chồng nhau không quyết định một so sánh; thêm hướng dẫn đọc. |
| 39 | Báo cáo | 136 hành động (RQ1) và 102 truy vấn (RQ2) là các đơn vị khác nhau, không phải sai lệch. |
