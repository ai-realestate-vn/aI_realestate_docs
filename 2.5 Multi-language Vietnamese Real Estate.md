## 🌏 Chức năng 2.5 – Hỗ trợ đa ngôn ngữ (Multi-language)

### 🎯 Mục tiêu & Phạm vi
**Mục tiêu:** Người dùng có thể nói/gõ vi/en (hoặc trộn), hệ thống hiểu chính xác, phản hồi theo ngôn ngữ ưa thích và chuẩn định dạng địa phương (đơn vị, ngày giờ, tiền tệ).  
**Phạm vi:** Text + Voice (STT/TTS), NLU/DM/NLG, RAG, Cards/Quick replies, UI i18n, đơn vị/format.

---

### 🧭 Kiến trúc ngôn ngữ (Router tổng)
Ba mô hình vận hành:
1. **Native-per-lang:** NLU riêng cho từng ngôn ngữ → chính xác cao, bảo trì nhiều.
2. **Canonical-lang pivot:** Dịch input → ngôn ngữ chuẩn (vi-VN) → NLU một ngôn ngữ → đơn giản.
3. **Multilingual models:** Dùng mT5, XLM-R, LaBSE, multilingual-mpnet → linh hoạt, phụ thuộc model.

**Khuyến nghị MVP:** Hybrid = langid → nếu vi/en thuần → native; nếu trộn → pivot_vi.

---

### 🚦 Luồng runtime (Text & Voice)
#### LangID & Code-switch detect
```json
lang = {"primary":"vi","share":0.8,"codeswitch":true|false}
```
- CLD3/FastText + heuristic (tỷ lệ token vi-en, dấu tiếng Việt).

#### Normalize theo locale
- **vi-VN:** dấu câu, viết hoa, ITN: “hai phẩy năm tỷ” → 2.5 tỷ.
- **en-US:** punctuation, unit (“70 sqm”, “$1200/mo”).

#### Routing
- codeswitch=false → NLU_vi hoặc NLU_en.
- codeswitch=true → translate_to=canonical (vi-VN) → NLU_vi.

#### NLU → DM → Tools/RAG
- RAG: embeddings đa ngôn ngữ, hoặc dịch query_en→vi.
- Slots chuẩn hoá ISO datetime, VND/USD normalized.

#### NLG & Output Language
- Phản hồi theo `output_locale` (primary hoặc user_profile.locale).
- Glossary/Terminology song ngữ: “sổ hồng” ↔ “pink book (ownership certificate)”.

#### TTS/STT (nếu voice)
- STT: chọn model theo lang đoạn.
- TTS: giọng theo `output_locale`; barge-in như 2.2.

#### Cards/Quick replies
- Dịch label theo locale; payload giữ schema gốc.

---

### 🧱 Hợp đồng dữ liệu (locale & profile)
```json
{
  "session_id": "s_ulid",
  "lang_runtime": {
    "primary": "vi-VN",
    "detected_turn": "en-US",
    "codeswitch": true,
    "route_strategy": "pivot_vi"
  },
  "user_profile": {
    "preferred_locale": "vi-VN",
    "fallback_locales": ["en-US"]
  },
  "i18n": {
    "currency_default": "VND",
    "unit_area": "m²",
    "date_tz": "Asia/Ho_Chi_Minh"
  }
}
```

---

### 🌐 RAG đa ngôn ngữ (chiến lược)
- Index song ngữ: mỗi tài liệu giữ `lang`, `title_localized`, `chunk_lang`.
- Embeddings đa ngôn ngữ (LaBSE/multilingual-mpnet) → cross-lang search.
- Re-rank ưu tiên cùng ngôn ngữ.
- Nếu chỉ có tài liệu vi → dịch query_en→vi → trả kết quả English + chú giải thuật ngữ.

---

### 🧩 Dịch & Glossary (chống lệch nghĩa)
**Glossary cứng:**
- “Sổ hồng” ↔ “Pink book (ownership certificate)”  
- “Phí quản lý” ↔ “Management fee”  
- “Hướng nhà” ↔ “House orientation”  

**Chính sách dịch:**
- Không dịch địa danh (“Phú Mỹ Hưng”, “Quận 7”).
- Nếu dùng USD → hiển thị cả hai: “$1,200/tháng (~29 triệu VND)”.
- Ngày giờ theo `Asia/Ho_Chi_Minh` và format theo locale.

---

### 🔌 UI i18n (FE & BE)
FE: dùng i18n keys + message catalogs (`vi.json`, `en.json`); Intl.* để format số, tiền, ngày.
BE: NLG templates song ngữ; `card_mapper` sinh title/subtitle theo locale.

**Ví dụ:**
```json
// vi.json
{
  "cta.book_weekend": "Đặt lịch cuối tuần",
  "label.area": "Khu vực",
  "msg.no_result": "Hiện chưa có căn nào phù hợp."
}
// en.json
{
  "cta.book_weekend": "Book this weekend",
  "label.area": "Area",
  "msg.no_result": "No matching listings yet."
}
```

---

### 🧠 Bộ nhớ & đa ngôn ngữ (1.4 kết nối)
- **STM:** lưu slot chuẩn + `last_user_locale`.
- **LTM:** preferred_locale, honorific (Anh/Chị vs Mr./Ms.).
- Khi switch ngôn ngữ giữa phiên → đổi output_locale ngay, không mất context.

---

### 🧪 Tình huống tiêu biểu
1. **User gõ English:**
   - “Find me 2-bed apartments in District 7 under 120k USD.”
   - LangID: en-US → NLU_en → DM → NLG_en → Cards_en.
   - Hiển thị: “$120k (~3.0 tỷ VND)”.

2. **Code-switch:**
   - “Căn 2PN ở Q7, budget around 1200 dollars per month.”
   - LangID: mix → pivot_vi → dịch input → NLU_vi → output vi-VN.
   - Hiển thị: “~$1,200/tháng (~29 triệu VND)”.

3. **RAG pháp lý:**
   - Query English → dịch en→vi → tìm → trả English + note:  
     “According to the Housing Law 2014 (Luật Nhà ở 2014)…”.

---

### 🧰 Pseudo Implementation
**Router (TypeScript)**
```ts
function routeByLanguage(utter: string, ctx: Ctx): RoutedInput {
  const det = langid(utter);
  const pref = ctx.profile?.preferred_locale ?? det.primary;

  if (!det.codeswitch) {
    return { text: utter, nlu_lang: det.primary, output_locale: pref, route: "native" };
  }
  const pivot = "vi-VN";
  const textVi = translate(utter, det.primary, pivot, { glossary: GLOSSARY });
  return { text: textVi, nlu_lang: "vi-VN", output_locale: pref, route: "pivot_vi" };
}
```

**NLG chọn template theo locale**
```ts
function t(key: string, locale: string, vars?: Record<string,string>) { /* i18n lookup */ }
function priceLabel(vnd?: number, usd?: number, locale = "vi-VN") {
  // format ưu tiên locale; bổ sung quy đổi nếu cần
}
```

**RAG truy hồi đa ngôn ngữ (Python)**
```python
def rag_search(query, user_locale):
    q = query
    if user_locale.startswith("en") and CORPUS_LANG == "vi":
        q = translate(query, "en", "vi")
    hits = vector_db.search(emb(q, m_use_multilingual))
    return rerank(hits, prefer_lang=user_locale)
```

---

### 🔒 Guardrails & Chính sách
- Không dịch sai thuật ngữ pháp lý; nếu mơ hồ → chú giải ngoặc.
- PII: không dịch hoặc tiết lộ thông tin nhạy cảm.
- Currency/Unit: hiển thị rõ đơn vị, quy đổi có ký hiệu “~”.
- Nếu không có TTS/STT phù hợp locale → fallback text.

---

### 📈 KPI gợi ý
| Chỉ số | Mục tiêu |
|---------|-----------|
| LangID accuracy | > 97% |
| Code-switch detect | > 90% |
| Task success parity | ±3% giữa vi/en |
| Glossary adherence | > 98% |
| Format correctness | > 99% |
| Locale retention | ≈ 100% |

---

### ✅ Checklist MVP 2.5
- [x] LangID + code-switch detector
- [x] Router native/pivot_vi
- [x] NLU_vi & NLU_en (hoặc multilingual) + slot normalizer
- [x] i18n FE/BE (message catalogs, Intl format)
- [x] RAG đa ngôn ngữ (embeddings, rerank, dịch query)
- [x] Glossary song ngữ + quy tắc đơn vị/tiền tệ/ngày giờ
- [x] STT/TTS đa giọng + fallback text
- [x] Telemetry: lang-routing, glossary adherence, format correctness

