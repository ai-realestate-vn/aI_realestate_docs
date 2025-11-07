**CHỨC NĂNG CƠ BẢN CỦA 1 CHATBOT AI**

1\. Nhóm Chức năng Cốt lõi (Core AI Functions)

  -----------------------------------------------------------------------
  **Chức năng**             **Mô tả ngắn gọn**  **Ví dụ**
  ------------------------- ------------------- -------------------------
  **1.1. Hiểu ngôn ngữ tự   Giúp chatbot hiểu ý Người dùng gõ "Tôi muốn
  nhiên (NLU -- Natural     định (intent) và    xem nhà ở Quận 7 giá dưới
  Language Understanding)** trích xuất thông    3 tỷ" → chatbot hiểu **ý
                            tin quan trọng từ   định = tìm bất động
                            câu người dùng.     sản**, **điều kiện = Quận
                                                7, \<3 tỷ**

  **1.2. Sinh phản hồi (NLG Tạo ra câu trả lời  "Tôi đã tìm thấy 3 căn hộ
  -- Natural Language       tự nhiên, rõ ràng   ở Quận 7 phù hợp yêu cầu
  Generation)**             và phù hợp ngữ      của bạn."
                            cảnh.               

  **1.3. Quản lý hội thoại  Theo dõi tiến trình Sau khi người dùng nói
  (Dialogue Management)**   cuộc hội thoại,     "Tôi muốn mua", chatbot
                            biết đang ở bước    nhớ "ngữ cảnh = nhu cầu
                            nào, giữ ngữ cảnh   mua" để hỏi tiếp "Bạn
                            cho các câu hỏi     muốn mua nhà hay đất?"
                            tiếp theo.          

  **1.4. Xử lý ngữ cảnh và  Lưu tạm thông tin   Ghi nhớ "Quận 7" cho các
  bộ nhớ tạm (Context &     để dùng lại trong   câu hỏi tiếp theo mà
  Memory)**                 các câu sau.        không cần người dùng lặp
                                                lại.

  **1.5. Học và cải thiện   Cho phép chatbot    Ghi nhận khi người dùng
  theo dữ liệu              học từ tương tác    đánh giá "Câu trả lời
  (Learning/Feedback        người dùng để cải   không đúng", sau đó dùng
  Loop)**                   thiện độ chính xác. để huấn luyện lại.
  -----------------------------------------------------------------------

2\. Nhóm Giao tiếp & Tương tác (Interaction Features)

  ------------------------------------------------------------------------
  **Chức năng**        **Mô tả**               **Ví dụ**
  -------------------- ----------------------- ---------------------------
  **2.1. Nhắn tin hai  Giao tiếp qua tin nhắn  Messenger, Web Chat UI
  chiều (Text-based    văn bản.                
  chat)**                                      

  **2.2. TTS/STT --    Text-to-Speech và       Người dùng nói "Tìm nhà ở
  Giọng nói**          Speech-to-Text để người Đà Nẵng", chatbot nhận và
                       dùng nói chuyện bằng    phản hồi bằng giọng.
                       giọng.                  

  **2.3. Gợi ý nhanh / Gợi ý sẵn các câu trả   "Chọn loại bất động sản: 🏠
  Quick Replies**      lời hoặc tùy chọn.      Nhà ở

  **2.4. Hiển thị thẻ  Trả kết quả có hình     Hiển thị thẻ "Căn hộ 2PN --
  thông tin (Cards /   ảnh, nút, bản đồ, biểu  2.8 tỷ -- Sunrise City" kèm
  Rich Messages)**     đồ,...                  ảnh và nút "Xem chi tiết"

  **2.5. Hỗ trợ đa     Hỗ trợ người dùng nói   Người Việt gõ tiếng Anh →
  ngôn ngữ             bằng nhiều ngôn ngữ.    chatbot vẫn hiểu.
  (Multi-language)**                           
  ------------------------------------------------------------------------

3\. Nhóm Tích hợp & Kết nối (Integration & Backend)

  -----------------------------------------------------------------------
  **Chức năng**         **Mô tả**             **Ví dụ**
  --------------------- --------------------- ---------------------------
  **3.1. Kết nối API    Lấy dữ liệu thật từ   Lấy danh sách dự án từ DB
  hoặc cơ sở dữ liệu**  backend hoặc nguồn    hoặc API của Sở Xây dựng.
                        ngoài.                

  **3.2. RAG / Search   Tìm câu trả lời trong Chatbot AI đọc Luật Nhà ở
  Engine Integration**  tài liệu hoặc hệ      hoặc tài liệu nội bộ để trả
                        thống tri thức.       lời chính xác.

  **3.3. Webhook /      Kích hoạt hành động   "Gửi báo cáo", "Đặt lịch
  Event Trigger**       khi người dùng ra     xem nhà" → gọi API backend.
                        lệnh.                 

  **3.4. Authentication Nhận biết người dùng, Người dùng đăng nhập →
  / Personalization**   lưu thông tin tài     chatbot nhớ tên và lịch sử
                        khoản.                tư vấn.
  -----------------------------------------------------------------------

4\. Nhóm Quản trị & Hệ thống (Admin & Analytics)

  ------------------------------------------------------------------------
  **Chức năng**             **Mô tả**                **Ví dụ**
  ------------------------- ------------------------ ---------------------
  **4.1. Ghi log & lịch sử  Lưu lại toàn bộ tương    Admin xem lại các
  hội thoại**               tác để tra cứu.          cuộc trò chuyện.

  **4.2. Dashboard thống    Phân tích số lượng người Biểu đồ "Tổng số lượt
  kê**                      dùng, câu hỏi phổ biến,  hỏi trong tuần".
                            tỷ lệ hài lòng.          

  **4.3. Flagged questions  Gắn cờ câu trả lời sai   "Chatbot trả lời sai
  / lỗi AI**                để cải thiện.            → gắn cờ → QA team xử
                                                     lý."

  **4.4. Quản lý tri thức   Thêm, sửa, xóa tài liệu  Cập nhật thêm chính
  (Knowledge Base           hoặc câu hỏi mẫu.        sách mới vào kho dữ
  Management)**                                      liệu.
  ------------------------------------------------------------------------

5\. Nhóm Bảo mật & Quyền riêng tư

  -----------------------------------------------------------------------
  **Chức năng**                   **Mô tả**
  ------------------------------- ---------------------------------------
  **5.1. Xác thực và phân quyền   Kiểm soát ai được truy cập, cấp độ
  (Auth)**                        quyền của người dùng.

  **5.2. Ẩn/ẩn danh dữ liệu cá    Bảo vệ thông tin cá nhân trong hội
  nhân**                          thoại.

  **5.3. Cơ chế xóa / lưu trữ dữ  Tuân thủ chính sách lưu trữ (GDPR, Nghị
  liệu theo quy định**            định 13/2023 VN).
  -----------------------------------------------------------------------

6\. (Tuỳ chọn Nâng cao)

  -----------------------------------------------------------------------
  **Chức năng**                      **Ứng dụng**
  ---------------------------------- ------------------------------------
  **6.1. Kết nối LLM / Vector DB     Dùng RAG để tìm câu trả lời chính
  (Qdrant, Pinecone, etc.)**         xác theo tài liệu Việt Nam.

  **6.2. Fine-tuning / Custom        Tùy chỉnh hành vi chatbot cho từng
  Prompts**                          lĩnh vực.

  **6.3. Memory dài hạn (Persistent  Nhớ lịch sử trò chuyện lâu dài giữa
  Memory)**                          các phiên.

  **6.4. Tự động tóm tắt             Tạo tóm tắt nội dung hội thoại hoặc
  (Summarization)**                  văn bản dài.

  **6.5. Multi-agent / Tool-using    Chatbot biết gọi API, tính toán, tra
  Chatbot**                          cứu dữ liệu,... giống "AI agent".
  -----------------------------------------------------------------------
