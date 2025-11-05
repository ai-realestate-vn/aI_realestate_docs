## 📊 Chức năng 4.2 – Dashboard thống kê (Admin & Analytics)

### 🎯 Mục tiêu & phạm vi
- Nhìn nhanh sức khỏe hệ thống (latency, lỗi), hiệu quả AI (intent, fallback, groundedness), hiệu quả kinh doanh (CTR, conversion), chi phí (tokens/cost).
- Hỗ trợ drill-down đến phiên/hội thoại cụ thể.
- Phân quyền theo tenant/role (admin/manager/analyst).

---

### 🧱 Kiến trúc dữ liệu
```
[Logs 4.1 | Events | DB giao dịch]  ──►  [Ingest/ETL]
      (messages, metrics, flags)          (stream/batch)
                 │                              │
                 ▼                              ▼
          [ODS/Staging]  ─────────────►  [Warehouse/Star Schema]
                 │                              │
                 └────────►  [Materialized Views / Aggregates]
                                         │
                                         ▼
                                   [Dashboards]
                             (Usage, Quality, Business, Cost)
```

**Nguồn chính**
- `messages`, `conversations`, `message_flags` (4.1/4.3)
- Metrics (Prometheus export: `latency`, `error_rate`, `token_usage`)
- Hành vi FE (`clicked_quick_reply`, `card_click`, `conversion`)

---

### 📦 Lược đồ dữ liệu (Star schema rút gọn)
**Dimension**
- `dim_date(date_key, y, q, m, d, dow)`
- `dim_tenant(tenant_id, name)`
- `dim_channel(channel_id, name)` – *web, mobile, partner*
- `dim_intent(intent_id, name, group)`
- `dim_model(model_id, name, provider)` – *Llama-x, vLLM, …*

**Fact**
- `f_session_daily(date_key, tenant_id, channel_id, sessions, dau, wau, mau, avg_turns, session_duration_sec)`
- `f_message_daily(date_key, tenant_id, channel_id, model_id, intent_id, messages, user_msgs, bot_msgs, fallback_msgs, thumbs_up, thumbs_down, token_in, token_out, latency_p50_ms, latency_p95_ms, error_rate)`
- `f_business_daily(date_key, tenant_id, channel_id, quick_impr, quick_click, card_impr, card_click, leads, bookings, conv_rate)`  *(conv_rate = bookings/sessions)*
- `f_rag_quality_daily(date_key, tenant_id, rag_queries, citation_present, grounded_score_avg, no_result_count)`

**Materialized View (ví dụ)**
```sql
CREATE MATERIALIZED VIEW mv_daily_summary AS
SELECT
  d.date::date AS dt, m.tenant_id,
  SUM(s.sessions) AS sessions,
  AVG(s.avg_turns) AS avg_turns,
  SUM(m.messages) AS messages,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY m.latency_p95_ms) AS latency_p95,
  SUM(m.fallback_msgs)::float / NULLIF(SUM(m.bot_msgs),0) AS fallback_rate,
  SUM(b.bookings)::float / NULLIF(SUM(s.sessions),0) AS conv_rate,
  SUM(m.token_in) AS token_in, SUM(m.token_out) AS token_out
FROM dim_date d
LEFT JOIN f_session_daily s ON s.date_key=d.date_key
LEFT JOIN f_message_daily m ON m.date_key=d.date_key AND m.tenant_id=s.tenant_id
LEFT JOIN f_business_daily b ON b.date_key=d.date_key AND b.tenant_id=s.tenant_id
GROUP BY 1,2;
```

---

### 🔄 Ingest/ETL (batch + near-real-time)
- **Near-real-time (60–120s):** stream sự kiện UI & metrics → ODS (Kafka/Redis Stream → worker).
- **Batch hàng giờ/ngày:** tổng hợp vào `f_*_daily`, cập nhật MV.
- **Chuẩn hoá/PII:** mask email/phone trước khi vào warehouse.

**ETL pseudo**
```ts
// aggregate messages -> f_message_daily
for (const msg of stream("messages")) {
  const key = dailyKey(msg.ts, msg.tenant, msg.channel, msg.model, msg.intent);
  agg[key].messages++;
  if (msg.kind === "user") agg[key].user_msgs++;
  if (msg.kind === "assistant") agg[key].bot_msgs++;
  if (msg.is_fallback) agg[key].fallback_msgs++;
  agg[key].token_in += msg.token_in || 0;
  agg[key].token_out += msg.token_out || 0;
  agg[key].lat_p95.add(msg.latency_ms);
}
flushHourlyToWarehouse(agg);
```

---

### 📊 Bộ widget/KPI đề xuất
**1) Usage**
- Sessions/Day, DAU/WAU/MAU, Avg turns/session
- Heatmap giờ cao điểm (dow × hour)
- Top intent (pareto 80/20)

**2) Quality (AI)**
- Intent accuracy, Fallback rate
- Thumbs up/down rate
- RAG: citation coverage, groundedness avg, no-result
- Flagged rate + link 4.3 queue

**3) Performance (SLA)**
- Latency p50/p95 (E2E & breakdown NLU/RAG/NLG)
- Error rate (4xx/5xx) theo service
- Queue backlog (workers)

**4) Business**
- CTR quick replies/cards
- Conversion funnel: View → Click → Lead → Booking
- Bookings/Day theo kênh/intent

**5) Cost**
- Token In/Out theo model
- Cost/100 sessions; Cost/booking
- Model mix (stacked bar)

---

### 🖥️ UX Dashboard (Admin Console)
- **Header filter:** date range, tenant, channel, model, intent group.
- **Layout:** 2 cột (overview KPI cards) → hàng biểu đồ → bảng chi tiết.
- **Drill-down:** click KPI → mở list hội thoại (4.1) đã filter; xem timeline & flags.
- **Export:** PNG/CSV/PDF, schedule email (hằng tuần).
- **Compare mode:** WoW/MoM, A/B (model A vs B).

*Thành phần FE (React):* `KpiCard`, `TimeSeriesChart`, `BarRank`, `Heatmap`, `Funnel`, `DataTable` (virtualized, skeleton, SWR/RTK Query cache).

---

### 🔌 API cho Dashboard
```
GET /admin/analytics/overview?from=...&to=...&tenant=...&channel=...
GET /admin/analytics/quality?from=...&to=...&intent=...
GET /admin/analytics/perf?from=...&to=...&service=...
GET /admin/analytics/business?from=...&to=...&funnel=true
GET /admin/analytics/cost?from=...&to=...&model=...
GET /admin/analytics/drilldown/sessions?filter=...
```

**Định nghĩa một số chỉ số**
- `intent_accuracy = 1 - (fallback_msgs / bot_msgs)` *(MVP; về sau dùng tập gán nhãn/eval tự động)*
- `qr_ctr = quick_click / quick_impr`
- `card_ctr = card_click / card_impr`
- `conv_rate = bookings / sessions`
- `cost = Σ(token_in/out * price_per_token[model])`

---

### 🔒 RBAC & phân tách dữ liệu
- **admin:** xem mọi tenant + export.
- **manager:** chỉ tenant của mình, có drill-down.
- **analyst:** chỉ dashboard, không mở raw chat.
- **RLS:** row-level filter theo `tenant_id`.
- **Masking:** PII luôn ẩn trong bảng drill-down trừ admin.

---

### ⚡ Tối ưu hiệu năng
- MV & pre-aggregation theo ngày/giờ → đáp ứng < 300ms.
- Columnar DB (ClickHouse/BigQuery) cho truy vấn lớn.
- API cache (Redis) 60–300s cho widget nặng.
- Pagination + cursor cho bảng.

---

### 🧪 Test & dữ liệu chuẩn
- Độ đúng số liệu: đối soát ODS ↔ MV *(sai lệch < 1%)*.
- Tải: 50 concurrent viewers, TTFB < 500ms cho widget nhẹ.
- Drill-down: filter liên thông (overview → danh sách phiên → timeline).
- Bảo mật: test RBAC/RLS; PII không lọt ra widgets.

---

### 📈 KPI vận hành dashboard
- API dashboard error rate < 1%.
- Refresh widgets p95 < 1s (pre-agg).
- ETL freshness < 5 phút (near-real-time) / < 1h (batch).
- Adoption: số lần dùng drill-down/tuần.

---

### ✅ Checklist MVP 4.2
- [x] Star schema + MV `mv_daily_summary`, `f_*_daily`  
- [x] ETL (stream + batch) với PII masking  
- [x] API `/admin/analytics/*` + cache  
- [x] Dashboard FE (overview, quality, perf, business, cost) + drill-down 4.1  
- [x] RBAC/RLS + export + schedule email report  
- [x] Định nghĩa KPI & công thức thống nhất (doc i18n)  
- [x] Test tính đúng/sớm & bảo mật dữ liệu

