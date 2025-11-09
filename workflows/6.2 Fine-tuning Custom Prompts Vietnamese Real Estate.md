# Workflow 6.2 – Fine‑tuning & Custom Prompts (Vietnamese Real Estate)

> **Mục tiêu:** đạt độ chính xác, giọng điệu nhất quán, giảm ảo tưởng và chi phí suy luận. Tương thích 1.x (NLU/DM/NLG), 5.x (Auth/Privacy/Retention) và 6.1 (RAG/Vector DB).

---

## 🎯 Mục tiêu & Phạm vi

**Custom Prompts:** thiết kế hệ thống prompt (system/instruction, planner, tool‑use, guardrail) + few‑shot chuẩn hoá.

**Fine‑tuning (PEFT/LoRA/QLoRA):** tinh chỉnh cho nhiệm vụ hẹp (clarify slot, trả lời có trích dẫn nguồn, booking flow).

**Không mục tiêu:** “tự học online” tại runtime. Luôn theo chu kỳ **train → eval → canary → A/B**.

---

## 🧱 Lựa chọn chiến lược

- **Chỉ Prompting** khi: miền hẹp, dữ liệu vàng ít (< 2k cặp), yêu cầu cập nhật nhanh.
- **PEFT/QLoRA** khi: cần hành vi nhất quán, giảm token đầu ra, xử lý tiếng Việt dài, hoặc tool‑use mẫu hoá.
- **RAG + Prompting (khuyến nghị nền tảng):** mọi câu trả lời đều phải dựa nguồn (6.1).

---

## 🗃️ Chuẩn dữ liệu (Instruction/Chat Format)

**Schema tối thiểu (JSONL)**

```json
{"instruction":"Tạo câu hỏi làm rõ thiếu slot","input":"User: Căn 2PN Q7\nSlots thiếu: dien_tich","output":"Để lọc chính xác hơn, anh/chị cho em biết diện tích tối thiểu (ví dụ 65 m²) ạ?"}
{"messages":[
  {"role":"system","content":"Bạn là trợ lý BĐS Việt Nam..."},
  {"role":"user","content":"Theo Luật Nhà ở 2014, điều kiện cấp sổ hồng chung cư?"},
  {"role":"assistant","content":"Theo Điều ... [Nguồn: ...]"}
]}
```

**Tag meta** (lọc/đánh giá):

- `domain`: `real_estate` | `learning`
- `task`: `clarify` | `result_list` | `law_grounded` | `booking_confirm`
- `tone`: `polite-pro` | `neutral`
- `language`: `vi-VN`

---

## 🧩 Thiết kế Prompt Stack

### 1) System (bất biến, ngắn, ràng buộc)
- Vai trò: Trợ lý BĐS Việt Nam, **lịch sự – ngắn gọn – tự nhiên**.
- **Không bịa**; luôn trích dẫn nguồn khi dựa tài liệu (6.1).
- Không cam kết pháp lý/lợi nhuận; tôn trọng 5.1–5.3 (mask PII, privacy/retention).

### 2) Planner (ẩn / tool‑use)
- Nếu `intent=tim_bds` & thiếu slot → **sinh 1 câu clarify** (< 20 từ).
- Nếu có kết quả → **tóm tắt 1 câu + 3 bullet + CTA**.
- Nếu RAG không đủ → **xin phép hỏi lại** hoặc **từ chối lịch sự**.

### 3) Few‑shot theo act
- 3–6 ví dụ/act (clarify, booking confirm, no‑result, law grounded).

### 4) Output Schema Guard
- Yêu cầu **JSON hợp lệ** khi FE/BE cần (messages/cards/quick_replies).
- Kèm **function‑call schema** nếu dùng tool API.

---

## 🔧 Fine‑tuning (PEFT/QLoRA) – Quy trình

### 1) Chọn model nền
- Nhẹ, tiếng Việt tốt: **Qwen 1.5/2 (1.8B–7B)**, **Llama‑3‑Instruct 8B**, **Gemma‑2 9B**.
- Hạ tầng VRAM hạn chế: **QLoRA 4‑bit** + gradient checkpointing (**Unsloth** hỗ trợ tốt).

### 2) Chuẩn dữ liệu FT
- Gom dialogs vàng từ 1.5 (Learning): *low‑conf, fail* → chuẩn hoá.
- **Ẩn PII (5.2)**; loại turn nhạy cảm. Balance theo task/domain.
- Tối thiểu **5–10k mẫu** đa dạng (hành vi cốt lõi); có thể bắt đầu **1–2k** (PEFT).

### 3) Huấn luyện (ví dụ **Unsloth + TRL**)
- Mục tiêu: **giảm độ dài phản hồi**, **giữ schema**, **cải thiện Vietnamese fluency**.
- Loss: SFT tiêu chuẩn; tuỳ chọn **constraint loss** phạt ngoài schema.
- **Config gợi ý** (QLoRA, 7B, 1×24GB):
  - `bnb_4bit=True`, `r=16–32`, `lora_alpha=32–64`, `lora_dropout=0.05`
  - `lr=1e-4 ~ 2e-4`, `bsz_eff=64` (grad accumulation), `epochs=2–3`
  - `target_modules`: attention proj + MLP up/down (tuỳ model)
  - `max_seq_len=2k`; **packing** nếu phân phối độ dài không đều

```python
from unsloth import FastLanguageModel
from trl import SFTTrainer
from transformers import TrainingArguments

model, tok = FastLanguageModel.from_pretrained(
  "Qwen/Qwen2-7B-Instruct", load_in_4bit=True, max_seq_length=2048
)
model = FastLanguageModel.get_peft_model(
  model, r=32, target_modules=["q_proj","v_proj","k_proj","o_proj","gate_proj","up_proj","down_proj"],
  lora_alpha=64, lora_dropout=0.05, bias="none", task_type="CAUSAL_LM"
)

args = TrainingArguments(
  output_dir="ft-qwen2-7b-r32",
  learning_rate=2e-4, num_train_epochs=2,
  per_device_train_batch_size=2, gradient_accumulation_steps=32,
  logging_steps=20, save_steps=500, bf16=True, lr_scheduler_type="cosine",
  warmup_ratio=0.05
)

trainer = SFTTrainer(
  model=model, train_dataset=train_ds, dataset_text_field="text",
  max_seq_length=2048, tokenizer=tok, args=args, packing=True
)
trainer.train(); model.save_pretrained("artifacts/lora")
```

### 4) Đánh giá
- **Offline:** exact‑match **schema JSON**; *BLEU/ROUGE* < *Task Success* (clarify đúng slot, có trích dẫn).
- **Constitutional tests:** “no‑source → từ chối”; “policy claim → hedge”.
- **Regression:** bộ 200–300 hội thoại chuẩn vàng.

### 5) Triển khai
- **Merge‑weights** (optional) hoặc **attach LoRA** khi suy luận.
- Tag phiên bản: `llm-base@ver + lora@ver + prompt@ver`.
- **Canary 5–10%**; rollback nếu *hallucination↑* hoặc *schema error↑*.

---

## 🛡️ Guardrails & An toàn
- Prompt rule: **không bịa nguồn**, luôn “Theo [Nguồn: …]”.
- **Max tokens** + phong cách “short‑answer” để tránh lan man.
- **Safety checker** (rule & LLM‑critic): pháp lý/đầu tư, PII, cam kết giá.
- **Schema validator** trước khi trả về FE (sai → auto‑repair hoặc fallback template).

---

## 🧪 Bộ prompt mẫu (rút gọn, vi‑VN)

### System
> Bạn là Trợ lý BĐS Việt Nam. Luôn: 1) Ngắn gọn, lịch sự, tự nhiên. 2) Dựa nguồn (6.1). Nếu thiếu nguồn: xin phép hỏi thêm hoặc từ chối. 3) Không khẳng định pháp lý/lợi nhuận; dùng “Theo [Nguồn:…]”. 4) Khi được yêu cầu UI, trả **JSON** đúng schema.

### Planner – Clarify
**Nhiệm vụ:** Nếu thiếu slot `{slot_list}`, hãy hỏi **đúng 1 câu rõ ràng** (< 20 từ), không gợi ý dư.  
**Ngữ cảnh:** `{context_summary}`  
**Đầu ra:** chỉ **câu hỏi** làm rõ.

### Answer w/ citations
- **Câu hỏi:** `{query_norm}`
- **Ngữ cảnh (≤ 3 đoạn):** `{ctx_1}\n{ctx_2}\n{ctx_3}`
- **Yêu cầu:** trả lời ≤ 3 câu, cuối cùng chèn: `[Nguồn: {doc_id}:{section} ({date})]`.  
Nếu không đủ ngữ cảnh: nói *"Chưa đủ dữ liệu để kết luận."* và gợi ý **1 câu hỏi làm rõ**.

### JSON UI (cards)
**Trả về đúng schema**:

```json
{"messages":[{"type":"text","content":""}],"cards":[],"quick_replies":[]}
```

> **Chỉ JSON, không thêm chữ thường.**

---

## 📈 Chỉ số & A/B

- **Task Success** (đặt lịch/ mở chi tiết/ câu làm rõ đúng): ↑ so baseline ≥ **+5–10%**
- **Hallucination Rate** (không nguồn/nguồn sai): **< 1–2%**
- **Schema Error Rate** (JSON invalid): **< 0.5%**
- **Avg Tokens Out:** giảm **≥ 15%** (sau FT/prompt tuning)
- **Latency p95:** không tăng > **10%** so baseline

---

## 🔌 Tích hợp với 6.1 & 1.x, 5.x

- **6.1** cấp context + citation → dùng prompt **Answer w/ citations**.
- **1.3 (Policy)** chọn act: Clarify vs Result List → chọn prompt/act tương ứng.
- **5.1** áp AuthZ filter trước khi RAG; **5.2** che PII trong prompt; **5.3** kiểm soát retention.

---

## ✅ Checklist MVP (1–2 tuần)

- **Prompt stack** (system/planner/acts) + **20–40 few‑shot** chuẩn vàng.
- **Dataset SFT 2–5k** cặp (clarify, law‑grounded, result_list, booking_confirm).
- **Train QLoRA (7B)** + **eval offline** + **regression 200** hội thoại.
- **Inference attach‑LoRA** + **schema validator** + **canary 10%**.
- **Dashboard:** Task Success, Hallucination, Schema Error, Tokens Out, Latency.
- **Playbook rollback** & **prompt‑hotfix** (không cần retrain khi lệch văn phong).

