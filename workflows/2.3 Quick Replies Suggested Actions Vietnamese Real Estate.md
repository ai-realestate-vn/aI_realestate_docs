## ⚡ Chức năng 2.3 – Quick Replies / Suggested Actions (Gợi ý nhanh)

### 🎯 Mục tiêu & I/O
**Mục tiêu:** Từ `nlu_result` + `dialog_state` + `results` (nếu có) sinh ra 3–5 gợi ý ngắn, có ích, bấm-là-làm.  
**Input:** intent, slots, missing_slots, policy_plan, search_results?, profile?  
**Output (schema):**
```json
{
  "quick_replies": [
    {"label":"Lọc 2PN + ban công","payload":{"action":"refine","filters":{"so_phong":2,"ban_cong":true}}},
    {"label":"Tăng ngân sách","payload":{"action":"suggest_budget_step","delta":200000000}},
    {"label":"Đặt lịch cuối tuần","payload":{"action":"book_visit","time_hint":"weekend"}}
  ],
  "max_visible": 4,
  "expires_in_turns": 2
}
```

---

### 🧭 Chiến lược sinh gợi ý (Planner → Generator → Ranker)
#### 1. Planner (chọn nhóm gợi ý theo bối cảnh)
- **Theo intent:**
  - `tim_bds` → Refine, Discover, CTA.
  - `dat_lich_xem` → Confirm, Resched, Share.
  - `clarify` → Fill Slot.
  - `no_result` → Relax, Switch Area.

- **Theo state:**
  - PRESENTING → ưu tiên CTA.
  - COLLECTING → ưu tiên Fill Slot.
  - REPAIR → ưu tiên Relax.

- **Theo profile:** ưu tiên thói quen (ví dụ hay xem nhà cuối tuần).

#### 2. Generator (tạo candidates)
- **Rule templates (MVP – ổn định, rẻ):**
  - Refine: Lọc {so_phong}PN, Thêm {tien_ich}, Diện tích ≥ {m2}.
  - Relax: Tăng ngân sách +{step}, Xem khu gần {khu_vuc}.
  - CTA: Đặt lịch {time_hint}, Gọi tư vấn.
  - Fill Slot: Ngân sách ~ {gợi_ý}, Khu vực {top_k}.
- **Heuristics:** sử dụng market stats và khu lân cận từ gazetteer.

#### 3. Ranker (xếp hạng 3–5 item)
- **Utility score:**
  `u = w_c*completeness_gain + w_b*business_value + w_p*personalization + w_h*history_diversity`
- Khử trùng lặp ngữ nghĩa (Jaccard/BERT cosine).
- Đảm bảo đa dạng loại: 1 CTA + 1 Refine + 1 Relax.

---

### 🧱 Bộ quy tắc gợi ý (YAML mẫu)
```yaml
intents:
  tim_bds:
    candidates:
      - type: refine.slot
        when: missing_slots: []
        template: "Lọc {so_phong}PN"
        vars: {so_phong: [2,3]}
      - type: refine.feature
        template: "Ban công rộng"
        payload: {action: "refine", filters: {"ban_cong": true}}
      - type: relax.budget
        when: no_result: true
        template: "Tăng ngân sách +{step}"
        vars: {step: [200000000, 500000000]}
      - type: cta.book
        template: "Đặt lịch cuối tuần"
        payload: {action: "book_visit", time_hint: "weekend"}
      - type: discover.nearby
        template: "Xem khu gần {khu}"
        vars_from: market.nearby(khu_vuc, 2)
  dat_lich_xem:
    candidates:
      - type: confirm
        template: "Xác nhận {time}"
      - type: resched
        template: "Đổi sang {time_alt}"
```

---

### 🔌 Payload hành động (Action Contract)
Postback gửi thẳng vào Orchestrator/Policy (không qua NLU):
```json
{ "action":"refine", "filters":{"so_phong":2,"ban_cong":true} }
{ "action":"book_visit", "time_hint":"weekend" }
{ "action":"relax_budget", "delta":200000000 }
{ "action":"switch_area", "area_suggestion":"Phú Mỹ Hưng" }
{ "action":"fill_slot", "slot":"ngan_sach_max", "value":3000000000 }
```

---

### 🧠 Luồng runtime
1. Sau NLG 1.2 chọn plan → gọi `quickReply.service(plan, ctx, results, profile)`.
2. Generate candidates từ rules + market stats + profile.
3. Rank và dedupe, chọn top-K (K=3–5).
4. Package vào quick_replies và gửi kèm message.
5. FE OnClick → POSTBACK payload (bỏ qua NLU) → Policy 1.3 xử lý.
6. Logging: impression, click, conversion.

---

### 🖥️ FE/UI – nguyên tắc
- Tối đa 4 nút hiển thị 1 dòng; text ≤ 20–24 ký tự.
- Dùng động từ đầu câu: “Lọc…”, “Tăng…”, “Đặt…”.
- Disable & fade sau click; hiển thị spinner ngắn.
- Expire sau `expires_in_turns` hoặc khi state đổi.

---

### 🔒 Guardrails
- Không gợi ý khẳng định pháp lý/lợi nhuận.
- Nếu `no_result` → không gợi ý CTA “Đặt lịch”.
- Ưu tiên payload có cấu trúc, không text tự do.
- Ẩn/bỏ gợi ý PII nếu chưa consent.

---

### 📈 KPI theo dõi
| Chỉ số | Mục tiêu |
|--------|-----------|
| QR-CTR | ≥ 25–35% (PRESENTING) |
| QR→Success | ≥ 12–20% |
| Time-to-action | Giảm ≥ 20% |
| Duplication rate | < 5% |
| Staleness rate | < 3% |

---

### 🛠️ Pseudo Implementation
**TypeScript – service**
```ts
export function buildQuickReplies(ctx: Ctx, plan: Plan, results?: Listing[], profile?: Profile): QuickReply[] {
  const cand: QuickReply[] = [];
  const I = ctx.dialog.intent;

  if (I === "tim_bds") {
    if (ctx.dialog.missing_slots.length) {
      const s = ctx.dialog.missing_slots[0];
      cand.push(qrFillSlot(s, ctx, profile));
    } else {
      cand.push(qrCTA_BookWeekend(ctx));
      cand.push(qrRefineRoom(ctx, profile));
      if (!results?.length) cand.push(qrRelaxBudget(ctx));
      else cand.push(qrDiscoverNearby(ctx));
    }
  }
  const ranked = rankByUtility(cand, ctx, results, profile);
  return dedupe(ranked).slice(0, 4);
}

function qrRefineRoom(ctx, profile): QuickReply {
  const pn = ctx.dialog.slots.so_phong ?? 2;
  return { label:`Lọc ${pn}PN`, payload:{action:"refine", filters:{so_phong: pn}} };
}
function qrRelaxBudget(ctx): QuickReply {
  return { label:"Tăng ngân sách", payload:{action:"relax_budget", delta: 200_000_000} };
}
function qrCTA_BookWeekend(ctx): QuickReply {
  return { label:"Đặt lịch cuối tuần", payload:{action:"book_visit", time_hint:"weekend"} };
}
```

---

### 🧪 Test kịch bản (MVP)
| ID | Tên | Kết quả mong đợi |
|----|------|-------------------|
| TC01 | `tim_bds` đủ slot | Hiển thị CTA + Refine + Discover |
| TC02 | Thiếu slot | Gợi ý Fill Slot đúng slot ưu tiên |
| TC03 | `no_result` | Thay Relax + Switch Area, bỏ CTA |
| TC04 | Click POSTBACK | Policy xử lý, UI cập nhật đúng |
| TC05 | Profile “cuối tuần” | Nút “Đặt lịch cuối tuần” top-1 |

---

### ✅ Checklist MVP 2.3
- [x] Bộ rules/templates cho 6 tình huống (tim_bds, clarify, presenting, dat_lich_xem, rag_faq, no_result)
- [x] Payload POSTBACK chuẩn hoá (refine/relax/book/switch/fill_slot)
- [x] Ranker utility + dedupe + cap 3–5 nút
- [x] FE hiển thị + expire theo state; disable sau click
- [x] Log impression/click/conversion → dashboard A/B
- [x] Guardrails: không CTA sai ngữ cảnh; tôn trọng consent

