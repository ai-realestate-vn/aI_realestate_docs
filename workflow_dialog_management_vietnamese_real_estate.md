## 💬 Workflow Chức năng 1.2 – Dialog Management (Quản lý hội thoại)

### 🎯 Mục tiêu & I/O
**Mục tiêu:** Biến `{intent, slots, context, knowledge}` thành phản hồi có cấu trúc (text + rich cards + gợi ý hành động) theo giọng điệu nhất quán.  
**Input:** `nlu_result (intent/slots/confidence)` · `context (lịch sử, persona)` · `kb_hits (kết quả RAG)` · `policy (quy tắc ngành)`.  
**Output:**
```json
{
  "messages": [
    {"type": "text", "content": "Em thấy 3 căn phù hợp ở Quận 7 dưới 3 tỷ. Anh/chị muốn xem nhanh không?"}
  ],
  "cards": [
    {
      "title": "Sunrise Riverside - 2PN - 2.85 tỷ",
      "subtitle": "70 m² • Quận 7 • Hướng Đông",
      "image": "https://.../sunrise.jpg",
      "actions": [
        {"type": "open_url", "label": "Xem chi tiết", "url": "https://..."},
        {"type": "postback", "label": "Đặt lịch xem", "payload": {"action":"book_visit","id":"SR-2PN-2850"}}
      ],
      "badges": ["Sổ hồng", "Full nội thất"]
    }
  ],
  "quick_replies": ["Lọc theo 2PN", "Tăng ngân sách", "Xem lịch cuối tuần"],
  "metadata": {"tone":"polite-pro", "grounded": true, "source_ids": ["kb:duan:SR", "db:listing:123"]}
}
```

---

### 🚦 Luồng runtime (NLG Pipeline)

#### **1. Response Planning (kế hoạch phản hồi)**
- Chọn kiểu hồi đáp theo intent & slots:
  - `tim_bds` → liệt kê rút gọn + đề xuất lọc thêm.
  - `dat_lich_xem` → xác nhận thời gian + CTA “Xác nhận lịch”.
  - `khong_hieu` / `clarify` → hỏi 1–2 câu rõ ràng, ngắn gọn.
- Xác định mục tiêu bước kế tiếp (Next Best Action): gợi ý lọc, đặt lịch, hoặc xin thêm thông tin.

#### **2. Content Selection (chọn nội dung)**
- Ưu tiên dữ liệu grounded: kết quả từ DB/RAG > suy đoán.
- Tránh hallucination (không bịa dự án/giá). Nếu thiếu → nêu rõ và gợi ý hỏi thêm.

#### **3. Micro-planning (bố cục câu)**
- Cấu trúc: **Kết luận chính → Chi tiết → Lời mời hành động.**
- Giới hạn 2–3 câu; tối đa 3 item trong danh sách.

#### **4. Surface Realization (tạo câu)**
- Dùng template trước; LLM chỉ để làm tự nhiên hơn khi cần.
- Chuẩn hoá đơn vị (m², tỷ/triệu VND); định dạng tiêu đề: “Tên – PN – Giá”.

#### **5. Tone & Persona**
- Giọng “polite-pro”: lịch sự, chuyên nghiệp, không quảng cáo quá mức.
- Tùy biến theo ngữ cảnh người dùng (`anh/chị`, persona đã lưu).

#### **6. Safety & Policy Guardrails**
- Không hứa hẹn pháp lý/lợi nhuận.
- Nếu thông tin nhạy cảm → thêm cảnh báo hoặc điều kiện.
- Câu hỏi ngoài phạm vi → từ chối khéo + hướng dẫn lại.

#### **7. Multilingual & Style**
- Nếu `language=en` → chuyển ngữ, giữ số & đơn vị.
- Hạn chế viết tắt, chuẩn hoá dấu tiếng Việt.

#### **8. Packaging (đóng gói UI)**
- Sinh `text`, `cards`, `quick_replies`, `actions`.
- Thêm `metadata.source_ids` để truy vết nguồn.

#### **9. Post-check (QA tự động)**
- Kiểm tra: dữ liệu trống, giá vô lý, link lỗi.
- Nếu `kb_hits` rỗng & intent cần dữ liệu → chuyển sang clarify.

---

### 🧱 Thư viện Templates (mẫu câu – rút gọn)

#### **A) Kết quả tìm BĐS (có item)**
```
[Opening]
Em tìm thấy {count} căn phù hợp {khu_vuc?} {gia?}.

[List ≤3]
• {ten_du_an} – {pn}PN – {gia} – {dien_tich} m²
• {ten_du_an_2} – ...

[CTA]
Anh/chị muốn xem chi tiết hay đặt lịch tham quan?
```

#### **B) Không có kết quả**
```
Hiện chưa có căn nào đúng {bộ lọc}.
Anh/chị muốn nới ngân sách lên khoảng {gia_goi_y} hoặc đổi khu vực gần {khu_vuc_lan_can} không?
```

#### **C) Thiếu slot quan trọng (clarify)**
```
Để em lọc chính xác hơn, anh/chị cho em biết ngân sách tối đa khoảng bao nhiêu ạ?
```

#### **D) Xác nhận đặt lịch**
```
Em đã đặt lịch tham quan {du_an} vào {thoi_gian}.
Anh/chị kiểm tra giúp em thông tin liên hệ {phone/email} nhé.
```

#### **E) Câu trả lời từ tài liệu (RAG)**
```
Theo [Nguồn: {source_name}], {kết_luận_ngắn_gọn}.
Nếu cần, em gửi toàn văn điều khoản liên quan.
```

---

### 🧪 Ví dụ đầu vào → đầu ra

**Input (từ NLU):**
```json
{
  "intent": "tim_bds",
  "slots": {"khu_vuc": "quận 7", "ngan_sach_max": 3000000000, "so_phong": 2, "loai_bds":"chung cư"},
  "confidence": 0.82
}
```

**Output (NLG):**
```json
{
  "messages": [
    {"type": "text", "content": "Em thấy 3 căn chung cư 2PN ở Quận 7 trong tầm dưới 3 tỷ."}
  ],
  "cards": [
    {
      "title": "Sunrise Riverside - 2PN - 2.85 tỷ",
      "subtitle": "70 m² • Quận 7 • Hướng Đông",
      "image": "https://.../sr.jpg",
      "actions": [
        {"type":"postback","label":"Đặt lịch xem","payload":{"action":"book_visit","id":"SR-2850"}},
        {"type":"open_url","label":"Xem chi tiết","url":"https://.../SR-2850"}
      ],
      "badges": ["Sổ hồng", "Ban công thoáng"]
    },
    {"title": "Eco Green - 2PN - 2.75 tỷ", "subtitle": "68 m² • Quận 7", "actions": [...]}
  ],
  "quick_replies": ["Lọc 2PN + ban công", "Tăng ngân sách", "Xem lịch cuối tuần"],
  "metadata": {"tone":"polite-pro", "grounded": true, "source_ids":["db:listing:..."]}
}
```

---

### 🧩 Kiến trúc triển khai gợi ý

**Layer 1 — Planner:** chọn chiến lược (`RESULT_LIST`, `CLARIFY`, `NO_RESULT`).  
**Layer 2 — Realizer:** điền biến vào template + tự nhiên hoá (LLM khi cần).  
**Layer 3 — UI Packager:** đóng gói kết quả cho Web, Zalo OA, Messenger…

```typescript
const plan = planResponse(nlu, kb, context);
const tpl  = pickTemplate(plan, locale="vi-VN");
const text = fillTemplate(tpl.text, variables);
const cards = buildCards(results.slice(0,3));
const quick = buildQuickReplies(plan, nlu.missing_slots);
return { messages:[{type:"text", content:text}], cards, quick_replies:quick, metadata };
```

---

### 🔒 Guardrails (quan trọng cho BĐS)
- **Không bịa:** nếu thiếu dữ liệu → nói rõ “hiện chưa có…”.
- **Không khẳng định pháp lý:** dùng “theo hồ sơ/nguồn A…”.
- **Giá/diện tích:** hiển thị khoảng hoặc nguồn, kèm cảnh báo “giá có thể thay đổi”.
- **Nhạy cảm:** không tiết lộ thông tin chủ sở hữu, số nhà nếu bị cấm.

---

### 📈 Đo lường chất lượng NLG
- **Fluency:** Điểm trung bình ≥ 4.2/5.
- **Task success:** tỷ lệ click CTA hoặc đặt lịch tăng rõ rệt.
- **Grounding score:** ≥ 98% câu có nguồn hợp lệ.
- **Avg length:** 1–3 câu, tránh “wall-of-text”.

