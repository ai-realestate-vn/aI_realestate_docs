## 🧩 Workflow Chức năng 2.1 – Nhắn tin hai chiều (Text-based chat)

### 🎯 Mục tiêu & I/O
**Mục tiêu:** Gửi/nhận tin nhắn ổn định, realtime, có thứ tự, an toàn; gắn với NLU→DM→NLG.  
**Input:** user_text/client_events từ FE; context phiên; JWT người dùng.  
**Output:** assistant_messages (text + payload rich), events (typing, read), mã lỗi chuẩn.

---

### 🧱 Kiến trúc tổng quát
**FE (Web)**
- UI Chat (virtualized list, composer, attachments nhẹ)
- State: messages, connection, pendingQueue
- Transport: WebSocket + fallback HTTP POST (send) & SSE (receive)

**BE (Gateway)**
- Auth (JWT), rate limit, moderation
- Message Broker in-proc (MVP) → async queue
- Orchestrator: gọi 1.1/1.3/1.2 và adapters (DB/RAG)

**Stores**
- conversations, messages, sessions (ULID/UUID v7, idempotency)

---

### 🔌 Giao thức & Schema
#### 1) WebSocket events (bi-directional)
**Client → Server**
```json
{ "type":"AUTH", "token":"<JWT>" }
{ "type":"SEND", "client_id":"c_ulid", "conversation_id":"conv_ulid",
  "message": { "kind":"user", "text":"căn 2pn q7 dưới 3 tỷ", "locale":"vi-VN" } }
{ "type":"ACK", "message_id":"m_ulid" }
{ "type":"READ", "conversation_id":"conv_ulid", "upto":"m_ulid" }
{ "type":"TYPING", "conversation_id":"conv_ulid", "is_typing":true }
```

**Server → Client**
```json
{ "type":"READY", "session_id":"s_ulid" }
{ "type":"DELIVERED", "client_id":"c_ulid", "message_id":"m_ulid" }
{ "type":"MESSAGE",
  "message": {
    "id":"m_ulid","ts":1730431200,"kind":"assistant",
    "content": [{ "type":"text","text":"Em thấy 3 căn..." }],
    "cards":[{ "title":"Sunrise Riverside - 2PN - 2.85 tỷ", "actions":[...] }],
    "quick":["Lọc 2PN","Tăng ngân sách","Xem lịch cuối tuần"],
    "meta":{"source_ids":["db:listing:..."],"tone":"polite-pro"}
  }
}
{ "type":"TYPING", "conversation_id":"conv_ulid", "is_typing":true }
{ "type":"ERROR", "code":"RATE_LIMIT_EXCEEDED", "retry_after_ms":5000 }
```

**Ordering:** dùng ULID/UUIDv7 + ts để sắp xếp, client hợp nhất theo (ts, id).

#### 2) HTTP REST (fallback & tiện ích)
- `POST /v1/chat/send` – gửi 1 tin (idempotency key = client_id)
- `GET /v1/chat/history?conversation_id&cursor=...` – phân trang
- `POST /v1/chat/typing` · `POST /v1/chat/read`
- `POST /v1/chat/new` – tạo hội thoại mới (guest → registered)

**Payload gửi:**
```json
{
  "conversation_id": "conv_ulid",
  "client_id": "c_ulid",
  "message": { "kind":"user", "text":"...", "locale":"vi-VN" }
}
```

**Đáp ứng:**
```json
{ "status":"queued", "message_id":"m_ulid" }
```

---

### 🔒 Bảo mật & an toàn
- **Auth:** JWT (sub, roles, exp), refresh via `/auth/refresh`.
- **CORS/CSRF:** chỉ whitelisted origins; CSRF token cho HTTP POST.
- **Rate limit:** 10 req/10s/conv (MVP); WS flood control.
- **Moderation:** trước khi route đến NLU → regex/keyword (PII, nhạy cảm) + hành vi (spam > N msg/Δt).
  - Nếu vi phạm → `ERROR{code:"CONTENT_BLOCKED"}` + hướng dẫn lịch sự.
- **Privacy:** ẩn/băm PII; theo NĐ 13/2023: export/delete conversation.

---

### 🔁 Dòng xử lý 1 lượt gửi (server)
1. Auth + Limit + Moderation
2. Persist inbound (`messages(kind="user", status="received")`)
3. Emit DELIVERED (map client_id → message_id)
4. Typing=true → push event
5. Orchestrator: gọi NLU (1.1) → Policy (1.3) → NLG (1.2)
6. (tuỳ intent) gọi search_listings / book_visit / RAG
7. Persist outbound (`messages(kind="assistant")`)
8. Emit MESSAGE (text + cards + quick replies)
9. Typing=false
10. Metrics: latency, tokens, tool_calls; push events (2.5/1.5)

---

### 🧠 Trạng thái UI (FE)
```ts
type Msg = {
  id?: string; client_id?: string; ts?: number;
  kind: "user"|"assistant"|"system";
  content?: {type:"text", text:string}[];
  cards?: Card[]; quick?: string[];
  status?: "pending"|"sent"|"delivered"|"failed";
  error?: {code:string, detail?:string};
};
```

**Luồng FE send:**
1. Tạo client_id ULID, push msg pending
2. Gửi WS SEND
3. Khi nhận DELIVERED → set sent + id
4. Khi nhận MESSAGE (assistant) → append
5. Retry exponential backoff nếu failed (giữ client_id để idempotent)

---

### 🧳 Lưu trữ (bảng tối thiểu)
```
conversations(id, user_id, created_at, last_active_at, channel)
messages(id, conversation_id, sender, kind, text, payload_json, ts, status)
message_index(conversation_id, ts, id) -- để phân trang
sessions(id, user_id, device, ws_status, last_seen)
```

---

### 🧭 State Machine rút gọn (client connection)
```
DISCONNECTED → CONNECTING → AUTHENTICATED → READY
READY → RETRYING (backoff 1s,2s,5s,10s) → READY
```

- Offline queue: xếp tin pending; khi READY → flush.

---

### 🧩 Tích hợp 1.x (điểm nối)
- Inbound → 1.1 (NLU) → 1.3 (Policy) → gọi tools → 1.2 (NLG) → Outbound.
- Context/Memory 1.4: nạp STM/LTM trước 1.3; cập nhật sau 1.2.
- Learning 1.5: log sự kiện gửi/nhận, CTR cards, read/typing, error.

---

### 🧪 Kiểm thử quan trọng (MVP)
| ID | Tên kiểm thử | Mô tả |
|----|----------------|-------------------------|
| TC01 | Gửi nhanh 5 tin/2s | Giữ đúng thứ tự + DELIVERED |
| TC02 | Mất mạng giữa chừng | reconnect + resend idempotent |
| TC03 | Lỗi moderation | CONTENT_BLOCKED + UI nhẹ nhàng |
| TC04 | Tin song ngữ/emoji | normalize OK |
| TC05 | Long text & markdown | render đúng |
| TC06 | Pagination history | mượt, không trùng tin |

---

### 📈 Quan sát & giới hạn
- **p95 send→delivered:** <150 ms (nội bộ)
- **p95 user→assistant:** <1.5 s (MVP)
- **Drop rate WS:** <1%
- **Duplicate rate:** ≈ 0%
- **Spam shield:** >3 tin/second → tạm chặn 5s / cảnh báo UI

---

### 🛠️ Mẫu triển khai nhanh
**Express (send)**
```js
app.post("/v1/chat/send", auth, async (req, res) => {
  const { conversation_id, client_id, message } = req.body;
  if (await isDuplicate(conversation_id, client_id)) {
    return res.json({ status:"duplicate" });
  }
  const id = newULID();
  await saveUserMessage({ id, conversation_id, client_id, message });
  broker.publish("inbound", { id, conversation_id });
  res.json({ status:"queued", message_id: id });
});
```

**WebSocket (server)**
```js
ws.on("message", async (raw) => {
  const evt = JSON.parse(raw);
  if (evt.type === "SEND") {
    const id = await handleInboundWS(ws.user, evt);
    ws.send(JSON.stringify({ type:"DELIVERED", client_id: evt.client_id, message_id: id }));
  }
});
```

---

### ✅ Checklist MVP 2.1
- [x] WS server + fallback REST/SSE; ACK/DELIVERED + client_id idempotent
- [x] Lưu messages + conversations, phân trang theo ts,id
- [x] Typing/Read receipts, offline queue FE
- [x] Rate limit, moderation, lỗi chuẩn hoá (ERROR{code, retry_after_ms})
- [x] Markdown cơ bản + quick replies + cards
- [x] Monitoring latency, error, drop/retry; logs cho 1.5
- [x] E2E tests: ordering, reconnect, spam, pagination, Unicode

