# 6.4 – Tự động tóm tắt (Summarization)

> **Mục tiêu:** tạo tóm tắt ngắn – chính xác – có căn cứ nguồn cho hội thoại, tài liệu luật/BĐS, và nội dung vận hành (log, ticket). Tối ưu tiếng Việt, gắn chặt với 1.x, 5.x, 6.1–6.3.

---

## 🎯 Mục tiêu & I/O

### Mục tiêu
Rút gọn thông tin để:
- **Tổng kết phiên chat** (episodic/turn summary)
- **Tóm tắt tài liệu pháp lý/điều khoản** (doc summary)
- **Tạo ghi chú nghiệp vụ** (actionable summary)
- **Tạo snippet ngắn** (card preview trong UI)

### Input / Output
- **Input:** văn bản (chat log, PDF/DOCX sau OCR, trang web, chunk từ 6.1), policy/role, context (tenant, khu vực, hiệu lực).
- **Output:** `{title?, bullets[], key_values?, citations?, next_actions?, risks?}` theo preset (`short`/`medium`/`long`).

---

## 🧱 Kiến trúc tổng quan
1. **Ingest nội dung** → chuẩn hoá (Unicode, dấu tiếng Việt, đơn vị m²/VND)
2. **Segment/Chunk** theo cấu trúc (Chương/Mục/Điều; heading; đoạn hội thoại)
3. **Select phần quan trọng** (keyword, TF-IDF, embed-sim + trọng số domain)
4. **Summarize (LLM)** – hướng dẫn “factual & grounded” + **citations**
5. **Post-process** (dedupe, chuẩn thuật ngữ, định dạng tiền/ngày)
6. **Guardrails** (không bịa, cảnh báo nếu thiếu nguồn/hết hiệu lực)
7. **Package cho UI** (cards/snippets) và lưu episodic summary vào STM/LTM (6.3) nếu có consent.

---

## 🧩 Kiểu tóm tắt (Presets)
| Preset | Dung lượng | Mục đích |
|---------|-------------|-----------|
| Turn Summary | ≤120 ký tự | Ghi ngắn mỗi lượt để 1.3/6.3 giữ mạch |
| Session Digest | ~3–5 bullet | Kết phiên: yêu cầu, ràng buộc, slot rõ, mục tiêu |
| Doc Executive | 5–7 bullet + cảnh báo | Tóm tắt điều khoản chính & hiệu lực |
| Actionable Note | checklist + người phụ trách | Ghi chú hành động nghiệp vụ |
| Card Snippet | ≤160 ký tự | Mô tả thẻ dự án/listing trong UI |

---

## 🚦 Luồng runtime

### Preprocess
- Chuẩn dấu/Unicode, tách câu, chuẩn hoá số tiền/diện tích.
- Loại bỏ rác (chữ ký email, menu PDF, số trang).

### Segmentation
- **Luật/Nghị định:** Chương → Mục → Điều → Khoản (giữ breadcrumb)
- **Chat:** gom theo tác vụ (intent) hoặc 10–20 lượt/khối.

### Salience Selection
- Từ khoá miền: *chủ sở hữu, sổ hồng, điều kiện, lệ phí…* (BĐS) / *mục tiêu, deadline, blocker…* (tác vụ).
- **Embed-sim + TF-IDF** chọn n đoạn cho tóm tắt.

### Summarization (LLM)
- Prompt “grounded summarization”: chỉ dùng nội dung cung cấp; chèn `[Nguồn: doc_id:section]`.
- Kiểm soát độ dài (max_tokens, ≤N câu/bullets).

### Post-process & Normalize
- Chuẩn tiền (tỷ/triệu), m², ngày (ISO + hiển thị vi-VN).
- Hợp nhất đồng nghĩa (chung cư ↔ căn hộ).

### Guardrail & Validation
- Kiểm tra citation; nếu thiếu → thêm “(Chưa có nguồn trực tiếp)”.
- Phát hiện suy luận ngoài nguồn → đánh dấu hoặc loại.

### Packaging
- Xuất `bullets`, `key_values`, `next_actions`, `risks`.
- Với listing: sinh `Card Snippet + badges`.

---

## 🧠 Prompt khung (vi-VN)

### Grounded Doc Summary
```
Bạn là trợ lý pháp lý BĐS. Tóm tắt 5–7 bullet, ngắn gọn, không bịa.
- Chỉ dùng thông tin trong văn bản.
- Nếu câu trích ý quan trọng, chèn [Nguồn: {doc_id}:{section}].
- Nếu có ngày hiệu lực/hết hiệu lực, nêu rõ.
- Nếu thiếu dữ liệu: ghi "Chưa đủ dữ liệu".
Văn bản:
{chunks}
```

### Session Digest
```
Tạo "Tổng kết phiên":
1) Nhu cầu & ràng buộc chính
2) Điểm đã rõ
3) Việc tiếp theo (checklist)
4) Ngày/cách liên hệ (nếu có)
Trả JSON: {"bullets":[],"next_actions":[]}
Ngữ cảnh: {turn_summaries}
```

### Card Snippet
```
Viết mô tả ≤160 ký tự cho thẻ listing: PN/diện tích/giá/khu vực, không quảng cáo.
Input: {attrs_json}
```

---

## 🧪 Ví dụ
**Input:** Điều kiện cấp sổ hồng chung cư.  
**Output:**
- Chủ đầu tư bàn giao nhà, hoàn thành nghĩa vụ tài chính. [Nguồn: LNO2014:Điều X]  
- Hồ sơ: hợp đồng mua bán, biên bản bàn giao, biên lai... [Nguồn: …]  
- Thời hạn giải quyết: …, lệ phí: … [Nguồn: …]

**Input (Session turns):** “2PN, Q7, dưới 3 tỷ; ưu tiên cuối tuần; né hướng Tây.”  
**Output:**
- Nhu cầu: căn hộ 2PN, Q7, <3 tỷ; tránh hướng Tây.  
- Next: gửi 3 căn phù hợp, đề xuất xem cuối tuần; xác nhận liên hệ.

---

## 📊 Đánh giá chất lượng
- **Factuality:** ≥ 4.5/5  
- **No-hallucination:** ≥ 98%  
- **Coverage:** ≥ 85% (so với gold bullets)  
- **Compression:** 70–85%  
- **Usefulness (CSAT):** ≥ 4.3/5  
- **Latency p95:** Turn ≤150ms, Doc ≤1200ms

---

## 🔒 Bảo mật & Quyền riêng tư
- **5.2:** không ghi PII thô, thay `[PERSON]`, tổng quát địa chỉ.  
- **5.3:** TTL theo purpose `runtime_chat/training`; hỗ trợ DSAR delete.  
- **5.1:** role gating – chỉ người có quyền mới xem citation nội bộ.

---

## ⚙️ Tích hợp module khác
- **1.3 (DM):** dùng Turn Summary để quyết định bước kế tiếp.
- **6.3 (LTM):** lưu Session Digest (không PII) cho personalization.
- **6.1 (RAG):** Doc Executive hiển thị cùng câu trả lời có citation.
- **1.5 (Learning):** log feedback “tóm tắt hữu ích?”.

---

## 🛠️ API (rút gọn)
```
POST /summarize/turn
{ "text": "...", "preset": "turn", "context": {...} }

POST /summarize/session
{ "turn_summaries": [...], "preset": "session_digest", "user_id": "...", "consent": true }

POST /summarize/doc
{ "doc_id": "...", "chunks": [...], "preset": "doc_exec", "need_citations": true, "jurisdiction": "hcm" }
```
**Response:**
```json
{
  "title": "...",
  "bullets": ["..."],
  "citations": [{"doc_id": "...","section": "..."}],
  "next_actions": ["..."],
  "risks": ["..."]
}
```

---

## 📦 Mô hình & Tham số gợi ý
- **LLM:** Qwen2-7B, Llama-3-8B Instruct; model nhỏ 3B cho turn/preview.
- **Rút ngắn:** prompt constraint + max_tokens + validator (≤N bullet/ký tự).
- **Nguồn:** reranking bằng bge-m3 hoặc BM25 trước tóm tắt.

---

## ✅ MVP Checklist (1–2 tuần)
- Presets: turn, session_digest, doc_exec, card_snippet, actionable_note.
- Pipeline: preprocess → select → summarize → validate → package.
- Prompt khung + 10–20 few-shot/preset (VN).
- Citation enforcer + validator (need_citations=true).
- PII scrub (5.2) + TTL (5.3) + role gating (5.1).
- API + unit test: độ dài, citation, factuality.
- Dashboard: factuality, coverage, latency, adoption.

---

## 🧩 Mở rộng (Phase 2)
- **MapView Summary:** tạo tóm tắt đồ hoạ (tiện ích/khoảng cách).
- **Structured Summary:** sinh key_values (Giá, Diện tích, Hướng, Pháp lý…).
- **Contrastive Summary:** so sánh 2–3 dự án, nêu pro/con + fit score.
- **Batch Summarization:** cron tóm tắt tài liệu mới ingest (6.1).