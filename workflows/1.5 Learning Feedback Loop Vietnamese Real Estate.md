## 🔄 Workflow Chức năng 1.5 – Learning / Feedback Loop (Học & Cải thiện theo dữ liệu)

### 🎯 Mục tiêu & Phạm vi
**Mục tiêu:** Thu thập tín hiệu chất lượng từ người dùng/hệ thống → gán nhãn/chuẩn hoá → huấn luyện/làm mới mô hình → triển khai có kiểm thử & A/B test.

**Phạm vi:** Intent/slot (1.1), policy (1.3), template & wording (1.2), và gợi ý UI/CTA.

**Không mục tiêu:** Online-learning “tự động viết lại mô hình” ngay tại runtime (rủi ro cao). Sử dụng chu kỳ định kỳ + canary/A/B.

---

### 🧭 Vòng đời dữ liệu (E2E Loop)

#### **1) Thu thập tín hiệu (Signals)**
- **Explicit:** nút 👍/👎, “Câu trả lời không đúng”, chỉnh sửa bộ lọc, “báo cáo lỗi”.
- **Implicit:** bỏ cuộc (drop-off), backtrack, time-to-first-CTA, mở card chi tiết, chuyển kênh gọi người thật.
- **Hệ thống:** confidence thấp, `no_result`, `conflict_slots`, guardrail triggers.

#### **2) Gộp & Chuẩn hoá (Curation)**
- Lọc PII, ẩn danh, gắn `session_id`/`user_id` (băm), kèm version model.
- Gợi ý nhóm mẫu cần gán nhãn: low-confidence, bất đồng (mô hình A ≠ B), high-impact (tạo lịch thành công/thất bại).

#### **3) Gán nhãn (Annotation)**
- **Công cụ:** Doccano/Label Studio.
- **Schema nhãn:** Intent (1 giá trị), Slots (BIO), Correction (nếu user sửa), Quality tags (hallucination, off-topic, policy-violate).
- **Quy tắc QA 2 tầng:** annotator → reviewer.

#### **4) Tập dữ liệu huấn luyện**
- Tách train/dev/test **theo thời gian** (time-based split) để đo drift.
- Augmentation có kiểm soát: paraphrase, biến thể đơn vị tiền (tỷ/triệu/USD), alias địa danh (Q7/Quận 7/Q.7).

#### **5) Huấn luyện & Đánh giá**
- Train NLU (intent/slots), reranker tìm kiếm (nếu có RAG), NLG templates ranking (nếu học hoá).
- **Đánh giá offline:** F1 macro intent/slot, Exact match các slot chính, Robustness (lỗi chính tả, viết tắt).
- **Regression suite:** chạy lại 100–300 hội thoại “chuẩn vàng”.

#### **6) Triển khai an toàn**
- **Model Registry** (vX.Y.Z), Canary 5–10% traffic, A/B có phân tầng (kênh, địa bàn).
- Rollback tức thì nếu KPI tụt.

#### **7) Giám sát sau triển khai**
- Live dashboards: Success rate, Clarify turns, No-result rate, CTR card/CTA, Complaint rate.
- Drift detection: phân phối intent/slot, OOD tăng bất thường → cảnh báo.

---

### 📦 Data Contracts (payload & bảng)

#### **Event (mỗi lượt/turn)**
```json
{
  "ts": 1730431200,
  "session_id": "s-uuid",
  "user_id_hash": "u-***",
  "channel": "web",
  "model_version": "nlu-1.4.2|nlg-0.9.1|policy-0.7.0",
  "input_text": "căn 2pn q7 dưới 3 tỷ",
  "nlu": {
    "intent": "tim_bds",
    "confidence": 0.82,
    "slots": {"khu_vuc":"quận 7","so_phong":2,"ngan_sach_max":3000000000}
  },
  "dm_state": "COLLECTING",
  "response_type": "RESULT_LIST|CLARIFY|NO_RESULT|CONFIRM",
  "metrics": {"latency_ms": 85, "cards_shown": 3},
  "user_actions": {"clicked_card_ids": ["SR-2850"], "cta":"book_visit"},
  "feedback": {"thumb":"up", "reason": null}
}
```

#### **Annotation (gán nhãn sau)**
```json
{
  "event_id": "e-uuid",
  "gold_intent": "tim_bds",
  "gold_slots": {"khu_vuc":"quận 7","so_phong":2,"ngan_sach_max":3000000000},
  "quality_tags": ["ok"],
  "reviewed_by": "annotator02",
  "round": "2025-11-R1"
}
```

#### **Model Registry (bảng)**
```
model_name | version | dataset_round | git_sha | train_time | eval_intent_f1 | eval_slot_f1 | notes
nlu        | 1.4.3   | 2025-11-R1    | a1b2c3  | 2025-11-01 | 0.923          | 0.882        | add Q.Thủ Đức aliases
```

---

### 🧪 Chiến lược Active Learning (ưu tiên mẫu cần gán nhãn)
- **Low confidence:** P(intent) < 0.6 hoặc slot avg < 0.6.
- **Disagreement:** mô hình cũ vs mới khác ý định/slot.
- **No-result / Fail cases:** tìm kiếm trả rỗng, đặt lịch thất bại.
- **High business impact:** người dùng có ý định “đặt lịch/leave contact” nhưng rơi vào clarify lặp.
- **Coverage gaps:** địa danh/alias mới, cụm từ mới (ví dụ "xì tai penthouse", "view sông Sài Gòn").
- **Batch:** mỗi tuần chọn top N (500–1,500) mẫu để gán nhãn.

---

### ⚙️ Huấn luyện & kiểm thử (mẫu quy trình)

#### **Pipeline (pseudo bash)**
```bash
# 1) trích xuất & làm sạch
python ingest_events.py --from 2025-10-01 --to 2025-10-31 --drop-pii

# 2) chọn mẫu Active Learning
python select_samples.py --strategy lowconf,disagree,noresult --n 1000

# 3) hợp nhất với gold data cũ
python build_dataset.py --round 2025-11-R1 --split time

# 4) train
python train_intent.py --cfg configs/nlu_intent.yaml
python train_slots.py  --cfg configs/nlu_slots.yaml

# 5) eval + regression
python eval_suite.py --gold test_2025-11.json --regression gold_dialogs_300.json

# 6) push model + metadata
python register_model.py --name nlu --version 1.4.3 --metrics metrics.json
```

#### **Policy/NLG tuning (nếu có)**
- Bandit/A-B lựa chọn template variants: **T1 (ngắn gọn)** vs **T2 (CTA kép)** → đo CTR đặt lịch, dwell time.

---

### 🧰 Cải tiến theo từng lớp

#### **(A) NLU (1.1)**
- Thêm gazetteer alias mới phát hiện từ logs.
- Fine-tune intent/slot với batch mới; regularization để giữ kiến thức cũ (EWC/early-stop).
- Kiểm tra **catastrophic forgetting** bằng regression suite.

#### **(B) Dialogue Policy (1.3)**
- Tối ưu trọng số `score(SEARCH vs CLARIFY)` bằng grid search trên replay logs.
- Điều chỉnh `threshold_clarify` theo hiệu quả clarify.

#### **(C) NLG (1.2)**
- Học xếp hạng template theo conversion/CTR (pairwise ranking).
- Tối ưu câu hỏi làm rõ để giảm mệt mỏi hội thoại.

---

### 📈 KPI theo dõi (mức hệ thống)

**Chất lượng hiểu**  
- Intent F1 macro ≥ **0.90** (dev).  
- Slot F1 ≥ **0.85** (khu_vuc, ngan_sach, loai_bds…).

**Hiệu quả hội thoại**  
- Task Success Rate (mở chi tiết/đặt lịch thành công): +X%/tháng.  
- Avg Turns to Success ≤ **6**.  
- Clarify Efficiency ≤ **1.5** câu/tác vụ.  
- No-result Rate giảm dần (theo khu/giá).

**Trải nghiệm người dùng**  
- CSAT/Thumb-up rate ≥ **80%**.  
- Complaint/Hallucination rate < **1%**.

**Vận hành**  
- Canary error rate ≈ baseline, p-value A/B < **0.05** cho cải thiện chính.

---

### 🔒 An toàn & Quyền riêng tư
- Ẩn danh/băm `user_id`; không lưu câu gốc gắn PII nhạy cảm (địa chỉ chính xác, số ĐT bên thứ ba).
- **Opt-in** cho việc dùng dữ liệu để cải thiện; tôn trọng export/delete theo yêu cầu.
- Bộ lọc nội dung: loại bỏ turn “sensitive” khỏi tập huấn luyện (danh mục regex/policy).

---

### 🧱 Hạ tầng khuyến nghị
- **Event bus:** Kafka (topics: `chat_events`, `feedback_events`).
- **Feature store:** Parquet/S3 + DuckDB/BigQuery.
- **Model registry/experiments:** MLflow/W&B.
- **Dashboards:** Metabase/Superset (funnel, cohort, heatmap intent).
- **CI/CD:** GitHub Actions → build docker → deploy canary (K8s/Render/Vercel BE) → auto-rollback.

---

### 🗂️ Bộ bảng tối thiểu (SQL gợi ý)
```sql
-- Sự kiện thô
CREATE TABLE events (
  ts TIMESTAMP,
  session_id TEXT,
  user_id_hash TEXT,
  channel TEXT,
  model_version TEXT,
  intent TEXT,
  intent_conf NUMERIC,
  slots JSONB,
  response_type TEXT,
  actions JSONB,
  feedback JSONB
);

-- Nhãn vàng
CREATE TABLE annotations (
  event_id TEXT PRIMARY KEY,
  gold_intent TEXT,
  gold_slots JSONB,
  tags TEXT[],
  reviewed_by TEXT,
  round TEXT
);

-- Đánh giá mô hình
CREATE TABLE model_evals (
  model_name TEXT,
  version TEXT,
  metric TEXT,
  value NUMERIC,
  split TEXT,
  ts TIMESTAMP
);
```

---

### 🧪 Tình huống điển hình (BĐS)
- **Clarify lặp lại** nhiều ở “Thủ Đức” → logs thiếu `dien_tich`  
  ⇒ Cập nhật policy: nếu có 2PN & ngân sách rõ mà thiếu `dien_tich` → mặc định 65–75 m² + cho sửa.

- **No-result** tăng ở 2PN < 2.3 tỷ, Q7  
  ⇒ NLG gợi ý “Q7 lân cận: Nhà Bè, Phú Mỹ Hưng; hoặc tăng ngân sách +0.2 tỷ”.

- **Alias mới:** người dùng hay gõ “PMH” → map “Phú Mỹ Hưng”  
  ⇒ Thêm alias vào gazetteer + test hồi quy.

---

### ✅ Checklist MVP cho 1.5
- Thu thập events + feedback (payload như trên) kèm `model_version`.
- Bộ lọc PII + ẩn danh.
- Bộ chọn mẫu active learning (lowconf, disagree, no-result, high-impact).
- Quy trình gán nhãn 2 tầng + audit.
- Script train/eval/register/deploy + regression suite.
- Canary & A/B + dashboard KPI (task success, clarify, no-result, CTR).
- Chính sách privacy (opt-in, export/delete).

