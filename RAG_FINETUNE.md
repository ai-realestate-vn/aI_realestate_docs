# Kết Hợp RAG Và Fine-Tune Trong Ứng Dụng AI Sales Bất Động Sản

## Tóm Tắt Yêu Cầu

Bạn hỏi liệu có thể kết hợp **RAG** và **Fine-tune** trong ứng dụng AI Sales Bất động sản (BĐS) và muốn hiểu cách áp dụng, dựa trên việc:

- Sử dụng RAG cho dữ liệu động (giá, dự án, chính sách, sản phẩm).
- Sử dụng Fine-tune cho phong cách tư vấn, giọng sales, và phân loại intent.
- Ví dụ: RAG tìm thông tin dự án, Fine-tune đảm bảo câu trả lời mang giọng điệu sales chuyên nghiệp.

**Trả lời ngắn gọn**: Có, kết hợp RAG và Fine-tune là khả thi và rất hiệu quả cho ứng dụng AI Sales BĐS.

## Kết Hợp RAG Và Fine-Tune: Phân Tích Kỹ Thuật

**Tính khả thi**:

- RAG và Fine-tune bổ trợ lẫn nhau:
  - **RAG**: Cung cấp thông tin chính xác, cập nhật từ dữ liệu động (giá, dự án, chính sách).
  - **Fine-tune**: Tùy chỉnh giọng điệu, kịch bản sales (truyền cảm hứng, CTA), và xử lý intent phức tạp.
- Kết hợp giúp tối ưu cả **độ chính xác** (RAG) và **trải nghiệm người dùng** (Fine-tune).

**Ứng dụng trong AI Sales BĐS**:

- **RAG**:
  - Truy xuất thông tin dự án (vị trí, giá, tiện ích) từ vector DB (Weaviate/pgvector).
  - Đảm bảo câu trả lời dựa trên dữ liệu mới nhất, ví dụ: "Dự án A có giá từ 2 tỷ, gồm 3 phòng ngủ."
- **Fine-tune**:
  - Huấn luyện mô hình trả lời theo kịch bản sales: thân thiện, thuyết phục, có CTA (ví dụ: "Anh/chị muốn đặt lịch xem dự án không?").
  - Phân loại intent (ví dụ: hỏi giá → tư vấn → từ chối → follow-up) để dẫn dắt hội thoại.
- **Ví dụ luồng xử lý**:
  - Câu hỏi: "Dự án X giá bao nhiêu?"
  - RAG: Truy xuất thông tin từ vector DB → "Dự án X giá từ 2.5 tỷ, vị trí quận 7."
  - Fine-tune: Định dạng câu trả lời: "Dự án X hiện có giá từ 2.5 tỷ tại quận 7, rất phù hợp cho gia đình trẻ. Anh/chị muốn em gửi thêm thông tin chi tiết không?"

## Kiến Trúc Đề Xuất (High-Level)

```
[Web/App Client] --> [API Gateway (FastAPI)] --> [Orchestrator]
                                  |
       +--------------------------+--------------------------+
       |                          |                          |
  [RAG Pipeline]          [Fine-tuned LLM]          [Cache/Logging]
       |
 [Vector DB] --> [Doc Store]
```

- **Client**: Web widget hoặc app, tích hợp CRM (Salesforce/HubSpot).
- **API Gateway**: FastAPI, xử lý 3 endpoint chính:
  - `POST /query`: Nhận câu hỏi, trả lời từ RAG + Fine-tune.
  - `POST /feedback`: Lưu đánh giá CSAT.
  - `GET /analytics`: Báo cáo KPI (latency, groundedness).
- **RAG Pipeline**:
  - Vector DB: Weaviate/pgvector.
  - Embedding: `bge-m3` (miễn phí, hiệu quả).
  - Re-ranker: Cohere Rerank hoặc ColBERT.
- **Fine-tuned LLM**: Llama 3.1 (8B) hoặc Mistral, sử dụng LoRA cho hiệu quả.
- **Cache**: Redis, lưu kết quả truy xuất phổ biến để giảm latency.
- **Logging**: OpenTelemetry, trace conversation, monitor cost/latency.
- **Storage**: S3/GCS cho tài liệu (hợp đồng, thông tin dự án).

## AI/ML Chi Tiết

### RAG

- **Mô hình**:
  - Embedding: `bge-m3` (miễn phí, hỗ trợ đa ngôn ngữ, hiệu quả cao).
  - Re-rank: Cohere Rerank (tăng độ chính xác tài liệu).
- **Chiến lược**:
  - **Chunking**: Chia tài liệu thành đoạn ~200–500 token, giữ ngữ cảnh (ví dụ: thông tin dự án, điều khoản pháp lý).
  - **Indexing**: HNSW index trong Weaviate/pgvector, tối ưu tốc độ truy xuất.
  - **Prompt**:
    ```
    Bạn là trợ lý AI Sales BĐS. Dựa trên tài liệu sau, trả lời câu hỏi chính xác và ngắn gọn:
    Tài liệu: {context}
    Câu hỏi: {query}
    Trả lời: [Câu trả lời ngắn gọn, kèm thông tin từ tài liệu]
    ```
  - **Guardrail**: Kiểm tra groundedness (≥80% câu trả lời dựa trên tài liệu).
  - **Eval**: Factuality (so sánh với CRM), groundedness, hallucination (<5%).
- **Latency**: p95 ~1–2s (tùy quy mô vector DB và caching).

### Fine-Tune

- **Mô hình**: Llama 3.1 (8B) hoặc Mistral (OSS, nhẹ, dễ triển khai).
- **Dữ liệu huấn luyện**:
  - 1000–5000 mẫu hội thoại sales (từ CRM hoặc ghi âm).
  - Cấu trúc: {query, response, intent} (ví dụ: hỏi giá → tư vấn → CTA).
  - Ví dụ mẫu:
    ```
    {
      "query": "Dự án X giá bao nhiêu?",
      "response": "Dự án X giá từ 2.5 tỷ, vị trí quận 7, rất tiềm năng. Anh/chị muốn em gửi thông tin chi tiết?",
      "intent": "consult_price"
    }
    ```
- **Phương pháp**: LoRA (tiết kiệm compute, ~1–2GB VRAM cho 8B model).
- **Eval**:
  - Instruction-following: BLEU/ROUGE so sánh với kịch bản chuẩn.
  - CSAT: ≥4.4/5 từ khách hàng.
  - Intent accuracy: ≥90% phân loại đúng intent.
- **Latency**: p95 ~0.5–1s (nhanh hơn RAG).

### Kết Hợp RAG + Fine-Tune

- **Luồng xử lý**:
  1. **Phân loại intent**: Fine-tuned model phân loại câu hỏi (ví dụ: hỏi giá, pháp lý, tư vấn).
  2. **Truy xuất RAG**: Nếu cần dữ liệu động (giá, dự án), gọi vector DB để lấy tài liệu liên quan.
  3. **Tạo câu trả lời**: Fine-tuned model định dạng câu trả lời theo giọng sales, kết hợp thông tin từ RAG.
- **Prompt mẫu**:
  ```
  Bạn là chuyên viên sales BĐS chuyên nghiệp. Dựa trên thông tin từ tài liệu, trả lời câu hỏi theo giọng điệu thân thiện, thuyết phục, kèm CTA:
  Tài liệu: {context}
  Câu hỏi: {query}
  Trả lời: [Câu trả lời ngắn gọn, truyền cảm hứng, có CTA]
  ```
- **Ví dụ**:
  - Câu hỏi: "Dự án A có gì nổi bật?"
  - RAG: Truy xuất → "Dự án A: 3 phòng ngủ, hồ bơi, trung tâm quận 7."
  - Fine-tune: "Dự án A tọa lạc ngay trung tâm quận 7, với hồ bơi sang trọng và căn hộ 3 phòng ngủ rộng rãi. Anh/chị muốn em sắp xếp lịch xem nhà mẫu không?"

## Hệ Thống/Dev

- **Backend**: FastAPI (Python) cho tốc độ và dễ tích hợp AI.
- **DB Schema**:
  - `conversations`: {id, user_id, query, response, intent, timestamp, csat}.
  - `documents`: {id, project_id, content, embedding, last_updated}.
- **API Spec**:
  - `POST /query`:
    ```json
    {
      "query": "Dự án X giá bao nhiêu?",
      "user_id": "123",
      "context": { "crm_data": {...} }
    }
    Response: {
      "response": "Dự án X giá từ 2.5 tỷ...",
      "intent": "consult_price",
      "sources": ["doc_id_1", "doc_id_2"]
    }
    ```
  - `POST /feedback`: Lưu CSAT (1–5).
  - `GET /analytics`: Báo cáo latency, groundedness, CSAT.
- **AuthN/AuthZ**: OAuth2 (Keycloak), role-based (admin, sale, customer).
- **CI/CD**: GitHub Actions, deploy trên AWS ECS hoặc Lambda.
- **Chi phí ước lượng**: ~$700–1500/tháng (vector DB, LLM API, fine-tune, cloud).

## Ưu/Nhược Điểm & Rủi Ro

**Ưu điểm**:

- **RAG**: Cập nhật dữ liệu động nhanh chóng, đảm bảo tính chính xác.
- **Fine-tune**: Tăng trải nghiệm khách hàng với giọng điệu chuẩn sales, xử lý intent phức tạp.
- **Kết hợp**: Tối ưu cả độ chính xác (RAG) và trải nghiệm (Fine-tune), phù hợp cho AI Sales BĐS.

**Nhược điểm**:

- **RAG**: Latency cao hơn (~1–2s), phụ thuộc chất lượng dữ liệu.
- **Fine-tune**: Chi phí ban đầu cao (dữ liệu, compute), cần bảo trì khi kịch bản thay đổi.
- **Kết hợp**: Phức tạp hơn trong triển khai và bảo trì.

**Rủi ro & giải pháp**:

- **Rủi ro**: Truy xuất sai tài liệu (RAG) → Khắc phục: Re-ranking, eval groundedness tự động.
- **Rủi ro**: Fine-tune lệch giọng điệu hoặc overfit → Khắc phục: Eval định kỳ, bổ sung dữ liệu đa dạng.
- **Rủi ro**: Chi phí vượt dự toán → Khắc phục: Cache truy xuất, dùng OSS model (Llama 3.1), tối ưu compute.

## Roadmap Triển Khai

- **Tuần 1–2 (PoC)**:
  - Xây RAG pipeline: Vector DB (Weaviate), embedding (`bge-m3`), API (FastAPI).
  - Tích hợp LLM (GPT-4o hoặc Llama 3.1).
  - Thử nghiệm với 10 câu hỏi mẫu từ CRM.
- **Tuần 3–4 (Pilot)**:
  - Thu thập 1000+ mẫu hội thoại sales cho fine-tune.
  - Fine-tune Llama 3.1 với LoRA (kịch bản sales, intent).
  - Tích hợp CRM, thử nghiệm với 20 sale.
  - Eval: Groundedness ≥80%, CSAT ≥4.2/5.
- **Tháng 2–3 (Prod)**:
  - Triển khai full, tối ưu latency (p95 ≤1.5s).
  - Mở rộng 100 sale, tích hợp feedback loop.
  - Báo cáo KPI: Answer-rate ≥85%, lead → SQL ≥20%.

**Checklist bàn giao**:

- [ ] RAG pipeline (vector DB, embedding, re-ranker).
- [ ] Fine-tuned model (LoRA, Llama 3.1) với kịch bản sales.
- [ ] API tích hợp CRM, logging/trace (OpenTelemetry).
- [ ] Báo cáo eval: Groundedness, latency, CSAT.
- [ ] Tài liệu: API spec, cách cập nhật dữ liệu, hướng dẫn fine-tune.

## KPI & Eval

- **Chất lượng**:
  - Answer-rate: ≥85% câu hỏi được trả lời đúng.
  - Groundedness: ≥80% câu trả lời dựa trên tài liệu.
  - Hallucination: <5% câu trả lời sai lệch.
- **Hiệu năng**:
  - p95 latency: ≤1.5s (RAG + Fine-tune).
  - Throughput: ≥100 queries/phút.
- **Sản phẩm**:
  - CSAT: ≥4.4/5.
  - Lead → SQL: ≥20%.
- **Chi phí**: ≤$0.07/conversation (RAG + Fine-tune).

## Bảo Mật & Tuân Thủ

- **PII**: Mã hóa dữ liệu khách hàng (AES-256 at-rest, TLS in-transit).
- **Phân tách dữ liệu**: Tách tài liệu theo dự án/khách hàng trong vector DB.
- **Logging**: Không lưu PII, chỉ lưu metadata (conversation_id, intent, timestamp).
- **Tuân thủ**: GDPR (nếu có khách hàng quốc tế), luật BĐS Việt Nam.

## Tài Liệu Tham Khảo

- [LangChain RAG Guide](https://python.langchain.com/docs/use_cases/question_answering/) – Hướng dẫn RAG.
- [Hugging Face: LoRA Fine-tuning](https://huggingface.co/docs/peft/task_guides/lora) – Hướng dẫn LoRA.
- [Weaviate Docs](https://weaviate.io/developers/weaviate) – Setup vector DB.
- [Cohere Rerank](https://cohere.com/rerank) – Re-ranking tài liệu.

## Artifact Code (rag_finetune_workflow.py)

```python
import asyncio
from typing import Dict, List
from fastapi import FastAPI
from pydantic import BaseModel
from sentence_transformers import SentenceTransformer
from qdrant_client import QdrantClient
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

# FastAPI app
app = FastAPI()

# Models
embedder = SentenceTransformer("BAAI/bge-m3")
llm_model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.1-8B", torch_dtype=torch.bfloat16)
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B")
qdrant_client = QdrantClient(url="http://localhost:6333")

class QueryRequest(BaseModel):
    query: str
    user_id: str

class QueryResponse(BaseModel):
    response: str
    intent: str
    sources: List[str]

async def search_rag(query: str, top_k: int = 5) -> List[Dict]:
    # Encode query
    query_vector = embedder.encode(query, normalize_embeddings=True).tolist()

    # Search vector DB
    results = qdrant_client.query_points(
        collection_name="real_estate_docs",
        query=query_vector,
        limit=top_k,
        with_payload=True
    )

    docs = [
        {
            "id": r.id,
            "content": r.payload.get("content", ""),
            "project_id": r.payload.get("project_id", ""),
            "score": float(r.score)
        } for r in results.points
    ]
    return docs

async def generate_response(query: str, docs: List[Dict], intent: str) -> str:
    # Build prompt with RAG context
    context = "\n".join([f"Document {i+1}: {doc['content']}" for i, doc in enumerate(docs)])
    prompt = f"""
    Bạn là chuyên viên sales BĐS chuyên nghiệp. Dựa trên tài liệu sau, trả lời câu hỏi theo giọng điệu thân thiện, thuyết phục, kèm CTA:
    Tài liệu: {context}
    Câu hỏi: {query}
    Intent: {intent}
    Trả lời: [Câu trả lời ngắn gọn, truyền cảm hứng, có CTA]
    """

    # Generate response with fine-tuned model
    inputs = tokenizer(prompt, return_tensors="pt")
    outputs = llm_model.generate(**inputs, max_length=200)
    response = tokenizer.decode(outputs[0], skip_special_tokens=True)
    return response

@app.post("/query", response_model=QueryResponse)
async def query_endpoint(request: QueryRequest):
    # Step 1: Classify intent (fine-tuned model)
    intent_prompt = f"Classify intent: {request.query}"
    intent_inputs = tokenizer(intent_prompt, return_tensors="pt")
    intent_outputs = llm_model.generate(**intent_inputs, max_length=50)
    intent = tokenizer.decode(intent_outputs[0], skip_special_tokens=True).strip()

    # Step 2: Retrieve documents (RAG)
    docs = await search_rag(request.query)

    # Step 3: Generate response (fine-tuned model + RAG context)
    response = await generate_response(request.query, docs, intent)

    return QueryResponse(
        response=response,
        intent=intent,
        sources=[doc["id"] for doc in docs]
    )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```
