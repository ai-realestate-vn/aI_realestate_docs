# Workflow 6.1 – Kết nối LLM / Vector DB (Qdrant, Pinecone…) cho RAG

**Tên file chuẩn**: `workflow_llm_vector_db_vietnamese_real_estate.md`

## 🎯 Mục tiêu
Biến tài liệu (luật nhà ở, biểu mẫu Sở XD, FAQs, mô tả dự án/listing…) thành tri thức truy xuất để LLM trả lời **đúng nguồn – đúng bối cảnh Việt Nam**, có lọc theo **khu vực/giá/tenant**, độ trễ thấp.

---

## 1) Kiến trúc tổng quan (RAG online)
**Chuỗi xử lý:**
1. **Ingest** → đọc file (PDF/DOCX/HTML/CSV/API listings).
2. **Chuẩn hoá & Chunking** → tách đoạn theo cấu trúc (tiêu đề, điều khoản, mục).
3. **Enrich** → làm sạch tiếng Việt, chuẩn hoá đơn vị (tỷ/triệu/m²), địa danh (Quận/TP Thủ Đức), trích meta (loại tài liệu, thời điểm hiệu lực).
4. **Embedding** → tạo vector (text + *optional*: image caption/plan).
5. **Index** → đẩy vào **Qdrant (khuyến nghị)** với payload lọc.
6. **Query (online)** → **Hybrid search** (dense + BM25) + **Reranker** (*BAAI/bge-reranker-v2-m3*) → lấy top-k.
7. **Grounded Answer** → LLM sinh câu trả lời + **trích dẫn nguồn** + guardrail (không bịa).
8. **Feedback loop (6.1 ↔ 1.5)** → lưu đánh giá, cải thiện bộ lọc/thuật toán.
9. **Lớp Policy/Safety**: che PII (5.2), điều khoản pháp lý (5.3), và phân quyền (5.1) áp dụng ngay ở bước **filter** khi truy vấn.

Sơ đồ rút gọn:
```
[Ingest] -> [Normalize/Chunk] -> [Enrich] -> [Embed] -> [Index: Qdrant]
                                                           |
[Client Query] -> [AuthZ Filter] -> [Hybrid Search] -> [Rerank] -> [LLM Compose]
                                                           |                   
                                                        [Citations/Guardrails]
```

---

## 2) Thiết kế Collection & Payload (Qdrant)
- **Collection**: `kb_realestate_v1`
- **Vectors**: `text_vector` (768/1024 dim, tuỳ model)
- **HNSW**: `m=32`, `ef_construct=256` (MVP); **quantization** (scalar/IVF-PQ) nếu cần giảm RAM.

**Payload (filterable fields khuyến nghị):**
```json
{
  "source_type": "law|guideline|faq|listing|project_profile|form",
  "jurisdiction": "vn|hcm|hn|danang|...",
  "effective_date": "YYYY-MM-DD",
  "tenant_id": "uuid",
  "topics": ["thu_tuc_phap_ly","quy_hoach","gia_ban"],
  "project": "<slug_ten_du_an>",
  "geo": {"lat": 10.73, "lon": 106.70},
  "hash": "<sha256_text_norm>",
  "doc_id": "<id_nguon>",
  "section": "<breadcrumb>",
  "page": 45,
  "url": "https://...#page=45",
  "text": "<raw_chunk_text_for_reranker>"
}
```
> **PII tối thiểu**: không lưu số nhà/căn chính xác trong payload. Nếu bắt buộc, lưu ở **vault** khác và chỉ **join** khi người dùng có quyền.

---

## 3) Chunking tiếng Việt (mẹo thực chiến)
- **Luật/Nghị định**: cắt theo **Chương → Mục → Điều → Khoản → Điểm**; giữ **breadcrumb** để hiển thị nguồn.
- **Biểu mẫu**: mỗi **mục điền** là 1 chunk + ví dụ điền.
- **Listing/Dự án**: gộp **tiêu đề + bullets thuộc tính** (diện tích, PN, hướng, pháp lý) → **≤ 800–1200 ký tự**.
- **Heuristic**: **200–350 token/chunk (MVP)**, **overlap 15–25%**.
- **Chuẩn hoá**: số đo (m²), tiền (VND/USD), alias địa danh (“PMH” → “Phú Mỹ Hưng”).

---

## 4) Embedding & Reranking
- **Embedding (VN)**: `sentence-transformers/paraphrase-multilingual-mpnet-base-v2` (ổn định) hoặc **`bge-m3`** (multi-vector + sparse).
- **Sparse** (BM25/SPLADE): để **hybrid search**; Qdrant hỗ trợ **text + sparse_vector** (tuỳ phiên bản).
- **Reranker**: `BAAI/bge-reranker-v2-m3` (chính xác, nhanh), điểm **cosine/sigmoid** để xếp lại **top-k (k=30 → rerank top-7)**.
> **Gợi ý**: với tài liệu luật/thuật ngữ VN, `bge-m3` + reranker **cải thiện đáng kể**.

---

## 5) Truy vấn (online) – Hybrid + Filter + Geo
- **Filter trước (AuthZ)**: `tenant_id`, `jurisdiction`, `source_type` theo ngữ cảnh.
- **Hybrid scoring** (ví dụ): `score = 0.65 * dense + 0.35 * bm25` → chọn **top-k** → **rerank**.

**Ví dụ filter thực tế**
1. **User HCM** hỏi “thủ tục sang tên căn hộ chung cư”  
   `filter`: `source_type in [law,guideline,form]` **AND** `jurisdiction in [vn,hcm]` **AND** `effective_date <= today`.
2. **Hỏi nhà ở PMH**: “căn 2PN gần Phú Mỹ Hưng”  
   `filter`: `source_type = listing` **AND** `geo within radius 5–8km của PMH` **AND** `attrs.so_phong >= 2`.

---

## 6) Trả lời có trích dẫn (Grounded Answering)
- **Prompt LLM** gồm: câu hỏi đã **chuẩn hoá**, **top-k context** (≤ 3–5 đoạn ngắn), hướng dẫn **trích dẫn**.
- **Yêu cầu**: Luôn ghi `[Nguồn: doc_id/Điều X, Luật Y, ngày...]`.
- **Nếu thiếu context đủ** → **từ chối/đề nghị nguồn thay thế**.
- **Guardrail**: chặn **cam kết lợi nhuận/pháp lý**; câu trả lời dạng: “**Theo Điều … (nguồn)**”.

---

## 7) Độ trễ & tối ưu
- **Ngân sách p95 (web)**: **≤ 1200ms**
  - Vector search: **80–180ms**
  - Rerank (7–10 cặp): **60–120ms** (FP16)
  - LLM answer (short): **500–800ms**
- **Cache**: Query cache (key = `normalized_query+filters`), Passage cache (`doc_id → embedding`).
- **Qdrant tuning**: `ef_search=128–256`, `exact=false`, **quantization** khi RAM căng.
- **Batching**: gom nhiều truy vấn nhỏ; **connection pooling**.

---

## 8) Pipeline Ingest (offline/batch)
1. **Crawler/Importers**: PDF→text, DOCX→markdown, API listings.
2. **Deduplicate**: theo **hash(text_norm)**.
3. **Chunk & Enrich**: add breadcrumbs + meta.
4. **Embed**: GPU/CPU; ghi **parquet** dự phòng.
5. **Upsert Qdrant**: `point_id=uuid`, **vector + payload**.
6. **Quality gates**: phát hiện chunk “rỗng/ngắn/dài quá”, lỗi TV, thiếu meta.
7. **Versioning**: `doc_version`, `ingest_round` để rollback/xoá theo (5.3).

---

## 9) Truy vấn Qdrant – ví dụ (Python, hybrid + rerank)
```python
from qdrant_client import QdrantClient, models
from sentence_transformers import SentenceTransformer
from transformers import AutoModelForSequenceClassification, AutoTokenizer
import torch, numpy as np

qdr = QdrantClient(host="QDRANT_HOST", api_key="...")
emb = SentenceTransformer("sentence-transformers/paraphrase-multilingual-mpnet-base-v2")

tok = AutoTokenizer.from_pretrained("BAAI/bge-reranker-v2-m3")
rer = AutoModelForSequenceClassification.from_pretrained("BAAI/bge-reranker-v2-m3").eval()

def rerank(query, docs):
    pairs = [(query, d["payload"]["text"]) for d in docs]
    batch = tok([p[0] for p in pairs], [p[1] for p in pairs], padding=True, truncation=True, return_tensors="pt")
    with torch.no_grad():
        scores = rer(**batch).logits.squeeze(-1).tolist()
    order = np.argsort(scores)[::-1]
    return [docs[i] | {"rerank_score": scores[i]} for i in order]

def search(query, filters):
    qv = emb.encode([query])[0]
    cond = models.Filter(
        must=[
            models.FieldCondition(key="jurisdiction", match=models.MatchAny(any=filters["jurisdiction"])),
            models.FieldCondition(key="source_type", match=models.MatchAny(any=filters["source_type"]))
        ]
    )
    res = qdr.search(
        collection_name="kb_realestate_v1",
        query_vector=("text_vector", qv),
        limit=30,
        with_payload=True,
        score_threshold=None,
        query_filter=cond,
        search_params=models.SearchParams(hnsw_ef=128, exact=False)
    )
    # Optional: combine with BM25 via qdrant sparse, hoặc merge từ OpenSearch
    top = rerank(query, res)[:7]
    return top
```

---

## 10) Pinecone/Weaviate (thay thế nhanh)
- **Pinecone**: ít công setup, tính phí theo throughput + dung lượng; **payload filter** đơn giản hơn Qdrant nhưng đủ dùng MVP.
- **Weaviate**: schema-full, có **BM25/hybrid** tích hợp, **GraphQL** mạnh; phù hợp khi cần nhiều loại vector & module sẵn.

> Với yêu cầu **lọc nhiều trường (tenant/địa bàn/pháp lý/geo)** và tự chủ **on-prem/edge**, **Qdrant** là lựa chọn **rất hợp**.

---

## 11) Đánh giá chất lượng truy xuất
- **Tập đánh giá**: 200–500 cặp **query → gold passages** từ logs gán nhãn.
- **Chỉ số**: `Recall@k`, `nDCG@k`, `MRR`; theo miền (**luật** vs **listing**).
- **A/B**: weight hybrid (0.5/0.5 vs 0.65/0.35), rerank top-k (10/30), kết hợp bộ lọc địa bàn.
- **Drift monitor**: tỷ lệ **no-support** (LLM phải trả lời không có nguồn) tăng → rà soát ingest.

---

## 12) Bảo mật & Quyền riêng tư
- **Filter theo tenant_id** trước truy vấn (**PEP/PDP** từ 5.1).
- **Không** nhét **PII** vào payload; nếu cần liên hệ, dùng **token/vault**.
- **Retention**: `effective_date`, `doc_version`, `ingest_round` để xóa/quét lại (5.3).
- **PII scrub** trong ingest (5.2) **trước khi embed**.

---

## 13) Checklist MVP (1–2 tuần)
- [ ] Tạo **`kb_realestate_v1`** (Qdrant) + schema payload ở trên.
- [ ] Ingest 3 nhóm: **Luật/Nghị định**, **Biểu mẫu Sở XD HCM**, **FAQ nội bộ** (≥ **3000 chunks**).
- [ ] Embedding **multilingual-mpnet** + Reranker **bge-reranker-v2-m3**.
- [ ] **Hybrid search + filter** (jurisdiction, source_type, tenant_id).
- [ ] **LLM answer** có **trích dẫn nguồn** (doc_id/điều/page/url).
- [ ] **Dashboard**: nDCG@10, Recall@50, No-support rate, Latency p95.
- [ ] **Jobs** cập nhật/rollback theo `doc_version` + **DSAR purge hooks**.

---

## 14) Gợi ý mở rộng (phase 2)
- **Cross-encoder rerank → ColBERT-v2** nếu cần precision rất cao cho tài liệu dài.
- **Multi-vector (`bge-m3`)** để cải thiện truy vấn **từ khoá tiếng Việt + chính tả sai**.
- **Geo-aware ranking**: **boost** theo khoảng cách bán kính (PMH ± 5km).
- **Learning-to-rank** từ tín hiệu **click/đặt lịch** (pairwise).
- **Image-RAG**: caption/image-embedding cho **mặt bằng/ảnh dự án** (nếu cần).

