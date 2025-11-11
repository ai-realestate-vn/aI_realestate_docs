# Hệ thống AI Tư Vấn Bất Động Sản - Tài Liệu MVP

## 1. Đối tượng người dùng

## 1.1. Đối tượng chính (Người dùng cá nhân – sử dụng thường xuyên, nhu cầu cao)

### Người mua/thuê BĐS cá nhân

- **Đặc điểm**: Cá nhân, gia đình tìm kiếm nhà ở, căn hộ, đất đai hoặc BĐS thương mại để sử dụng.
- **Nhu cầu**: Tìm thông tin về giá cả, vị trí, tiện ích, pháp lý; so sánh các lựa chọn; nhận tư vấn nhanh thay cho môi giới.
- **Lý do ưu tiên**: Nhóm đông đảo nhất, chiếm tỷ trọng lớn trong giao dịch BĐS.

### Người bán/cho thuê BĐS cá nhân

- **Đặc điểm**: Chủ sở hữu muốn bán hoặc cho thuê nhà, đất, BĐS thương mại.
- **Nhu cầu**: Định giá tài sản, đăng tin hiệu quả, tiếp cận khách hàng tiềm năng, tư vấn xu hướng thị trường.
- **Lý do ưu tiên**: Giúp giảm chi phí và phụ thuộc vào môi giới truyền thống.

### Người dùng cần thông tin nhanh

- **Đặc điểm**: Cá nhân/tổ chức chỉ muốn tham khảo thông tin thị trường (giá cả, pháp lý, xu hướng).
- **Nhu cầu**: Tra cứu tổng quan, so sánh giá, kiểm tra minh bạch.
- **Lý do**: Tệp khách hàng tiềm năng, thường ở giai đoạn tìm hiểu trước khi giao dịch.

---

## 1.2. Đối tượng thứ cấp (Nhà đầu tư / Doanh nghiệp – sử dụng chuyên sâu, nhu cầu cụ thể)

### Nhà đầu tư BĐS

- **Đặc điểm**: Cá nhân hoặc tổ chức đầu tư vào BĐS để sinh lời (mua đi bán lại, cho thuê dài hạn).
- **Nhu cầu**: Phân tích dữ liệu thị trường, dự báo xu hướng giá, so sánh cơ hội đầu tư, tính toán lợi nhuận/rủi ro.
- **Lý do**: Nhóm đòi hỏi AI có khả năng xử lý dữ liệu lớn, đưa ra phân tích chuyên sâu.

### Doanh nghiệp BĐS / Sàn giao dịch / Chủ đầu tư

- **Đặc điểm**: Công ty môi giới, sàn phân phối, hoặc chủ đầu tư dự án.
- **Nhu cầu**: Tự động hóa tư vấn khách hàng, quản lý danh mục sản phẩm, phân tích thị trường, tích hợp AI vào nền tảng kinh doanh.
- **Lý do**: Giúp nâng cao hiệu quả, tối ưu chi phí và mở rộng khả năng tiếp cận khách hàng.

---

## 1.3. Đối tượng quản trị (Vận hành hệ thống – sử dụng nội bộ)

### Quản trị viên & Nhà quản lý dữ liệu

- **Đặc điểm**: Admin, đội ngũ kỹ thuật, hoặc nhà quản lý chịu trách nhiệm duy trì AI.
- **Nhu cầu**:
  - Quản lý và cập nhật dữ liệu thị trường, pháp lý, tin tức chính sách.
  - Kiểm tra độ chính xác và tính minh bạch của hệ thống.
  - Giám sát hiệu quả vận hành và cải thiện chất lượng tư vấn.
- **Lý do**: Đảm bảo AI hoạt động ổn định, cập nhật kịp thời, phù hợp quy định pháp luật.

---

## 2. Yêu cầu MVP

## 2.1. Yêu cầu MVP cho Người dùng cá nhân

### Tra cứu và cung cấp thông tin bất động sản

- Cung cấp thông tin cơ bản về giá, vị trí, loại hình bất động sản (nhà phố, chung cư, đất nền), và tiện ích xung quanh dựa trên câu hỏi tự nhiên (ví dụ: “Giá căn hộ 2 phòng ngủ ở Quận 7?”).
- Hiển thị nguồn gốc dữ liệu từ các nguồn uy tín (sàn giao dịch, bài đăng trên X, cơ quan quản lý).

### Tư vấn cá nhân hóa và gợi ý

- Thu thập yêu cầu qua chat bằng tiếng Việt (ví dụ: vị trí, giá, diện tích, tiện ích).
- Đề xuất danh sách bất động sản phù hợp với tiêu chí người dùng (ví dụ: căn hộ dưới 3 tỷ tại TP.HCM).
- Cho phép so sánh cơ bản giữa các bất động sản dựa trên giá, vị trí, và tiện ích.

### Giao diện thân thiện và đa nền tảng

- Hỗ trợ giao tiếp qua chatbot trên web để tra cứu và đặt lịch (ví dụ: “Đặt lịch xem nhà ở Quận 7 vào thứ Bảy”).
- Giao diện đơn giản, dễ sử dụng, không yêu cầu đăng nhập cho tra cứu cơ bản.
- Lưu trữ lịch sử tra cứu hoặc danh sách bất động sản yêu thích (nếu người dùng đăng nhập).

### Hỗ trợ 24/7 và đặt lịch

- Trả lời tức thì các truy vấn cơ bản qua text hoặc voice mode.
- Hỗ trợ đặt lịch tư vấn hoặc xem bất động sản trực tiếp qua Zoom/Zalo, với xác nhận qua thông báo.

### Tính minh bạch và bảo mật

- Đảm bảo thông tin trung lập, không ưu tiên dự án hoặc chủ đầu tư cụ thể.
- Hỗ trợ tùy chọn ẩn danh cho tra cứu, không lưu trữ dữ liệu cá nhân nếu chưa đồng ý.
- Tuân thủ cơ bản Luật Bảo vệ Dữ liệu Cá nhân 2025.

## 2.2. Yêu cầu MVP cho Nhà đầu tư/doanh nghiệp

### Quản lý lịch hẹn

- Cung cấp giao diện calendar để admin doanh nghiệp xem và phân công lịch hẹn (tư vấn, xem bất động sản, ký hợp đồng).
- Đồng bộ calendar với Google Calendar hoặc Zalo để gửi thông báo.

## 2.3. Yêu cầu MVP cho Quản trị viên & Nhà quản lý dữ liệu

### Quản lý và cập nhật dữ liệu

- Hỗ trợ nhập và cập nhật dữ liệu bất động sản (giá, pháp lý, tiện ích) từ các nguồn uy tín.
- Cung cấp công cụ kiểm tra tính hợp lệ của dữ liệu trước khi đưa vào hệ thống.

### Giám sát hiệu suất

- Cung cấp dashboard cơ bản để theo dõi số lượng truy vấn, tỷ lệ phản hồi, và lỗi hệ thống.
- Hỗ trợ xem lịch sử tư vấn để đánh giá tính chính xác của AI.

### Bảo mật và tuân thủ pháp luật

- Cung cấp phân quyền truy cập đơn giản (quản trị viên cấp cao chỉnh sửa dữ liệu, cấp thấp chỉ xem).
- Đảm bảo hệ thống tuân thủ Luật Bảo vệ Dữ liệu Cá nhân 2025 với kiểm tra cơ bản.
- Lưu trữ nhật ký hoạt động của quản trị viên (cập nhật dữ liệu, chỉnh sửa hệ thống).

---

## 3. Yêu cầu kỹ thuật cốt lõi cho MVP

## 3.1. Yêu cầu kỹ thuật cho Frontend (FE)

### Công nghệ

- Sử dụng **React** với **JSX** để xây dựng giao diện web và mobile app, đảm bảo tái sử dụng component và hiệu suất cao.
- Sử dụng **Tailwind CSS** để thiết kế giao diện nhanh, linh hoạt, và nhất quán.
- Tích hợp **WebSocket** hoặc **Socket.IO** cho chatbot thời gian thực và thông báo.
- Sử dụng **WebRTC** hoặc thư viện tương tự (như **Agora**) để hỗ trợ voice mode tiếng Việt.

### Chức năng FE

- **Tra cứu và tư vấn**:
  - Hiển thị danh sách bất động sản (giá, vị trí, loại hình, tiện ích) với bộ lọc cơ bản (khu vực, giá, diện tích).
  - Hỗ trợ nhập câu hỏi tự nhiên qua text hoặc voice mode (tiếng Việt).
  - Hiển thị thông tin tổng quan thị trường (giá trung bình, xu hướng) và cảnh báo rủi ro cơ bản.
- **Giao diện người dùng**:
  - Giao diện tối giản với bố cục rõ ràng, sử dụng bảng, biểu đồ đơn giản để so sánh bất động sản.
  - Hỗ trợ bản đồ tương tác (dựa trên **Google Maps API** hoặc **OpenStreetMap**) để hiển thị vị trí và tiện ích xung quanh.
  - Form đặt lịch tư vấn/xem bất động sản, tích hợp với calendar.
- **Đa nền tảng**:
  - Hoạt động mượt mà trên web (Chrome, Firefox, Safari) và mobile app (iOS, Android) thông qua **React Native** hoặc **Progressive Web App (PWA)**.
  - Không yêu cầu đăng nhập cho tra cứu cơ bản, hỗ trợ đăng nhập qua **OAuth2** (Google, Zalo) để lưu lịch sử tra cứu.

### Yêu cầu UX/UI

- **UX**: Trải nghiệm mượt mà, phản hồi tức thì (dưới 2 giây) cho tra cứu và đặt lịch. Hỗ trợ voice mode tiếng Việt tự nhiên, ưu tiên người không rành công nghệ.
- **UI**: Màu sắc trung tính (xanh, trắng, xám), font chữ dễ đọc (Roboto, Noto Sans), bố cục chia thành các phần rõ ràng (tra cứu, danh sách gợi ý, lịch hẹn).
- **Tính tiếp cận**: Hỗ trợ kích thước chữ điều chỉnh, tương thích với người khuyết tật (WCAG 2.1).

## 3.2. Yêu cầu kỹ thuật cho Backend (BE)

### Công nghệ

- Sử dụng **Express** và **Python** với **FastAPI** để xây dựng REST API hiệu suất cao.
- Tích hợp **WebSocket** cho chatbot và thông báo thời gian thực.
- Sử dụng **NLP** (Natural Language Processing) cơ bản
- Triển khai trên dịch vụ đám mây (**VPS**) với **Docker** để dễ dàng mở rộng.

### Chức năng BE

- **Xử lý truy vấn người dùng**:
  - Phân tích câu hỏi tự nhiên (text/voice) để trả về thông tin bất động sản hoặc gợi ý.
  - Tích hợp dữ liệu từ nguồn bên ngoài (sàn giao dịch, bài đăng trên X) qua API hoặc **web scraping** (nếu được phép).
- **API cho doanh nghiệp**:
  - Cung cấp REST API để tra cứu bất động sản, quản lý lịch hẹn, và báo cáo xu hướng khách hàng.
  - Hỗ trợ xác thực API bằng **JWT** hoặc **OAuth2**.
- **Quản lý lịch hẹn**:
  - Lưu trữ và đồng bộ lịch hẹn với Google Calendar/Zalo qua API.
  - Gửi thông báo qua email hoặc Zalo cho người dùng và nhân viên doanh nghiệp.
- **Bảo mật**:
  - Mã hóa dữ liệu truyền tải bằng **TLS/SSL**.
  - Lưu trữ mật khẩu bằng **bcrypt** hoặc **Argon2**.
  - Phân quyền truy cập dựa trên vai trò (người dùng, quản trị viên, doanh nghiệp).

## 3.3. Yêu cầu cơ sở dữ liệu (DB)

### Công nghệ

- **MongoDB** cho dữ liệu phi cấu trúc: lịch sử chat, phản hồi người dùng, nhật ký quản trị viên.
- **VectorDB** Qdrant lưu trữ data về bất động sản

### Cấu trúc DB

- **Bảng bất động sản (MongoDB, QDrant)**: Hybrid
  - Trường: ID, giá, vị trí, loại hình, diện tích, tiện ích, pháp lý, nguồn dữ liệu, thời gian cập nhật.
- **Bảng lịch hẹn (MongoDB)**:
  - Trường: ID, người dùng, thời gian, loại hẹn (tư vấn/xem nhà), trạng thái, nhân viên phụ trách.
- **Bảng người dùng (MongoDB)**:
  - Trường: ID, tên, email, vai trò (người dùng cá nhân/doanh nghiệp/quản trị viên), thông tin xác thực.
- **Tài liệu lịch sử chat (MongoDB)**:
  - Trường: ID, người dùng, nội dung câu hỏi, câu trả lời, thời gian.
- **Tài liệu nhật ký quản trị viên (MongoDB)**:
  - Trường: ID, quản trị viên, hành động (cập nhật dữ liệu, chỉnh sửa hệ thống), thời gian.

### Yêu cầu bảo mật DB

- Mã hóa dữ liệu nhạy cảm (thông tin người dùng, lịch sử giao dịch) bằng **AES-256**.
- Sao lưu dữ liệu định kỳ (hàng ngày) trên dịch vụ đám mây.
- Tuân thủ **Luật Bảo vệ Dữ liệu Cá nhân 2025** với kiểm tra quyền truy cập.

## 3.4. Yêu cầu tích hợp

- Tích hợp với **OpenStreetMap** để hiển thị bản đồ tương tác.
