## 🧩 Chức năng 3.1 – Kết nối API hoặc Cơ sở dữ liệu (Backend Integration Layer)

### 🎯 Mục tiêu & I/O
**Mục tiêu:** Cung cấp Data Access Layer (DAL)/Service Layer nhất quán: chuẩn hoá schema, cache, retry, bảo mật, đo lường.  
**Input:** yêu cầu từ Orchestrator/Policy (intent 1.3 + slots 1.1).  
**Output:** kết quả đã chuẩn hoá theo hợp đồng dữ liệu (contracts) cho NLG 1.2 & Cards 2.4.

---

### 🧱 Kiến trúc lớp truy cập (tổng quan)
```
[Orchestrator 1.3]
      │
      ▼
[Service Facade / UseCases]
      │   (compose, fallback, join)
      ▼
[Domain Services] ──► listings, projects, schedules, user-profile, rag, pricing
      │
      ▼
[Connectors] REST/GraphQL/gRPC/DB drivers
      │
      ├─► Internal DB (Postgres/Mongo/Elastic/Qdrant)
      ├─► External APIs (bản đồ, tỉ giá, pháp lý…)
      └─► File/Blob (ảnh, PDF pháp lý)
```
**Nguyên tắc:**
- FE/Chat chỉ gọi **Facade** (1 entrypoint/intent).
- **Connectors** chịu trách nhiệm auth, retry, rate limit, mapping raw→domain.
- **Domain model thống nhất:** diện tích (m²), giá (VND), vị trí (lat/lng), ngày (ISO), tiền tệ (ISO 4217).

---

### 📜 Data Contracts (ví dụ BĐS)
**Request (từ Orchestrator):**
```json
{
  "filters": {
    "area": {"district":"quận 7","city":"hcm"},
    "budget": {"max": 3000000000, "currency": "VND"},
    "rooms": 2,
    "min_floor_area": 65,
    "property_type": "apartment"
  },
  "paging": {"page":1,"page_size":10},
  "sort": {"by":"match_score","order":"desc"},
  "context": {"session_id":"s_ulid","user_id":"u_123"}
}
```
**Response (chuẩn hoá cho 2.4):**
```json
{
  "items": [
    {
      "id":"listing-SR-2850",
      "project":"Sunrise Riverside",
      "type":"apartment",
      "bedrooms":2,
      "bathrooms":2,
      "floor_area_m2":70,
      "orientation":"east",
      "price_vnd":2850000000,
      "price_label":"2.85 tỷ",
      "district":"quận 7","city":"hcm",
      "geo":{"lat":10.73,"lng":106.70},
      "legal_tags":["pink_book"],
      "amenities":["balcony","river_view"],
      "media":[{"url":"https://.../sr.jpg","kind":"image","w":1200,"h":800}],
      "source":{"provider":"internal-db","doc_id":"123"},
      "match_score":0.92
    }
  ],
  "paging":{"page":1,"page_size":10,"total":134}
}
```

---

### 🔌 Connectors (REST/GraphQL/DB) — quy tắc chuẩn
- **Timeout:** 800–1500ms mỗi hop (mặc định 1s).
- **Retry:** 2–3 lần, exponential backoff (100ms, 300ms, 900ms), retry-eligible = 5xx/timeout.
- **Circuit breaker:** mở sau 5 lỗi/30s, half-open 10s.
- **Idempotency** cho POST nhạy cảm (đặt lịch): `Idempotency-Key: <uuid>`.
- **Auth:** mTLS hoặc OAuth2 client credentials (external); JWT nội bộ (microservices).
- **Rate limit:** token bucket/Leaky bucket; trả `429 + Retry-After`.
- **Observability:** OpenTelemetry trace/span + tags (provider, latency, status).

---

### 🧮 Chuẩn hoá & kiểm tra dữ liệu
**Units & Currency**
- Diện tích → m² (float); chuyển ft² → m².  
- Giá → VND nội bộ + `price_label` (2 chữ số: *2.85 tỷ*).  
- Nếu nguồn USD → convert ≈ (ký hiệu `~` nếu là tỷ giá gần đúng).

**Địa lý**
- Chuẩn tên hành chính (alias: “q7” → “quận 7”), gắn mã quận/huyện (nếu có).  
- Chuẩn lat/lng WGS84; độ chính xác ≤ 6 chữ số thập phân.

**Pháp lý**
- Map nhãn: `pink_book` (sổ hồng), `red_book` (sổ đỏ), `offplan`.

**Validation**
- JSON Schema/TypeBox/Zod; **reject** hoặc **fix-and-warn** (gắn `warnings[]`).

---

### 🧠 Chiến lược tìm kiếm & xếp hạng
- **Match score** = w1*(tuân thủ slots) + w2*(khoảng giá) + w3*(phù hợp profile 1.4) + w4*(độ mới).  
- **Pagination:** seek-based (cursor) ưu tiên; fallback offset.  
- **Search engines:** Elastic/OpenSearch (text + filters) + *reranker* (optional).  
- **Hybrid:** keyword + vector (khi cần tìm theo mô tả).

---

### 🧳 Cache & hiệu năng
- **Read-through cache** (Redis/Memcached) theo filters hash + page.  
- **TTL:** listings 30–120s; dictionary 24h.  
- **Per-item cache** (listing by id) TTL 5–15 phút.  
- **Negative caching** cho truy vấn rỗng 10–30s.

---

### 🛡️ Bảo mật & tuân thủ
- **Field-level filtering:** ẩn PII theo vai trò.  
- **Row-level security (Postgres RLS)** cho dữ liệu đối tác/kênh.  
- **Audit log** truy vấn nhạy cảm (pháp lý, hồ sơ KH).  
- **PII policy (NĐ 13/2023):** export/delete, data minimization.

---

### 🔁 Fallback & Degradation
- Nguồn A lỗi → **fallback** nguồn B (hoặc cache gần nhất).  
- **No-result** → trả **gợi ý** (khu lân cận, tăng ngân sách).  
- **Slow path** (> p95 budget) → gửi **partial** (top 3) rồi stream thêm.

---

### 🧩 Service Facade (Use-cases) — pseudo
```ts
// listings.facade.ts
export async function findListings(req: FindReq, ctx: Ctx): Promise<FindRes> {
  const normalized = normalizeFilters(req.filters, ctx.locale); // alias, currency, units
  const cacheKey = cacheKeyFrom(normalized, req.paging, req.sort);

  const cached = await cache.get(cacheKey);
  if (cached) return cached;

  const [primary, secondary] = [provider.internal, provider.partnerA];
  let res = await safeSearch(primary, normalized, req);
  if (!res || res.items.length === 0) {
    res = await safeSearch(secondary, normalized, req); // fallback
  }

  const ranked = rank(res.items, normalized, ctx.profile);
  const page = paginate(ranked, req.paging);
  const out: FindRes = { items: page.items, paging: page.paging };

  cache.set(cacheKey, out, ttlByResult(out));
  return out;
}

async function safeSearch(p, f, req) {
  try { return await p.search(f, req.paging, req.sort); }
  catch (e) { logger.warn({e}, "provider_search_failed"); return null; }
}
```

---

### 🗄️ Lựa chọn CSDL & khi nào dùng
- **Postgres:** dữ liệu quan hệ (dự án, listing, lịch, người dùng), RLS, JSONB linh hoạt.  
- **MongoDB:** schema linh hoạt cho listing/ảnh/thẻ; chú ý index (compound, text).  
- **Elasticsearch/OpenSearch:** full-text + filter + sort; tìm kiếm listing.  
- **Qdrant/FAISS:** RAG/semantic search tài liệu pháp lý.  
- **Redis:** cache, rate-limit, queue nhẹ.  
- **MVP gợi ý:** Postgres + Elastic + Redis (sau thêm Qdrant nếu RAG mạnh).

---

### 🧪 Test & quan sát (bắt buộc)
- **Contract tests** (raw→domain): mọi connector mapping giữ chuẩn.  
- **Latency SLO:** p95 < 500–800ms end-to-end Facade.  
- **Error budget:** ≤ 1% lỗi provider/tháng; alert khi 5xx tăng.  
- **Data quality:** record thiếu trường quan trọng < 1%.

---

### 🧰 Migrations & đồng bộ
- ORM (Prisma/TypeORM/SQLAlchemy) + migrations versioned.  
- **CDC** (Debezium/Outbox) đẩy sang Elastic.  
- **Backfill jobs:** điền `price_vnd`, `area_m2`, `legal_tags` từ dữ liệu cũ.

---

### 🧯 Xử lý lỗi & trả về có ý nghĩa
```json
{
  "error": {
    "code": "UPSTREAM_TIMEOUT",
    "message": "Nguồn dữ liệu chậm bất thường, đã trả kết quả gần nhất.",
    "fallback_used": true,
    "trace_id": "tr-abc123"
  },
  "data": { "items": [/* from cache */], "stale": true }
}
```

---

### 🧭 Tích hợp với 1.x & 2.x
- **1.1/1.3 → 3.1:** nhận filters chuẩn từ NLU/Policy.  
- **3.1 → 1.2/2.4:** trả domain model sẵn sàng render card.  
- **2.3 (quick replies):** cung cấp bậc thang ngân sách & khu lân cận.  
- **1.5 (learning):** log provider_latency, fallback_used, no_result_reason.

---

### ✅ Checklist MVP 3.1
- [x] Service Facade: `findListings`, `getListingById`, `getNearby`, `bookVisit`, `lawLookup`  
- [x] Connectors: timeout/retry/circuit + mapping + metrics  
- [x] Normalizer: alias địa danh, tiền & diện tích, orientation, legal tags  
- [x] Schema contracts (Zod/JSON Schema) + contract tests  
- [x] Cache: per-query & per-item; TTL thông minh  
- [x] Pagination: cursor-based + sort hợp lệ  
- [x] Security: field/row-level filter, audit log, rate limit  
- [x] Observability: tracing, logs chuẩn, dashboard p95/p99, error rate  
- [x] Fallback: nguồn phụ + cache gần nhất + no-result suggestions

---

### 🌾 Ví dụ end-to-end (rút gọn)
**Orchestrator 1.3** tạo yêu cầu:
```ts
const res = await listingsFacade.findListings({
  filters: { area:{district:"q7"}, budget:{max:3e9, currency:"VND"}, rooms:2, min_floor_area:65, property_type:"apartment" },
  paging: {page:1, page_size:10},
  sort: {by:"match_score", order:"desc"}
}, ctx);
```
**3.1:** chuẩn hoá alias (q7→quận 7), gọi Elastic + fallback, xếp hạng, cache, trả kết quả.  
**1.2/2.4:** render 3 card top + quick replies “Đặt lịch cuối tuần”, “Lọc 2PN + ban công”, “Diện tích ≥ 70 m²”.

