# EduExam

EduExam là giải pháp chuyển đổi số hỗ trợ giáo viên Tiếng Anh tự động trích xuất nội dung từ tệp PDF để tạo đề thi chuẩn hóa bằng AI, đồng thời cung cấp môi trường kiểm tra và đánh giá trực tuyến toàn diện.

## Giới thiệu tổng quan

EduExam rút ngắn quy trình soạn đề từ hàng giờ xuống chỉ còn vài phút bằng cách tích hợp mô hình ngôn ngữ lớn để phân tích ngữ liệu văn bản giảng dạy. Ứng dụng đảm bảo độ chính xác theo chuẩn khung tham chiếu Châu Âu (CEFR), kiểm soát chặt chẽ tính hợp lệ của tài liệu đầu vào và tối ưu hóa hạ tầng vận hành trên nền tảng đám mây.

## Công nghệ sử dụng

| Lớp kiến trúc | Công nghệ triển khai | Vai trò chính |
| --- | --- | --- |
| **Client Application** | Flutter (Dart) | Đa nền tảng, xử lý giao diện người dùng và tiền kiểm tra tệp tin |
| **Backend & Cloud** | Firebase Cloud Functions (v2) | Xử lý logic nghiệp vụ serverless, điều phối pipeline sinh đề thi |
| **Database & Cache** | Firebase Cloud Firestore | Lưu trữ cấu trúc đề thi, hồ sơ giáo viên, quản trị transaction |
| **Security & Auth** | Firebase Auth & App Check | Xác thực danh tính và bảo vệ API backend trước truy cập trái phép |
| **Storage & Messaging** | Cloud Storage & Firebase FCM | Lưu trữ PDF gốc, gửi thông báo đẩy trạng thái xử lý |
| **AI Engine** | OpenAI GPT-4o-mini | Phân tích ngữ cảnh, ánh xạ CEFR và xuất đề thi định dạng JSON |

## Cơ chế sinh đề thi thông minh (AI Exam Generation Pipeline)

### 1. Dạng câu hỏi và tiêu chuẩn CEFR

Hệ thống cho phép cấu hình linh hoạt theo 6 bậc trình độ từ A1 đến C2 với 4 dạng bài tập cốt lõi:

- **Multiple Choice:** Câu hỏi trắc nghiệm nhiều lựa chọn tập trung vào từ vựng, ngữ pháp và suy luận ngữ cảnh.
- **Fill in the blank:** Điền vào chỗ trống nhằm kiểm tra cách dùng từ, cụm từ cố định (collocations) và chia thì.
- **True/False:** Đánh giá mức độ nắm bắt thông tin thực tế theo văn bản.
- **Reading Comprehension:** Đọc hiểu văn bản mở rộng, phân tích đại ý và chi tiết.

### 2. Quy trình kiểm tra hợp lệ 2 lớp (Validation Pipeline)

Để tiết kiệm chi phí gọi API và tránh lỗi ngoại lệ, tài liệu đầu vào phải vượt qua hai chốt kiểm định:

- **Lớp 1 (Client-side - Flutter):**
  - Kiểm tra định dạng tệp (.pdf) và giới hạn dung lượng tải lên.
  - Phát hiện và chặn tài liệu dạng ảnh quét (scanned file/non-selectable text).
  - Đo lường sơ bộ tỷ lệ ngôn ngữ để chắc chắn nguồn tài liệu là Tiếng Anh.
- **Lớp 2 (Server-side - Cloud Functions v2):**
  - Phân tích tổng thể độ dài chuỗi ký tự hợp lệ.
  - Tái thẩm định ngôn ngữ chuyên sâu trước khi tiến hành đóng gói prompt.

### 3. Quản trị hạn mức và khả năng chịu lỗi (Quota & Fault Tolerance)

- **Quản lý hạn mức:** Giới hạn mỗi tài khoản giáo viên tối đa 3 lượt sinh đề/ngày. Trạng thái cập nhật thông qua **Firestore Transaction** nhằm loại bỏ xung đột dữ liệu (race conditions).
- **Cơ chế Retry (Exponential Backoff):** Trường hợp kết nối OpenAI API gặp sự cố, hệ thống thực hiện thử lại tối đa 3 lần với khoảng hoãn tăng dần theo chu kỳ: `0s -> 2s -> 4s`.
- **Cơ chế Rollback:** Nếu cả 3 lần thử đều thất bại, hệ thống tự động hoàn lại hạn mức sinh đề cho giáo viên và ghi nhận log hệ thống.

### 4. Chiến lược thiết kế Prompt (Prompt Engineering)

- **System Prompt:**
  - Định hình vai trò chuyên gia khảo thí Tiếng Anh quốc tế.
  - Ràng buộc cấu trúc phản hồi bắt buộc tuân theo Schema JSON nghiêm ngặt.
  - Áp đặt quy tắc nghiêm cấm sao chép nguyên văn (anti-verbatim copying) nhằm tăng tính sáng tạo và khả năng suy luận cho câu hỏi.
- **User Prompt:**
  - Tích hợp toàn bộ cấu hình kỳ thi kèm ngữ liệu văn bản.
  - Thuật toán cắt thông minh (smart truncation) được kích hoạt nếu nội dung vượt mức 40.000 ký tự để giữ nguyên tính toàn vẹn của câu cú.
  - Tham số mô hình duy trì ở mức `Temperature = 0.7` nhằm cân bằng giữa độ chính xác học thuật và sự linh hoạt của văn phong.
  - Nhúng trực tiếp các đặc tả ngữ pháp chi tiết của từng cấp độ CEFR vào context.

### 5. Xử lý ngữ cảnh chuyên biệt và Danh sách cấm (Ban List)

- **Đối với Reading Comprehension:** Mô hình AI chủ động trích xuất ngữ liệu cô đọng vào trường `passage` của đối tượng JSON và xây dựng các câu hỏi tương ứng bám sát ngữ cảnh đoạn trích.
- **Đối với các dạng bài khác:** Khi cấu hình sinh đề không chứa Reading Comprehension, cơ chế **BAN LIST** lập tức được chèn vào prompt, cấm AI sử dụng các từ phụ thuộc văn bản như: `"passage"`, `"text"`, `"author"`, `"paragraph"`, `"line"`. Điều này đảm bảo câu hỏi mang tính độc lập hoàn toàn.
