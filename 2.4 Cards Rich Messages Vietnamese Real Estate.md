## 🏙️ Chức năng 2.4 – Cards / Rich Messages (Thẻ thông tin giàu nội dung)

### 🎯 Mục tiêu & I/O
**Mục tiêu:** Biến kết quả từ 1.2 (NLG + data) thành thẻ giàu nội dung: ảnh, tiêu đề, mô tả, thuộc tính, huy hiệu, nút hành động.  
**Input:** plan (từ 1.2), results (DB/RAG/search), context/profile.  
**Output:**
```json
{
  "messages":[{"type":"text","content":"Em thấy 3 căn phù hợp ở Quận 7 dưới 3 tỷ."}],
  "cards":[],
  "card_layout":{"type":"carousel","page":1,"total_pages":2},
  "quick_replies":[],
  "telemetry":{"source_ids":["db:listing:..."]}
}
```

---

### 🧱 Card Schema (tối thiểu & mở rộng)
```json
{
  "id":"listing-SR-2850",
  "title":"Sunrise Riverside - 2PN - 2.85 tỷ",
  "subtitle":"70 m² • Quận 7 • Hướng Đông",
  "image":"https://cdn/.../sr.jpg",
  "badges":["Sổ hồng","Full nội thất","View sông"],
  "fields":[
    {"k":"Diện tích","v":"70 m²"},
    {"k":"Phòng ngủ","v":"2"},
    {"k":"Phí QL","v":"~18k/m²"}
  ],
  "price":{"value":2850000000,"currency":"VND","label":"2.85 tỷ"},
  "meta":{"project":"Sunrise Riverside","address_hint":"Nguyễn Hữu Thọ, Q7"},
  "actions":[
    {"type":"postback","label":"Đặt lịch xem","payload":{"action":"book_visit","id":"listing-SR-2850"}},
    {"type":"open_url","label":"Xem chi tiết","url":"https://.../SR-2850"},
    {"type":"postback","label":"Lưu","payload":{"action":"save","id":"listing-SR-2850"}}
  ]
}
```

**Nguyên tắc:** thông tin quan trọng trên 1–2 dòng đầu; thuộc tính nhanh trong fields; CTA chính đứng đầu danh sách actions.

---

### 🧭 Pipeline sinh thẻ (server)
1. **Plan chọn layout:**
   - RESULT_LIST → carousel|grid
   - NO_RESULT → thẻ gợi ý (relax/switch area)
   - RAG_ANSWER → thẻ “Nguồn/Trích dẫn”
2. **Mapping kết quả → thẻ:**
   - Chuẩn hoá units (m², tỷ/triệu)
   - Rút gọn title (Tên – PN – Giá)
   - Ảnh 1200×800 (16:10/3:2), có placeholder
3. **Enrichment (tuỳ chọn):**
   - Huy hiệu (legal, nội thất, view)
   - Khoảng cách tới trung tâm hoặc trường học (nếu có geocode)
4. **Rank & Cap:**
   - Xếp hạng theo độ phù hợp → giới hạn ≤10/thẻ/trang
5. **Packaging & Telemetry:**
   - Gắn source_ids, tracking_id để đo CTR

---

### 🖥️ FE Render (React/Next) – component hoá
```ts
export type CardAction =
  | { type: "postback"; label: string; payload: any }
  | { type: "open_url"; label: string; url: string };

export interface Card {
  id: string;
  title: string;
  subtitle?: string;
  image?: string;
  badges?: string[];
  fields?: { k: string; v: string }[];
  price?: { value?: number; currency?: "VND"|"USD"; label?: string };
  meta?: Record<string, any>;
  actions: CardAction[];
}
```

#### CardItem component
```tsx
function CardItem({ c }: { c: Card }) {
  return (
    <div className="rounded-2xl shadow p-3 w-[280px] bg-white dark:bg-zinc-900">
      <div className="aspect-[3/2] rounded-xl bg-neutral-200 overflow-hidden">
        {c.image ? <img src={c.image} alt={c.title} className="w-full h-full object-cover" /> : <div className="w-full h-full" />}
      </div>
      <h3 className="mt-2 text-sm font-semibold line-clamp-2">{c.title}</h3>
      {c.subtitle && <p className="text-xs text-neutral-500 line-clamp-2">{c.subtitle}</p>}
      {c.badges && <div className="mt-1 flex flex-wrap gap-1">
        {c.badges.slice(0,3).map(b => <span key={b} className="text-[10px] px-2 py-0.5 rounded-full border">{b}</span>)}
      </div>}
      {c.fields && <dl className="mt-2 grid grid-cols-2 gap-x-3 gap-y-1">
        {c.fields.slice(0,4).map(f => (<div key={f.k}><dt className="text-[10px] text-neutral-500">{f.k}</dt><dd className="text-xs">{f.v}</dd></div>))}
      </dl>}
      <div className="mt-2 flex items-center justify-between">
        <span className="text-sm font-bold">{c.price?.label}</span>
        <div className="flex gap-2">
          {c.actions.slice(0,2).map((a,i)=>(
            a.type==="postback" ? <button key={i} className="text-xs underline" data-payload={JSON.stringify(a.payload)}>{a.label}</button>
            : <a key={i} className="text-xs underline" href={a.url} target="_blank" rel="noreferrer">{a.label}</a>
          ))}
        </div>
      </div>
    </div>
  );
}
```

#### Carousel & Pagination
- Horizontal scroll (mobile) + grid 3–4 cột (desktop)
- Pagination: `{action:"paginate_cards", page:2}`
- Dùng skeleton khi loading ảnh

---

### 🔌 Hợp đồng hành động (Action Contract)
```json
{ "action":"book_visit", "id":"listing-SR-2850", "time_hint":"weekend" }
{ "action":"refine", "filters":{"so_phong":2,"dien_tich_min":70} }
{ "action":"save", "id":"listing-SR-2850" }
{ "action":"open_map", "lat":10.73, "lng":106.70 }
```

FE bắt sự kiện click → gửi POSTBACK qua WS/REST; nhận MESSAGE kế tiếp.

---

### 🧠 Quy tắc trình bày (Real-estate best-practice)
- **Tiêu đề:** Tên – PN – Giá
- **Subtitle:** Diện tích • Khu vực • Hướng
- **Price:** Rút gọn (2.85 tỷ, 12.5 triệu/tháng)
- **Badges:** tối đa 3 – ưu tiên pháp lý, view, nội thất
- **Fields:** 3–4 dòng (m², PN, WC, phí QL)
- **CTA:** “Đặt lịch xem” hoặc “Xem chi tiết” luôn hiển thị
- **Ảnh:** ưu tiên ảnh sáng, rõ; fallback placeholder
- **A11y:** alt ảnh, aria-label cho nút, hỗ trợ keyboard

---

### 🔒 Guardrails & Chính sách
- Không bịa dữ liệu; trường trống → ẩn (không ghi “N/A”)
- Giá/Pháp lý → thêm nhãn “giá có thể thay đổi”
- Liên kết ngoài → mở tab mới, có nguồn rõ ràng
- Không hiển thị PII (số nhà/số điện thoại riêng)

---

### 📈 Telemetry & KPI
| Metric | Target |
|---------|---------|
| Card CTR | ≥ 25–35% |
| CTA Conversion | ≥ 12–20% |
| Dwell time | đo hover/expand |
| Image load success | ≥ 98% |
| First paint | < 1.2s |
| No-result fallback CTR | > 10% |

Tracking payload:
```json
{
  "event":"card_click",
  "card_id":"listing-SR-2850",
  "action":"book_visit",
  "conversation_id":"conv_ulid",
  "ts":1730431200
}
```

---

### 🔁 No-result & Fallback Cards
```json
{
  "id":"fallback-1",
  "title":"Chưa có căn phù hợp",
  "subtitle":"Thử nới ngân sách +200 triệu hoặc xem khu gần Phú Mỹ Hưng?",
  "actions":[
    {"type":"postback","label":"+200 triệu","payload":{"action":"relax_budget","delta":200000000}},
    {"type":"postback","label":"Gần PMH","payload":{"action":"switch_area","area_suggestion":"Phú Mỹ Hưng"}}
  ]
}
```

Error card (dịch vụ lỗi): xin lỗi ngắn + nút thử lại.

---

### 🧪 Test kịch bản (MVP)
| ID | Tên | Mục tiêu |
|----|-----|----------|
| TC01 | 3–5 thẻ listing | Đầy đủ title/subtitle/price/CTA + ảnh fallback |
| TC02 | Ảnh hỏng | Placeholder, layout ổn định |
| TC03 | Click CTA | POSTBACK đúng schema |
| TC04 | Paginate | Cập nhật đúng page/state |
| TC05 | Dark mode | Tương phản ≥ WCAG AA |

---

### ✅ Checklist MVP 2.4
- [x] Schema Card & CardAction thống nhất giữa BE/FE
- [x] Mapper DB→Card + chuẩn hoá đơn vị/giá
- [x] Layout carousel|grid + skeleton + pagination
- [x] Postback contract + telemetry (impression/click/conversion)
- [x] Guardrails: ẩn trường trống, nhãn giá/pháp lý, không PII
- [x] A11y + dark mode + responsive
- [x] Bộ test UI & hợp đồng dữ liệu (contract tests)

