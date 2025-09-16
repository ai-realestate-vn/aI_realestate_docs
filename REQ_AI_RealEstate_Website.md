# 📘 YÊU CẦU HỆ THỐNG WEBSITE AI SALE BẤT ĐỘNG SẢN

# 1. 🎯 Mục tiêu tổng thể

Phát triển một nền tảng website sử dụng AI (dựa trên Large Language Models - LLMs) để tư vấn và bán bất động sản, tập trung vào trải nghiệm cá nhân hóa, minh bạch và tiện lợi cho người dùng cuối. Website sẽ train model trên tập dữ liệu bất động sản (bao gồm thông tin chi tiết về vị trí, giá cả, pháp lý, tiện ích và cuộc trò chuyện của Agent thực) để cung cấp tư vấn chính xác. Tương tác chính qua Text, Audio và Video. Mục tiêu là giải quyết vấn đề tìm kiếm, tư vấn và giao dịch bất động sản một cách nhanh chóng, đáng tin cậy, giảm thiểu rủi ro và tăng tính cá nhân hóa, lấy cảm hứng từ các hệ thống gợi ý địa điểm du lịch nhưng áp dụng cho lĩnh vực bất động sản.

# 2. 👥 Đối tượng sử dụng

- Người dùng cuối: Cá nhân hoặc gia đình tìm mua/rent bất động sản (căn hộ, nhà đất, đất nền), như người trẻ có ngân sách 2-4 tỷ VND, cần tư vấn về vị trí gần tiện ích (trường học, bệnh viện).
- Môi giới/Doanh nghiệp bất động sản: Cung cấp dữ liệu, dịch vụ và kết nối trực tiếp với người dùng qua nền tảng.
- Quản trị viên: Quản lý dữ liệu, train model AI và kiểm duyệt nội dung.

# 3. 🧱 Chức năng chính

## A. Tìm kiếm và Gợi ý Cá nhân hóa

- Gợi ý bất động sản dựa trên câu hỏi chi tiết: AI hỏi về nhu cầu (ngân sách, số phòng ngủ, vị trí ưu tiên, loại hình như căn hộ gần trường học) rồi gợi ý danh sách phù hợp từ dữ liệu train. Tích hợp tìm kiếm thông minh với từ khóa tự nhiên (ví dụ: "tôi cần mua căn hộ 2 phòng ngủ gần trường học, giá dưới 2 tỷ").
- Bộ lọc nâng cao với bản đồ tương tác: Sử dụng bản đồ (Google Maps hoặc OpenStreetMap + Leaflet) để hiển thị vị trí bất động sản, khoảng cách đến tiện ích (trường học, bệnh viện, siêu thị, giao thông công cộng), mật độ dân cư và môi trường xung quanh.
- So sánh bất động sản: Chọn 2-3 bất động sản để so sánh song song về giá, diện tích, tiện ích, pháp lý và đánh giá.

## B. Thông tin Chi tiết

- Thông tin minh bạch và cập nhật thời gian thực: Hiển thị chi tiết như giấy tờ pháp lý (sổ đỏ, giấy phép xây dựng), lịch sử giá, tình trạng (mới xây, đang cho thuê), cảnh báo rủi ro (ngập nước, tranh chấp). AI tổng hợp đánh giá khu vực từ cộng đồng (an ninh, môi trường sống, giao thông).
- Dự đoán tương lai: AI ước tính giá trị tăng trưởng trong 5-10 năm dựa trên xu hướng thị trường và quy hoạch đô thị.

## C. Tư vấn Tài chính và Pháp lý

- Tính toán chi phí toàn diện: Nhập ngân sách, AI tính tổng chi phí (giá nhà + thuế + phí notary + lãi vay); gợi ý gói vay từ ngân hàng đối tác (?) . Bao gồm mô phỏng chi phí duy trì.
- Hỗ trợ pháp lý cơ bản: AI giải thích hợp đồng mẫu, checklist kiểm tra giấy tờ; kết nối với luật sư qua chat/video. Cung cấp mẫu hợp đồng và giải thích điều khoản khó hiểu.

## D. Tương tác và Hỗ trợ Đa kênh

- Chuyển đổi mượt mà giữa Text/Audio/Video: Bắt đầu chat text, chuyển sang audio (voice call) hoặc video để tư vấn trực quan; AI hỗ trợ tiếng Việt tự nhiên, nhận diện giọng nói địa phương. AI giải thích, gợi ý phương án phù hợp theo nhu cầu tài chính; tóm tắt ưu/nhược điểm từng bất động sản.
- Chatbot 24/7 với lịch sử lưu trữ: Lưu cuộc trò chuyện để quay lại, AI nhớ ngữ cảnh (ví dụ: update về căn hộ quận 7 từ lần trước). Dự đoán giá & xu hướng thị trường trong 3-5 năm.
- Kết nối với agent thực: Chuyển sang gọi video với nhân viên bán hàng nếu cần đàm phán.

## E. Cộng đồng và Đánh giá (?)

- Review và đánh giá từ người dùng thực: Xem đánh giá 5 sao, bình luận, ảnh thực tế; AI tóm tắt (ví dụ: "80% hài lòng với vị trí").
- Forum hoặc Q&A cộng đồng: Người dùng hỏi đáp về khu vực; AI moderate và trả lời nhanh.
- Hồ sơ cá nhân: Lưu danh sách yêu thích, lịch sử tìm kiếm; nhận thông báo về bất động sản mới phù hợp hoặc giảm giá qua web, email, Zalo, SMS.

## F. Đặt lịch tư vấn với Agent thực

- Lịch hẹn xem nhà trực tuyến: Đặt lịch qua lịch tích hợp (Google Calendar), với nhắc nhở qua email/SMS.
- Đặt lịch tư vấn với mối giới thật.
- Theo dõi sau bán hàng: Gửi cập nhật tiến độ xây dựng; gợi ý dịch vụ liên quan (nội thất, chuyển nhà). Kết nối môi giới/người bán thật để đặt lịch gặp trực tiếp.

# 4. ⚙️ Yêu cầu hệ thống

## Giao diện người dùng

- Responsive: Hỗ trợ desktop, mobile (React + Tailwind CSS).
- Dark/light mode: Tùy chọn giao diện sáng/tối.
- UX tối ưu: Giao diện đơn giản, dễ thao tác; hỗ trợ đa ngôn ngữ (VN, EN).
- Accessibility: Tuân thủ WCAG cơ bản.

## Backend & API

- RESTful API: Xây bằng Nest.js, endpoint cho tìm kiếm, gợi ý, lưu hồ sơ, đặt lịch.
- AI Integration: Train LLMs trên tập dữ liệu bất động sản; sử dụng TensorFlow.js hoặc API như Dialogflow cho chatbot.
- API bên thứ ba: Google Maps/Leaflet cho bản đồ.

## Dữ liệu & bảo mật

- Cơ sở dữ liệu: PostgreSQL (Supabase, Prisma); bảng cho bất động sản (vị trí, giá, pháp lý), người dùng (hồ sơ, lịch sử), đánh giá.
- Bảo mật: HTTPS, JWT cho xác thực; mã hóa thanh toán và dữ liệu cá nhân; tuân thủ GDPR.
- Dữ liệu tĩnh: JSON chứa dữ liệu bất động sản ban đầu (crawl từ nguồn công khai như Batdongsan.com.vn).

## Hiệu năng & khả năng mở rộng

- Hiệu năng: Tải trang < 2s; hỗ trợ 1.000 user đồng thời.
- Triển khai: Vercel (miễn phí); kiến trúc monolith ban đầu.
- Cache: Lưu dữ liệu bản đồ và gợi ý để giảm truy vấn.

# 5. 📚 Công cụ

- Frontend: React, Tailwind CSS.
- Backend: Nest.js, Supabase, Prisma.
- Bản đồ: OpenStreetMap, Leaflet.
- AI: TensorFlow.js, Dialogflow, Huggingface.
- Triển khai: Vercel, Render,...
- Phát triển: VS Code, GitHub, Postman.

# 6. 🧑‍🤝‍🧑 Các bên liên quan

- Người dùng cuối: Tìm kiếm, tư vấn và giao dịch.
- Môi giới/Doanh nghiệp: Cung cấp dữ liệu và dịch vụ.
- Quản trị viên: Quản lý hệ thống và train AI.

# 7. 📈 Kết quả mong đợi

- Website chạy local hoặc Vercel, hỗ trợ tư vấn AI qua Text/Audio/Video, gợi ý cá nhân hóa và giao dịch an toàn.
- Nền tảng học hỏi về AI, LLMs và ứng dụng bất động sản.
- Sẵn sàng mở rộng với dữ liệu lớn hơn và tính năng AR/VR.
