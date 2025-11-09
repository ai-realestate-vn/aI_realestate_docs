# 6.3 – Memory dài hạn (Persistent Memory)

> **Mục tiêu:** giúp chatbot “nhớ bền vững” những gì **nên nhớ** để cá nhân hoá tốt hơn qua nhiều phiên, nhưng vẫn **an toàn – minh bạch – xoá được** (tuân thủ 5.1–5.3).

---

## 🎯 Mục tiêu & Phạm vi
- **Lưu – truy xuất – cập nhật**: sở thích bền, hồ sơ nhẹ, thói quen tương tác, tóm tắt dài hạn → tăng tỉ lệ hoàn tất tác vụ, giảm hỏi lặp.
- **Không lưu**: PII nhạy cảm/chi tiết (địa chỉ chính xác, số giấy tờ…), nội dung nhất thời (OTP), dữ liệu không có consent.

---

## 🧱 Lớp Memory (4 tầng dài hạn)
1) **Profile nhẹ (Identity-lite)**  
   `display_name`, `xưng_hô`, `locale`, `kênh_ưa_thích` (text/voice), `timezone`.

2) **Preferences (Sở thích bền)**  
   - BĐS: `khu_ưa_thích`, `ngân_sách_vùng`, `PN_ưa_thích`, `tiện_ích_bắt_buộc/né`, `ngày_hẹn_ưa_thích`, `kênh_liên_lạc`.
   - Học CNTT: `ngôn_ngữ_học`, `mục_tiêu_kỳ_thi`, `thời_lượng_mỗi_buổi`.

3) **Interaction Signals (Hành vi)**  
   Lịch sử click/CTA, tỷ lệ phản hồi, khung giờ hay hoạt động.

4) **Episodic Summaries (Tóm tắt nhiều phiên)**  
   “Mẩu” 1–3 câu tổng hợp sau mỗi phiên thành **long‑term digest** *(không chứa PII thô)*.

> **TTL mặc định:** 6–12 tháng (config theo `purpose=personalization`), xoá theo 5.3 khi người dùng yêu cầu.

---

## 🗃️ Lược đồ dữ liệu (gợi ý)
```json
{
  "user_id": "u-123",
  "tenant_id": "t-abc",
  "consent": true,
  "consent_scopes": ["personalization","recommendations"],
  "profile": {
    "display_name": "Minh",
    "honorific": "Anh",
    "locale": "vi-VN",
    "timezone": "Asia/Ho_Chi_Minh",
    "channels": ["web","zalo"]
  },
  "preferences": {
    "real_estate": {
      "khu_ua_thich": ["Quận 7","Thủ Đức"],
      "ngan_sach_vnd": {"min": 2000000000, "max": 3200000000},
      "phong_ngu": [2],
      "tien_ich_bat_buoc": ["ban_cong","view_song"],
      "tien_ich_ne": ["huong_tay"],
      "ngay_hen_ua_thich": ["Thứ 7","Chủ nhật"],
      "lien_lac": "zalo"
    },
    "learning": {
      "muc_tieu": "Python cơ bản → Data Analyst",
      "lich": "tối 3-5-7",
      "buoi": 45
    }
  },
  "signals": {
    "cta_click_rate": 0.42,
    "active_hours": ["19:00-22:00"],
    "last_projects_viewed": ["PMH-Sunrise","EcoGreen"]
  },
  "digest": [
    {"ts": "2025-10-30T21:10:00+07:00", "text": "Ưa Q7, 2PN, <3.2 tỷ; thích cuối tuần; tránh hướng Tây."}
  ],
  "meta": {"version": 3, "retention_days": 365}
}
```

---

## 🧭 Luồng runtime (Ghi – Đọc – Áp dụng)
### A. Nhận diện & Đồng ý
- Lần đầu/đăng nhập: hiển thị **banner consent** (mục đích, TTL, xoá được).
- `consent=false` → **không ghi LTM**; chỉ dùng STM (1.4).

### B. Ghi nhớ (Write Path)
- **Observe**: sau tác vụ thành công/clarify rõ ràng → trích facts bền.
- **Normalize**: map alias (PMH→Phú Mỹ Hưng), chuẩn hoá tiền/đơn vị.
- **Score & Update** *(confidence + recency + engagement)*:  
  `score = 0.5*new_conf + 0.3*recency + 0.2*engagement_weight`  → chỉ ghi nếu **delta ≥ ngưỡng**.
- **Summarize**: cập nhật digest 1–2 câu/phiên *(không PII)*.

### C. Truy xuất (Read Path)
- Orchestrator (1.3) tải LTM (nếu consent) → tự **tiền điền slot**, điều chỉnh **tone/xưng hô**, gợi ý **quick replies** (VD: “Xem lịch cuối tuần”).

### D. Quên có chủ đích (Decay/Prune)
- Giảm trọng số sở thích nếu không được nhắc ≥ *N* phiên.  
- Ưu tiên xoá preferences ít liên quan (VD: đã đổi khu).

---

## ⚖️ Quy tắc hợp nhất (Merge Rules)
- **Priority:** user override > explicit selection > inferred by behavior > history.
- **Conflicts:** user nói “không Q7 nữa, chọn Thủ Đức” → đánh dấu **deprecated** Q7 (không xoá ngay), giảm trọng số Q7 xuống `0.2`, tăng Thủ Đức lên `0.9`.
- **Multi‑tenant isolation:** mọi bản ghi có `tenant_id`; **không chia sẻ chéo tenant**.

---

## 🔒 Riêng tư & Tuân thủ
- Chỉ lưu **facts bền** được người dùng xác nhận qua lời nói/hành vi; **không suy diễn nhạy cảm** (thu nhập, tôn giáo…).
- **Không lưu PII thô** trong LTM (email/sđt/địa chỉ chính xác) — nếu cần, dùng **token/vault** (5.2).
- **DSAR:** `GET /privacy/export` (JSON/ZIP) & `POST /privacy/delete` (xóa LTM + liên quan).
- **Retention:** `retention_days` theo 5.3; hết hạn → **purge tự động**.

---

## 🔌 API gợi ý (FastAPI pseudo)
```python
@app.get("/memory")
def get_memory(user_id: str, tenant_id: str):
    mem = ltm.get(tenant_id, user_id)
    return filter_by_scope(mem, scope="personalization")

@app.post("/memory/upsert")
def upsert_prefs(user_id: str, tenant_id: str, patch: dict):
    mem = ltm.get_or_create(tenant_id, user_id)
    mem = merge(mem, patch, strategy="confidence_recency")
    ltm.save(tenant_id, user_id, mem)
    return {"ok": True, "version": mem["meta"]["version"]+1}

@app.post("/privacy/consent")
def set_consent(user_id: str, consent: bool, scopes: list[str]):
    prof = ltm.get_meta(user_id)
    prof["consent"] = consent
    prof["consent_scopes"] = scopes
    ltm.save_meta(prof)
    if not consent:
        ltm.delete_all(user_id)  # giữ nguyên booking theo luật/5.3
    return {"ok": True}
```

---

## 🔧 Tích hợp với các mô-đun khác
- **1.1 (NLU):** khi thiếu slot → ưu tiên gợi ý dựa LTM (VD: mặc định `khu_ua_thich[0]`).
- **1.2 (NLG):** giọng điệu + CTA phù hợp (VD: ưu tiên “cuối tuần”).
- **1.3 (DM):** giảm lượt clarify bằng auto‑fill slot hợp lý; tránh hỏi lại.
- **6.1 (RAG):** filter mặc định theo khu ưa thích/tenant trước truy vấn.
- **5.1–5.3:** AuthZ kiểm soát quyền đọc/ghi; PII mask; retention & DSAR.

---

## 🧪 Tình huống tiêu biểu
**Cá nhân hoá BĐS**  
Sau 2–3 phiên, user luôn click **2PN, <3.2 tỷ, Q7/PMH, né hướng Tây**.  
→ Lần sau chỉ hỏi “căn 2PN”: hệ thống **tự áp** Q7/PMH + gợi ý “tăng +0.2 tỷ nếu khan hiếm”.

**Học CNTT**  
User theo lộ trình “Python cơ bản → Pandas”, hay học **tối 3‑5‑7**.  
→ Tuần sau: gợi ý “**Ôn lại Pandas 20’ tối nay?**”.

---

## 📈 Chỉ số & Mục tiêu
- **Redundant‑ask rate** (hỏi lại thông tin đã biết): `< 5%`.
- **Auto‑fill rate** (dùng LTM để điền slot): `≥ 50%` sau 4 tuần.
- **Task success** (đặt lịch/mở chi tiết/hoàn thành bài học): `+5–10%` so baseline.
- **Opt‑in rate** (đồng ý cá nhân hoá): `≥ 65%`.
- **DSAR SLA:** Export `≤ 7 ngày`, Delete `≤ 15–30 ngày`.

---

## 🧰 Triển khai nhanh (MVP 1–2 tuần)
- **Schema LTM** (ở Postgres JSONB hoặc RedisJSON + snapshot).
- **Consent flow** (banner + API) + policy: chưa consent → **chỉ STM**.
- **Merge engine** (confidence + recency + override), unit test ≥ 15 case.
- **Summarizer** tạo digest `≤ 120 ký tự/phiên` (không PII).
- **Readers**: hàm `personalize(ctx, ltm)` cho 1.1/1.2/1.3/6.1.
- **Privacy hooks**: DSAR export/delete + retention cron (theo 5.3).
- **Dashboard**: auto‑fill rate, redundant‑ask, task success, opt‑in.

---

## 🔎 Anti‑creepiness (tránh cảm giác “rình rập”)
- Chỉ nhắc lại **thông tin user từng nói rõ**:
  > “Anh hay đi xem nhà cuối tuần, em ưu tiên lịch T7/CN nhé?”
- **Không suy diễn** cá nhân nhạy cảm; luôn cho **tắt cá nhân hoá** dễ dàng.
- Thông báo khi dùng dữ liệu cũ để gợi ý:  
  > “Dựa trên lần trước, em lọc sẵn Q7/2PN <3.2 tỷ, **có đổi không ạ?**”

