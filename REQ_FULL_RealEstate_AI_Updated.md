# Realtor: Artificial Intelligence Real Estate Sales Assistant - Hệ Thống AI Hỗ Trợ Bất Động Sản Việt Nam

**Phiên bản**: 1.0 (Ngày cập nhật: 29/09/2025)  
**Mô tả dự án**: Realtor: Artificial Intelligence Real Estate Sales Assistant là nền tảng AI đa chức năng hỗ trợ bất động sản (BDS) tại Việt Nam, tập trung vào tư vấn cá nhân hóa, phân tích thị trường, đánh giá tài sản, và hỗ trợ tài chính/pháp lý. Dự án mở rộng từ chatbot hỗ trợ sales BDS, sử dụng RAG (Retrieval-Augmented Generation), finetuning model, và tích hợp công cụ như GIS, AR/VR. Phát triển với scale nhỏ (2 người: AI Engineer lead, Fullstack Dev support), timeline 3 tháng (29/09/2025 - 29/12/2025), budget 500-1000 USD. MVP nhắm đến accuracy >85% trong matching dự án, tích hợp multichannel (web, Zalo, FB). Dựa trên yêu cầu REQ_FULL_RealEstate_AI.md và roadmap chatbot BDS, bổ sung từ nguồn như Omicall cho kịch bản sales và FPT.AI cho lợi ích AI trong BDS.

## 1. 🎯 Mục Tiêu Tổng Thể

Phát triển nền tảng AI hỗ trợ BDS đa chức năng giúp người dùng:

- Nắm bắt thông tin thị trường BDS chính xác, cập nhật liên tục (dữ liệu realtime từ team Môi giới cung cấp)
- Đưa ra tư vấn cá nhân hóa theo nhu cầu, ngân sách và mục tiêu đầu tư.
- So sánh, đánh giá và phân tích bất động sản.
- Cung cấp thông tin pháp lý rõ ràng, đáng tin cậy và cảnh báo rủi ro tiềm ẩn (kết hợp với 1 AI Law Agent khác).
- Hỗ trợ tìm kiếm, lọc và theo dõi BDS từ nhiều nguồn dữ liệu.
- Tư vấn tài chính và phương án vay vốn/đầu tư tối ưu.
- Mang đến trải nghiệm trực quan, dễ dàng qua giao diện đa kênh.
- Phân tích hình ảnh và tài liệu liên quan đến BDS.
- Đảm bảo tính minh bạch, trung lập và bảo mật dữ liệu người dùng theo tiêu chuẩn cao (tuân thủ GDPR và Luật An ninh Mạng VN).

## 2. 👥 Đối Tượng Sử Dụng

- Người mua nhà để ở.
- Nhà đầu tư cá nhân hoặc tổ chức (ngắn hạn, dài hạn, cho thuê).
- Môi giới và sàn giao dịch BDS.
- Doanh nghiệp BDS, chủ đầu tư dự án.

## 3. 🧱 Chức Năng Chính

### A. Phân Tích Thị Trường

- Thu thập dữ liệu thời gian thực.
- Báo cáo tùy chỉnh: Tạo biểu đồ/bản đồ (Matplotlib/Google Maps API) về thị trường, so sánh khu vực, tích hợp hạ tầng/quy hoạch.
- Cảnh báo thay đổi: Push notification/email về biến động giá, chính sách (Luật Đất đai).

### B. Tư Vấn Cá Nhân Hóa

- Thu thập dữ liệu người dùng: Qua chat tự nhiên, lưu ngân sách/mục đích/sở thích (xóa dữ liệu tùy chọn cho bảo mật).
- Gợi ý cá nhân hóa: AI recommendation (collaborative/content-based) đề xuất BDS phù hợp, e.g., "Dựa ngân sách 2 tỷ, gợi ý quận 7 gần trường".
- Hồ sơ lâu dài: Cải thiện gợi ý từ lịch sử, gửi cảnh báo cơ hội mới.
- Hỗ trợ đa ngôn ngữ/ngữ cảnh: Điều chỉnh văn hóa (phong thủy), ngôn ngữ (VN/EN/CN/KR).
- Kết nối môi giới

### C. So Sánh & Đánh Giá Tài Sản

- Công cụ so sánh: So theo giá/m²/vị trí/tiện ích, với 3-5 tài sản tương tự trong bán kính.
- Đánh giá khách quan: Điểm số 1-10 từ lịch sử giá/chi phí/quy hoạch (từ GIS data).

### D. Thông Tin Pháp Lý

- Hướng dẫn quy trình: Checklist mua/bán, timeline.
- Tra cứu luật: DB luật BDS, tóm tắt giấy tờ.
- Mẫu tài liệu: Mẫu hợp đồng, kiểm tra sổ đỏ (qua PDF tools).

### G. Thông Tin Địa Phương

- Bản đồ: Google Maps API hiển thị tiện ích, thời gian di chuyển.
- Dữ liệu cộng đồng: Đánh giá từ X user search/web search.
- Phân tích môi trường: Ngập lụt/quy hoạch (GIS từ Mendeley).
- Cập nhật realtime: Tin hạ tầng từ web search.

### H. Tương Tác & Giao Diện

- Giao tiếp tự nhiên: NLP hiểu câu hỏi, trả lời như tư vấn viên.
- Đa kênh: Chat/voice, bảng/hình ảnh/báo cáo.
- Hướng dẫn: Gợi ý câu hỏi, chế độ chi tiết/tóm tắt.

### I. Phân Tích Hình Ảnh & Tài Liệu

- Nhận diện ảnh: Tình trạng BDS (view_image tools).
- Xử lý tài liệu: PDF hợp đồng/quy hoạch (search/browse pdf attachment).
- Đánh giá: Nhận xét giá trị từ ảnh/video (view_x_video).
- Video tour: Phân tích từ X/nguồn khác.

## 4. ⚙️ Yêu Cầu Hệ Thống

### Giao Diện Người Dùng

- Hỗ trợ desktop/tablet/mobile, dark/light mode, WCAG.
- Đa ngôn ngữ (VN/EN/CN/KR).

### Backend & API

- RESTful/GraphQL, API phân tích realtime.
- Tích hợp BDS/quy hoạch/ngân hàng/Bộ Xây dựng.
- Push notification: Biến động/thông báo.
- Phân quyền: Cá nhân/môi giới/doanh nghiệp/admin.

### Dữ Liệu & Bảo Mật

- PostgreSQL chính, Redis caching.
- HTTPS, JWT/OAuth2.
- Bảo mật GDPR/Luật VN, xóa dữ liệu.
- DB: BDS/người dùng/giao dịch/pháp lý/quy hoạch/lịch sử.

### Hiệu Năng & Mở Rộng

- 10.000 users đồng thời, phản hồi <2s.
- Microservices: Phân tích/tài chính/pháp lý/AI.
- Docker, CI/CD.

## 5. 🧪 Tính Năng Mở Rộng Tương Lai

- AR/VR tour ảo (ví dụ Realsee/FIDOVN, House3D)
- Big Data dự báo xu hướng.
- Phân tích tác động chính sách.

## 6. 📚 Tài Liệu Yêu Cầu

- BRD (Business Requirement Document).
- SRS (Software Requirement Specification).
- DB Diagram & API Spec.
- User Manual: Người dùng/Môi giới/Doanh nghiệp/Quản trị viên.

## 7. 🧑‍🤝‍🧑 Các Bên Liên Quan

- Người dùng cuối: Mua/thuê/đầu tư.
- Môi giới/sàn: Cung cấp/quản lý info.
- Doanh nghiệp: Chủ đầu tư/môi giới lớn.
- Admin: Kiểm duyệt/quản lý data.
- Cơ quan quản lý: Bộ Xây dựng/địa phương.
- DevOps/kỹ sư: Triển khai/bảo trì.

## Flowchart Luồng Chạy (RAG End-to-End Cho BDS)

**Mô tả tổng quát**: Flowchart RAG end-to-end cho trả lời query BDS. Bắt đầu "User Question", kết thúc "Answer" với kiểm tra/tối ưu. Chia Input/Retrieval/Generation/Verification/Ops.

<p align="center">
  <img src="/documents/images/mermaid-diagram.svg" alt="Flowchart RAG End-to-End" width="800"/>
</p>
**Cấu trúc chi tiết**:

- **Input**: "User Question" → "QA/FastAPI /ask" → "Query Understanding" (chuẩn hóa, extract entity).
- **Retrieval**: "Hybrid Retrieval" (BM25 + Embeddings) từ OpenSearch/Vector Store → "Top K" → "Rerank" → "Diversity" → "Availability Filter" → "Context Builder".
- **Generation**: "Fine-tune LLM RAG Prompt".
- **Verification**: Check "Citations/availability" với PostgresQL. OK: "Answer + Links + Confidence". Mismatch: "Auto-correct".
- **Ops**: Nightly refresh, Evaluation (Recall/Lead Quality), Logs, Monitoring, Ingest.

**Công nghệ**: FastAPI, Hybrid Retrieval (Sentence Transformers), Qdrant, PostgresQL, Vietcuna/PhoGPT, Cron/ELK.

## Phân Tích Flowchart Hình Ảnh

- **Tổng Quan**: 4 giai đoạn từ data BDS đến chatbot.
- **Chuẩn Bị Data**: Scrape listings, QA personalization, clean/chunk.
- **Xây Dựng RAG**: Eval embedding, pipeline hybrid/re-ranking, embed Qdrant.
- **Training Model**: Eval base/finetuned, train SFT/LoRA, RLHF từ sales.

**Tổng Kết**: Linh hoạt, review hàng tháng. Rủi ro: Data biến động – realtime API. Output: Tăng lead 30-50% (FPT.AI/Omicall).
