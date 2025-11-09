## 🔐 Chức năng 5.1 – Xác thực & Phân quyền (AuthN/AuthZ)

### 🎯 Mục tiêu
- **AuthN (Xác thực):** đảm bảo “bạn là ai?” (login, SSO, 2FA, token).
- **AuthZ (Phân quyền):** đảm bảo “bạn được phép làm gì?” (RBAC/ABAC, policy).
- **Phi chức năng:** an toàn (OWASP ASVS), nhanh, dễ mở rộng, tuân thủ **NĐ 13/2023 (VN)**.

---

### 🧱 Kiến trúc tổng quan
```
Client (Web/Chat UI/Apps)
   ↔ API Gateway (rate limit, WAF)
      ↔ Auth Service (OIDC Provider, token, session)
         ↔ Resource APIs (Listings, Booking, RAG)
            ↔ Policy Engine (PDP/OPA)
               ↔ DB/VectorDB
```
- **Kho Bí mật (Secret Manager):** khóa JWT/Private key, TOTP secret, SMTP/SMS keys.

---

### 👤 Vai trò & mô hình quyền
- **RBAC cơ bản:** `buyer`, `investor`, `broker`, `admin`, `superadmin` (multi-tenant: thêm `tenant_admin`).
- **ABAC mở rộng:** thuộc tính `tenant_id`, `region`, `verified_broker=true`, `kyc_level`, `data_sensitivity`.
- **Mô hình kết hợp:** RBAC cho **coarse-grained**, ABAC/policy cho **fine-grained**.

---

### 🗄️ Lược đồ dữ liệu (tối thiểu)
```sql
-- users & sessions
users(
  id, email, phone, password_hash, status, tenant_id,
  mfa_enabled, mfa_secret, last_login_at
)

roles(id, name, tenant_scope)
user_roles(user_id, role_id)

permissions(id, action, resource)
role_permissions(role_id, permission_id)

sessions(
  session_id, user_id, device_id, ip, ua, created_at, revoked_at
)

refresh_tokens(
  refresh_token_hash, user_id, session_id, exp, rotated_from, revoked
)

policy_rules(
  id, effect(allow/deny), action, resource, conditions JSON
)
```

---

### 🔐 Chu kỳ AuthN (JWT/OIDC) – chuẩn triển khai
**Đăng nhập**
- Email/phone + password → kiểm tra `status`, hash **argon2id/bcrypt(12+)**, rate limit + kiểm tra rủi ro IP.
- Nếu `mfa_enabled`: yêu cầu **TOTP** (Authenticator) hoặc **OTP SMS** (fallback).

**Cấp token**
- **Access Token** (JWT, 5–15 phút): `sub, tenant_id, roles, scopes, iat, exp`.
- **Refresh Token** (opaque, 30–90 ngày): **lưu hash** trong DB, **rotate** mỗi lần dùng.

**Làm mới**
- Client gửi refresh → kiểm tra **chưa thu hồi** + **chưa dùng trước** (rotation).
- Cấp **cặp token mới**; **vô hiệu** refresh cũ (chống replay).

**Đăng xuất**
- Xoá session, revoke refresh; access **tự hết hạn** (stateless).

**Magic Link (tùy chọn)**
- Onboarding nhanh: link 1 lần (5–10 phút), scope hạn chế; yêu cầu đặt mật khẩu nếu cần full.

**SSO/OIDC**
- Google (email verified), Zalo/Facebook (ràng buộc phone), đồng bộ vào `users` + ràng buộc `tenant_id`.

---

### 🧾 Chu kỳ AuthZ (RBAC/ABAC + Policy Engine)
- **PEP** (Policy Enforcement Point): tại API Gateway/Resource API.
- **PDP** (Policy Decision Point): Policy Engine (OPA/Rego hoặc module custom).
- **Input đánh giá:** `user.claims` (roles, tenant_id, kyc_level), `resource.attrs` (owner_tenant, sensitivity), `action`, `context` (ip, time, device_trust).
- **Quyết định:** `allow/deny` + **obligations** (ví dụ: mask PII nếu không đủ cấp).
- **Audit:** lưu mọi quyết định quan trọng (admin ops, dữ liệu nhạy cảm).

**Ví dụ policy**
- `action=READ resource=listing`: **allow** nếu `tenant_id` trùng **hoặc** `listing.visibility='public'`.
- `action=READ resource=owner_phone`: **allow** nếu `role ∈ {broker, admin}` **và** `kyc_level ≥ 2`.

---

### 🧭 Luồng điển hình (sequence)
**A) Email/Password + TOTP**
1. Client → `/auth/login`: email+pass  
2. Auth Service: verify + risk checks → `mfa_required=true`  
3. Client → `/auth/mfa/verify`: TOTP  
4. Auth: cấp `access_jwt` + `refresh_token`, tạo **session**  
5. Client gắn `Authorization: Bearer <jwt>` khi gọi API

**B) Refresh rotation**
- Client gửi refresh → Auth kiểm tra `revoked=false` & `rotated_from=null` → cấp cặp mới & set `rotated_from` cũ → **revoke** cũ.

**C) Gọi API tài nguyên**
- Client → `/listings?region=q7`  
- API: PEP giải mã JWT → **hỏi PDP** → allow + **field-level filter** (mask phone nếu thiếu quyền).

---

### 🛡️ Biện pháp an toàn (then chốt)
- **Mật khẩu:** argon2id/bcrypt(12+), password-policy + HaveIBeenPwned check (nếu có).
- **MFA:** TOTP mặc định; SMS chỉ fallback. **Recovery codes** (10 mã 1 lần).
- **JWT:** ký **RS256/EdDSA**; `kid` + **key rotation (JWKS)**. Ngắn hạn (≤ 15’), thêm `jti`.
- **Refresh:** **opaque + hash** trong DB, **ROTATION BẮT BUỘC**, **reuse detection**.
- **Session binding:** gắn `device_id` + IP(ASN) + UA fingerprint (nhẹ) → phát hiện bất thường.
- **Rate limiting:** login/OTP/refresh → 5–10 req/min/account; WAF **block IP** độc hại.
- **PII minimization:** tách bảng PII, **column-level encryption** (phone, exact address).
- **Audit trail:** ghi who/when/what cho hành động nhạy cảm; **immutable log** (WORM/S3 Object Lock).
- **CSP/CSRF/XSS:** nếu dùng cookie (SameSite=strict + HttpOnly + Secure); hoặc **pure Bearer** qua header.
- **Secrets:** không lưu plain; dùng **Secret Manager/ENV**; **không commit**.

---

### 🔍 Phân quyền chi tiết (mẫu)
| Resource | Action | buyer | investor | broker | admin |
|---|---|:--:|:--:|:--:|:--:|
| listings | read_public | ✅ | ✅ | ✅ | ✅ |
| listings | read_owner_phone | ❌ | ❌ | ✅ (kyc2+) | ✅ |
| bookings | create | ✅ | ✅ | ✅ | ✅ |
| bookings | approve | ❌ | ❌ | ✅ | ✅ |
| users | invite | ❌ | ❌ | ❌ | ✅ |
| policies | update | ❌ | ❌ | ❌ | ✅ |

**ABAC bổ sung:** `tenant_id` phải trùng; `region` trong danh sách được phép; `data_sensitivity ≤ role_clearance`.

---

### 🧪 Test & Giám sát
**Unit/E2E**
- Login + MFA; refresh rotation + reuse detection; **denial** khi cross-tenant; **field-masking**.

**Security tests**
- Brute force, token replay, **JWT kid swap**, **IDOR**, privilege escalation.

**Metrics**
- Login success %, MFA adoption %, **token reuse detected**, **auth latency p95**, deny/allow ratio theo resource.

---

### 📜 Tuân thủ Nghị định 13/2023 (Việt Nam) – tóm lược thực thi
- **Cơ sở pháp lý & Mục đích:** thông báo khi thu thập; chỉ dùng cho xác thực/cá nhân hoá.
- **Opt-in/Opt-out:** bật/tắt lưu phiên dài hạn, ghi nhớ thiết bị.
- **Quyền chủ thể dữ liệu:** API export/delete; thời hạn lưu rõ ràng.
- **Tối thiểu hoá dữ liệu:** không lưu trữ quá mức (đặc biệt địa chỉ chính xác, giấy tờ).
- **Ghi nhận & báo cáo sự cố:** quy trình ứng phó, nhật ký truy cập.

---

### 🔌 API mẫu (rút gọn)
```
POST /auth/login
POST /auth/mfa/verify
POST /auth/refresh
POST /auth/logout
GET  /auth/jwks.json

GET  /me
GET  /listings                # PEP → PDP
POST /bookings                # PEP → PDP
POST /admin/users/invite      # admin only
POST /admin/policies          # admin only
```

**Pseudo policy check (Node/TS)**
```ts
const decision = await pdp.evaluate({
  subject: { sub, roles, tenant_id, kyc_level },
  action:  "read",
  resource:{ type:"listing", tenant_id: listing.tenant_id, sensitivity: listing.sensitivity },
  context: { ip, time, device_trust }
});
if (!decision.allow) throw new ForbiddenError();
applyObligations(decision, listing); // mask fields if required
```

---

### ✅ Checklist MVP (gợi ý 1–2 tuần)
- [x] Đăng nhập email/password + **TOTP**; argon2id/bcrypt; rate limit.
- [x] **JWT RS256** ngắn hạn + **refresh opaque** có **rotation**.
- [x] **RBAC** 4 vai trò + `tenant_id`; **PEP** tại API; **PDP** đơn giản (rule JSON).
- [x] **Audit log** cho hành động nhạy cảm; **mask PII** theo quyền.
- [x] **Secret management**, **key rotation**, **JWKS endpoint**.
- [x] Bộ test: login, mfa, refresh-reuse, cross-tenant deny, mask field.
- [x] Trang cài đặt người dùng: bật/tắt **MFA**, quản lý thiết bị/phiên.
- [x] Tài liệu privacy (mục đích, TTL, export/delete) theo **NĐ 13**.

