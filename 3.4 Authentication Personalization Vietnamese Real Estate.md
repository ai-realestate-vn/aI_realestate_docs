## 🔐 Chức năng 3.4 – Authentication / Personalization (Xác thực & Cá nhân hoá)

### 🎯 Mục tiêu
- **AuthN + AuthZ:** bảo mật truy cập và phân quyền theo RBAC/ABAC.
- **Đa kênh:** Web chat, mobile, API partner.
- **Cá nhân hoá có consent:** lưu sở thích/locale/hành vi; không lưu PII nhạy cảm nếu chưa đồng ý.

---

### 🧱 Sơ đồ tổng quan
```
[Clients: Web/Mobile/Partner]
           │  (OAuth2/OIDC, Email OTP, Magic Link)
           ▼
   [Auth Gateway / IdP]
           │ issues JWT (short) + Refresh (long)
           ▼
[API Gateway / Chat Orchestrator]
           │  verify JWT → enforce RBAC/ABAC
           ▼
     [Services / DAL 3.1 / RAG 3.2 / Actions 3.3]
           │
           └─► Personalization Service (Profile + Consent + Preferences)
```

---

### 🔐 Luồng xác thực (AuthN)
**A) Người dùng cuối (buyer/investor)**
- **Đăng nhập:** Magic Link / Email OTP (MVP); OIDC Social (Google/Apple) nếu cần.
- **Token:** Access (JWT RS256, TTL 15–30m) + Refresh (TTL 7–30d, rotating + revoke list).
- **Guest → Registered:** bắt đầu với `session_id`. Khi đăng nhập, **merge** lịch sử hội thoại vào `user_id`.

**B) Đối tác / Agent / Admin**
- OIDC (Auth Code) hoặc Client Credentials (server-to-server).
- **MFA** (TOTP/WebAuthn) bắt buộc với admin.
- **IP allowlist** cho dashboard quản trị, webhook ingress.

---

### 🧾 Cấu trúc JWT (ví dụ)
```json
{
  "iss":"https://idp.example.com",
  "sub":"u_123",
  "aud":"chat-api",
  "exp":1730563200,
  "iat":1730561400,
  "nonce":"ulid",
  "sid":"s_ulid",
  "roles":["buyer"],
  "scopes":["chat:send","listings:read","booking:create"],
  "tenant":"default",
  "attrs":{ "channel":"web", "locale":"vi-VN", "tier":"free" }
}
```

---

### 🛂 Phân quyền (AuthZ)
**RBAC (role-based)**
- buyer/investor: `chat:send`, `listings:read`, `booking:create`, `profile:read/update`.
- broker/agent: thêm `lead:read`, `booking:list/all`, (tuỳ nội bộ) `listings:write`.
- admin: `*` (giới hạn theo tenant), truy cập log/audit.

**ABAC (attribute-based)**
- Ràng buộc: `tenant == token.tenant`; `resource.owner_id == token.sub`;
  `role==broker && agency_id==listing.agency_id` mới sửa listing.
- Engine: Casbin/OPA hoặc policy nội bộ (YAML/rego).

**Ví dụ policy (Casbin-like):**
```
p, buyer, listings, read, tenant==req.tenant
p, buyer, booking, create, tenant==req.tenant
p, broker, listings, write, tenant==req.tenant && subject.agency_id==object.agency_id
p, admin, *, *, tenant==req.tenant
```

---

### 🙋 Cá nhân hoá (Personalization)
**Dữ liệu hồ sơ (Profile)**
```json
{
  "user_id":"u_123",
  "honorific":"Anh/Chị",
  "preferred_locale":"vi-VN",
  "preferred_contact":{"phone":"+84...", "email_masked":"m***@x.com"},
  "preferences":{
    "area_favs":["quận 7","thủ đức"],
    "bedrooms":[2],
    "avoid_orientations":["tây"],
    "visit_days":["sat","sun"],
    "budget_max_vnd":3000000000
  },
  "consent":{
    "personalization": true,
    "data_improvement": false,
    "email_marketing": false
  },
  "meta":{"created_at":1730000000, "retention_days":365}
}
```
**Nguyên tắc**
- **Opt-in rõ ràng** theo mục đích: personalization, model improvement (1.5), marketing.
- **Data minimization:** chỉ lưu field ảnh hưởng trải nghiệm.
- **Right to Forget/Export:** `DELETE /v1/profile`, `GET /v1/profile/export`.

**Điểm nối với 1.4 (Memory)**
- Khi `consent.personalization=true`: nạp profile → tiền xử lý **NLG 1.2** (xưng hô, locale) và **Policy 1.3** (mặc định bộ lọc). Cập nhật `preferences` theo click/booking.
- Khi `consent=false`: chỉ dùng **STM** trong phiên (không nhớ đa phiên).

---

### 🌐 Đa ngôn ngữ & định dạng (phối hợp 2.5)
- Lấy `preferred_locale` → định dạng tiền/diện tích/ngày với `Intl.*`.
- Khi user đổi ngôn ngữ trong phiên → cập nhật `last_user_locale` (STM), không bắt buộc đổi profile.

---

### 🔒 Bảo mật & tuân thủ
- **HTTPS bắt buộc**, HSTS.
- **CORS:** allowlist origin; cookie `SameSite=Lax` nếu dùng cookie.
- **CSRF:** token cho endpoint thay đổi trạng thái (khi dùng cookie).
- **PII:** ẩn/băm trong log; tách bảng PII + **encryption-at-rest**.
- **Rate limit** theo `sub + ip + session_id`.
- **Audit:** ghi truy cập dữ liệu nhạy cảm, thay đổi consent.
- **Retention:** TTL theo `retention_days`; **auto-purge**.
- **MFA** bắt buộc với admin/agent quan trọng.

---

### 🔌 API đề xuất
**Auth**
- `POST /auth/magic-link` → gửi link/OTP.  
- `POST /auth/verify` → đổi OTP/Magic link lấy access+refresh.  
- `POST /auth/refresh` → cấp access mới (rotate refresh).  
- `POST /auth/logout` → revoke refresh.

**Profile & Consent**
- `GET/PUT /v1/profile` (whitelist field).  
- `GET/PUT /v1/profile/consent`.  
- `DELETE /v1/profile` (xoá toàn bộ) / `GET /v1/profile/export`.

**Session**
- `GET /v1/session` → `sid`, `last_user_locale`, `features`.

---

### 🧩 Middlewares (Gateway)
- **JWT Verify** (kid, JWKS cache, clock skew ±60s).  
- **Tenant Resolver** (từ domain/host header hoặc claim).  
- **RBAC/ABAC Enforcer** (Casbin/OPA).  
- **Consent Guard**:
  - Nếu `consent.personalization=false` → tắt nạp LTM, chỉ dùng STM.  
  - Nếu gọi 1.5 (learning) mà `data_improvement=false` → không ghi sự kiện gắn user_id; chỉ thống kê ẩn danh.  
- **PII Scrubber:** che PII trong log/telemetry.

---

### 🛠️ Pseudo (Express/FastAPI)
**Verify + Enforce**
```ts
app.use(async (req, res, next) => {
  const token = extractBearer(req);
  const claims = await verifyJWT(token, jwks);
  req.user = claims;

  const allowed = await enforcer.enforce(claims.roles, req.resource, req.action, {tenant:req.tenant, subject:claims, object:req.object});
  if (!allowed) return res.status(403).json({error:"forbidden"});
  next();
});
```

**Consent guard cho Personalization**
```ts
app.get("/v1/profile", async (req, res) => {
  const prof = await profileRepo.get(req.user.sub);
  const out = filterWhitelisted(prof);
  res.json(out);
});
```

---

### 🧪 Test & KPI
**Kiểm thử**
- **TC01:** Magic link/OTP → nhận access/refresh; refresh rotate hoạt động.  
- **TC02:** RBAC/ABAC – buyer không thể gọi `listings:write`.  
- **TC03:** Consent off → không nạp LTM; on → NLG đổi xưng hô, quick replies ưu tiên cuối tuần.  
- **TC04:** Revoke refresh → token cũ không dùng lại được.  
- **TC05:** Admin bắt buộc MFA; thiếu MFA → 401.

**KPI**
- Auth success rate > 98%.  
- Token verification latency p95 < 20 ms (JWKS cache).  
- Privacy incidents = 0; PII-in-log = 0.  
- Personalization uplift: CTR CTA +10–15%, giảm số lượt đến hành động ≥ 15–20%.

---

### ✅ Checklist MVP 3.4
- [x] IdP/OIDC hoặc module auth (magic link/OTP + JWT RS256 + refresh rotate)  
- [x] RBAC vai trò: buyer, broker, admin; ABAC theo tenant/agency/owner  
- [x] Consent model + API; Personalization Service (profile/preferences)  
- [x] Middlewares: JWT verify, tenant resolver, enforcer, consent guard, PII scrubber  
- [x] Rate limit, CORS/CSRF, encryption-at-rest, audit log  
- [x] Contract tests + e2e cho guest→login→merge session, consent on/off  
- [x] Tích hợp 1.4 (Memory), 1.5 (Learning), 2.5 (i18n), 3.1/3.2/3.3 (services)

