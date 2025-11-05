## 🔔 Chức năng 3.3 – Webhook / Event Trigger

### 🎯 Mục tiêu & Phạm vi
**Mục tiêu:** Orchestrator (1.3) có thể gọi hành động (trigger) và nhận sự kiện (webhook-in) đáng tin cậy, có retry, idempotency, chữ ký (HMAC) và audit.  
**Phạm vi:**
- **Outbound:** chatbot → dịch vụ ngoài (HTTP/gRPC/Event bus)
- **Inbound:** dịch vụ ngoài → chatbot (webhook callback)
- **Job async:** hàng đợi/worker cho tác vụ dài (gửi mail, đồng bộ lịch, xuất PDF…)

---

### 🧱 Kiến trúc tổng quát
```
[Chatbot Orchestrator]
        │  (Trigger API)
        ▼
 [Event Facade / Actions]
        │
        ├──► Sync HTTP (quick action ≤ 800ms)
        ├──► Async Queue (BullMQ/Celery/SQS) → Workers
        └──► Outbox → Webhook Dispatcher (retries + HMAC)
        
  (Inbound)
  Provider/Partner ──► [Webhook Ingress] ─► Verify (HMAC/IP) ─►
                         Dedup/Idempotent ─► Event Router ─► Orchestrator
```

---

### 📜 Hợp đồng dữ liệu
**1) Yêu cầu kích hoạt hành động (Orchestrator → Actions)**
```json
{
  "action": "book_visit",
  "idempotency_key": "c4f1e61d-9d5a-4a7f-9b4f-7c8c8e2a3c6e",
  "user": { "id":"u_123", "name":"Anh Minh" },
  "payload": {
    "listing_id": "listing-SR-2850",
    "preferred_time": "2025-11-02T15:00:00+07:00",
    "contact": { "phone":"+84xxxxxxxxx", "email":"minh@example.com" }
  },
  "context": { "session_id":"s_ulid", "conversation_id":"conv_ulid" }
}
```
**Đáp ứng (sync, nhanh):**
```json
{
  "status": "accepted",
  "request_id": "req_ulid",
  "tracking_url": "https://.../jobs/req_ulid",
  "estimate_ms": 400,
  "next": "await_callback"
}
```

**2) Webhook Callback (Provider → Chatbot)**
- **Headers bắt buộc**
  - `X-Signature: sha256=<hex>` (HMAC body với secret)
  - `X-Request-Id: <uuid>` (idempotent)
  - `X-Provider: partnerA`
- **Body ví dụ**
```json
{
  "event": "booking.confirmed",
  "request_id": "req_ulid",
  "data": {
    "booking_id": "bk_789",
    "listing_id": "listing-SR-2850",
    "time": "2025-11-02T15:00:00+07:00",
    "status": "confirmed",
    "agent": { "name":"Ms. Lan", "phone":"+84..." }
  },
  "occurred_at": "2025-11-01T10:05:30+07:00"
}
```
**Phản hồi của chatbot (bắt buộc)**
```json
{ "received": true, "request_id":"req_ulid" }
```

---

### 🔒 Bảo mật & Tin cậy
- **Chữ ký HMAC (SHA-256):** kiểm tra body + timestamp (chống replay), clock skew ≤ 5 phút.
- **IP allowlist:** chỉ nhận webhook-in từ IP đối tác.
- **Idempotency:** mọi trigger POST có `Idempotency-Key`; lưu dedup window 24–72h.
- **Retry:**
  - Outbound HTTP: 3 lần (100ms, 400ms, 1.2s), chỉ retry 5xx/timeout.
  - Inbound webhook: đối tác retry khi 4xx/5xx; dedup theo `X-Request-Id`.
- **Timeout:** sync ≤ 800ms; nếu lâu → đẩy queue và trả `queued`.
- **Least privilege:** token scoping.
- **PII minimization:** che/băm số ĐT/email trong log.

---

### 🚦 Luồng runtime tiêu chuẩn
**A) Outbound (gọi hành động)**
1. Orchestrator 1.3 sinh Plan `act="BOOK_VISIT"`.
2. Actions Facade kiểm tra `Idempotency-Key` → dedup.
3. Tác vụ nhanh → **sync HTTP**; tác vụ dài → **enqueue** `jobs:book_visit`.
4. Worker gọi API đối tác, retry/backoff, ghi **Outbox** sự kiện.
5. Khi có webhook callback → cập nhật `booking.confirmed` → đẩy message cho người dùng (2.1/2.4).

**B) Inbound (nhận callback)**
1. Webhook Ingress kiểm `HMAC/IP` → reject nếu fail.
2. Dedup `X-Request-Id`.
3. Event Router map `booking.confirmed` → `onBookingConfirmed()`.
4. Gọi NLG 1.2 tạo thông báo + thẻ; cập nhật Memory 1.4.

---

### 🧩 Event Types (chuẩn hoá)
```
booking.confirmed | booking.failed | booking.rescheduled
lead.created | lead.assigned
email.sent | email.bounced | email.opened
doc.generated | doc.signed
payment.authorized | payment.failed
```
*Mỗi event có schema (Zod/JSON Schema) và `event_version: "1.0"`.*

---

### 🧰 Pseudo Implementation
**TypeScript – Actions Facade (Express/Fastify)**
```ts
app.post("/v1/actions/book_visit", auth, async (req, res) => {
  const { idempotency_key } = req.body;
  if (await isDuplicate(idempotency_key)) {
    return res.json({ status:"duplicate", request_id: await getReqId(idempotency_key) });
  }
  const request_id = newUlid();
  await saveIdempotency(idempotency_key, request_id);

  if (isFastPartner()) {
    const ok = await callPartnerQuick(req.body);
    return res.json({ status: ok ? "done" : "queued", request_id });
  } else {
    await queue.add("book_visit", { request_id, payload: req.body }, { attempts: 3, backoff: { type:"exponential", delay: 200 }});
    return res.json({ status:"queued", request_id, next:"await_callback" });
  }
});
```

**Webhook Ingress (Node)**
```ts
app.post("/webhooks/partnerA", async (req, res) => {
  if (!verifyHmac(req)) return res.status(401).end();
  if (await isWebhookDuplicate(req.headers["x-request-id"])) return res.json({received:true});

  const event = req.body;
  await recordWebhook(event); // audit
  await routeEvent(event);    // booking.confirmed → orchestrator
  res.json({ received: true, request_id: event.request_id });
});
```

**Router sự kiện → nhắn người dùng**
```ts
async function onBookingConfirmed(evt) {
  await db.updateBooking(evt.data.booking_id, { status: "confirmed" });
  const msg = nlg.bookingConfirmed(evt.data); // 1.2
  await chat.pushMessage(evt.data.user_id, msg); // 2.1
}
```

---

### 🗄️ Bảng & Log gợi ý
```
idempotency(idempotency_key PK, request_id, created_at)
jobs(id PK, type, status, attempts, last_error, payload_json, created_at, updated_at)
webhook_events(id PK, provider, event, request_id, signature_valid, body_json, received_at)
action_audit(id PK, action, request_id, user_id, status, latency_ms, trace_id, ts)
bookings(id, user_id, listing_id, time, status, provider, external_id)
```

---

### 🧯 Lỗi & Fallback
- **Webhook thất bại:** worker **polling** tạm (nếu partner có API status).
- **Timeout:** đánh dấu `pending`, gửi card: “Đang tạo lịch, sẽ nhắn khi xác nhận xong.”
- **Mất đối xứng:** reconciler job mỗi 5–10 phút so trạng thái.

---

### 🧪 Kiểm thử bắt buộc (MVP)
- **TC01:** Idempotency – gọi 2 lần cùng key → chỉ tạo 1 booking.  
- **TC02:** Retry – partner 5xx → worker retry backoff.  
- **TC03:** Webhook HMAC sai → 401; HMAC đúng → 200 + dedup.  
- **TC04:** Mất kết nối → `queued`; khi callback đến, người dùng nhận thông báo.  
- **TC05:** Reconcile – partner confirmed nhưng chưa cập nhật → job sửa trạng thái.

---

### 📈 Quan sát & SLA
- **Trigger p95 (sync):** < 800 ms.  
- **Queue success:** > 99%.  
- **Webhook validation pass:** ≈ 100%, dedup rate < 1%.  
- **Orphan jobs (pending > 15m):** < 0.5%.  
- **End-to-end booking time (median):** theo đối tác.

---

### 🔌 Nối với 1.x & 2.x
- **1.3 (Policy):** quyết định khi nào trigger + theo dõi `request_id`.  
- **1.2 (NLG):** tạo thông báo trạng thái (accepted/queued/confirmed/failed).  
- **2.1 (Chat):** nhận push message khi có callback.  
- **1.5 (Learning):** log action_success, provider_latency, retry_count.

---

### ✅ Checklist MVP 3.3
- [x] Actions Facade (book_visit, send_email, create_lead, generate_doc)  
- [x] Idempotency + Outbox/Queue + Workers + Retry/Backoff  
- [x] Webhook Ingress: HMAC/IP allowlist + dedup + router  
- [x] Audit + Tracing (trace_id xuyên suốt trigger ↔ webhook)  
- [x] Reconciler (đồng bộ trạng thái định kỳ)  
- [x] Contract tests với 1 đối tác mẫu + mock provider  
- [x] Thông điệp người dùng cho các trạng thái (accepted/queued/confirmed/failed)

