## 🧩 Workflow Chức năng 1.4 – Context & Memory (Xử lý ngữ cảnh và bộ nhớ)

### 🎯 Mục tiêu & Phạm vi
**Mục tiêu:** Lưu – truy xuất – cập nhật thông tin từ hội thoại và hồ sơ người dùng để:
- Giữ mạch ngữ cảnh trong phiên hiện tại (short-term).
- Cá nhân hóa có kiểm soát cho các phiên sau (long-term, có consent).

**Không mục tiêu:** ghi nhớ nhạy cảm trái quy định; suy diễn vượt dữ liệu.

---

### 🧱 Các lớp bộ nhớ (4 tầng)

#### **1. Turn Memory (tạm thời)**
- Ghi nội dung lượt nói gần nhất: văn bản đã chuẩn hoá, NLU output.
- TTL: 1 lượt.

#### **2. Short-Term Memory (STM – theo phiên)**
- Chứa: trạng thái hội thoại, slots đã có/thiếu, intent stack, episodic summary.
- TTL: hết phiên hoặc sau 30–120 phút không hoạt động.

#### **3. Long-Term Memory (LTM – đa phiên, có consent)**
- Lưu hồ sơ và sở thích người dùng (ví dụ: thích 2PN, tầng trung, có ban công).
- TTL: 6–12 tháng; tuân theo chính sách và quyền xóa.

#### **4. Knowledge Snippets (Contextual KB)**
- Mẩu tri thức ngắn liên quan trực tiếp tác vụ (FAQ, chính sách vay, v.v.).
- TTL: theo phiên.

---

### 🗺️ Luồng runtime (ghi nhớ & dùng lại)

#### **1. Observe → Extract**
- Lấy intent/slots từ NLU, phát hiện “chỉnh sửa” hoặc phủ định.
  > Ví dụ: “Không quận 7, chuyển Thủ Đức” → xóa `khu_vuc=Q7`, thêm `Thủ Đức`.

#### **2. Merge (Cập nhật STM)**
- Quy tắc hợp nhất:
  - **Newest wins** nếu cùng slot được nhắc lại.
  - **Constraint check** nếu xung đột (2PN vs studio) → hỏi xác nhận.
  - **Inference an toàn** cho đồng nghĩa ("căn hộ" ~ "chung cư").

#### **3. Summarize (Tóm tắt lượt)**
- Sinh **episodic summary** ≤ 120 ký tự, tránh dữ liệu nhạy cảm.
- Cập nhật `missing_slots` để DM (1.3) xác định bước tiếp.

#### **4. Retrieve (Truy xuất LTM nếu phù hợp)**
- Nếu user đã đồng ý → nạp profile (xưng hô, khu ưa thích).
- Nếu không → chỉ dùng STM.

#### **5. Personalize (Ứng dụng)**
- Điều chỉnh tone, quick replies, lọc mặc định theo sở thích.
  > Ví dụ: user hay chọn cuối tuần → đề xuất lịch Thứ 7/Chủ nhật.

#### **6. Decay & Prune (Quên có chủ đích)**
- Giảm confidence cho slot ít dùng theo thời gian.
- Xóa STM khi hết phiên; LTM chỉ lưu nếu consent=true và thuộc whitelist.

---

### 🧱 Cấu trúc dữ liệu khuyến nghị

#### **STM (per session)**
```json
{
  "session_id": "uuid",
  "turn": 12,
  "dialog_state": "COLLECTING",
  "slots": {
    "khu_vuc": {"value": "quận 7", "confidence": 0.9, "ts": 1730431200},
    "loai_bds": {"value": "chung cư", "confidence": 0.86},
    "so_phong": {"value": 2, "confidence": 0.8},
    "ngan_sach_max": {"value": 3000000000, "confidence": 0.88}
  },
  "missing_slots": ["dien_tich"],
  "episodic_summary": "Tìm chung cư 2PN Q7 <3 tỷ; thiếu diện tích tối thiểu.",
  "intent_stack": [{"name": "tim_bds", "status": "active"}]
}
```

#### **LTM (per user, nếu consented)**
```json
{
  "user_id": "u_123",
  "consent": true,
  "consent_scopes": ["preferences","greeting_tone"],
  "locale": "vi-VN",
  "honorific": "Anh/Chị",
  "preferences": {
    "khu_ua_thich": ["quận 7", "thủ đức"],
    "phong_ngu": [2],
    "huong_tranh": ["tây"],
    "ngay_hen_ua_thich": ["thứ 7", "chủ nhật"]
  },
  "meta": {"created_at": 1730000000, "retention_days": 365}
}
```

#### **Retention & Policy**
```yaml
retention:
  stm_ttl_minutes: 120
  ltm_days: 365
privacy:
  pii_whitelist: [honorific, locale, preferences]
  pii_blocklist: [cmnd_cccd, so_nha_chinh_xac, sdt_ben_thu_ba]
  export_enabled: true
  delete_on_request: true
```

---

### ⚖️ Quy tắc hợp nhất (Merge Rules)
- **Priority:** user override > explicit slot > inferred synonym > historical preference.
- **Ambiguity Handling:** nếu có 2 giá trị hợp lệ → hỏi xác nhận hoặc giữ giá trị min/max hợp lý.
- **Confidence-weighted merge:**
  ```
  score = w_c*confidence + w_r*recency + w_s*source_weight
  ```
  Chỉ cập nhật nếu `new_score ≥ old_score + delta`.

---

### 🧠 Decay (quên dần) – công thức gợi ý
```
confidence_t = confidence_{t-1} * exp(-λ * Δturns)   (λ ≈ 0.15–0.25)
```
Nếu < 0.4 → đánh dấu cần xác nhận lại.

---

### 🔒 Quyền riêng tư & tuân thủ (VN)
- **Opt-in:** Không lưu LTM nếu chưa có đồng ý rõ ràng.
- **Minh bạch:** Thông báo mục đích, TTL, quyền xem/xoá (theo NĐ 13/2023).
- **PII minimization:** chỉ lưu thông tin cần thiết; băm hoặc ẩn danh nếu thống kê.
- **Data Subject Rights:** API `export_profile`, `delete_profile`.

---

### 🔧 API gợi ý (FastAPI/Python pseudo)
```python
class MemoryStore:
    def __init__(self, kv_stm, kv_ltm):
        self.stm, self.ltm = kv_stm, kv_ltm

    def get_session(self, sid): ...

    def upsert_slot(self, sid, key, value, conf, source):
        s = self.stm.hgetall(f"stm:{sid}:slots") or {}
        old = s.get(key)
        if not old or self._better(value, conf, old):
            s[key] = {"value": value, "confidence": conf, "ts": now(), "source": source}
        self.stm.hset(f"stm:{sid}:slots", mapping=s)

    def summarize_turn(self, sid, last_uttr, slots):
        summary = make_compact_summary(last_uttr, slots)
        self.stm.setex(f"stm:{sid}:sum", ttl=7200, value=summary)

    def get_profile(self, uid):
        prof = self.ltm.get(f"ltm:{uid}")
        return prof if (prof and prof.get("consent")) else None

    def save_profile(self, uid, prof):
        assert prof.get("consent") is True
        self.ltm.set(f"ltm:{uid}", prof, ex=prof.get("retention_days", 365)*86400)
```

---

### 🔌 Tích hợp với 1.1, 1.2, 1.3
```python
nlu = nlu_parse(text, ctx)
mem.upsert_slot(ctx.sid, "khu_vuc", nlu.slots.get("khu_vuc"), nlu.conf_slot("khu_vuc"), "nlu")
ctx = merge_context(ctx, mem.get_session(ctx.sid))

profile = mem.get_profile(ctx.user_id)  # LTM (optional)
personalize(ctx, profile)

plan = decide_next(ctx, nlu)            # (1.3)
out  = nlg.render(plan, ctx, profile)   # (1.2)
```

---

### 🧪 Tình huống tiêu biểu

#### **1) Chuỗi rút gọn nhiều lượt**
```
U: “Căn 2PN q7 dưới 3 tỷ” → STM ghi slots.
U: “tầm 70m²+” → merge dien_tich>=70.
U: “thêm ban công” → thêm tien_ich={"ban_cong":true}.
→ NLG không hỏi lại slot đã rõ.
```

#### **2) Chỉnh sửa & phủ định**
```
U: “Không Q7 nữa, chuyển Thủ Đức” → xoá khu_vuc=Q7, set Thủ Đức.
→ NLG: “Đã chuyển sang Thủ Đức. Em lọc trong tầm <3 tỷ, 2PN, ≥70m².”
```

#### **3) Cá nhân hoá đa phiên (có consent)**
```
Phiên trước: user hay chọn cuối tuần.
Phiên này: gợi ý “Xem lịch thứ 7/Chủ nhật”.
```

---

### 📈 KPI gợi ý
- **Redundant-ask rate:** < 5%.
- **Auto-fill rate:** > 40% (MVP), > 60% (mở rộng).
- **Correction success:** > 95%.
- **Consent clarity:** ≥ 4.5/5.