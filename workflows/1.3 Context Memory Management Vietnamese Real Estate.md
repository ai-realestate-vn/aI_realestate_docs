## 🔁 Workflow Chức năng 1.3 – Context & Memory Management

### 🎯 Mục tiêu & Input/Output
**Mục tiêu:** Xác định “bước tiếp theo tốt nhất” (Next Best Action — NBA) cho mỗi lượt nói dựa trên trạng thái hội thoại.  
**Input:** `user_utterance`, `nlu_result`, `context_state`, `kb/db hits`, `policies`.  
**Output:** `dialogue_act (PLAN)` → gọi tool/API cần thiết → `nlg_payload`.

---

### 🧠 Mô hình điều phối: Hybrid Policy
- **Rule-first:** cho các luồng chắc chắn (`greeting`, `slot-filling`, `confirm`, `booking`).
- **Score-based policy:** khi có cạnh tranh mục tiêu (liệt kê ↔ hỏi làm rõ ↔ đề xuất mở rộng).
- **Safety guardrails:** phủ ngoài (chặn bịa, chặn khẳng định pháp lý).

---

### 🗺️ State Machine (rút gọn)

**States:** `IDLE → COLLECTING → READY → ACTING → PRESENTING → CONFIRMING → COMPLETED ⇄ REPAIR`

#### **Transitions (tiêu biểu)**
- `IDLE + intent=tim_bds` → `COLLECTING` (thiếu slot) / `READY` (đủ).
- `COLLECTING + đủ slot` → `READY`.
- `READY` → `ACTING` (gọi API).
- `ACTING(ok)` → `PRESENTING`; `(rỗng)` → `REPAIR`.
- `PRESENTING + chọn “Đặt lịch”` → `CONFIRMING`.
- `CONFIRMING + confirm ok` → `COMPLETED`.
- Bất kỳ + `low confidence` → `REPAIR`.
- Timeout hoặc loop bảo vệ → `IDLE`.

---

### 🧱 Cấu trúc Context & Session
```json
{
  "session_id": "uuid",
  "user_profile": {"name": "Anh Minh", "lang": "vi", "persona": "polite-pro"},
  "dialog": {
    "turn": 14,
    "state": "COLLECTING",
    "intent_stack": [
      {"name": "tim_bds", "confidence": 0.82, "status": "active"},
      {"name": "smalltalk", "confidence": 0.61, "status": "sidelined"}
    ],
    "slots": {
      "khu_vuc": "quận 7",
      "loai_bds": "chung cư",
      "so_phong": 2,
      "ngan_sach_max": 3000000000
    },
    "missing_slots": ["dien_tich"],
    "history": [
      {"role": "user", "text": "Tôi muốn căn 2PN ở q7 dưới 3 tỷ"},
      {"role": "assistant", "act": "ASK_SLOT", "text": "Diện tích tối thiểu mong muốn là bao nhiêu ạ?"}
    ]
  },
  "policies": {"threshold_clarify": 0.75, "max_clarify_turns": 2, "max_noresult_retries": 1},
  "tools": {"search_listings": {...}, "book_visit": {...}, "rag_law": {...}}
}
```

---

### 🚦 Luồng runtime (chi tiết)

#### **1. Parse (1.1)**
- Lấy `intent`, `slots`, `confidence`, kiểm tra OOD.

#### **2. Context Merge**
- Kế thừa slot cũ nếu người dùng rút gọn ("tăng lên 3 tỷ nhé").
- Xoá/mặc định khi người dùng chỉnh sửa ("không quận 7, chuyển Thủ Đức").

#### **3. Goal Check & Slot-Filling**
- Kiểm tra slot bắt buộc theo intent:
  - `tim_bds`: `khu_vuc` hoặc `du_an`, `ngan_sach_max`, `loai_bds?`, `so_phong?`.
- Nếu thiếu → tạo **Clarify Plan** (hỏi slot quan trọng nhất).

#### **4. Policy Scoring (nếu đủ slot)**
- Tính điểm cho hành động: `SEARCH`, `CLARIFY`, `EDUCATE`.
- Ví dụ:
  ```
  score(SEARCH) = w1*coverage(slots) + w2*intent_conf + w3*market_density(khu_vuc)
  ```
- Chọn hành động có điểm cao nhất.

#### **5. Action Execution**
- `SEARCH` → gọi `search_listings(khu_vuc, budget, filters)`.
- `RAG` → gọi `rag_law(query)`.
- `BOOK` → gọi `book_visit(project, datetime, contact)`.

#### **6. Result Handling**
- Có kết quả → `PRESENTING` (≤3 thẻ, có CTA).
- Không kết quả → `REPAIR` (nới tiêu chí, chỉ thử ≤2 lần).

#### **7. Handover/Abort**
- Nếu người dùng muốn liên hệ người thật → tạo ticket và kết thúc.

#### **8. Update Memory**
- Lưu sở thích (2PN, Q7, ngân sách) nếu được phép.

---

### 📏 Quy tắc & Guardrails
- **Anti-loop:** không hỏi cùng slot >2 lần; nếu chưa có → gợi ý mặc định hoặc bỏ qua.
- **Interruption Handling:** nếu user ngắt với intent mới → push intent cũ, xử lý intent mới rồi resume.
- **Multi-intent:** ưu tiên intent tác vụ > smalltalk; phần còn lại thành quick replies.
- **Safety:** nếu đụng pháp lý/đầu tư → dùng RAG + câu "Theo nguồn A…".

---

### 🔧 Mô-đun & API đề xuất

#### **Policy (TypeScript pseudo)**
```ts
type State = "IDLE"|"COLLECTING"|"READY"|"ACTING"|"PRESENTING"|"CONFIRMING"|"COMPLETED"|"REPAIR";

function decideNext(state: State, ctx: Ctx, nlu: NLU): Plan {
  if (nlu.confidence < ctx.policies.threshold_clarify) return { act: "ASK_REPHRASE" };

  const goal = topIntent(ctx, nlu);
  const missing = requiredSlots(goal).filter(s => !ctx.dialog.slots[s]);

  if (missing.length) return { act: "ASK_SLOT", slot: missing[0] };

  if (goal.name === "tim_bds") {
    const s = scoreSearch(ctx);
    if (s > 0.6) return { act: "SEARCH_LISTINGS" };
    return { act: "ASK_REFINEMENT" };
  }

  if (goal.name === "dat_lich_xem") return { act: "CONFIRM_BOOKING" };

  return { act: "SMALLTALK_OR_HELP" };
}
```

#### **Reducer cập nhật Context**
```ts
function reducer(ctx: Ctx, event: Event): Ctx {
  switch (event.type) {
    case "USER_UTTERANCE":
      return mergeSlots(ctx, event.nlu.slots, event.nlu.corrections);
    case "SEARCH_RESULT":
      return { ...ctx, search: { items: event.items, ts: Date.now() }, dialog: { ...ctx.dialog, state: "PRESENTING" }};
    case "NO_RESULT":
      return bumpNoResult(ctx);
    case "BOOK_CONFIRMED":
      return { ...ctx, dialog: { ...ctx.dialog, state: "COMPLETED" }};
    default: return ctx;
  }
}
```

---

### 🧪 Tình huống tiêu biểu (real estate)

#### **1) Slot-filling gọn**
```
User: “Căn 2PN q7 dưới 3 tỷ” → NLU: intent=tim_bds, thiếu dien_tich.
→ ASK_SLOT: “Anh/chị muốn diện tích tối thiểu khoảng bao nhiêu m² ạ?”
User: “>65” → đủ slot → SEARCH_LISTINGS → PRESENTING 3 căn + CTA.
```

#### **2) Ngắt lời + resume**
```
Đang ở PRESENTING, user hỏi “Cho hỏi lãi suất vay hiện giờ?”
→ push intent cũ → xử lý RAG → trả lời ngắn + nguồn → hỏi lại “Tiếp tục với các căn ở Q7 nhé?”
```

#### **3) Không có kết quả**
```
NO_RESULT → đề xuất “2.8–3.2 tỷ hoặc chuyển Phú Mỹ Hưng?”
Nếu từ chối → kết thúc lịch sự, lưu sở thích.
```

---

### 📈 KPI gợi ý
- **Task Success:** ≥ 65% người dùng mở chi tiết.
- **Avg Turns to Success:** ≤ 6 lượt.
- **Clarify Efficiency:** ≤ 1.5 câu hỏi/tác vụ.
- **Loop Rate:** < 2%.
- **Interruption Resume Rate:** ≥ 90%.

---

### 📦 Mẫu cấu hình (YAML)
```yaml
intents:
  tim_bds:
    required_slots: [khu_vuc|du_an, ngan_sach_max]
    optional_slots: [loai_bds, so_phong, dien_tich, huong_nha]
    actions:
      ready: SEARCH_LISTINGS
      missing: ASK_SLOT
  dat_lich_xem:
    required_slots: [du_an, thoi_gian, contact_phone]
    actions:
      ready: CONFIRM_BOOKING
  hoi_chinh_sach:
    required_slots: [topic]
    actions:
      ready: RAG_LOOKUP
policies:
  threshold_clarify: 0.75
  max_clarify_turns: 2
  max_noresult_retries: 1
```

---

### 🔌 Tích hợp 1.1 & 1.2 (orchestrator — Python pseudo)
```python
def turn(user_text, ctx):
    nlu = nlu_parse(user_text, ctx)
    ctx = merge_context(ctx, nlu)

    plan = decide_next(ctx, nlu)

    if plan.act == "ASK_SLOT":
        return nlg.ask_slot(plan.slot, ctx)
    if plan.act == "SEARCH_LISTINGS":
        items = search_listings(ctx.dialog.slots)
        if items:
            ctx = reducer(ctx, {"type": "SEARCH_RESULT", "items": items})
            return nlg.present_list(items, ctx)
        else:
            ctx = reducer(ctx, {"type": "NO_RESULT"})
            return nlg.suggest_relax(ctx)
    if plan.act == "CONFIRM_BOOKING":
        return nlg.confirm_booking(ctx)
    if plan.act == "ASK_REPHRASE":
        return nlg.ask_rephrase()
    return nlg.help()
```

---

### 🧪 Bộ test nhanh (gợi ý)
- **TC01:** Thiếu 1 slot → hỏi đúng slot ưu tiên.
- **TC02:** User sửa slot (“không q7, thủ đức”) → cập nhật & tiếp tục.
- **TC03:** Ngắt lời hỏi lãi suất → trả lời RAG → resume.
- **TC04:** No-result → gợi ý nới tiêu chí, nếu từ chối → kết thúc lịch sự.
- **TC05:** Low confidence → hỏi lại (ASK_REPHRASE).

