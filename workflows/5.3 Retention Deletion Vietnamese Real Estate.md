# Workflow 5.3 – Retention & Deletion (Cơ chế Xóa / Lưu trữ Dữ liệu theo Quy định)

> **Phạm vi:** Chatbot AI đa tenant cho BĐS (web/app/API). Tuân thủ Nghị định 13/2023 (VN). Tương thích các lớp dữ liệu: DB giao dịch, log sự kiện, VectorDB (RAG), kho file (S3/GCS), backups.

---

## 🎯 Mục tiêu
- **Purpose binding:** Xác định mục đích xử lý và gán **TTL/chu kỳ lưu trữ** tương ứng.
- **DSAR:** Cung cấp **Export / Delete / Rectify** theo yêu cầu chủ thể dữ liệu.
- **Xóa an toàn:** Không còn dấu vết trong **backup, cache, vector index, ML datasets**.

---

## 🧱 Kiến trúc tổng quan (các lớp)
- **Retention Policy Registry (YAML/DB):** ma trận TTL theo **purpose** & **data class**.
- **Lifecycle Engine (job scheduler):** lên lịch **soft-delete → purge** theo chính sách.
- **DSAR Service (API):** tiếp nhận & thực thi **export/delete/rectify** có SLA.
- **Data Lineage & Manifests:** theo dõi nơi dữ liệu đi qua (DB, S3, VectorDB, logs, datasets).
- **Legal Hold:** cơ chế “đóng băng” bản ghi khi có yêu cầu điều tra/kiện tụng (tạm dừng purge).
- **Audit Trail:** ghi nhận mọi hành động xóa/lưu trữ (**WORM/immutable**).

---

## 🗺️ Luồng vận hành (E2E)
### A) Định nghĩa & gán chính sách
- **Classify** bản ghi ngay khi tạo: `purpose`, `tenant_id`, `data_class`, `consent_scope`.
- Tra **Retention Registry** để set: `ttl_days`, `soft_delete_days`, `backup_purge_days`.

**Retention Registry (ví dụ)**
```yaml
purposes:
  runtime_chat:
    dialog_events:     { ttl_days: 90,  soft_delete_days: 90,  backup_purge_days: 180 }
    stm_memory:        { ttl_days: 120, soft_delete_days: 120, backup_purge_days: 180 }
  personalization:
    ltm_profile:       { ttl_days: 365, soft_delete_days: 0,   backup_purge_days: 180 }
  booking:
    booking_records:   { ttl_days: 730, soft_delete_days: 0,   backup_purge_days: 365 } # ràng buộc nghiệp vụ
  training_analytics:
    scrubbed_datasets: { ttl_days: 365, soft_delete_days: 0,   backup_purge_days: 365 }

data_classes:
  pii_high:   { storage: "vault_enc",   policy: "mask_by_default" }
  pii_medium: { storage: "db_enc",      policy: "mask_on_read" }
  non_pii:    { storage: "db_plain",    policy: "standard" }
```

### B) Xóa theo lịch (Lifecycle)
1) **Discover:** Job quét theo TTL → lập **Deletion Tasks** theo resource type.

2) **Soft-Delete (nếu có):** đặt `deleted_at`, ẩn khỏi UI & API, tách khỏi index tìm kiếm.

3) **Grace Window:** chờ `soft_delete_days` (cho phép khôi phục nếu sai sót/`legal_hold`).

4) **Hard Purge:** xóa triệt để ở **mọi nơi**:
- **DB:** `DELETE` vật lý (hoặc **VACUUM FULL** với Postgres), xoá tombstone sau purge.
- **VectorDB (Qdrant/Pinecone):** delete theo `payload.user_id`, chạy **optimizer/compaction**.
- **S3/GCS:** Lifecycle rule → **Expiration** + xóa **previous versions** (Object Lock policy cho audit).
- **Caches:** Redis/CloudFront purge theo key pattern.
- **Logs/Lakes:** xóa partition theo ngày + compaction (Iceberg/Delta).
- **Backups:** đánh dấu bản sao liên quan để **không khôi phục** dữ liệu đã xoá (selective restore policy) + xóa theo `backup_purge_days`.

### C) DSAR (Data Subject Access Requests)
- **Tiếp nhận** (trang Privacy hoặc API) → **xác minh danh tính** (AuthN + MFA).
- **Orchestrate:**
  - **Export:** gom dữ liệu từ DB, VectorDB, S3, Logs (bản **đã pseudonym** theo 5.2) → **ZIP + manifest JSON**.
  - **Delete:** tạo **Deletion Order** theo `user_id` hoặc global identifiers (email/phone **hashed mapping**).
- **Thực thi có thứ tự:** **Vault PII → DB → VectorDB → Logs → Datasets → Backups policy**.
- **Chứng từ & Audit:** sinh **Proof of Deletion** (hash danh sách bản ghi + timestamp + version hệ thống).
- **SLA:** Export ≤ **7 ngày**; Delete ≤ **15–30 ngày** (có **legal hold** ngoại lệ).

---

## 🧩 Hợp đồng dữ liệu (Data Contracts)
### Bản ghi dữ liệu (ví dụ DB)
```sql
CREATE TABLE user_profile (
  id UUID PRIMARY KEY,
  tenant_id UUID,
  email_hash TEXT,          -- không lưu email thô
  phone_token TEXT,         -- FPE/tokens
  consent_personalize BOOLEAN,
  purpose TEXT,             -- personalization
  data_class TEXT,          -- pii_medium
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ,
  deleted_at TIMESTAMPTZ,   -- soft delete
  legal_hold BOOLEAN DEFAULT FALSE
);

CREATE INDEX ON user_profile (deleted_at);
CREATE INDEX ON user_profile (tenant_id, purpose);
```

### Sổ lệnh xóa (Deletion Orders)
```sql
CREATE TABLE deletion_orders (
  id UUID PRIMARY KEY,
  user_id UUID,
  scope TEXT,               -- all|profile|events|vector|datasets
  reason TEXT,
  status TEXT,              -- queued|running|completed|failed|on_hold
  created_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ
);
```

### Manifest DSAR Export
```json
{
  "user_id": "u-123",
  "generated_at": "2025-11-01T12:00:00+07:00",
  "datasets": [
    {"name":"profile","records":1,"pii":"masked"},
    {"name":"events","records":153,"pii":"masked"},
    {"name":"vector_points","records":321,"pii":"removed"}
  ],
  "hash": "sha256:ab...ef"
}
```

---

## 🔧 Orchestrator (pseudo)
```python
def run_deletion_order(order_id):
    order = db.get_order(order_id)
    if is_legal_hold(order.user_id):
        return db.update(order, status="on_hold")

    # 1) transactional DB
    soft_delete_db(order.user_id)
    purge_db(order.user_id)

    # 2) VectorDB
    qdrant.delete(collection="docs", filter={"must": [{"key":"user_id","match":{"value": order.user_id}}]})

    # 3) Files/S3
    s3_delete_prefix(f"tenants/{tenant}/users/{order.user_id}/")
    s3_mark_versions_for_expiration(...)

    # 4) Logs/Lake
    delete_partitions("events", user_id=order.user_id)

    # 5) Datasets (training/analytics)
    mark_dataset_rows_removed("round>=current-12mo", user_id=order.user_id)
    rebuild_manifests()

    # 6) Backups policy
    register_backup_exclusion(order.user_id)

    write_audit("delete", order.user_id, details=...)
    db.update(order, status="completed", completed_at=now())
```

---

## 🧭 Chính sách giữ liệu theo mục đích (Purpose Binding)
| Purpose           | TTL mặc định | Ghi chú |
|-------------------|--------------|--------|
| runtime_chat      | 90 ngày      | Nhật ký hội thoại đã mask (5.2); phục vụ vận hành/sự cố. |
| personalization   | 365 ngày     | LTM profile; cần consent=true; người dùng có thể xoá bất kỳ lúc nào. |
| booking           | 24–36 tháng  | Phục vụ quyết toán/CSKH; có thể kéo dài theo nghĩa vụ pháp lý/kiện tụng. |
| training          | 12 tháng     | Chỉ dữ liệu đã scrub; dừng dùng ngay khi có DSAR delete. |
| security_audit    | 12–24 tháng  | Bản ghi truy cập/điều khiển; WORM/immutable; ngoại lệ legal hold. |

> **Legal hold** luôn đình chỉ purge cho bản ghi liên quan đến vụ việc.

---

## 🔒 An toàn & Tuân thủ (điểm mấu chốt)
- **Default deny:** nếu thiếu `purpose/data_class` → không ghi log; hoặc gán **TTL ngắn nhất**.
- **Immutable audit:** S3 **Object Lock / WORM** cho log quyết định xóa.
- **Selective restore:** quy trình **restore** không khôi phục dữ liệu đã purge (có **deletion ledger**).
- **Cross-tenant isolation:** lệnh xóa chỉ ảnh hưởng trong `tenant_id`.
- **Thứ tự xoá:** **Vault/PII gốc → liên kết → chỉ mục phụ/Vector → dữ liệu thứ cấp (datasets, logs) → backup policy**.

---

## 🧪 Test kịch bản bắt buộc
- **TC01:** Xóa **toàn bộ** hồ sơ người dùng (DB+Vector+S3+Logs+Datasets) → kiểm tra **không còn truy xuất** được.
- **TC02:** Xóa **hội thoại phiên** (`runtime_chat`) nhưng **giữ hồ sơ booking** (nghĩa vụ pháp lý).
- **TC03:** **DSAR Export** không chứa **PII thô**; chỉ masked/token/hash.
- **TC04:** **Legal hold** bật → mọi purge **dừng**, audit ghi nhận.
- **TC05:** **Backup restore** thử nghiệm → xác nhận exclusion list ngăn dữ liệu đã xoá quay lại.

---

## 📈 KPI & SLA gợi ý
- **DSAR Export SLA:** ≤ **7 ngày**; **Delete SLA:** ≤ **15–30 ngày**.
- **Residual Rate** sau purge: **0** bản ghi còn sót (theo re-scanner).
- **Policy Coverage:** **100%** bảng/collection có `purpose`, `data_class`, `ttl`.
- **Restore Violations:** **0** lần khôi phục nhầm dữ liệu đã purge.
- **Legal Hold Accuracy:** **100%** bản ghi liên quan bị đóng băng đúng thời điểm.

---

## ✅ Checklist MVP (1–2 tuần)
- **Retention Registry** (YAML/DB) + middleware gán `purpose`, `data_class`.
- **Lifecycle jobs:** soft-delete → purge; lịch chạy **hàng ngày**.
- **DSAR API:** `/privacy/export`, `/privacy/delete`, `/privacy/status/:id`.
- **Tích hợp** VectorDB/S3/Logs vào pipeline xoá.
- **Deletion ledger** + **immutable audit** (S3 Object Lock).
- **Legal hold flag** & **UI** cho admin pháp chế.
- **Bộ test E2E** 5 kịch bản & **rescan tool** sau purge.

