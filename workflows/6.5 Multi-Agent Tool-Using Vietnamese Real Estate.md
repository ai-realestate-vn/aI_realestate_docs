# 6.5 – Multi-Agent / Tool-Using Chatbot (Vietnamese Real Estate)

> **Mục tiêu:** xây dựng kiến trúc chatbot nhiều tác nhân (multi-agent) có khả năng lập kế hoạch, gọi công cụ, hợp tác, và dừng rõ ràng. Hướng đến vận hành thông minh, an toàn, đa miền (BĐS & học tập), tích hợp các mô-đun 1.x, 5.x, 6.1–6.4.

---

## 🎯 Mục tiêu & Phạm vi
- **Mục tiêu:** Cho phép chatbot tách bài toán thành các bước, giao cho agents chuyên trách (tìm listing, tra luật, định giá, đặt lịch…), gọi tool/API đúng cách, hợp nhất kết quả và trả lời grounded.
- **Phạm vi:** runtime online; hỗ trợ multi-tenant, RBAC/ABAC (5.1), PII masking (5.2), retention (5.3), RAG (6.1), Memory (6.3), Summarization (6.4).

---

## 🧱 Kiến trúc tổng quan (tầng)

### Orchestrator (điều phối trung tâm)
- Nhận yêu cầu → tạo **Kế hoạch (Plan)** → giao nhiệm vụ → hợp nhất kết quả.
- Kiểm soát dừng, retry, logging, tracing, guardrail.

### Agents chuyên trách (stateless, có prompt riêng)
- **Router/Planner:** phân rã nhiệm vụ, sắp thứ tự, chọn công cụ.
- **RAG-Researcher:** truy xuất tri thức (6.1) + trích dẫn nguồn.
- **Listing-Finder:** tìm bất động sản (lọc, geo, giá, rerank).
- **Price-Estimator:** ước tính giá/tham chiếu thị trường (không cam kết).
- **Booking-Scheduler:** gợi ý & đặt lịch (calendar/CRM).
- **Law-QA:** trả lời theo luật/biểu mẫu (có citation, hedge).
- **Summarizer:** tóm tắt (6.4) cho UI & memory.
- **Memory-Agent:** đọc/ghi STM/LTM (6.3).
- **Guardrail/Critic:** kiểm tra an toàn, format JSON, tính nhất quán nguồn.

### Tooling Layer (Tool Registry)
- Định nghĩa **function schema (JSON Schema)** cho từng tool.
- **Adapters:** Qdrant, SearchListings, Calendar, CRM, Payments, Maps, OCR/PDF…
- **PEP/PDP:** kiểm quyền (5.1) trước khi gọi tool.

### Observability
- **Traces/Spans:** mỗi lời gọi tool → 1 span (latency, status, payload size).
- **Replay logs:** phục vụ 1.5 (Learning) & debug.
- **Idempotency & Retries:** backoff exponential cho lỗi mạng.

---

## 🗺️ Luồng tổng quát (Plan → Execute → Verify → Respond)

### 1. Intake & Routing
- Nhận `user_text` + `ctx` (STM/LTM, tenant, role).
- Router → dự đoán `task type`: `tim_bds` | `law_qa` | `booking` | `learning` | `smalltalk`.
- Nếu multi-intent: ưu tiên tác vụ hành động, phần còn lại thành subtasks.

### 2. Planning
- Planner sinh **Plan Graph (DAG)** gồm các bước có preconditions.
- Ví dụ (tim_bds): `Normalize → SearchListings → Rerank → Summarize → NLG cards`.

### 3. Execution
- Orchestrator chạy từng step:
  - Kiểm quyền (5.1), che PII (5.2), timeout 2–5s/tool.
  - Retry 2 lần (backoff) cho lỗi mạng, không retry với 401/403.
  - Ghi trace + cache TTL ngắn.
  - Nếu step fail → Repair step (nới filter/Clarify user).

### 4. Verification (Critic)
- Kiểm: có nguồn chưa (RAG)? JSON hợp lệ? có vi phạm policy?
- Nếu lỗi nhỏ → tự sửa (self-heal 1 vòng); nếu không → fallback template.

### 5. Respond & Update Memory
- Trả payload (text + cards + quick replies).
- Cập nhật STM & LTM (nếu consent & fact bền, theo 6.3).

### 6. Stop Condition
- Khi **hoàn tất kế hoạch**, hoặc **đạt mục tiêu (CTA executed)**, hoặc **yêu cầu Clarify**.

---

## 🧩 Plan Graph (ví dụ)
```json
{
  "goal": "tim_bds",
  "steps": [
    {"id": "S1", "type": "normalize", "inputs": ["user_text","ltm"], "outputs": ["filters"]},
    {"id": "S2", "type": "tool", "tool": "SearchListings", "inputs": ["filters"], "outputs": ["listings"]},
    {"id": "S3", "type": "tool", "tool": "Rerank", "inputs": ["listings","query"], "outputs": ["topk"]},
    {"id": "S4", "type": "agent", "agent": "Summarizer", "inputs": ["topk"], "outputs": ["cards"]},
    {"id": "S5", "type": "agent", "agent": "NLG", "inputs": ["cards","ctx"], "outputs": ["reply"]}
  ],
  "repair": {"no_result": "relax_budget_or_area", "low_conf": "clarify_missing_slot"}
}
```

---

## 🔌 Tool Registry (mẫu định nghĩa)
```json
{
  "SearchListings": {
    "auth": "RBAC:buyer|broker|admin",
    "rate_limit": "30rpm",
    "schema": {
      "type": "object",
      "properties": {
        "khu_vuc": {"type": "string"},
        "ngan_sach_min": {"type": "number"},
        "ngan_sach_max": {"type": "number"},
        "so_phong": {"type": "integer"},
        "dien_tich_min": {"type": "number"},
        "geo": {"type": "object", "properties": {"lat": {"type": "number"}, "lon": {"type": "number"}, "radius_km": {"type": "number"}}}
      },
      "required": ["khu_vuc"]
    }
  },
  "RAG": {
    "auth": "RBAC:any",
    "schema": {"type": "object", "properties": {"query": {"type": "string"}, "filters": {"type": "object"}}, "required": ["query"]}
  },
  "BookVisit": {
    "auth": "RBAC:buyer|broker|admin",
    "idempotency_key": true,
    "schema": {"type": "object", "properties": {"project": {"type": "string"}, "time_iso": {"type": "string"}, "contact": {"type": "string"}}, "required": ["project","time_iso"]}
  }
}
```

---

## ⚙️ Orchestrator (pseudo-code)
```python
def handle_turn(user_text, ctx):
    task = router.detect(user_text, ctx)
    plan = planner.build(task, user_text, ctx)

    for step in plan.steps:
        enforce_authz(step, ctx)   # 5.1
        redact_inputs(step, ctx)   # 5.2
        res = execute_step(step, ctx)
        trace(step, res)

        if res.error:
            if can_repair(step, res, plan):
                plan = repair_step(plan, step, res, ctx)
                continue
            return fallback_clarify(ctx, res)

        ctx = merge_ctx(ctx, res)
        if stop_condition(plan, ctx): break

    reply = critic.verify_and_fix(nlg.compose(ctx))
    memory.update(ctx, reply)
    return reply
```

---

## 🛡️ Guardrails & Chính sách
- **Policy trước tool:** kiểm tenant_id/role/scope → từ chối sớm.
- **PII:** lọc/mask trước RAG/Search; vault cho contact (5.2).
- **Legal/Lợi nhuận:** Law-QA luôn “Theo [Nguồn: …]”.
- **Rate limit & Budget:** hạn chế step/tool/turn; circuit breaker khi lỗi.
- **JSON Schema Validation:** mọi output UI JSON qua validator; lỗi → auto-repair/fallback.

---

## 🔍 Quan sát & Chẩn đoán
- **OpenTelemetry spans:** router, planner, mỗi tool_call, critic, nlg.
- **Metrics:** Task success, No-result rate, Clarify turns, Tool-error rate, Latency p95/p99.
- **Replay/Time-travel:** lưu Plan Graph + tool IO (đã mask) để mô phỏng.
- **A/B:** thử planner-policy, rerank weight, max step.

---

## 🧪 Kịch bản tiêu biểu
### A) Tìm căn 2PN, Q7, <3 tỷ → gợi ý lịch
- Router: tim_bds.
- Planner: Normalize → SearchListings → Rerank → Summarize → NLG cards.
- Repair: relax_budget_or_area (+0.2 tỷ / mở rộng khu).
- User click “Đặt lịch” → Booking-Scheduler gọi BookVisit.
- Critic check JSON, Summarizer tạo session digest, Memory cập nhật (6.3).

### B) Hỏi pháp lý “điều kiện cấp sổ hồng”
- Router: law_qa.
- RAG-Researcher: filter jurisdiction=vn|hcm → top-k + citations.
- Law-QA: trả lời ≤3 câu + [Nguồn: …].
- Critic: has_citations=true → pass → NLG.

---

## 🧩 Prompt khởi tạo Agents (vi-VN)

### Planner
```
Nhiệm vụ: Phân rã yêu cầu thành các bước tối thiểu, có thứ tự, mỗi bước ghi rõ tool/agent.
Ràng buộc: tối đa 5 bước; nếu dữ liệu thiếu → chèn bước CLARIFY.
Đầu ra JSON: {"steps":[{"id":"","type":"tool|agent|clarify","tool?":"","inputs":[],"outputs":[]}],"stop":"when goal reached"}
```

### Guardrail/Critic
```
Kiểm tra: (1) Có nguồn khi câu trả lời dựa tài liệu; (2) Không hứa hẹn pháp lý/lợi nhuận;
(3) JSON đúng schema UI; (4) Số liệu/đơn vị hợp lý.
Nếu lỗi nhỏ → tự sửa; nếu thiếu nguồn → yêu cầu RAG step bổ sung.
```

---

## 📈 KPI mục tiêu
- **Task Success:** +8–15% vs single-agent.
- **Suggestion Rate:** ≥ 90% khi no-result.
- **Tool-error rate:** < 2%; **Schema error:** < 0.5%.
- **p95 latency (2–4 tool calls):** ≤ 1.6s.

---

## ✅ Checklist MVP (1–2 tuần)
- Orchestrator (plan→execute→verify→respond) + stop conditions.
- Agents: Router/Planner, RAG-Researcher, Listing-Finder, Summarizer, Guardrail/Critic, Booking-Scheduler.
- Tool Registry + JSON Schema + idempotency + retry policy.
- Policy hooks (RBAC/ABAC) & PII masking middleware.
- Validator UI JSON + fallback templates.
- Tracing (OTel) + dashboards (latency, success, errors).
- Replay logger (ẩn danh) cho 1.5 (Learning) & debug.

