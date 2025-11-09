## 📈 Chức năng 4.1 – Logging, Monitoring & Analytics

### 🎯 Mục tiêu
- Ghi nhận toàn bộ hoạt động và lỗi trong hệ thống (từ NLU → NLG → API → Webhook).
- Cung cấp dashboard thời gian thực về hiệu suất, độ chính xác, tỉ lệ lỗi, hành vi người dùng.
- Hỗ trợ giám sát sự cố (incident detection) và phân tích hành vi (chatflow analytics, funnel).
- Nền tảng dữ liệu cho học liên tục (1.5) và tối ưu hóa mô hình.

---

### 🧱 Kiến trúc giám sát tổng thể
```
[Chatbot Modules]
 ├─ 1.x NLU/NLG/Policy
 ├─ 2.x UI/Interaction
 ├─ 3.x Integrations
 └─ System Runtime (API, DB, Queue)
          │
          ▼
   [Telemetry SDK]
          │
          ▼
   [Collector Layer]
   ├─ Logs (ELK / Loki)
   ├─ Metrics (Prometheus)
   ├─ Traces (Jaeger / OpenTelemetry)
   └─ Events (Kafka / Redis Stream)
          │
          ▼
   [Analytics & Dashboards]
   ├─ Grafana (system metrics)
   ├─ Kibana (logs & errors)
   ├─ Metabase/Superset (business KPIs)
   └─ Alerting (PagerDuty / Discord / Email)
```

---

### 🧩 Thành phần dữ liệu & loại log
| Loại | Mục đích | Ví dụ |
|---|---|---|
| **System Logs** | Lỗi hệ thống, cảnh báo, trạng thái | Timeout, API 5xx, circuit breaker open |
| **App Logs** | Hành động nghiệp vụ, intent, message flow | "User intent=tim_bds, plan=filter_area" |
| **Audit Logs** | Theo dõi thay đổi nhạy cảm | "Admin A cập nhật cấu hình chatbot" |
| **Interaction Logs** | Ghi lại hội thoại đã ẩn PII | "User hỏi ‘căn hộ Q7’ → Bot trả lời…" |
| **Metrics** | Đếm/đo/xu hướng (Prometheus) | latency, throughput, token usage |
| **Traces** | Theo dõi request xuyên microservice | span: NLU → Policy → RAG → NLG |
| **Events** | Hành vi người dùng (event analytics) | clicked_quick_reply, booked_visit |

---

### 🧭 Luồng xử lý dữ liệu logging
**1️⃣ Ứng dụng gửi log**  
Dùng OpenTelemetry SDK (Python/TS) hoặc logger (Winston/Pino) kèm metadata:
```json
{
  "trace_id":"tr-abc123",
  "session_id":"s_ulid",
  "user_id":"u_123",
  "module":"nlu",
  "level":"info",
  "message":"Intent detected: tim_bds",
  "latency_ms":43,
  "timestamp":"2025-11-01T15:00:03Z"
}
```
**2️⃣ Collector layer**  
- Loki/Elasticsearch cho log text.  
- Prometheus cho metrics (counter/histogram/gauge).  
- Jaeger/Tempo cho distributed tracing.  
- Truyền `trace_id` xuyên suốt (NLU → RAG → NLG → API).

**3️⃣ Lưu trữ & phân tích**  
- Hot storage (7–30 ngày): Elasticsearch/Loki.  
- Cold storage (≥ 6 tháng): S3/MinIO cho ML (1.5).  
- Retention: PII ẩn/băm, auto-delete logs sau 90 ngày.

**4️⃣ Truy cập & Dashboard**  
- Grafana: CPU/RAM, latency, throughput, error rate.  
- Kibana: tìm kiếm lỗi, exception trace.  
- Metabase: hành vi (click rate, session length, conversion).

---

### 📊 Mô hình dữ liệu sự kiện (Event Schema)
```json
{
  "event_id":"evt_20251101_00023",
  "timestamp":"2025-11-01T08:30:00+07:00",
  "session_id":"s_ulid",
  "user_id":"u_123",
  "event_type":"clicked_quick_reply",
  "intent":"tim_bds",
  "payload":{ "label":"Đặt lịch cuối tuần", "action":"book_visit" },
  "context":{ "channel":"web", "locale":"vi-VN", "model":"Llama-3.1-8B", "latency_ms":312 }
}
```

---

### ⚙️ Hệ thống giám sát (Monitoring Metrics)
| Nhóm | Metric | Đơn vị | Mục tiêu |
|---|---|---|---|
| **System** | CPU, RAM, Disk, Uptime | % | CPU < 80%, uptime > 99% |
| **API** | Latency p95, Error rate | ms / % | < 800ms, < 1% lỗi |
| **NLU/NLG** | Accuracy, Latency | % / ms | > 85%, < 300ms |
| **RAG** | Recall@10, Latency | % / ms | > 90%, < 1500ms |
| **Conversation** | Avg msg/turn, CTR quick reply | số / % | CTR ≥ 25% |
| **User Flow** | Conversion rate (view→book) | % | ≥ 5–10% |
| **Infra** | Queue backlog, DB connections | số | backlog < 100, connections < limit |

---

### 🧠 Liên kết tới các module khác
| Module | Log/Metric gửi ra | Ý nghĩa giám sát |
|---|---|---|
| **1.1–1.3 (Core AI)** | intent_confidence, error_confidence_low | đánh giá chất lượng hiểu người dùng |
| **2.1–2.4 (UI)** | click_rate, reply_delay | đo UX & tốc độ phản hồi |
| **3.1–3.3 (Backend)** | api_latency, provider_error | lỗi tích hợp ngoài |
| **3.4 (Auth)** | login_fail, token_expired | bảo mật & quyền truy cập |
| **1.5 (Learning)** | feedback_pos/neg | dữ liệu huấn luyện |
| **RAG (3.2)** | retrieval_latency, groundedness_score | độ chính xác câu trả lời |

---

### 🧰 Pseudo Implementation
**Logger config (Node/TypeScript)**
```ts
import pino from "pino";
import { traceId } from "./telemetry";

export const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  formatters: { level: (label) => ({ level: label }) },
  base: { service: "chatbot-core" },
  timestamp: () => `,"time":"${new Date().toISOString()}"`
});

export function logEvent(level, msg, meta: any = {}) {
  logger[level]({ trace_id: traceId(), ...meta, msg });
}
```
**Metrics exporter (Prometheus)**
```ts
import client from "prom-client";

const reqDuration = new client.Histogram({
  name: "chatbot_request_duration_ms",
  help: "Request latency in ms",
  labelNames: ["module", "endpoint"]
});

export function recordLatency(module: string, endpoint: string, ms: number) {
  reqDuration.labels(module, endpoint).observe(ms);
}
```

---

### 📈 Alerting (Cảnh báo tự động)
| Cảnh báo | Ngưỡng | Hành động |
|---|---|---|
| Error Rate > 5% | 5 phút liên tục | gửi cảnh báo Discord + email devops |
| Latency > 2s (p95) | 10 phút | đánh dấu “slow region”, auto-scale pod |
| Intent Confidence < 0.5 (avg) | 1 giờ | gắn cờ nhóm NLP tuning |
| DB Connection > 90% limit | 5 phút | restart pooler / mở rộng |
| RAG recall giảm >10% | 1 ngày | cảnh báo nhóm AI retraining |
| Queue backlog > 500 | 10 phút | auto-scale worker |
| Uptime < 99%/24h | — | báo cáo SLA |

---

### 🧾 Audit Logging
- Lưu thay đổi nhạy cảm: quyền, cấu hình model, token API, consent.  
- Trường: **who, what, when, before_after**; bảng riêng, chỉ admin audit xem.
```json
{
  "audit_id":"au_20251101_1001",
  "user_id":"admin_01",
  "action":"update_model_config",
  "before":{"model":"llama-3-8b"},
  "after":{"model":"llama-3-70b"},
  "timestamp":"2025-11-01T10:00:00Z"
}
```

---

### 🧪 Test Cases chính
- ✅ **TC01:** Log → Collector → hiển thị Grafana ≤ 10s.  
- ✅ **TC02:** API lỗi → alert Discord ≤ 1 phút.  
- ✅ **TC03:** RAG latency > 1.8s → cảnh báo nhóm AI.  
- ✅ **TC04:** User feedback “👎” → tăng `feedback_neg=1`.  
- ✅ **TC05:** PII trong log → bị mask (***).

---

### 📊 Báo cáo & phân tích (Metabase / Superset)
| Báo cáo | Chỉ số chính | Mục tiêu |
|---|---|---|
| Hiệu suất hội thoại | số phiên/ngày, độ dài trung bình | giữ engagement cao |
| Tỷ lệ click quick reply | click / impression | ≥ 25% |
| Tỷ lệ hoàn tất đặt lịch | số lịch / tổng phiên | ≥ 5% |
| Intent Accuracy theo loại | accuracy từng intent | ≥ 85% |
| Top lỗi thường gặp | danh sách lỗi phổ biến | fix cycle ≤ 7 ngày |
| SLA Backend | API latency trung bình | < 800 ms |

---

### 🔒 Bảo mật & Tuân thủ
- **PII masking** trước khi log (email → e***@mail.com, phone → ***1234).  
- **Encrypted at rest** (AES-256/S3 KMS).  
- **GDPR / NĐ13/2023**: user có quyền ẩn danh hóa dữ liệu log khi yêu cầu.  
- **Least privilege:** DevOps/AI lead xem tập trung, không raw chat.

---

### ✅ Checklist MVP 4.1
- [x] Telemetry SDK (OpenTelemetry / custom logger)  
- [x] Central Collector (Prometheus, Loki/ELK, Jaeger)  
- [x] Trace ID xuyên service  
- [x] Log masking PII  
- [x] Metrics key (latency, throughput, error_rate, intent_conf, rag_recall)  
- [x] Alert rules & notifications  
- [x] Grafana/Kibana dashboard mẫu  
- [x] Audit log module  
- [x] Retention & backup policy

