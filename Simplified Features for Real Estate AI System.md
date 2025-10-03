# Hệ thống AI Tư Vấn Bất Động Sản - Tài Liệu Tinh Gọn

## 0. Giới thiệu, phạm vi & mục tiêu dự án

### Mục tiêu
Hệ thống AI tư vấn bất động sản hỗ trợ người dùng (cá nhân, sàn BĐS, quản trị viên) tra cứu, tư vấn, phân tích, và quản lý giao dịch bất động sản nhanh chóng, minh bạch, tuân thủ pháp luật, thay thế một phần môi giới truyền thống.

### Phạm vi
- **In-scope**: Tra cứu thông tin, tư vấn cá nhân hóa, phân tích thị trường cơ bản, định giá, hỗ trợ pháp lý, quản lý lịch hẹn/danh mục, API cho sàn BĐS, quản trị hệ thống.
- **Out-of-scope**: Ký hợp đồng chính thức, chịu trách nhiệm pháp lý giao dịch, tích hợp AR/VR, dự báo thị trường dài hạn (1-5 năm) trong MVP.

---

## 1. Đối tượng người dùng

### 1.1. Người dùng cá nhân
- **Người mua/thuê**: Tìm nhà ở, căn hộ, đất đai; cần thông tin giá, vị trí, pháp lý, tư vấn nhanh.
- **Người bán/cho thuê**: Định giá, đăng tin, tiếp cận khách hàng, tư vấn xu hướng.
- **Nhà đầu tư**: Phân tích thị trường, dự đoán ROI, tìm cơ hội đầu tư.

### 1.2. Sàn BĐS
- Vừa là end-user (sử dụng chatbot để tư vấn khách hàng) vừa là đối tác cung cấp dữ liệu BĐS, sử dụng dashboard và API để tự động hóa tư vấn, quản lý danh mục, và phân tích khách hàng.

### 1.3. Quản trị viên
- Quản lý dữ liệu, giám sát hệ thống, đảm bảo minh bạch và tuân thủ pháp luật.

---

## 2. Danh sách chức năng hệ thống

### 2.1. Tra cứu và tư vấn bất động sản
- **Chức năng**:
  - Tra cứu thông tin giá, vị trí, loại hình, tiện ích qua câu hỏi tự nhiên (text/voice).
    - **Ví dụ**: Hỏi “Căn hộ 2PN Quận 7 dưới 3 tỷ?” → Chatbot trả danh sách 5 căn hộ, giá 2.5–2.8 tỷ, hiển thị vị trí trên bản đồ.
  - Tích hợp dữ liệu từ sàn giao dịch, X, cơ quan quản lý; hiển thị nguồn gốc.
    - **Ví dụ**: Hiển thị “Nguồn: Batdongsan.com.vn, cập nhật 01/10/2025” cho căn hộ Quận 7.
  - Cung cấp tổng quan thị trường (giá trung bình, xu hướng khu vực).
    - **Ví dụ**: Hỏi “Xu hướng giá Quận 7?” → Chatbot trả “Giá trung bình 40 triệu/m², tăng 5% trong 6 tháng.”
  - Giải thích thuật ngữ (sổ đỏ, quy hoạch) bằng ngôn ngữ đơn giản.
    - **Ví dụ**: Hỏi “Sổ đỏ là gì?” → Chatbot trả “Giấy chứng nhận quyền sử dụng đất, cấp cho cá nhân/tổ chức.”
  - Gợi ý bất động sản theo tiêu chí (vị trí, giá, diện tích).
    - **Ví dụ**: Nhập tiêu chí “Căn hộ 2PN, dưới 3 tỷ, gần trường học” → Chatbot gợi ý 3 căn hộ ở Quận 7.
  - Hỗ trợ đặt lịch tư vấn/xem nhà qua chatbot (text/voice).
    - **Ví dụ**: Nói “Đặt lịch xem nhà thứ Bảy” → Chatbot xác nhận, gửi email/Zalo.
  - Gửi thông báo tự động (email/Zalo) về lịch hẹn, cập nhật giá.
    - **Ví dụ**: Gửi email “Lịch xem nhà: 10h, Thứ Bảy, Quận 7” sau khi đặt lịch.
  - DeepSearch mode: Phân tích dữ liệu từ X, sàn giao dịch.
    - **Ví dụ**: Hỏi “Căn hộ tiềm năng Quận 2?” → Chatbot phân tích bài đăng X, trả danh sách 4 căn hộ.
  - Xử lý tiếng Việt tự nhiên (MVP); hỗ trợ tiếng Anh cơ bản.
    - **Ví dụ**: Hỏi “Apartment in District 7?” → Chatbot trả danh sách căn hộ bằng tiếng Anh.
- **Đối tượng**: Người dùng cá nhân (bao gồm nhà đầu tư), Sàn BĐS (tư vấn khách hàng).
- **Giao diện**: Chatbot trên web/mobile app.

### 2.2. Phân tích thị trường và đánh giá rủi ro
- **Chức năng**:
  - So sánh bất động sản (giá, vị trí, tiện ích) qua bảng/biểu đồ.
    - **Ví dụ**: Hỏi “So sánh căn hộ Quận 2 và Quận 7?” → Chatbot hiển thị bảng giá/m², tiện ích.
  - Phân tích xu hướng giá, cung cầu (6-12 tháng).
    - **Ví dụ**: Hỏi “Xu hướng giá Quận 7?” → Chatbot trả biểu đồ tăng 5% trong 6 tháng.
  - Dự đoán ROI và thanh khoản cơ bản dựa trên dữ liệu lịch sử.
    - **Ví dụ**: Hỏi “ROI căn hộ Quận 7?” → Chatbot trả “6%/năm, thanh khoản cao.”
  - Cảnh báo rủi ro pháp lý (sổ đỏ, tranh chấp) và thị trường (quy hoạch, biến động kinh tế).
    - **Ví dụ**: Hỏi về căn hộ Quận 2 → Chatbot cảnh báo “Khu vực có quy hoạch metro, tiềm năng cao.”
- **Đối tượng**: Người dùng cá nhân (bao gồm nhà đầu tư), Sàn BĐS (tư vấn khách hàng).
- **Giao diện**: Chatbot (kết quả cơ bản).

### 2.3. Hỗ trợ giao dịch và định giá
- **Chức năng**:
  - Hướng dẫn pháp lý (hợp đồng, công chứng, thuế) theo quy định Việt Nam 2025.
    - **Ví dụ**: Hỏi “Làm hợp đồng mua nhà cần gì?” → Chatbot liệt kê các bước: kiểm tra sổ đỏ, công chứng.
  - Cung cấp mẫu hợp đồng PDF cơ bản.
    - **Ví dụ**: Hỏi “Mẫu hợp đồng mua nhà?” → Chatbot cung cấp PDF hợp đồng mua bán.
  - Đề xuất giá bán/cho thuê dựa trên thị trường.
    - **Ví dụ**: Hỏi “Giá bán căn hộ 70m² Quận 7?” → Chatbot trả “2.8 tỷ, dựa trên 40 triệu/m².”
  - Kiểm tra giá niêm yết hợp lý.
    - **Ví dụ**: Nhập giá 3 tỷ → Chatbot cảnh báo “Giá cao hơn thị trường 10%.”
  - Tư vấn tài chính (vay ngân hàng, trả góp).
    - **Ví dụ**: Hỏi “Vay ngân hàng mua nhà?” → Chatbot gợi ý “Vietcombank: vay 70%, lãi 7.5%/năm.”
  - Tính toán chi phí tổng (giá mua, thuế, lãi vay).
    - **Ví dụ**: Hỏi “Chi phí mua căn hộ 2 tỷ, vay 50%?” → Chatbot trả “Tổng: 2.2 tỷ, trả góp 15 triệu/tháng.”
- **Đối tượng**: Người dùng cá nhân (bao gồm nhà đầu tư), Sàn BĐS (tư vấn khách hàng).
- **Giao diện**: Chatbot (hướng dẫn, tải mẫu, tính toán).

### 2.4. Quản lý danh mục và lịch hẹn
- **Chức năng**:
  - Cập nhật danh sách bất động sản (giá, tình trạng).
    - **Ví dụ**: Sàn nhập 10 căn hộ, cập nhật giá từ 2–5 tỷ trên dashboard.
  - Theo dõi giá trị và trạng thái giao dịch (đặt cọc, ký hợp đồng).
    - **Ví dụ**: Dashboard hiển thị “Danh mục: 50 tỷ, căn hộ A đã đặt cọc.”
  - Quản lý lịch hẹn (tư vấn/xem nhà) qua calendar.
    - **Ví dụ**: Khách đặt lịch qua chatbot → Dashboard hiển thị “10h, Thứ Bảy, Quận 7.”
  - Đồng bộ với Google Calendar/Zalo; phân công nhân viên.
    - **Ví dụ**: Admin phân công nhân viên A cho lịch hẹn, đồng bộ Google Calendar.
  - Gửi thông báo lịch hẹn cho khách hàng/nhân viên.
    - **Ví dụ**: Gửi Zalo “Lịch xem nhà: 10h, Thứ Bảy” cho khách và nhân viên.
- **Đối tượng**: Sàn BĐS.
- **Giao diện**: Dashboard sàn BĐS, chatbot (đặt lịch).

### 2.5. Phân tích khách hàng và tích hợp kinh doanh
- **Chức năng**:
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

### 2.6. Quản trị hệ thống và bảo mật
- **Chức năng**:
  - Nhập, chỉnh sửa dữ liệu BĐS và chính sách (thuế, quy hoạch).
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
  - Thu thập phản hồi (nút 1-5 sao trong chatbot).
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
- **Công nghệ**: React, JSX, Tailwind CSS, WebSocket (chatbot), WebRTC (voice mode).
- **Chức năng**:
  - Chatbot text/voice: Tra cứu, gợi ý, đặt lịch, thông báo cho người dùng cá nhân/sàn BĐS.
  - Dashboard: Quản lý danh mục, lịch hẹn, báo cáo cho sàn BĐS; quản trị cho quản trị viên.
  - Hiển thị danh sách, bảng/biểu đồ, bản đồ (Google Maps/OpenStreetMap).
  - Form đặt lịch, tải báo cáo PDF.
  - Đa nền tảng (web: Chrome/Firefox/Safari; mobile: PWA/React Native).
  - Đăng nhập OAuth2 (Google, Zalo) cho lưu lịch sử.
- **UX/UI**: Phản hồi <2s, giao diện tối giản (màu xanh/trắng/xám, font Roboto/Noto Sans), tuân thủ WCAG 2.1.
- **Tùy chỉnh**: Hiển thị chức năng theo vai trò (người dùng cá nhân, sàn BĐS, quản trị viên).

### 4.2. Backend (BE)
- **Công nghệ**: Node.js/Express hoặc Python/FastAPI, WebSocket, NLP (spaCy/Hugging Face).
- **Chức năng**:
  - Xử lý câu hỏi tự nhiên, tích hợp dữ liệu từ X/sàn giao dịch.
  - REST API cho tra cứu, lịch hẹn, báo cáo; xác thực JWT/OAuth2.
  - Đồng bộ Google Calendar/Zalo; thông báo email/Zalo.
- **Hiệu suất**: API trả lời <200ms (cơ bản), <500ms (phức tạp); hỗ trợ 10.000 người dùng đồng thời; uptime 99.9% (CDN Cloudflare).

### 4.3. Cơ sở dữ liệu (DB)
- **Công nghệ**: PostgreSQL (dữ liệu cấu trúc: BĐS, lịch hẹn, người dùng), MongoDB (lịch sử chat, nhật ký), Redis (cache).
- **Bảo mật**: Mã hóa AES-256, sao lưu hàng ngày, tuân thủ Luật Bảo vệ Dữ liệu Cá nhân 2025.

### 4.4. Hiệu suất và mở rộng
- **Hiệu suất**: Tra cứu <200ms, chatbot <2s (text)/<3s (voice), lịch hẹn <500ms.
- **Mở rộng**: Microservices, Docker, Kubernetes, Redis cache.
- **Tính sẵn sàng**: Uptime 99.9% (AWS Elastic Load Balancer).

### 4.5. Bảo mật
- Mã hóa TLS/SSL, JWT/OAuth2, phân quyền theo vai trò.
- Kiểm tra bảo mật định kỳ (OWASP ZAP).
- Nhật ký hoạt động, tuân thủ pháp luật.

### 4.6. Testing
- Unit test: Jest/pytest (80% coverage).
- Integration test: Cypress (FE), Postman (API).
- Load test: JMeter (10.000 người dùng).
- Security test: OWASP ZAP.

### 4.7. Triển khai
- Đám mây: AWS/Google Cloud, CI/CD (GitHub Actions).
- Giám sát: Prometheus, Grafana.
- Backup: Khôi phục trong 1 giờ.

---

## 5. Yêu cầu tích hợp
- Google Maps/OpenStreetMap (bản đồ tương tác).
- Google Calendar/Zalo API (lịch hẹn).
- Sàn giao dịch (Batdongsan.com.vn, Chợ Tốt) qua API/scraping (nếu được phép).
- X API (bài đăng BĐS).

---

## 6. Roadmap triển khai
- **Phase 1 (MVP – 3 tháng)**: Chatbot tra cứu, gợi ý, định giá, lịch hẹn, API cơ bản, dashboard cho sàn BĐS/quản trị viên.
- **Phase 2 (6–9 tháng)**: Phân tích thị trường nâng cao, báo cáo sàn BĐS, bản đồ tương tác.
- **Phase 3 (12 tháng)**: Hỗ trợ đa ngôn ngữ, cải tiến AI, tích hợp thêm sàn BĐS.