## 📚 Chức năng 4.4 – Quản lý Tri thức (Knowledge Base Management)

### 🎯 Mục tiêu & phạm vi
- Tập trung quản trị nguồn tri thức cho **RAG**: tài liệu pháp lý, hướng dẫn, FAQ, hồ sơ dự án, mô tả sản phẩm.
- Bảo đảm: **đầy đủ meta**, **phiên bản hoá**, **staging** (draft/review/published), **track usage**, **rollback** nhanh.
- Kết nối chặt với **3.2 (RAG)**, **4.3 (Flagged → KB fix)**, **4.2 (Dashboard chất lượng)**.

---

### 🧭 Vòng đời tài liệu (Lifecycle)
```
Upload → Parse → Chunk → Enrich Meta → Validate → Draft
              ↓
            Review (QA/Legal/SME)
              ↓ approve / request-change
         Published (RAG-visible)
              ↓
    Monitor usage → Update/Depublish → Archive
```
- **Effective dates:** `effective_from`, `effective_to` (bắt buộc với tài liệu pháp lý).
- **Scheduled publish:** tự động lên `Published` vào giờ đã định.

---

### 🧱 Lược đồ dữ liệu (tối thiểu)
```sql
kb_docs(
  id PK, title, source_url, type, lang, project?, jurisdiction?,
  year?, effective_from?, effective_to?,
  status ENUM('draft','review','published','archived'),
  version INT, checksum, uploader, reviewer?, published_by?,
  created_at, updated_at, published_at
)

kb_chunks(
  id PK, doc_id FK, chunk_idx, text, lang, page?, heading_path?,
  meta_json, vector_ptr, created_at
)

kb_jobs(
  id PK, doc_id FK, job_type ENUM('ingest','reindex','delete'),
  status ENUM('queued','running','failed','done'),
  log TEXT, started_at, finished_at
)

kb_usage_daily(
  dt DATE, doc_id, hits INT, cited INT, avg_rank FLOAT, last_query_ts TIMESTAMP
)
```
- **type gợi ý:** `law`, `guideline`, `project_info`, `faq`, `marketing`, `contract_template`.

---

### 🔄 Pipeline Ingest (tự động hóa)
**Upload**
- Hỗ trợ: **PDF, DOCX, HTML, Markdown, URL crawl** (whitelist domain).
- Lưu **checksum (SHA-256)** để chống trùng.

**Parse**
- PDF: tách text + giữ heading, mục/điều; **OCR fallback** (nếu scan).
- DOCX/HTML/MD: làm sạch, chuẩn hoá spacing, bảng (giữ nội dung text).

**Chunking (VN-friendly)**
- **Pháp lý:** chunk theo **Điều / Khoản / Mục** (ưu tiên cấu trúc).
- **Tài liệu thường:** 300–800 tokens, overlap 10–20%, giữ `heading_path`.
- Gắn `page` khi có (để deep-link `#page=45`).

**Enrich meta**
- Tự động suy luận `year`, `jurisdiction`, `project` từ tiêu đề/đường dẫn.
- Chuẩn hoá `lang` (vi-VN / en-US), đơn vị (m², VND).

**Validation (bắt buộc)**
- `title` ≥ 5 ký tự, `type`, `lang`.
- Nếu `type=law` → **phải có** `jurisdiction`, `effective_from`.
- Cảnh báo nếu `source_url` trống hoặc link chết.

**Index**
- **BM25 (OpenSearch)** + **Vector (Qdrant)** → ghi `vector_ptr`.
- Re-rank model cấu hình ở **3.2**.

**Staging**
- `status='draft'`. Yêu cầu **review (QA/Legal)**.
- **Review checklist:** tiêu đề rõ, meta đúng, trích dẫn hợp lệ, bản mới hơn?

---

### 🧩 Quy tắc xuất bản (Publishing)
- Chỉ `published` mới đi vào **RAG Router**.
- Cho phép **Scheduled publish** & **Depublish** (khi hết hiệu lực).

**Versioning**
- Mỗi publish **tăng version**.
- **Diff viewer:** so sánh vN ↔ vN-1 (highlight đoạn thay đổi).

**Rollback**
- Depublish phiên bản mới, **promote** lại phiên bản trước (1-click).

---

### 🔌 API Admin (chuẩn hoá)
```
POST   /admin/kb/docs                          # upload meta + file/url
GET    /admin/kb/docs?status&type&lang&project&from&to&page
GET    /admin/kb/docs/{id}                     # chi tiết + versions + usage
POST   /admin/kb/docs/{id}/parse               # parse lại (force)
POST   /admin/kb/docs/{id}/chunk               # re-chunk
POST   /admin/kb/docs/{id}/reindex             # re-embed + reindex
POST   /admin/kb/docs/{id}/submit-review       # chuyển 'review'
POST   /admin/kb/docs/{id}/approve             # reviewer duyệt
POST   /admin/kb/docs/{id}/publish             # lên 'published' (+schedule?)
POST   /admin/kb/docs/{id}/depublish           # hạ khỏi RAG
DELETE /admin/kb/docs/{id}                     # soft delete + reindex
GET    /admin/kb/usage?doc_id&from&to          # theo dõi hit/citation
```
**Webhooks (tùy chọn)**
- `kb.document.published`, `kb.document.depublished`, `kb.reindex.failed`

---

### 🖥️ UI (Console – vai trò Admin/QA/Legal)
- **Library View:** bộ lọc `status/type/lang/project/jurisdiction/effective`.
- **Upload Panel:** kéo-thả file / nhập URL (crawl một trang).
- **Doc Detail**
  - Preview nội dung + **chunk list** (có heading/page).
  - **Meta editor** (selectbox/bộ từ điển có kiểm soát).
  - **Version & Diff** tab.
  - **Usage:** hits, cited, avg_rank, top queries dẫn đến doc.
- **Bulk actions:** publish hàng loạt, reindex theo project/type/lang.

---

### 🔒 RBAC & Tuân thủ
**Vai trò:**
- `uploader`: upload, chỉnh meta, gửi review.
- `reviewer (QA/Legal)`: approve/reject, comment.
- `publisher/admin`: publish/depublish, rollback, xoá mềm.

**Audit log:** mọi thay đổi trạng thái/meta/version.  
**PII:** không cho phép upload tài liệu lộ PII nếu chưa được duyệt.  
**Chữ ký nguồn:** lưu `source_url`/`file_proof` để truy xuất nguồn gốc.

---

### 🌐 Đa ngôn ngữ & Freshness
- **Tài liệu song ngữ:** lưu **2 doc** riêng, liên kết `doc_group_id`.
- **RAG ưu tiên** doc cùng ngôn ngữ người dùng (2.5); nếu không có → cho phép cross-lingual + dịch kết luận.

**Freshness policy**
- Với pháp lý: cảnh báo “có bản mới hơn?” khi upload.
- Gắn `effective_to` để **auto depublish** khi hết hiệu lực.

---

### 🔎 Chất lượng & Quan sát (nối 4.2)
**Usage analytics:** `hits` (được truy hồi), `cited` (thực sự trích dẫn), `avg_rank`.
- **Dead chunk:** chunk **không bao giờ** được gọi → cân nhắc gộp/xoá.

**KB Impact:** sau publish, theo dõi:
- `fallback_rate` (intent liên quan) ↓
- `flagged_rate` (4.3) ↓
- `citation_coverage` ↑
- **Index latency:** từ upload → published → searchable (p95 < 2 phút).

---

### 🧪 Auto từ 4.3 (Flagged → KB)
- Khi flag `factual_error|hallucination`:
  - Gợi ý **top 5 doc** liên quan (3.2).
  - **Open in KB** → tạo **draft** từ nội dung QA biên soạn, gắn link flag.
- Khi publish:
  - Hệ thống tự ghi `flag_links.kb_doc_id` & đánh dấu **Verified** nếu **eval (1.5)** pass.

---

### 🛠️ Pseudo triển khai
**Chuẩn hoá & validate meta**
```ts
function normalizeMeta(meta: any): NormalizedMeta {
  return {
    title: meta.title?.trim(),
    type: assertOneOf(meta.type, ["law","guideline","project_info","faq","marketing","contract_template"]),
    lang: detectLang(meta.text) ?? meta.lang ?? "vi-VN",
    jurisdiction: meta.type==="law" ? meta.jurisdiction : undefined,
    effective_from: meta.type==="law" ? parseDate(meta.effective_from) : undefined,
    project: canonicalProject(meta.project),
    year: inferYear(meta),
  };
}
```

**Reindex task**
```ts
async function reindexDoc(docId: string) {
  const chunks = await db.kb_chunks.findMany({ where: { doc_id: docId }});
  for (const c of chunks) {
    const v = await embed(c.text);             // multilingual embeddings
    await qdrant.upsert({ id: c.id, vector: v, payload: { doc_id: docId, lang: c.lang, type: c.meta.type }});
    await opensearch.index({ id: c.id, body: { text: c.text, title: c.meta.title, doc_id: docId, lang: c.lang }});
  }
}
```

**Scheduler depublish khi hết hiệu lực**
```ts
cron("0 2 * * *", async () => {
  const docs = await db.kb_docs.findMany({ where: { status: "published", effective_to: { lt: now() }}});
  for (const d of docs) await depublish(d.id, "expired");
});
```

---

### 🧯 Lỗi & Fallback
- **Parse fail:** hiển thị log chi tiết, gợi ý OCR hoặc upload bản khác.
- **Crawl URL lỗi:** lưu **snapshot HTML** (để audit), đánh dấu “nguồn không ổn định”.
- **Embedding error:** retry + queue; nếu tiếp tục fail → publish **keyword-only** tạm thời (BM25) và cảnh báo.
- **Dead link:** job định kỳ check `source_url` → gắn cờ “link hỏng”.

---

### 📈 KPI cho 4.4
- **Citation coverage:** % câu trả lời có ≥1 trích dẫn hợp lệ (≥ **95%**).
- **Index latency p95:** < **2 phút**.
- **Dead-chunk rate:** < **20%**.
- **Freshness** (pháp lý còn hiệu lực): ≥ **98%**.
- **Post-publish impact:** giảm ≥ **30%** flagged quanh chủ đề trong **7–14 ngày**.

---

### ✅ Checklist MVP 4.4
- [x] Upload/Parse/Chunk/Embed/Index pipeline + UI  
- [x] Meta validator (type/lang/effective/jurisdiction)  
- [x] Staging draft → review → published, versioning + diff + rollback  
- [x] API reindex, depublish, schedule publish, usage analytics  
- [x] RBAC (uploader/reviewer/publisher/admin) + audit log  
- [x] Dashboard KB usage + dead-chunk + impact sau publish  
- [x] Job kiểm tra **dead link** + **auto depublish** khi hết hiệu lực  
- [x] Kết nối **4.3 (Flagged → KB fix)** & **3.2 (RAG)**

