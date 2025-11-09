## 🗣️ Workflow Chức năng 2.2 – STT/TTS (Giọng nói hai chiều)

### 🎯 Mục tiêu & I/O
**Mục tiêu:** Trò chuyện bằng giọng nói mượt, độ trễ thấp, chính tả & ngắt câu chuẩn Việt, có barge-in và endpointing chuẩn.  
**Input:** Audio người dùng (mic, 16–48 kHz, mono).  
**Output:**
- `stt_result`: {text, lang, timestamps, confidence} (streaming/immediate)
- `tts_audio`: PCM/Opus (48 kHz) hoặc MPEG-DASH/HLS (nếu dài)
- **Sự kiện:** PARTIAL_TEXT, FINAL_TEXT, TTS_START, TTS_END, BARGE_IN

---

### 🧱 Kiến trúc tổng quan
**Client (Web/Mobile)**
- Thu âm (getUserMedia) → VAD (voice activity detection) cục bộ → stream qua WebSocket (Opus 16k/24k/48k).
- Player TTS (WebAudio) + barge-in (tự động dừng phát khi người dùng nói).

**Gateway (BE Realtime)**
- WS Audio Ingest → chuẩn hoá (resample) → STT streaming → gửi partial/final.
- Orchestrator: NLU (1.1) → DM (1.3) → NLG (1.2) → TTS → stream audio về client.

**Engines**
- STT: streaming (Whisper/faster-whisper, Vosk, Riva, GCP/Azure…)
- TTS: low-latency neural TTS (có giọng Việt), chunked + cache.

---

### 🎧 Luồng STT (Streaming)
#### 1. Mic Capture & Preprocess (Client)
- getUserMedia({audio:true}) → resample 16/24 kHz mono → Opus frame 20ms.
- VAD (WebRTC/Silero VAD wasm): chỉ gửi khi có giọng nói.
- Endpointing: im lặng > 500–800ms → đóng segment.

#### 2. Truyền qua WS
```json
{ "type":"AUDIO_CHUNK", "seq":1, "pcm_or_opus":"<base64>", "ts_client":1730430000 }
```
- Heartbeat/keepalive 15s; backpressure nếu mạng yếu.

#### 3. Server Ingest
- Chuẩn hoá sample rate, RMS/AGC nhẹ.
- STT Engine (streaming):
  - PARTIAL_TEXT mỗi 200–400ms.
  - FINAL_TEXT khi endpoint kết thúc.
- Post-processing:
  - Punctuation & casing tiếng Việt.
  - ITN: “hai phẩy năm tỷ” → 2.5 tỷ.
  - Lexicon: “PMH”→“Phú Mỹ Hưng”, “q7”→“Quận 7”.

#### 4. Emit kết quả
```json
{ "type":"STT_PARTIAL", "text":"căn hai phòng...", "confidence":0.72 }
{ "type":"STT_FINAL", "text":"căn 2 phòng ngủ quận 7 dưới 3 tỷ", "confidence":0.87 }
```
- Sau đó, text → 1.1→1.3→1.2.

---

### 🔊 Luồng TTS (Low-latency, interruptible)
#### 1. Response Planning
- Tạo lời thoại ngắn (≤2 câu, 6–12 từ/câu), ưu tiên câu mở-đầu.

#### 2. Synthesis
- Tách chunks 1–2 câu → TTS song song.
- Giọng vi-VN (nữ/nam), tốc độ ~1.0–1.1x.
- Cache theo `hash(text+tone+voice)`.

#### 3. Streaming về client
```json
{ "type":"TTS_START", "voice":"vi-female-01", "sample_rate":48000 }
{ "type":"TTS_CHUNK", "seq":3, "data":"<base64>" }
{ "type":"TTS_END", "duration_ms":1450 }
```

#### 4. Barge-in
- Client dừng phát khi người dùng nói (hoặc server báo BARGE_IN=true).
- Server hủy phần TTS còn lại.

---

### 🧩 Giao thức WS (sự kiện chính)
**Client → Server**
```json
{ "type":"AUTH", "token":"<JWT>" }
{ "type":"AUDIO_START", "sr":16000, "codec":"opus", "lang_hint":"vi-VN" }
{ "type":"AUDIO_CHUNK", "seq":17, "data":"<base64>" }
{ "type":"AUDIO_END" }
{ "type":"BARGE_IN" }
```

**Server → Client**
```json
{ "type":"STT_PARTIAL", "text":"...", "confidence":0.7 }
{ "type":"STT_FINAL", "text":"...", "confidence":0.88 }
{ "type":"TTS_START", "voice":"vi-female-01", "sr":48000 }
{ "type":"TTS_CHUNK", "seq":3, "data":"<base64>" }
{ "type":"TTS_END" }
{ "type":"ERROR", "code":"AUDIO_TOO_NOISY" }
```

---

### 🧠 Tối ưu cho tiếng Việt
- Punctuation Restoration để NLU hiểu tốt hơn.
- ITN/NTN: số tiền, diện tích, ngày giờ (“hai giờ rưỡi chiều mai” → datetime).
- Alias địa danh: “quận bảy”, “bình thạnh”, “Q.Thủ Đức”.
- Noise handling: RNNoise/VAD; khuyến nghị mic định hướng.

---

### 🔒 Quyền riêng tư & tuân thủ
- Không lưu audio mặc định; chỉ lưu transcript ẩn danh nếu consent.
- API xoá: `DELETE /v1/voice/session/:id`.
- Thông báo rõ việc sử dụng dữ liệu giọng nói (NĐ 13/2023).

---

### 🧪 KPI & ngưỡng
| Thông số | Mục tiêu |
|-----------|-----------|
| STT partial latency | ≤ 400 ms |
| STT final segment | ≤ 800–1200 ms |
| TTS first audio | ≤ 300–800 ms |
| WER vi-VN | ≤ 10–14% |
| Barge-in delay | <150 ms |
| WS uptime | ≥ 99.5% |
| Packet loss | <1% |

---

### 🛠️ Triển khai nhanh (pseudo)
**Client mic → WS**
```js
const ws = new WebSocket(URL);
const recorder = new OpusRecorder({ sampleRate: 16000 });
await recorder.start();
ws.send(JSON.stringify({type:"AUDIO_START", sr:16000, codec:"opus", lang_hint:"vi-VN"}));
recorder.ondata = (opusFrame) => ws.send(JSON.stringify({type:"AUDIO_CHUNK", seq:++i, data:b64(opusFrame)}));
```

**Server STT (Python pseudo)**
```python
@ws.on("AUDIO_CHUNK")
def on_chunk(data):
    stt.feed(opus_to_pcm(data))
    for partial in stt.partial():
        ws.send({"type":"STT_PARTIAL", "text": punct(partial.text)})

@ws.on("AUDIO_END")
def on_end():
    final = stt.final()
    text = itn_vi(punct(final.text))
    emit_final(text)
    route_to_nlu(text)
```

**Server TTS (Node pseudo)**
```js
function speak(text, voice="vi-female-01") {
  ws.send({type:"TTS_START", voice, sr:48000});
  for (const chunk of tts.synthesizeStream(text, voice)) {
    if (session.barged_in) break;
    ws.send({type:"TTS_CHUNK", seq:nextSeq(), data:b64(chunk)});
  }
  ws.send({type:"TTS_END"});
}
```

---

### 🧳 Lưu trữ & logging (tối thiểu)
```
voice_sessions(id, user_id, started_at, consent, lang, stats_json)
voice_turns(session_id, seq, stt_text, confidence, t0_ms, t1_ms)
voice_errors(session_id, code, detail, ts)
voice_cache(hash_text_voice, audio_ptr, duration_ms, last_used_at)
```

---

### 🧰 Kiểm thử & vận hành
| ID | Tên kiểm thử | Mô tả |
|----|---------------|------------------------------|
| TC01 | Tiếng ồn quán cà phê | WER & latency vẫn đạt |
| TC02 | Câu dài 12–18s | Endpointing đúng |
| TC03 | Barge-in | Dừng <150ms |
| TC04 | Số tiền & đơn vị | ITN chính xác |
| TC05 | Mạng yếu 5% loss | Jitter buffer ổn định |

---

### ✅ Checklist MVP 2.2
- [x] WS audio (AUDIO_START/CHUNK/END) + heartbeat
- [x] STT streaming + partial/final + punctuation & ITN tiếng Việt
- [x] TTS streaming + cache + barge-in client/server
- [x] VAD/endpointing (client), AGC nhẹ, resample
- [x] Privacy: không lưu audio mặc định, consent rõ ràng
- [x] KPI monitor: latency, WER, barge-in, cache hit
- [x] E2E tests: noise, long utterance, reconnect, packet loss

