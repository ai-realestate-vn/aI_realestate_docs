## 📚 Chức năng 3.2 – RAG / Search Engine Integration

### 🎯 Mục tiêu & Phạm vi
**Mục tiêu:** Truy hồi chính xác đoạn liên quan + xếp hạng + tổng hợp câu trả lời grounded với citation.  
**Nguồn:** PDF/Docx, HTML, DB nội bộ, API công khai; cả listing lẫn pháp lý.  
**Ràng buộc:** Latency p95 ≤ 1200–1800 ms (truy hồi + rerank), câu trả lời có ≥1 nguồn hợp lệ.

---

### 🧱 Kiến trúc tổng quan
```
[NLU/Policy 1.1/1.3] ─► RAG.Router
                         ├─► Keyword Search (BM25/OpenSearch)
                         ├─► Vector Search (Qdrant/FAISS)
                         └─► Hybrid (BM25 ∪ Vector) + Reranker (cross-encoder)
                               │
                               ▼
                          Answer Composer (LLM+Templates)
                         (citation, quote-span, safety filters)
                               │
                               ▼
                          Cards 2.4 + NLG 1.2
```

---

### 📦 Data Contracts
**Doc Chunk (index)**
```json
{
  "doc_id":"law-2014-§121",
  "chunk_id":"law-2014-§121:7",
  "title":"Luật Nhà ở 2014 - Điều 121",
  "text":"Điều 121. Nội dung của hợp đồng về nhà ở...",
  "lang":"vi",
  "meta":{
    "type":"law", "jurisdiction":"VN", "year":2014,
    "url":"https://.../law2014.pdf#page=45",
    "page":45, "project": null, "effective_date":"2015-07-01"
  },
  "vector":[/* embedding */]
}
```

**RAG Query (từ Orchestrator)**
```json
{
  "query_text":"quy định đặt cọc khi mua căn hộ hình thành trong tương lai",
  "locale":"vi-VN",
  "filters":{"type":["law","guideline"],"year_gte":2014},
  "top_k":8,
  "mode":"hybrid"
}
```

**RAG Result**
```json
{
  "hits":[
    {"chunk_id":"law-2014-§121:7","score":0.83,"meta":{"url":"...#p=45","title":"Luật Nhà ở 2014 - Điều 121"}},
    {"chunk_id":"decree-99-2015:12","score":0.79,"meta":{"url":"...","title":"Nghị định 99/2015"}}
  ],
  "rerank_model":"bge-reranker-v2-m3",
  "used_filters":{"type":["law","guideline"],"year_gte":2014}
}
```

**Answer Payload (đưa ra NLG/2.4)**
```json
{
  "answer":"Theo Luật Nhà ở 2014, hợp đồng mua bán phải có ...",
  "citations":[
    {"title":"Luật Nhà ở 2014 - Điều 121","url":"...#p=45","quote":"Nội dung của hợp đồng ..."},
    {"title":"Nghị định 99/2015","url":"...","quote":"... quy định chi tiết ..."}
  ],
  "confidence":0.78,
  "grounded":true,
  "safety_flags":[]
}
```

---

### 🧭 Pipeline Indexing (Offline)
- **Ingest:** tải PDF/Docx/HTML/API → lưu raw + checksum.  
- **Parse:** PDF (giữ heading/bảng), HTML bỏ nav.  
- **Chunking (Vi-friendly):** theo mục/điều/khoản (law-aware), hoặc semantic 300–800 token + overlap 10–20%; bảo toàn anchor (doc_id, page, heading path).  
- **Normalization:** làm sạch dấu/khoảng trắng, quy đổi đơn vị.  
- **Embedding:** multilingual tốt cho VN (multilingual-e5, mGTE).  
- **Metadata:** type, year, jurisdiction, project, effective_date…  
- **Upsert:** Qdrant (vector+payload) + OpenSearch (BM25).  
- **QC:** orphan pages, độ phủ heading, duplicate-chunk (minhash).  
- **Listing:** đồng bộ từ DB 3.1 → chunk “facts” (mô tả, tiện ích) + metadata listing_id/project.

---

### 🔎 Retrieval (Online)
**Router**
- `mode="hybrid"` mặc định; chỉ BM25 khi câu rất ngắn & pháp lý chuẩn; chỉ vector khi mô tả mơ hồ.  
- Lang routing 2.5: query en → pivot sang vi nếu corpus chủ yếu vi.

**Candidate Generation**
- BM25 `top_k=50`, Vector `top_k=50` (cosine/IP).  
- **Filters (pre):** type, year, project, lang.  
- Hợp nhất ∪, loại trùng bằng `chunk_id`.

**Reranking (cross-encoder)**
- Model multilingual (ví dụ *bge-reranker v2 m3*).  
- Chọn `top_k_final=8–12`.  
- **MMR** để đa dạng hoá.

**Answer Composer**
- Template-first: "Kết luận ngắn → trích dẫn → lưu ý".  
- LLM gated: **chỉ** dùng passages truy hồi; cấm thông tin ngoài.  
- Trả citations kèm **quote-span** (1–2 câu) + URL/page.  
- **Caching:** theo hash(query+filters) 60–120s; cold-start partial.

---

### 🛡️ Guardrails & Chính sách
- **Grounding-only:** ràng buộc không bịa; thiếu căn cứ → nêu rõ.  
- **Citation Required** với `type in [law, policy]`.  
- **Freshness:** ưu tiên `effective_date` mới; cảnh báo nếu hết hiệu lực/dự thảo.  
- **Conflict:** nguồn mâu thuẫn → hiển thị cả hai + ngày hiệu lực.  
- **PII:** khi trích hồ sơ nội bộ → ẩn theo vai trò.

---

### 🧪 Đánh giá chất lượng
- **Retrieval:** nDCG@10, Recall@20.  
- **Rerank:** MRR@10, Hit@1.  
- **Answer:** Faithfulness/Groundedness ≥ 4.3/5; Citation coverage ≥ 98%; Hallucination < 1%.  
- **Latency:** p95 ≤ 1.8s (Hybrid).

---

### 🔌 Tích hợp với 1.x & 2.x
- **1.3 Policy** chọn `RAG_LOOKUP` cho intent: `hoi_chinh_sach`, `faq_du_an`, `hoi_ho_so`.  
- **1.2 NLG** nhận `answer+citations` → sinh text + thẻ nguồn (2.4).  
- **2.3 Quick replies**: "Mở điều khoản liên quan", "Xem toàn văn PDF".  
- **1.5 Learning log**: retrieval_hit, rerank_score, feedback 👍/👎 theo nguồn.

---

### 🛠️ Pseudo Code (TypeScript/Python)
**TypeScript — query RAG**
```ts
export async function ragAnswer(q: RAGQuery): Promise<RAGOut> {
  const qNorm = normalizeQuery(q);
  const bm25 = await os.search(qNorm.text, q.filters, 50);
  const vec = await qdrant.search(emb(qNorm.text), q.filters, 50);

  const merged = dedupeById([...bm25, ...vec]);
  const reranked = await crossEncodeRank(qNorm.text, merged).then(topK(10));
  const diversified = mmr(reranked, 0.3);

  const composerInput = diversified.map(x => ({ text: x.text, source: x.meta }));
  const { answer, citations, grounded } = await composeAnswer(composerInput, q.locale);
  return { answer, citations, grounded, confidence: score(diversified) };
}
```

**Python — compose answer (guarded)**
```python
def compose_answer(passages, locale="vi-VN"):
    sys_prompt = """Bạn chỉ được trả lời dựa trên 'PASSAGES' cung cấp.
    Nếu thiếu căn cứ, trả lời: 'Chưa đủ căn cứ để kết luận.' và gợi ý nguồn cần xem.
    Trả về trích dẫn (title+url+quote). Ngắn gọn, chính xác, không suy diễn."""
    content = format_passages(passages)
    out = llm.generate(sys_prompt, content, locale=locale, max_tokens=300)
    answer, cites = postcheck_and_extract_citations(out, passages)
    grounded = verify_quotes_in_passages(answer, cites, passages)
    return {"answer": answer, "citations": cites, "grounded": grounded}
```

---

### 🖥️ Cards 2.4 cho RAG (nguồn pháp lý)
```json
{
  "id":"cite-law-2014-121",
  "title":"Luật Nhà ở 2014 – Điều 121",
  "subtitle":"Nội dung của hợp đồng về nhà ở",
  "actions":[
    {"type":"open_url","label":"Mở PDF tại trang 45","url":"...#page=45"},
    {"type":"postback","label":"Xem thêm điều liên quan","payload":{"action":"rag_more","topic":"Hợp đồng mua bán"}}
  ]
}
```

---

### 🧯 Lỗi & Fallback
- **No-hit:** "Chưa tìm thấy nội dung phù hợp." → gợi ý mở rộng từ khoá / loại tài liệu khác.  
- **Upstream down:** dùng snapshot cache gần nhất + cảnh báo “dữ liệu có thể cũ”.  
- **PDF parse lỗi:** lưu ảnh trang + OCR fallback (tesseract/vietOCR) → đánh dấu độ tin cậy thấp.

---

### ✅ Checklist MVP 3.2
- [x] Ingest + Parser PDF/Docx/HTML, giữ doc_id/page/heading  
- [x] Chunking (300–800 tok, overlap 10–20%), law-aware  
- [x] Embeddings đa ngôn ngữ (ưu tiên VN) + Index Vector + BM25  
- [x] Hybrid retrieval + cross-encoder rerank + MMR  
- [x] Answer composer grounded-only + citations (quote + URL/page)  
- [x] Filters: type, year, lang, project + freshness  
- [x] Cache query & negative cache; latency p95 ≤ 1.8s  
- [x] Dashboard: nDCG/MRR/Recall, Hallucination rate, Citation coverage  
- [x] Fallbacks: snapshot cache, OCR, no-result suggestions

