# Kế Hoạch Phát Triển MVP Cho Hệ Thống AI Tư Vấn Bất Động Sản (3 Tháng)

## 0. Giới thiệu MVP

### Mục tiêu

MVP tập trung xây dựng phiên bản cốt lõi của hệ thống AI tư vấn bất động sản, ưu tiên chức năng thiết yếu để xác thực giá trị với người dùng cá nhân (bao gồm nhà đầu tư), sàn BĐS, và quản trị viên. MVP sẽ hỗ trợ tra cứu, tư vấn cơ bản, đặt lịch, và quản lý dữ liệu đơn giản, đảm bảo khả thi trong 3 tháng với đội ngũ 4-6 người (2 FE, 2 BE, 1 QA, 1 PM).

### Phạm vi MVP

- **In-scope**: Tra cứu thông tin BĐS qua chatbot, gợi ý cơ bản, đặt lịch, hỗ trợ pháp lý đơn giản, định giá cơ bản, quản lý lịch hẹn qua dashboard (cho sàn BĐS), quản lý dữ liệu cơ bản (cho quản trị viên).
- **Out-of-scope**: Phân tích chuyên sâu (ROI chi tiết, dự báo dài hạn), AR/VR, hỗ trợ đa ngôn ngữ (chỉ tiếng Việt), báo cáo phức tạp, tích hợp đầy đủ API (chỉ cơ bản).

### Giả định

- Đội ngũ có kinh nghiệm React/Node.js, NLP cơ bản (spaCy).
- Dữ liệu ban đầu từ sàn BĐS công khai (Batdongsan.com.vn, X).
- Triển khai trên AWS, test với 1.000 người dùng.

---

## 1. Đối tượng người dùng trong MVP

- **Người dùng cá nhân (bao gồm nhà đầu tư)**: Tra cứu, gợi ý, đặt lịch qua chatbot.
- **Sàn BĐS**: Tư vấn khách hàng qua chatbot, quản lý lịch hẹn/danh mục qua dashboard cơ bản.
- **Quản trị viên**: Quản lý dữ liệu, giám sát qua dashboard đơn giản.

---

## 2. Chức năng MVP (Ưu tiên từ file full features)

### 2.1. Tra cứu và tư vấn bất động sản (Ưu tiên cao - Tuần 1-4)

- Tra cứu thông tin giá, vị trí, loại hình, tiện ích qua câu hỏi tự nhiên (text/voice). (AI)
  - **Ví dụ**: Hỏi “Căn hộ 2PN Quận 7 dưới 3 tỷ?” → Chatbot trả danh sách 5 căn hộ, giá 2.5–2.8 tỷ (AI response), hiển thị vị trí trên bản đồ (Backend Handle).
- Tích hợp dữ liệu từ sàn giao dịch (sử dụng function calling để get data -> chuyển tiếp cho backend xử lý và frontend hiển thị), X; hiển thị nguồn gốc. (AI)
  - **Ví dụ**: Hiển thị “Nguồn: Batdongsan.com.vn, cập nhật 01/10/2025.”
- Cung cấp tổng quan thị trường (giá trung bình, xu hướng 6 tháng). (AI)
  - **Ví dụ**: Hỏi “Xu hướng giá Quận 7?” → Chatbot trả “Giá trung bình 40 triệu/m², tăng 5%.” (AI khi trả về -> FE xử lý lại hiển thị các chart trực quan trên chatbot)
- Giải thích thuật ngữ đơn giản. (AI Legal)
  - **Ví dụ**: Hỏi “Sổ đỏ là gì?” → Chatbot trả “Giấy chứng nhận quyền sử dụng đất.”
- Gợi ý BĐS theo tiêu chí (vị trí, giá, diện tích) (AI).
  - **Ví dụ**: Nhập “Căn hộ 2PN, dưới 3 tỷ, gần trường học” → Chatbot gợi ý 3 căn hộ Quận 7. (AI trả về data -> gửi cho Backend -> Backend xử l)
- Hỗ trợ đặt lịch tư vấn/xem nhà qua chatbot (Backend).
  - **Ví dụ**: Nói “Đặt lịch xem nhà thứ Bảy” → Chatbot xác nhận, gửi email/Zalo (Cái này thì AI trả về function calling để xử lý).
- Gửi thông báo tự động (email/Zalo) về lịch hẹn. (Backend)
  - **Ví dụ**: Gửi “Lịch xem nhà: 10h, Thứ Bảy, Quận 7.” Như trên
- **Đối tượng**: Người dùng cá nhân (bao gồm nhà đầu tư), Sàn BĐS.
- **Giao diện**: Chatbot trên web/mobile app.

### 2.2. Phân tích thị trường và đánh giá rủi ro (Ưu tiên trung bình - Tuần 5-8)

- So sánh BĐS (giá, vị trí, tiện ích) qua bảng/biểu đồ. (backend lấy xử lý rồi trả về FE, FE hiển thị modal so sánh trực quan giữa 2 BDS)
  - **Ví dụ**: Hỏi “So sánh căn hộ Quận 2 và Quận 7?” → Chatbot hiển thị bảng giá/m², tiện ích.
- Phân tích xu hướng giá, cung cầu (6 tháng).
  - **Ví dụ**: Hỏi “Xu hướng giá Quận 7?” → Chatbot trả biểu đồ tăng 5%.
- Dự đoán ROI và thanh khoản cơ bản. (AI)
  - **Ví dụ**: Hỏi “ROI căn hộ Quận 7?” → Chatbot trả “6%/năm, thanh khoản cao.”
- Cảnh báo rủi ro pháp lý (sổ đỏ, tranh chấp) và thị trường (quy hoạch).
  - **Ví dụ**: Hỏi về căn hộ Quận 2 → Chatbot cảnh báo “Quy hoạch metro, tiềm năng cao.”
- **Đối tượng**: Người dùng cá nhân (bao gồm nhà đầu tư), Sàn BĐS.
- **Giao diện**: Chatbot (kết quả cơ bản).

### 2.3. Hỗ trợ giao dịch và định giá (Ưu tiên cao - Tuần 3-6)

- Hướng dẫn pháp lý (hợp đồng, công chứng, thuế) theo quy định Việt Nam 2025. (AI)
  - **Ví dụ**: Hỏi “Làm hợp đồng mua nhà cần gì?” → Chatbot liệt kê các bước: kiểm tra sổ đỏ, công chứng.
- Cung cấp mẫu hợp đồng PDF cơ bản. (Backend)
  - **Ví dụ**: Hỏi “Mẫu hợp đồng mua nhà?” → Chatbot cung cấp PDF hợp đồng mua bán.
- Đề xuất giá bán/cho thuê dựa trên thị trường. (AI)
  - **Ví dụ**: Hỏi “Giá bán căn hộ 70m² Quận 7?” → Chatbot trả “2.8 tỷ, dựa trên 40 triệu/m².”
- Kiểm tra giá niêm yết hợp lý.
  - **Ví dụ**: Nhập giá 3 tỷ → Chatbot cảnh báo “Giá cao hơn thị trường 10%.”
- **Đối tượng**: Người dùng cá nhân (bao gồm nhà đầu tư), Sàn BĐS.
- **Giao diện**: Chatbot (hướng dẫn, tải mẫu, tính toán).

### 2.4. Quản lý danh mục và lịch hẹn (Ưu tiên trung bình - Tuần 7-10)

- Cập nhật danh sách BĐS (giá, tình trạng) (AI+BE).
  - **Ví dụ**: Sàn nhập 10 căn hộ, cập nhật giá từ 2–5 tỷ trên dashboard.
- Theo dõi giá trị và trạng thái giao dịch (đặt cọc, ký hợp đồng). (BE)
  - **Ví dụ**: Dashboard hiển thị “Danh mục: 50 tỷ, căn hộ A đã đặt cọc.”
- Quản lý lịch hẹn (tư vấn/xem nhà) qua calendar. (BE)
  - **Ví dụ**: Khách đặt lịch qua chatbot → Dashboard hiển thị “10h, Thứ Bảy, Quận 7.”
- Đồng bộ với Google Calendar/Zalo; phân công nhân viên. (BE)
  - **Ví dụ**: Admin phân công nhân viên A cho lịch hẹn, đồng bộ Google Calendar.
- Gửi thông báo lịch hẹn cho khách hàng/nhân viên. (BE)
  - **Ví dụ**: Gửi Zalo “Lịch xem nhà: 10h, Thứ Bảy” cho khách và nhân viên.
- **Đối tượng**: Sàn BĐS.
- **Giao diện**: Dashboard sàn BĐS, chatbot (đặt lịch).

### 2.5. Phân tích khách hàng và tích hợp kinh doanh (Ưu tiên thấp - Tuần 9-12)

- Phân tích hành vi khách hàng (tìm kiếm, ưu tiên).
  - **Ví dụ**: Dashboard hiển thị “60% khách tìm căn hộ dưới 3 tỷ.”
- Phân khúc khách hàng (mua để ở, đầu tư).
  - **Ví dụ**: Báo cáo “70% khách mua để ở, 30% đầu tư.”
- Cung cấp API tích hợp vào website/CRM sàn BĐS.
  - **Ví dụ**: API trả lời “Giá căn hộ Quận 7 từ 2.5 tỷ” trên website sàn.
- Tạo báo cáo tự động (giao dịch, xu hướng khách hàng).
  - **Ví dụ**: Gửi email “Tháng 10/2025: 50 giao dịch, 20% từ Quận 7.”
- **Đối tượng**: Sàn BĐS.
- **Giao diện**: Dashboard sàn BĐS, API.

### 2.6. Quản trị hệ thống và bảo mật (Ưu tiên cao - Tuần 1-4)

- Nhập, chỉnh sửa dữ liệu BĐS và chính sách (thuế, quy hoạch). (BE->AI)
  - **Ví dụ**: Quản trị viên nhập căn hộ Quận 7, giá 2.8 tỷ trên dashboard.
- Kiểm tra tính hợp lệ/minh bạch dữ liệu.
  - **Ví dụ**: Dashboard cảnh báo “Giá cao bất thường” cho căn hộ 3 tỷ.
- Giám sát chất lượng tư vấn, báo cáo lỗi.
  - **Ví dụ**: Dashboard hiển thị “Tỷ lệ phản hồi: 98%, chính xác: 95%.”
- Dashboard quản trị (theo dõi hiệu suất, phản hồi).
  - **Ví dụ**: Hiển thị “80% người dùng đánh giá 4 sao.”
- Phân quyền truy cập; mã hóa dữ liệu (AES-256, TLS/SSL).
  - **Ví dụ**: Chỉ admin cấp cao chỉnh sửa dữ liệu pháp lý.
- Tuân thủ Luật Bảo vệ Dữ liệu Cá nhân 2025.
  - **Ví dụ**: Mã hóa thông tin người dùng, lưu nhật ký hành động.
- Lưu nhật ký hoạt động quản trị viên.
  - **Ví dụ**: Lưu “Admin A cập nhật giá, 01/10/2025.”
- Thu thập phản hồi (nút 1-5 sao trong chatbot). (Sentiman anylytics)
  - **Ví dụ**: Người dùng nhấn 4 sao sau tư vấn, tổng hợp trên dashboard.
- Tùy chỉnh giao diện theo vai trò (người dùng, sàn BĐS, quản trị viên).
  - **Ví dụ**: Chatbot chỉ hiển thị tra cứu cho người dùng cá nhân, dashboard đầy đủ cho quản trị viên.
- **Đối tượng**: Quản trị viên.
- **Giao diện**: Dashboard quản trị.

---

## 3. Yêu cầu nghiệp vụ

### 3.1. Người dùng cá nhân (bao gồm nhà đầu tư)

- Tra cứu nhanh thông tin BĐS (giá, vị trí, pháp lý) qua chatbot.
- Nhận gợi ý theo tiêu chí (vị trí, giá, tiện ích).
- Xem tổng quan thị trường, so sánh cơ bản, cảnh báo rủi ro.
- Hướng dẫn pháp lý, tải mẫu hợp đồng, tính toán chi phí.
- Đặt lịch tư vấn/xem nhà qua text/voice; nhận thông báo.
- Bảo mật thông tin, tùy chọn ẩn danh.

### 3.2. Sàn BĐS

- Tư vấn khách hàng qua chatbot, quản lý danh mục/lịch hẹn qua dashboard.
- Phân tích khách hàng, tích hợp API vào hệ thống kinh doanh.
- Báo cáo giao dịch, xu hướng khách hàng.
- Phân khúc khách hàng để tối ưu marketing.

### 3.3. Quản trị viên

- Nhập, cập nhật, kiểm tra dữ liệu BĐS/chính sách.
- Giám sát tư vấn, xử lý lỗi, phân quyền truy cập.
- Phân tích phản hồi để cải tiến AI.
- Đảm bảo tuân thủ pháp luật, lưu nhật ký hoạt động.

---

## 4. Yêu cầu kỹ thuật

### 4.1. Frontend (FE)

- **Công nghệ**: React, JSX, Tailwind CSS, WebSocket (chatbot)
- **Chức năng**:
  - Chatbot text: Tra cứu, gợi ý, đặt lịch, thông báo cho người dùng cá nhân/sàn BĐS.
  - Dashboard: Quản lý danh mục, lịch hẹn, báo cáo cho sàn BĐS; quản trị cho quản trị viên.
  - Hiển thị danh sách, bảng/biểu đồ, bản đồ (Google Maps/OpenStreetMap).
  - Form đặt lịch, tải báo cáo PDF.
  - Đa nền tảng (web: Chrome/Firefox/Safari; mobile: PWA/React Native).
  - Đăng nhập OAuth2 (Google, Zalo) cho lưu lịch sử.
- **UX/UI**: Phản hồi <2s, giao diện tối giản (màu xanh/trắng/xám, font Roboto/Noto Sans), tuân thủ WCAG 2.1.
- **Tùy chỉnh**: Hiển thị chức năng theo vai trò (người dùng cá nhân, sàn BĐS, quản trị viên).

### 4.2. Backend (BE)

- **Công nghệ**: NestJS và Python/FastAPI, WebSocket, NLP (spaCy/Hugging Face).
- **Chức năng**:
  - Xử lý câu hỏi tự nhiên, tích hợp dữ liệu từ X/sàn giao dịch.
  - REST API cho tra cứu, lịch hẹn, báo cáo; xác thực JWT/OAuth2.
  - Đồng bộ Google Calendar/Zalo; thông báo email/Zalo.
- **Hiệu suất**: API trả lời <200ms (cơ bản), <500ms (phức tạp); hỗ trợ 10.000 người dùng đồng thời; uptime 99.9% (CDN Cloudflare).

(cần ghi chi tiết phần nghiên cứu AI nữa)

### 4.3. Cơ sở dữ liệu (DB)

- **Công nghệ**: MongoDB (lịch sử chat, nhật ký, dữ liệu cấu trúc: BĐS, lịch hẹn, người dùng), Redis (cache), QDrant.
