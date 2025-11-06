## 🛡️ Chức năng 5.2 – Ẩn danh/Giảm định danh dữ liệu cá nhân (Pseudonymization & Anonymization)

### 🎯 Mục tiêu & Phạm vi
**Mục tiêu:** Tự động phát hiện – phân loại – che/mã hoá – kiểm soát vòng đời dữ liệu cá nhân trong mọi luồng: **nhập liệu → xử lý → lưu trữ → huấn luyện → chia sẻ**.

**Phạm vi:**
- **PII trực tiếp:** tên, email, số điện thoại, CCCD/CMND, địa chỉ cụ thể…
- **PII gián tiếp:** ID phiên, cookies, IP…
- **Dữ liệu nhạy cảm miền BĐS:** số nhà, số sổ đỏ, số hợp đồng, toạ độ căn…

---

### 🧱 Kiến trúc lớp bảo vệ (Defense-in-Depth)
1) **Lớp phát hiện (Detection)**  
   Regex + từ điển (gazetteer) + heuristic (địa chỉ VN) + **ML NER** (tên riêng, địa danh).

2) **Lớp xử lý (Transform)**  
   **Masking** (che một phần), **Tokenization** (token tra ngược có kiểm soát), **Hash+Salt** (không thể khôi phục), **Encryption** (AES/GCM, **FPE** cho số).

3) **Lớp chính sách (Policy)**  
   Quy tắc theo **mục đích sử dụng** (chat runtime, analytics, training, sharing).

4) **Lớp kiểm soát truy cập**  
   Field-level access + **masking theo quyền** (buyer/broker/admin).

5) **Lớp vòng đời**  
   **TTL/retention**, **DSAR (export/delete)**, **audit** bất biến.

---

### 🗺️ Luồng xử lý dữ liệu (E2E)
**A. Ingress (trước NLU)**
- Nhận `user_text` → **PII detector** chạy đồng bộ.
- Gắn nhãn `data_tags` cho từng span (EMAIL, PHONE, CCCD, ADDRESS_EXACT, GEO, BANK…).
- Áp dụng policy theo kênh:
  - **Runtime:** giữ nguyên cho NLU, **nhưng** sinh **bản sao đã che** để ghi log.
  - **No-PII mode (tuỳ kênh):** che ngay trong request (privacy-first).

**B. Processing (NLU/RAG/Tools)**
- Truyền **bản gốc** chỉ nội bộ pipeline; **bản đã che** cho logging/analytics.
- Khi gọi **RAG/Tool ngoài**: **minimize** — gửi **chỉ** trường cần, ưu tiên **token hóa**.

**C. Egress (Logs, Datasets, Sharing)**
- Event Logger **luôn** ghi `masked_text` + **hashes** thay vì giá trị thô.
- **ETL huấn luyện:** thêm bước **scrub** (loại/che PII) + đánh dấu **consent**.
- **Dataset export:** enforce policy “no direct identifiers”, **scan lại** → ký **biên bản kiểm tra (QA)**.

---

### 🔎 Danh mục PII (VN) & Cách xử lý
| Loại | Ví dụ | Mặc định Log | Cho phân tích | Cho huấn luyện | Cho chia sẻ |
|---|---|---|---|---|---|
| Email | ten@abc.com | `t***@abc.com` | `hash(salt)` | `hash(salt)` | **remove** |
| SĐT VN | 09x/03x… (10 số) | `09*****123` | `token(FPE)` | `token(FPE)` | **remove** |
| CCCD/CMND | 12/9 số | `****(last4)` | **remove** | **remove** | **remove** |
| Địa chỉ cụ thể | Số 12/3 Lê Lợi, P.4… | làm mờ số nhà | **generalize** (phường/quận) | **generalize** | generalize/**remove** |
| Toạ độ | 10.762…, 106.68… | làm tròn 3–4 số | làm tròn 2–3 | **remove** nếu không cần | **remove** |
| Tên cá nhân | “Anh Nam”, “Chị Hoa” | giữ xưng hô, ẩn tên | ẩn/role | ẩn/role | ẩn/role |
| Số hợp đồng/biển số | HĐ-2025-… | mask trung tâm | token | token | **remove** |
| IP/UA | 113.x.x.x | /24 subnet | /16 subnet | **remove** | **remove** |

*`hash(salt)` = SHA-256 + **salt luân phiên** (key vault). `token(FPE)` = **Format-Preserving Encryption** (giữ định dạng số điện thoại/biểu mẫu).*

---

### 🧩 Bộ dò PII – Regex/Heuristic (VN, gợi ý)
- **Email:** `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b`
- **SĐT VN (10 số):** `\b(0|\+84)(3|5|7|8|9)\d{8}\b`
- **CCCD 12 số:** `\b\d{12}\b` *(kèm ngữ cảnh: “CCCD”, “Căn cước”)*
- **CMND 9 số:** `\b\d{9}\b` *(kèm ngữ cảnh: “CMND”)*
- **Địa chỉ (heuristic):** từ khoá *Số/Ngõ/Hẻm/Đường/Phố/Thôn/Ấp, P.\d+|Phường|Xã|Quận|Huyện|TP|Tỉnh*
- **Toạ độ:** `\b-?\d{1,2}\.\d{3,},\s*-?\d{1,3}\.\d{3,}\b`

Kết hợp **NER** (tên người/địa danh) để giảm false positive; dùng **denylist context** (“số căn”, “số nhà”, “lô”, “thửa”) để bắt địa chỉ BĐS.

---

### 🧪 Ví dụ che dữ liệu (Masking Examples)
**Input**  
“Em tên Nguyễn Minh Anh, sđt 0901234567, hẹn xem căn 12/3 Lê Lợi, P.4, Q.3 vào 3pm.”

**Log (masked)**  
“Em tên **[PERSON]**, sđt **09*567**, hẹn xem **[ADDR: Lê Lợi, P.4, Q.3]** vào 3pm.”

**Analytics (hashed/tokenized)**  
`phone_hash = sha256("0901234567" + salt)`  
`addr_level = Q.3` *(drop số nhà)*

---

### 🔐 Chiến lược chuyển đổi (Transform Strategy)
- **Masking** — hiển thị cho người có quyền nhưng che phần nhạy cảm.
- **Tokenization** — ánh xạ 1–1 bằng **token vault** (giải hoàn nguyên có kiểm soát).
- **Hash+Salt** — thống kê/so khớp **không thể khôi phục**.
- **Encryption** — lưu trữ PII thô (nếu bắt buộc) bằng **AES-256-GCM**, khóa trong **KMS/Secret Manager**.
- **FPE** — giữ định dạng (đặc biệt số điện thoại/thẻ) để UI/validation vẫn hoạt động.

---

### 🧭 Policy theo mục đích sử dụng (Purpose Binding)
```yaml
purposes:
  runtime_chat:
    allow: [masking, fpe]
    deny:  [export_raw]
  analytics:
    allow: [hash, generalize_region]
    deny:  [plain_phone, exact_address]
  training:
    allow: [remove_direct_identifiers, generalize_address]
  sharing_external:
    allow: []
    deny:  [any_pii]  # bắt buộc vô danh hóa hoàn toàn
```

---

### 🔌 API & Hook gợi ý
**Middleware (Node/TS pseudo)**
```ts
app.post("/chat", piiScan, purposeEnforce("runtime_chat"), (req,res,next) => next());

function piiScan(req,res,next){
  const text = req.body.text;
  const spans = detectPII(text); // regex+NER
  req.pii = spans;
  req.masked_text = mask(text, spans);
  next();
}

function purposeEnforce(purpose){
  return (req,res,next) => {
    const pol = policies[purpose];
    if (pol.deny.includes("export_raw")) req.body.forbid_raw_export = true;
    // chỉ ghi log bản đã che
    req.body._log_text = req.masked_text;
    next();
  };
}
```

**Event Logger (chỉ ghi bản đã che)**
```json
{
  "ts": 1730431200,
  "session_id": "s-uuid",
  "user_id_hash": "u-***",
  "masked_text": "Em tên [PERSON], sđt 09*****567, hẹn xem [ADDR: Q.3]...",
  "pii_tags": ["PHONE","PERSON","ADDRESS"],
  "channel": "web",
  "purpose": "runtime_chat"
}
```

---

### 🗄️ Lược đồ DB & kiểm soát quyền (gợi ý)
- `events(masked_text, pii_tags[], user_id_hash, purpose, ...)` — **không** lưu text gốc.
- `pii_vault(token, type, value_encrypted, created_by, access_scope)` — bảo vệ bằng **KMS**, truy cập **theo vai trò**, **full audit**.
- **Field-level masking** trong bảng nghiệp vụ (Postgres/RLS):
  - Cột `owner_phone`: view cho broker/admin trả `xxx***xxx` nếu `kyc_level < 2`.

---

### ⏳ TTL/Retention & DSAR (Yêu cầu chủ thể dữ liệu)
- **TTL log:** mặc định **90 ngày** (có thể 30/180 tuỳ rủi ro).
- **TTL dataset training:** gắn **round + manifest**; giữ **12 tháng**; xóa theo **DSAR**.
- **DSAR API:**
  - `GET /privacy/export` — trả JSON/ZIP dữ liệu cá nhân (**đã che** giá trị nhạy cảm của bên thứ ba).
  - `POST /privacy/delete` — xóa trong **STM/LTM**, **vault**, **datasets** (đánh dấu **purge** trong manifest & chạy job).

---

### 🧰 Tích hợp với các chức năng khác
- **1.1/1.2/1.3/1.4:** memory & context chỉ lưu **phiên bản đã giảm định danh**, ngoại trừ thông tin user chủ động cung cấp cho **giao dịch** (ví dụ số điện thoại đặt lịch) → lưu ở **vault**, hiển thị theo **quyền**.
- **5.1:** AuthZ quyết định có cho **unmask** hay không (*OBLIGATIONS: mask theo role*).
- **1.5 (Learning):** pipeline training lấy **scrubbed dataset** từ ETL; log guardrail **loại** turn nhạy cảm.

---

### 📈 KPI & Giám sát
- **PII Leak Rate (prod logs):** ~0% (watchdog regex quét đêm).
- **Mask Coverage:** ≥ **99%** hits đúng loại (precision/recall theo mẫu vàng).
- **False Positive:** < **3%** (địa chỉ/tên riêng).
- **DSAR SLA:** Export ≤ **7 ngày**, Delete ≤ **15 ngày**.
- **Access Violations:** **0** sự cố unmask trái quyền/tháng.

---

### ✅ Checklist MVP (1–2 tuần)
- [x] Bộ **regex/NER PII VN** + unit test (email, sđt, CMND/CCCD, địa chỉ, toạ độ).
- [x] Middleware **piiScan** + **purposeEnforce** + **logger** chỉ ghi `masked_text`.
- [x] **Token vault** (FPE/Encrypt) + **audit truy cập** + **field-level masking**.
- [x] **ETL scrub** cho training/analytics + **manifest dataset** (round/version).
- [x] **TTL/retention jobs** + **DSAR endpoints** (`/privacy/export`, `/privacy/delete`).
- [x] **Dashboard** giám sát PII leak & coverage.
- [x] **Tài liệu privacy (VN)** + **banner consent** (opt-in personalisation).

