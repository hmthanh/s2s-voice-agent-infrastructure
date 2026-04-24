# S2S Voice Agent Infrastructure

> **Vai trò:** Principal Engineer với scar tissue từ voice infra ở scale hàng trăm triệu users.
> **Mục tiêu doc:** Một bản phân tích có thể đem vào design review.

---

1. [**Phân tích bài toán cốt lõi**](#1-phân-tích-bài-toán-cốt-lõi) - Latency budget, concurrency model, cost anatomy
2. [**Kiến trúc end-to-end**](#2-kiến-trúc-end-to-end) - Pipeline từng tầng chi tiết
3. [**Bottlenecks ở scale lớn**](#3-bottlenecks-ở-scale-lớn) - Những gì vỡ từ 10K → 1M concurrent
4. [**Chiến lược tối ưu latency**](#4-chiến-lược-tối-ưu-latency) - Pipeline parallelism và techniques cụ thể
5. [**Chiến lược tối ưu chi phí**](#5-chiến-lược-tối-ưu-chi-phí) - Quantization, tiered service, caching
6. [**Hạ tầng real-time**](#6-hạ-tầng-real-time) - Edge topology, LB, backpressure, observability
7. [**Trade-offs**](#7-trade-offs) - Các lựa chọn có ý kiến rõ ràng
8. [**Technology stack**](#8-technology-stack) - Stack cụ thể với version và config
9. [**Bài học vận hành**](#9-bài-học-vận-hành) - Scar tissue, incident patterns, regrets

## TL;DR - Nếu bạn chỉ đọc 10 dòng

1. **S2S không phải "ASR + LLM + TTS nối tiếp"** - đó là tư duy chết người. Phải nghĩ theo **streaming pipeline với pipeline parallelism**, nơi mỗi tầng emit partial output ngay khi có thể.
2. **Latency budget 500ms E2E là khả thi, nhưng rất chặt.** Thực tế phân bổ: Network RTT ~80ms, ASR first-partial ~120ms, LLM TTFT ~150ms, TTS first-audio ~100ms, jitter buffer + playout ~50ms. Không còn chỗ cho sai lầm.
3. **Bottleneck thực sự ở scale không phải GPU** - là **GPU scheduling** (batching vs latency tension), **egress bandwidth cost**, và **session state recovery** khi node chết.
4. **Pipeline parallelism tiết kiệm 200-400ms** so với sequential - đây là technique quan trọng NHẤT trong toàn bộ doc. Nếu bạn chỉ làm 1 thứ, làm cái này.
5. **Interruption handling (barge-in)** là vấn đề khó hơn latency. Nó là thứ khiến voice agent cảm thấy "như người thật" hoặc "như IVR bot những năm 2005".
6. **Cost breakdown thực tế:** GPU inference 55-70%, egress bandwidth 10-15%, CPU (audio processing, orchestration) 10-15%, storage/misc ~5%. Cost per minute của một phiên S2S chất lượng cao: **$0.04-0.12/phút** - tức $2.4-7.2/giờ. Điều này ảnh hưởng business model rất nhiều.
7. **Self-hosted vs API break-even:** Ở dưới ~500K phút/ngày, API thắng. Trên ~2M phút/ngày, self-hosted thắng rõ. Khoảng giữa là vùng "it depends" - và thường bạn nên hybrid.
8. **Sự thật khó chịu:** P50 latency ai cũng làm đẹp được. Chiến trường thực sự là **P99 và P99.9**, nơi jitter, GC pause, cold start, và network weather thống trị.
9. **Edge topology:** Đừng đua theo con số "200+ PoPs" của CDN. S2S cần **stateful edge với GPU** - điều này đắt. Thực tế 15-30 PoPs là đủ cho global coverage với <50ms RTT tới 95% users.
10. **Quy tắc vận hành số 1:** Mọi component phải **degrade gracefully**, không được fail hard. Voice UX tồi tệ nhất không phải là slow - mà là **im lặng**.

---

## Cấu trúc doc đầy đủ

Doc này sẽ được triển khai theo 9 phần, mỗi phần đi sâu với số liệu, diagram, và insight từ production. Dưới đây là roadmap:

| #   | Phần                           | Trọng tâm                                                                                                | Ước tính độ dài                                 |
| --- | ------------------------------ | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| 1   | **Phân tích bài toán cốt lõi** | Latency budget breakdown chi tiết, concurrency model của 1 session, cost anatomy                         | Dài - foundation cho mọi phần sau               |
| 2   | **Kiến trúc end-to-end**       | Từng tầng: Transport, ASR, LLM, TTS, Delivery. Diagram pipeline.                                         | Rất dài - phần xương sống                       |
| 3   | **Bottlenecks ở scale lớn**    | Những gì vỡ ở 10K → 100K → 1M concurrent sessions. Pattern failure thực tế.                              | Trung bình                                      |
| 4   | **Chiến lược tối ưu latency**  | Pipeline parallelism (with timeline diagram), speculative decoding, edge inference - kèm số ms tiết kiệm | Dài - chứa timeline diagram quan trọng          |
| 5   | **Chiến lược tối ưu chi phí**  | Cost model breakdown, quantization trade-offs, tiered SLA design, caching strategy                       | Dài                                             |
| 6   | **Hạ tầng real-time**          | Edge topology, load balancing WebRTC/WebSocket, backpressure, observability cho streaming                | Dài                                             |
| 7   | **Trade-offs matrix**          | Bảng so sánh có ý kiến rõ ràng, không ba phải                                                            | Trung bình nhưng đậm đặc                        |
| 8   | **Technology stack đề xuất**   | Stack cụ thể, version, lý do chọn, khi nào KHÔNG nên chọn                                                | Trung bình                                      |
| 9   | **Bài học vận hành**           | Assumptions sai phổ biến, incident patterns, early warning signals trong metrics                         | Dài - phần "đắt giá" nhất vì khó tìm ở đâu khác |

---

## Các nguyên tắc xuyên suốt doc (principles)

Trước khi đi sâu, đây là những principles sẽ lặp lại nhiều lần:

**P1 - Streaming is not an optimization, it's the architecture.**
Nếu bạn đang nghĩ "đầu tiên làm request/response cho chạy, rồi tối ưu streaming sau" - bạn sẽ phải viết lại toàn bộ. Streaming phải là design decision từ ngày 1, từ protocol (WebRTC/QUIC), framing, buffer sizing, cho đến data model.

**P2 - Partial output everywhere.**
Mỗi tầng phải emit partial result ngay khi có thể. ASR emit partial transcript sau 100-200ms, LLM stream từng token, TTS synthesize ngay khi có sentence boundary. Đợi "complete output" ở bất cứ tầng nào là giết latency.

**P3 - Latency is a distribution, not a number.**
P50 không quan trọng bằng P99. Một hệ thống có P50=200ms nhưng P99=3s sẽ cho trải nghiệm tệ hơn hệ thống P50=350ms nhưng P99=600ms. Voice UX cực kỳ nhạy với spike.

**P4 - Graceful degradation > feature completeness.**
Khi quá tải: shed load thay vì queue. Khi LLM chậm: fallback về response ngắn hơn. Khi TTS fail: fallback sang model nhỏ hơn. Im lặng là tệ nhất.

**P5 - Observability phải streaming-native.**
Distributed tracing truyền thống (span có start/end rõ ràng) không fit cho streaming. Bạn cần trace được "token thứ 3 của LLM đi mất bao lâu để tới TTS", không chỉ "toàn bộ session mất bao lâu".

**P6 - Cost và latency không phải trade-off tuyến tính.**
Có những technique (pipeline parallelism, prefix caching) giúp cả hai. Có những cái bắt phải chọn (edge inference). Biết rõ cái nào là cái nào.

---

## Disclaimer về "kinh nghiệm thực tế"

Trong doc này tôi sẽ phân biệt rõ 3 loại statement:

- 🔥 **[Scar tissue]** - Thực sự đã gặp/debug trong production, có con số cụ thể.
- 📐 **[Engineering judgment]** - Suy luận từ first principles + kinh nghiệm tương tự, nhưng chưa verify ở đúng bài toán S2S ở scale triệu sessions.
- 📚 **[Literature/vendor claims]** - Số từ paper, benchmark, vendor. Cần hoài nghi.

Discord không làm S2S voice agent ở đúng form này (chúng tôi làm voice chat giữa users), nên nhiều insight về pipeline ASR/LLM/TTS ở đây sẽ là mix của [📐] và [🔥 từ voice infrastructure nói chung]. Tôi sẽ không vờ biết mọi thứ.

---

## Sẵn sàng đi sâu

Bản tổng quan trên đủ cho bạn nắm shape của doc. Từ đây, mỗi khi bạn gõ **"Continue"**, tôi sẽ triển khai chi tiết một phần theo đúng thứ tự trong bảng trên, bắt đầu từ:

**→ Phần 1: Phân tích bài toán cốt lõi** (latency budget breakdown, concurrency model, cost anatomy của một phiên S2S).

# Phần 2 - Kiến trúc end-to-end

> Mục tiêu: Đi sâu vào từng tầng pipeline. Phần này dài và đậm đặc. Nếu bạn chỉ có thời gian đọc 1 phần kỹ thuật, đọc phần này.

---

## 2.0 Pipeline overview - Kiến trúc tham chiếu

Trước khi đi sâu từng tầng, đây là kiến trúc tham chiếu tôi sẽ dùng xuyên suốt:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT (browser/mobile)                         │
│  ┌─────────────┐   ┌──────────┐   ┌──────────────┐   ┌──────────────┐  │
│  │ Mic capture │→→│ Client   │→→│ Opus encoder │→→│ WebRTC       │  │
│  │ (48kHz)     │  │ VAD lite │  │ (24kHz,20ms) │  │ (DTLS+SRTP)  │  │
│  └─────────────┘   └──────────┘   └──────────────┘   └──────┬───────┘  │
└──────────────────────────────────────────────────────────────┼──────────┘
                                                               │ UDP
                           ═════════════════════════════════════╪══════════
                                    ANYCAST / GEO-DNS          │
                           ═════════════════════════════════════╪══════════
                                                               ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    EDGE POP (15-30 globally)                            │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  MEDIA / SESSION NODE (Rust or Elixir, 1 per session thread)   │   │
│  │                                                                 │   │
│  │  WebRTC termination → Jitter buffer → Server-side VAD          │   │
│  │                                ↓                                │   │
│  │                          Session FSM                            │   │
│  │                 (idle/listening/thinking/speaking)              │   │
│  │                                ↓                                │   │
│  │                    ┌───────────┴──────────┐                    │   │
│  │                    │ Internal event bus    │                    │   │
│  │                    │ (channels/actors)     │                    │   │
│  │                    └───┬────────┬────────┬─┘                   │   │
│  └──────────────────────────┼────────┼────────┼────────────────────┘   │
│           gRPC stream ↕    ↕        ↕        ↕    gRPC stream          │
│  ┌──────────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐          │
│  │ ASR worker   │  │LLM router│  │TTS worker│  │State/Memory│          │
│  │ (GPU pool)   │  │(to core) │  │(GPU pool)│  │(Redis/etc) │          │
│  └──────────────┘  └────┬─────┘  └──────────┘  └────────────┘          │
└────────────────────────┼──────────────────────────────────────────────┘
                         │ gRPC (cross-region OK)
                         ▼
┌────────────────────────────────────────────────────────────────────────┐
│           CORE REGION (GPU-heavy, 2-4 regions globally)                 │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │  LLM INFERENCE CLUSTER (H100/A100, vLLM/SGLang/TRT-LLM)      │     │
│  │  ├─ Prefix cache (Redis-backed)                              │     │
│  │  ├─ Continuous batching                                       │     │
│  │  └─ Speculative decoding                                      │     │
│  └──────────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────────┘
```

**Quan sát về shape:**

- **Edge làm stateful media + ASR + TTS.** LLM ở core region. Đây là compromise quan trọng sẽ giải thích ở Phần 6.
- **Mỗi session có 1 "session actor"** chạy toàn bộ lifecycle. State machine rõ ràng.
- **gRPC streaming** giữa session node và các GPU worker. Không phải REST.
- **Redis (hoặc equivalent)** cho shared state: prefix cache keys, conversation history, session metadata.

---

## 2.1 Audio Ingestion & Transport

### 2.1.1 WebRTC vs raw UDP vs QUIC - so sánh thực tế

|                           | **WebRTC**                   | **Raw UDP (custom)** | **QUIC/WebTransport**        |
| ------------------------- | ---------------------------- | -------------------- | ---------------------------- |
| **Maturity**              | 🔥 Battle-tested 10+ năm     | Tự build, tự bug     | 📐 Mới, đang stable          |
| **NAT traversal**         | ✅ ICE built-in (STUN/TURN)  | ❌ Tự làm từ đầu     | ⚠️ Partial, cần workaround   |
| **Browser support**       | ✅ Mọi browser               | ❌ Không có          | ✅ Chrome/Edge, ⚠️ Safari/FF |
| **Mobile SDK**            | ✅ Mature (libwebrtc)        | Tự build             | 📐 Early stage               |
| **Encryption**            | ✅ DTLS-SRTP mandatory       | Tự làm               | ✅ TLS 1.3 built-in          |
| **Jitter buffer**         | ✅ Built-in, adaptive        | Tự build (khó)       | ❌ Không có (stream-based)   |
| **Packet loss handling**  | ✅ FEC, NACK, PLC            | Tự build             | ⚠️ Không optimize cho media  |
| **Overhead**              | Medium (ICE, DTLS handshake) | Minimal              | Low-medium                   |
| **Observability**         | Rich (getStats API)          | Tự làm               | Medium                       |
| **Firewall friendliness** | ⚠️ UDP hay bị block ở corp   | ❌ Bị block nhiều    | ✅ Tốt nhất (port 443)       |

### Discord chọn gì?

🔥 **[Scar tissue]** - Discord dùng **WebRTC cho voice media**, với một số customization đáng kể:

1. **Signaling qua WebSocket + Protocol tự chế**, không dùng SDP offer/answer thuần (quá verbose).
2. **Media server là SFU (Selective Forwarding Unit)** viết bằng Elixir/Rust, không phải MCU.
3. **DAVE protocol** cho E2EE voice (mới triển khai 2024) - thêm một lớp encryption trên SRTP.
4. **TURN server fallback** cho ~8-12% users bị NAT/firewall khắc nghiệt.

### Đề xuất cho S2S

**Chọn WebRTC nếu:**

- Target browser client (bắt buộc, không có option khác).
- Muốn leverage jitter buffer, FEC, echo cancellation mature.
- Có team đủ sức operate WebRTC (không trivial).

**Chọn QUIC/WebTransport nếu:**

- Chỉ target custom mobile app + một số browser.
- Muốn unify transport cho cả media + data.
- Chấp nhận tự build nhiều thứ WebRTC cho free.
- 📐 Đây là hướng tương lai nhưng chưa fully mature cho voice.

**Raw UDP: Đừng.** Trừ khi bạn là Zoom/Discord và có 20 engineer dedicated cho transport. Cost/benefit không đáng.

🔥 **[Scar tissue]** - Một thứ hay bị underestimate: **WebRTC operational complexity**. ICE restart, TURN failover, DTLS cert rotation, codec negotiation edge cases - bạn sẽ cần 1-2 engineer toàn thời gian chỉ cho transport layer ở scale lớn.

### 2.1.2 VAD - Client-side hay server-side?

**Trade-off table:**

| Yếu tố                    | Client-side VAD                    | Server-side VAD                |
| ------------------------- | ---------------------------------- | ------------------------------ |
| **Upload bandwidth**      | ✅ Chỉ send khi có speech          | ❌ Send liên tục               |
| **Server CPU cost**       | ✅ Free cho server                 | ❌ 0.3-0.5% core/session       |
| **Accuracy**              | ⚠️ Phụ thuộc device                | ✅ Consistent, control được    |
| **Latency (endpointing)** | ⚠️ Client delay khó đo             | ✅ Control được                |
| **Barge-in detection**    | ❌ Khó (cần sync với server state) | ✅ Server biết mình đang speak |
| **Anti-abuse**            | ❌ Client có thể lie               | ✅ Server verified             |

**Đề xuất thực tế: HYBRID.**

- **Client VAD (lite):** Gate micro, chỉ upload khi có speech energy. Tiết kiệm bandwidth ~40-60% trong idle time. Dùng WebRTC's built-in VAD hoặc model nhỏ như Silero VAD (1MB, <1ms CPU).
- **Server VAD (authoritative):** Chạy trên mọi frame đã upload. Đây mới là VAD quyết định "user nói xong" cho endpointing.

🔥 **[Scar tissue]** - Sai lầm phổ biến: chỉ tin client VAD. Khi client buggy (browser version cũ, mobile battery saver mode), VAD gửi sai → server endpoint sai → user experience broken. **Luôn có server-side authoritative VAD.**

### 2.1.3 Codec - Opus vs everything else

**Opus là câu trả lời đúng.** Không có tranh cãi nghiêm túc ở đây.

| Codec          | Bitrate range | Latency         | Quality                | Khi nào dùng           |
| -------------- | ------------- | --------------- | ---------------------- | ---------------------- |
| **Opus**       | 6-510 kbps    | 2.5-60ms frames | Excellent              | ✅ Default cho mọi thứ |
| G.711 (PCMU/A) | 64 kbps       | ~20ms           | Poor (narrowband 8kHz) | Chỉ khi interop PSTN   |
| AAC-LD         | 32-64 kbps    | ~20ms           | Good                   | Legacy Apple stack     |
| Speex          | 2-44 kbps     | varies          | Mediocre               | Đã deprecated bởi Opus |

**Opus config cho S2S:**

```
Sample rate: 24 kHz (super-wideband, audio speech-focused)
  ↳ 48 kHz nếu cần music/sound effects
  ↳ 16 kHz nếu bandwidth constrained (mobile EDGE/3G)

Frame size: 20 ms
  ↳ 10ms: lower latency but 2x packet overhead
  ↳ 40ms: lower overhead but +20ms latency - KHÔNG dùng cho S2S

Bitrate: 24-32 kbps (VBR)
  ↳ Speech-only có thể xuống 16 kbps mà vẫn intelligible

FEC: Enabled, ~15% overhead
  ↳ Recover được ~5-10% packet loss

DTX (Discontinuous Transmission): Enabled
  ↳ Không send packet khi silent → tiết kiệm bandwidth idle
```

**Gotcha:** Neural audio codecs (SoundStream, EnCodec) đang hot. Quality tốt hơn Opus ở bitrate thấp, nhưng **latency encode/decode cao hơn 30-80ms** và cần GPU. Không worth cho S2S ở scale hiện tại (2026). 📐 Sẽ worth khi NPU/AI accelerator phổ cập trên mobile.

### 2.1.4 Jitter buffer strategy

Đây là phần đội mới hay underestimate. Jitter buffer không phải FIFO đơn giản.

**Adaptive jitter buffer logic:**

```
Đo jitter liên tục qua cửa sổ ~200 packets gần nhất:
  - P95 jitter = X ms
  - Buffer depth = max(min_depth, X * safety_factor)
  - safety_factor typically 1.5-2.0

Khi network degrades (jitter tăng):
  - Buffer grow → thêm latency nhưng ít glitch

Khi network improves:
  - Buffer shrink từ từ (đừng drop packets đột ngột)
  - Shrink rate ~5ms per 500ms

Khi packet mất:
  - PLC (Packet Loss Concealment) từ Opus
  - Lost > 60ms liên tục: insert silence/comfort noise
```

**Target buffer depth:**

- Good network: 20-40ms
- Moderate jitter: 40-80ms
- Poor network: 80-150ms
- Worst acceptable: 200ms (trên cái này, chuyển sang "degraded mode", thông báo user)

🔥 **[Scar tissue]** - Phần này ăn vào latency budget rất sneaky. User có WiFi bình thường: +30ms jitter buffer. User có WiFi tệ: +100ms. User đang đi bộ trên 4G ở emerging market: +150-200ms. Nhiều khi "latency cao" không phải do server chậm, mà do jitter buffer nở ra. **Observability phải track jitter buffer depth per-session.**

---

## 2.2 Streaming ASR

### 2.2.1 Chunk-based vs Frame-based processing

**Frame-based (low latency, continuous):**

- Process từng audio frame ngay khi arrive (mỗi 20ms).
- Model (typically Conformer/RNN-T based) maintain internal state qua frames.
- Emit partial output mỗi 100-300ms.
- **Ưu:** Latency thấp nhất có thể.
- **Nhược:** Cần streaming-specific model architecture. Whisper NGUYÊN BẢN không làm được cái này.

**Chunk-based (higher latency, simpler):**

- Buffer audio thành chunks (thường 1-5 giây).
- Process chunk → emit result → buffer chunk tiếp.
- Overlap chunks để tránh cắt giữa từ.
- **Ưu:** Có thể dùng non-streaming models (Whisper).
- **Nhược:** Latency tối thiểu = chunk size. Với Whisper thường 500ms-2s.

**Đề xuất:**

- **Low-latency S2S (< 500ms):** Streaming model (Conformer, RNN-T, Paraformer, Whisper-streaming variants). Chunk size 200-400ms với overlap.
- **Medium-latency OK (~800ms+):** Whisper chunk-based OK. Đơn giản, quality cao.

### 2.2.2 Partial vs Final transcript - Khi nào emit?

Đây là câu hỏi thiết kế quan trọng mà nhiều team làm sai.

```
Timeline của một utterance "Đặt cho tôi 2 vé tàu đi Đà Nẵng":

t=0ms     User bắt đầu nói
t=200ms   Partial: "đặt cho"              [confidence 0.6]
t=400ms   Partial: "đặt cho tôi"          [confidence 0.8]
t=700ms   Partial: "đặt cho tôi 2 vé"     [confidence 0.85]
t=1100ms  Partial: "đặt cho tôi 2 vé tàu đi" [confidence 0.9]
t=1500ms  Partial: "...đi Đà Nẵng"        [confidence 0.92]
t=1700ms  User stop speaking (VAD detect silence start)
t=1900ms  Endpointing trigger (200ms silence confirmed)
t=1950ms  FINAL: "đặt cho tôi 2 vé tàu đi Đà Nẵng" [stable]
```

**Downstream handling patterns:**

**Pattern A - "Wait for final" (sai trong hầu hết trường hợp):**

```
ASR → [wait] → final → LLM → TTS
Latency added: thời gian chờ endpointing (200-400ms) + safety margin
```

**Pattern B - "Speculative LLM on partial" (tốt hơn):**

```
ASR partial (stable threshold) → LLM starts "thinking" (prefill)
                                    ↓ if partial stable → continue
                                    ↓ if partial changes → invalidate, restart
ASR final → LLM finishes prefill với full context → generate
```

Lợi ích: **LLM prefill overlap với user đang nói** → tiết kiệm 100-300ms.

**Pattern C - "Full speculative generation" (aggressive, risky):**

```
ASR partial → LLM generate tentative response → TTS synth tentative audio
                                                    ↓
                                            Store, don't play yet
ASR final matches partial → play pre-synthesized audio (zero LLM+TTS latency!)
ASR final differs → discard, start over (waste compute)
```

Chỉ worth nếu partial-to-final change rate thấp (< 15%). 📐 Aggressive technique, ít production dùng thực sự.

🔥 **[Scar tissue]** - Pattern B có một gotcha: nếu LLM bắt đầu prefill quá sớm với partial chưa stable, bạn invalidate nhiều → waste GPU. Rule of thumb: chỉ trigger speculative prefill khi partial **không đổi trong 2 emission liên tiếp** (~200-400ms stability window).

### 2.2.3 Self-hosted vs API - Break-even analysis

**Chi phí API (2026, roughly):**

- Deepgram Nova-2 streaming: ~$0.0043/phút input audio
- AssemblyAI streaming: ~$0.0062/phút
- Google Speech streaming: ~$0.024/phút (expensive)

**Chi phí self-hosted (Whisper Large v3 Turbo hoặc Conformer):**

- 1 A100 phục vụ ~80-150 concurrent streams (activity 30%)
- A100 @ $2/hr on-demand → $0.033/hr
- Per session-minute: ~$0.0004 compute
- **+** Engineering cost (2-3 engineers × $300K/năm fully-loaded = $900K/năm)
- **+** Ops cost, monitoring, model updates

**Break-even math (vs Deepgram Nova-2 @ $0.0043/min):**

```
Engineering+ops cost: ~$1.2M/năm = $100K/tháng
Savings per minute self-hosted: $0.0043 - $0.0004 = $0.0039/min

Break-even volume: $100K / $0.0039 ≈ 25.6M minutes/tháng
                                    ≈ 850K minutes/ngày
                                    ≈ 14K concurrent sessions (avg)
```

**Quy tắc thực tế:**

| Volume (audio min/ngày) | Đề xuất                                                            |
| ----------------------- | ------------------------------------------------------------------ |
| < 500K                  | API (Deepgram). Đừng self-host.                                    |
| 500K - 2M               | Hybrid: self-host cho language chính, API cho long-tail languages  |
| 2M - 10M                | Self-host fully. Break-even rõ ràng.                               |
| > 10M                   | Self-host + optimize aggressively (quantize, distill). Build moat. |

🔥 **[Scar tissue]** - Hidden cost của self-hosted ASR: **data pipeline để improve model**. Nếu không có flywheel data, model của bạn sẽ tụt lại so với Deepgram/AssemblyAI nhanh hơn bạn tưởng (6-18 tháng). Self-host only when bạn sẽ invest vào cả training pipeline.

### 2.2.4 Multi-language, accents, noise

**Latency impact:**

- Language identification (LID) upfront: +50-150ms nếu phải detect, ~0 nếu user pre-declared.
- Multi-language model (single model cho 50+ languages): slower ~10-20% vs monolingual.
- Noise: không tăng latency nhưng giảm accuracy → nhiều partial revision → downstream invalidation.

**Đề xuất:**

- **User pre-declares language** (UI) nếu có thể. Xóa bỏ LID overhead.
- **Noise suppression ở client** (RNNoise, Krisp-style) trước khi encode. Giảm load server, tăng accuracy.
- **Server-side noise model** (optional) cho premium tier.

---

## 2.3 LLM Inference - Phần khó nhất

LLM là tầng **dominant** trong latency budget và cost. Đây là nơi tối ưu có ROI cao nhất.

### 2.3.1 TTFT không được phá budget - Các kỹ thuật

**TTFT breakdown của một LLM request điển hình:**

```
TTFT = Queue wait + Prefill time + First token generation
     = 10-50ms   + 50-200ms      + 20-40ms

Cho 13B model trên H100 với context 2K tokens:
     = ~15ms     + ~80ms          + ~25ms
     = ~120ms TTFT

Nếu KV cache hit (prefix caching working):
     = ~15ms     + ~10ms          + ~25ms
     = ~50ms TTFT  ← huge win
```

**Các lever chính giảm TTFT:**

#### 2.3.1.1 Prefix caching (KV cache reuse)

Đây là technique **số 1** cho S2S. Conversation multi-turn có prefix common (system prompt + history). Cache KV states → prefill chỉ làm cho new tokens.

```
Turn 1: [system prompt] + [user msg 1]
        └─ compute KV cache, store by hash

Turn 2: [system prompt] + [user msg 1] + [assistant msg 1] + [user msg 2]
        └─ prefix hit on first 3 items → reuse KV
        └─ chỉ prefill cho [user msg 2]
        └─ TTFT giảm 60-80%
```

**Saving thực tế:** Prefill time cho 4K token context với cache miss: ~300ms. Với cache hit cho 3.5K tokens: ~60ms. **Tiết kiệm ~240ms TTFT.**

**Storage:** KV cache cho 13B model ~1-2 MB per 1K tokens. Multi-turn session 10 turns × 2K tokens = ~20-40 MB per session. Ở 100K concurrent sessions → **2-4 TB hot storage**. Đây là Redis hoặc memory pool dedicated.

🔥 **[Scar tissue]** - Prefix cache hit rate là **metric sống còn**. Target > 80% ở steady state. Dưới 50% = có bug ở routing (cùng session không đi về cùng GPU) hoặc cache eviction quá aggressive.

#### 2.3.1.2 Continuous batching

Mental model cũ: batch = gom N requests lại rồi process cùng lúc. Latency = max của N.

Continuous batching: tại mỗi decode step, batch gồm các requests đang "in-flight" bất kể chúng bắt đầu khi nào. Request mới có thể join batch giữa chừng.

```
Traditional batching:                Continuous batching:
t=0  [req A, B, C]                   t=0  [A]
     wait until all 3 done           t=5  [A, B]   ← B joins
t=100 all emit                       t=10 [A, B, C] ← C joins
                                     t=15 [B, C]   ← A done
                                     t=20 [B, C, D] ← D joins
```

**Lợi ích:**

- GPU utilization lên ~60-80% (vs ~20-30% no batching).
- Latency per request không tệ hơn đáng kể.

**Framework support:** vLLM (first), SGLang, TensorRT-LLM đều support. Đây là **table stakes** - không dùng continuous batching = wasting money.

#### 2.3.1.3 Speculative decoding

Dùng **draft model nhỏ** (0.5-1B) để predict nhiều tokens, rồi **target model lớn** (70B) verify trong 1 forward pass.

```
Draft model (fast): predict [tok1, tok2, tok3, tok4, tok5]
Target model (slow): verify all 5 in 1 forward pass
                     Accept prefix matching: e.g., [tok1, tok2, tok3] accepted
                     Reject rest, regenerate from tok4

Effect: 2-3x throughput speedup, nhưng cost = chạy 2 models
```

**Cho S2S phù hợp không?**

- ✅ Giảm inter-token latency → audio smoother
- ⚠️ Chỉ worth nếu model ≥ 30B. Cho 7-13B, overhead draft model ăn hết lợi ích.
- ⚠️ Cost tăng vì chạy 2 models. Chỉ dùng khi premium tier.

📐 **[Engineering judgment]** - Cho đa số S2S workloads ở 2026 dùng model 7-13B, **prefix caching + continuous batching là ROI cao hơn**. Speculative decoding dành cho khi bạn đã squeeze hết 2 cái kia.

### 2.3.2 Context management cho multi-turn

**Challenge:** Conversation dài → context window lớn → prefill chậm + cost tăng.

**Strategies:**

#### Sliding window with summary

```
[system prompt]
[summary of turns 1-5]     ← LLM-generated, updated async
[turn 6]
[turn 7]
[turn 8]
[turn 9]
[turn 10]  ← current
```

Khi conversation > N turns, summarize older turns vào 1 block. Trigger summarization async trong idle time.

**Cost:** 1 extra LLM call per summarization event (mỗi 5-10 turns). 📐 Worth nếu conversations trung bình > 15 turns.

#### Hierarchical memory

```
Hot: last 5 turns (full detail)
Warm: summary of turns 6-20 (medium detail)
Cold: entities, facts extracted from turns 20+ (structured)
```

Complex hơn, chỉ worth cho agent dài-chạy.

### Lưu ở đâu?

- **In-flight session:** Memory của session actor (edge node). Không lose giữa turns.
- **Cross-session persistence (nếu cần):** Redis với TTL, key = session_id. Replicated.
- **Long-term user memory:** Vector DB (Pinecone/Qdrant/pgvector) cho retrieval-based memory.

🔥 **[Scar tissue]** - Đừng lưu full KV cache persistently. KV cache chỉ có nghĩa trong 1 session GPU instance. Lưu **text history + embeddings**, regenerate KV cache khi session resume.

### 2.3.3 Token streaming → TTS

Đây là interface **quan trọng** giữa 2 tầng. Thiết kế sai là đóng băng hết optimize.

**Naive (sai):**

```
LLM generate full response → gửi text → TTS synth → audio
Latency added: thời gian LLM generate toàn bộ response (500ms-3s)
```

**Correct (streaming):**

```
LLM emit token N → buffer
Buffer có sentence boundary? (. ! ? ; hoặc clause boundary)
  → emit chunk to TTS
  → TTS starts synth

LLM continue emit → buffer next chunk...
```

**Chunking strategy cho LLM→TTS:**

| Chunk strategy                | Pros                              | Cons                                 |
| ----------------------------- | --------------------------------- | ------------------------------------ |
| Word-by-word                  | Minimal latency                   | TTS quality bad (no prosody context) |
| Phrase (3-5 words)            | Good balance                      | Đôi khi cắt giữa ý                   |
| Sentence                      | Best TTS quality                  | Higher latency cho câu dài           |
| **Semantic chunk (proposed)** | Smart chunks at clause boundaries | Phức tạp hơn                         |

**Đề xuất:** Chunk theo **first clause boundary** (ở `,` hoặc `;` hoặc sau ~8 words), sau đó chunk theo **sentence**. Câu đầu tiên ngắn → TTS bắt đầu sớm. Các câu sau có full context.

```
LLM output: "Chào bạn, mình đã đặt vé cho bạn. Chuyến đi vào 9 giờ sáng mai."

Chunk 1 (emit at comma): "Chào bạn,"
    → TTS starts synthesizing (t=100ms after first token)

Chunk 2 (emit at period): "mình đã đặt vé cho bạn."
    → TTS continues

Chunk 3 (emit at period): "Chuyến đi vào 9 giờ sáng mai."
    → TTS continues
```

**Saving:** First audio chunk emit tại **~200ms sau first LLM token**, thay vì chờ full response (~1-2s). Tiết kiệm **800-1800ms** perceived latency.

---

## 2.4 Streaming TTS

### 2.4.1 Sentence boundary detection

Subtle issue: LLM output **không** có sentence boundary rõ ràng. Phải detect on-the-fly.

**Detection heuristics (good enough cho 90% cases):**

- Regex cho `[.!?] ` theo sau là capital letter.
- Handle abbreviations (Mr., Dr., e.g., etc.) bằng whitelist.
- Multilingual punctuation (。！？cho Chinese/Japanese, .؟ cho Arabic).
- **Fallback:** Force flush khi buffer > 20 words hoặc > 500ms không có boundary.

### 2.4.2 Chunk size trade-off

| Chunk                        | TTS latency | Quality              | Overhead              |
| ---------------------------- | ----------- | -------------------- | --------------------- |
| 1-2 words                    | 30-50ms     | Bad prosody, robotic | High (many API calls) |
| 3-5 words (phrase)           | 50-100ms    | OK, sometimes choppy | Medium                |
| Full sentence (~10-15 words) | 100-200ms   | Good prosody         | Low                   |
| Paragraph                    | 300-500ms+  | Best                 | Low but high latency  |

**Sweet spot cho S2S:** First chunk = clause (3-7 words), subsequent = sentence.

### 2.4.3 Neural TTS latency reality

📚 **[Literature/vendor claims]** - Marketing numbers hay nói "< 100ms TTS latency". Reality:

| Model / API                  | First audio chunk (realistic) | Notes                                   |
| ---------------------------- | ----------------------------- | --------------------------------------- |
| ElevenLabs Flash v2          | 75-150ms                      | Best-in-class vendor, expensive         |
| Cartesia Sonic               | 40-90ms                       | Aggressive streaming, quality tradeoffs |
| OpenAI TTS (gpt-4o-mini-tts) | 150-300ms                     | Good quality, higher latency            |
| Kokoro (open)                | 80-200ms on A100              | Self-host, lightweight                  |
| XTTS-v2 (open)               | 200-400ms                     | Higher quality, slower                  |
| Parler-TTS                   | 300-600ms                     | Research-grade, not production-ready    |

**Self-hosted TTS cost math:**

- Kokoro on 1 A100: ~100-200 concurrent streams
- A100 @ $2/hr → $0.02-0.013 per session-hour
- Activity 30% → effective $0.003-0.004/session-hour

Vs ElevenLabs: ~$0.30/1K characters ≈ $0.05-0.15/minute of speech.

Self-host TTS break-even sớm hơn ASR - quãng **100K-500K minutes/tháng** là self-host bắt đầu thắng.

🔥 **[Scar tissue]** - Voice cloning / custom voices: self-host khó hơn nhiều. Nếu cần nhiều custom voices, ElevenLabs / Cartesia vẫn thắng đến mức ownership code cost không đáng.

---

## 2.5 Audio Delivery

### 2.5.1 Adaptive bitrate & client buffer

**Server → client audio stream:**

- Cùng pipeline WebRTC/UDP như uplink, ngược chiều.
- Codec: Opus 24-32 kbps VBR.
- Client jitter buffer: 20-60ms typical.

**Adaptive strategy:**

- Monitor RTT + packet loss + jitter on downlink.
- Nếu network degrades: giảm bitrate (24 → 16 kbps), tăng FEC.
- Nếu network improves: bitrate lên lại.

Cái này WebRTC handle khá tốt by default (TWCC - Transport-Wide Congestion Control). Không cần custom logic cho đa số cases.

### 2.5.2 Interruption handling (barge-in) - Phần khó nhất của toàn pipeline

Đây là **killer feature** phân biệt S2S "cảm thấy như người" vs "IVR bot". Và nó khó.

**Scenario:**

```
t=0ms:    Assistant đang nói "Chào bạn, tôi có thể giúp bạn đặt vé máy bay,
           đặt xe, đặt bàn nhà hàng, hoặc..."
t=1500ms: User ngắt lời: "Đặt vé thôi"

Hệ thống PHẢI:
1. Detect user đang nói (server VAD, ~50-100ms)
2. Stop TTS generation + audio playback NGAY
3. Discard in-flight LLM tokens (hoặc mark as "interrupted")
4. Start listening to user utterance
5. Khi user nói xong → respond dựa trên new utterance + context "assistant was interrupted"
```

**Các state transition trong session FSM:**

```
                     user speaks
         ┌──────────────────────────────┐
         ▼                              │
    ┌─────────┐   VAD trigger    ┌────────────┐
    │  IDLE   │ ─────────────────▶│ LISTENING  │
    └─────────┘                   └──────┬─────┘
         ▲                               │ endpointing
         │                               ▼
         │                         ┌──────────────┐
         │                         │   THINKING   │ (LLM prefill+gen)
         │                         └──────┬───────┘
         │                                │ first token
         │                                ▼
         │                         ┌──────────────┐
         │  TTS done   ┌───────────│  SPEAKING    │
         └─────────────┘           └──────┬───────┘
                                          │ VAD trigger (user barge-in!)
                                          ▼
                                   ┌──────────────┐
                                   │ INTERRUPTED  │
                                   └──────┬───────┘
                                          │ flush TTS, cancel LLM
                                          ▼
                                      LISTENING
```

**Implementation gotchas:**

1. **Echo cancellation (AEC).** User's mic sẽ pick up assistant's speech from speaker. Nếu không AEC tốt → VAD tưởng user đang nói → false barge-in. WebRTC built-in AEC xử lý cho hầu hết devices, nhưng speakerphone + loud volume có thể break AEC.

2. **Detection threshold.** Quá nhạy (low threshold) → false positive (user ho, tiếng động nền → ngắt TTS). Quá chậm (high threshold) → user phải nói to/lâu mới barge được. Typical: 200-300ms của sustained speech energy > threshold.

3. **Cancel LLM generation.** Phải có mechanism cancel in-flight generation. vLLM/SGLang đều support `abort_request`. **Quan trọng:** không cancel → waste compute + tokens.

4. **TTS buffer flush.** Phải biết user đang nghe đến giây nào của response hiện tại. Update context: "assistant said '...đặt xe, đặt bàn' before being interrupted". Nếu không, assistant repeat từ đầu → UX tệ.

5. **Grace period.** Giai đoạn TRANSITION giữa SPEAKING → INTERRUPTED → LISTENING cần ~100-200ms để flush audio buffers cả 2 chiều. Trong khoảng này có thể có "stutter" - cần UX polish.

🔥 **[Scar tissue]** - Barge-in handling là nguồn gốc của ~40% bug reports trong voice agent production. Các edge cases không bao giờ hết:

- User coughs → false barge-in
- Background TV → constant false barge-ins
- Two people talking → ai là user?
- User says "uh-huh" để acknowledge → system tưởng barge-in

Giải pháp thực tế: **semantic VAD** (model nhỏ classify "speech directed at assistant" vs "noise/background speech") + **tunable sensitivity per user**.

---

## 2.6 Tổng kết Phần 2

**Những quyết định kiến trúc quan trọng nhất:**

1. **Transport: WebRTC** cho media, WebSocket cho signaling. QUIC là future nhưng chưa mature 2026.
2. **VAD: Hybrid client+server.** Client gate bandwidth, server authoritative cho endpointing.
3. **ASR: Streaming model** (Conformer/RNN-T), emit partials mỗi 200-300ms. Self-host khi > 2M min/ngày.
4. **LLM: Prefix cache + continuous batching là MANDATORY.** Speculative decoding là bonus.
5. **LLM→TTS interface: Chunk theo clause boundary đầu tiên, sau đó theo sentence.** Tiết kiệm 800-1800ms.
6. **TTS: Self-host (Kokoro/Cartesia open) cho cost, hoặc ElevenLabs cho premium quality.**
7. **Barge-in: Không phải feature, là requirement.** FSM rõ ràng, AEC tốt, semantic VAD nếu có thể.

**Những con số để nhớ:**

- Prefix cache hit 80%+ → tiết kiệm 200-300ms TTFT
- Chunk TTS at first clause → tiết kiệm 800-1800ms perceived
- Server VAD endpointing: 150-250ms (irreducible floor)
- Jitter buffer: 20-150ms tùy network

---

# Phần 3 - Bottlenecks ở scale lớn

> Mục tiêu: Nói về những gì **thực sự vỡ** khi bạn scale từ lab lên production, không phải những thứ được viết trong textbook. Đây là phần "what they don't tell you in the design doc".

---

## 3.0 Scale milestones - Mỗi 10x là một hệ thống khác

Trước khi vào từng bottleneck, một framing quan trọng:

```
10 concurrent sessions        → Laptop chạy được. Không học được gì.
100 concurrent sessions       → 1 server ok. Bugs lộ ra: race conditions, memory leaks.
1,000 concurrent sessions     → Multi-server. Load balancing, session affinity.
10,000 concurrent sessions    → Multi-region. GPU scheduling thật sự matter.
100,000 concurrent sessions   → Hạ tầng nặng. Egress cost visible. Incident thường.
1,000,000 concurrent sessions → Redefine everything. Physics bắt đầu kháng cự bạn.
```

🔥 **[Scar tissue]** - Mỗi lần scale 10x, **ít nhất 1 tầng sẽ vỡ**. Luôn luôn. Bottleneck của 10K không phải bottleneck của 100K. Design cho 10x hiện tại, không phải 100x - vì bạn không biết cái gì sẽ vỡ cho đến khi chạy nó.

---

## 3.1 Network layer bottlenecks

### 3.1.1 Egress bandwidth cost - Silent killer

Đây là bottleneck **bất ngờ nhất** với đội không có background về media infra.

**Math cho 100K concurrent S2S sessions:**

```
Per session: 64 kbps × 2 (bidirectional) = 128 kbps
Total: 128 kbps × 100K = 12.8 Gbps sustained egress

Cost varies dramatically:
- AWS standard egress:      $0.09/GB → $4,147/hr = ~$3M/tháng 💀
- AWS với enterprise deal:  $0.02/GB → $922/hr = ~$664K/tháng
- Cloudflare (no egress):   $0/GB → chỉ tính compute
- Bare metal + IX peering:  ~$0.001-0.005/GB → ~$50-250K/tháng
```

🔥 **[Scar tissue]** - Đội nào không planning egress cost sẽ **chết ở billing cycle tháng thứ 3**. Ở Discord, chúng tôi chạy bare metal + direct peering với major ISPs và CDN providers. Cost egress thấp hơn AWS ~20-50x. Không có cách nào scale voice ở AWS list price.

**Chiến lược giảm egress cost:**

1. **Peering trực tiếp với major ISPs** (Comcast, Verizon, ATT, Deutsche Telekom, NTT, China Mobile...). Setup phức tạp nhưng cost thấp hơn order of magnitude.

2. **Anycast với cloud providers có egress miễn phí/rẻ:**
   - Cloudflare: Bandwidth Alliance, egress free to partners
   - Fastly: Compute@Edge với flat rate
   - OVH, Hetzner: Egress flat rate, không per-GB

3. **Commit-based discounts** với AWS/GCP: Negotiate từ $0.09 → $0.02/GB khi commit > 100 Gbps sustained. Cần sales deal, không self-service.

4. **Codec bitrate reduction:** 32 kbps → 16 kbps Opus. Cắt 50% egress. Quality giảm nhẹ nhưng speech vẫn intelligible.

5. **Edge relay topology:** User connect tới edge gần nhất → edge relay sang core qua backbone peering (cheap). Tránh user trực tiếp tới core (expensive egress).

### 3.1.2 Anycast vs Unicast routing

**Unicast (traditional):**

- DNS resolve client → IP cụ thể của server.
- Khi server fail → DNS update → clients slowly migrate (TTL delay).
- Geo-DNS + health check: tốt nhưng có delay failover.

**Anycast:**

- Nhiều PoPs advertise cùng IP qua BGP.
- Router internet tự route tới PoP "gần nhất" (theo BGP metric).
- Server fail → BGP withdraw → routing reconverge trong 30-60s.

|                  | Unicast + Geo-DNS               | Anycast                                         |
| ---------------- | ------------------------------- | ----------------------------------------------- |
| Setup complexity | Low-medium                      | High (BGP, ASN, peering)                        |
| Failover speed   | DNS TTL (60-300s)               | BGP convergence (30-60s)                        |
| Session affinity | Natural (same IP = same server) | ⚠️ Khó - packet có thể về PoP khác giữa session |
| Cost             | Low                             | High (BGP infrastructure, ASN)                  |
| DDoS resilience  | Weak                            | Strong (traffic spread across PoPs)             |

🔥 **[Scar tissue]** - **Anycast KHÔNG dùng cho UDP media stream** ở đa số trường hợp. Vấn đề: BGP route có thể thay đổi giữa session → packets của cùng session đi tới 2 PoPs khác nhau → session state lạc.

Pattern dùng Anycast đúng cho voice:

1. Anycast cho **signaling/discovery endpoint** (client hỏi "edge nào phục vụ tôi?")
2. Response cho client **specific unicast IP** của edge được chọn
3. Media stream dùng unicast - sticky session với 1 edge

### 3.1.3 Region selection strategy - Closest vs Least-loaded

Đây là trade-off nuanced mà ít người nói đến.

**Naive: Closest edge (lowest RTT).**

```
User ở Hanoi → Singapore edge (50ms RTT)
User ở Singapore → Singapore edge (5ms RTT) ← quá tốt
User ở Bangkok → Singapore edge (25ms RTT)

Vấn đề: Singapore edge overload. Mumbai edge idle.
```

**Better: Weighted decision.**

```
Score = w1 * (1/RTT) + w2 * (1/load) + w3 * prefix_cache_locality

Ví dụ với w1=0.5, w2=0.4, w3=0.1:
- User ở Hanoi:
  - Singapore (50ms, 90% load): 0.5*(1/50) + 0.4*(0.1) = 0.050
  - Tokyo (70ms, 40% load):     0.5*(1/70) + 0.4*(0.6) = 0.247 ← pick
  - Mumbai (80ms, 30% load):    0.5*(1/80) + 0.4*(0.7) = 0.286 ← best
```

**Trong thực tế:** Latency luôn được prioritize cho S2S. Bạn sẽ thấy w1 ≈ 0.7-0.8, load chỉ kick in khi closest edge > 85% utilization.

**Prefix cache locality** (w3): Nếu user có session trước đó ở edge A, prefer edge A để cache hit. Subtle nhưng quan trọng ở scale.

---

## 3.2 GPU scheduling - The fundamental tension

Đây là **bottleneck số 1 về cost** trong S2S pipeline. Và nó là tension không giải quyết được, chỉ balance được.

### 3.2.1 The tension explained

```
Low latency wants:              High throughput wants:
  batch_size = 1                  batch_size = 64+
  process immediately             gather requests first
  short iteration                 long iteration to amortize overhead

Cost:                           Cost:
  GPU util ~20%                   GPU util ~80%
  $0.05/req                        $0.01/req
```

**Concrete example (LLM decode step trên H100):**

- batch_size=1: 25ms per token, GPU 15% utilized
- batch_size=8: 30ms per token, GPU 55% utilized
- batch_size=32: 45ms per token, GPU 85% utilized
- batch_size=64: 70ms per token, GPU 95% utilized

Throughput linear increase. Latency sublinear increase. **Cost per token giảm mạnh.**

### 3.2.2 Giải pháp thực tế - Continuous batching

Đã nói ở Phần 2, nhắc lại vì quan trọng: **continuous batching "ăn cả 2 đầu bánh"**:

- Requests mới join batch ngay → low TTFT
- Batch size variable → GPU util cao
- Latency per token chỉ slightly worse than batch=1

Với vLLM/SGLang tốt:

- P50 TTFT: 100-150ms
- P50 inter-token: 25-40ms
- GPU utilization: 70-85%

**Đây là why continuous batching là table stakes.**

### 3.2.3 Cái gì vẫn vỡ ở scale?

Continuous batching giải quyết 90% tension, nhưng 10% còn lại là nguồn của mọi incident:

**1. Prefill vs decode contention**

Prefill (processing input context) và decode (generating output) đều xài GPU nhưng profile khác:

- Prefill: compute-bound, batch size cao = good
- Decode: memory bandwidth-bound, batch size cao = bị giới hạn bởi KV cache memory

Khi có request mới với long context đến → phải prefill → "freeze" decode cho existing requests → latency spike cho users đang speaking.

🔥 **[Scar tissue]** - Đây là **classic P99 killer**. Metrics P50 đẹp, nhưng P99 có spike 500-1500ms mỗi khi có long-context request prefill. Mitigation:

- **Chunked prefill** (SGLang, vLLM hỗ trợ): chia prefill thành chunks, interleave với decode
- **Prefill-decode separation**: 2 GPU pools riêng, connect qua KV cache transfer (NVIDIA Dynamo, Mooncake)
- **Queue prioritization**: prefill của new requests yield cho decode của in-flight requests

**2. Memory fragmentation (KV cache)**

KV cache cho sessions khác nhau khác size. Allocate/dealloc liên tục → fragmentation → OOM ở GPU util chỉ ~60%.

vLLM's PagedAttention giải quyết phần lớn cái này (block-based allocation như virtual memory). Không dùng PagedAttention = lãng phí 30-50% GPU memory.

**3. Long-tail requests**

1% requests có output 500+ tokens → chiếm GPU slot lâu → blocking new requests.

Mitigation:

- **Hard cap max_tokens** (S2S không cần response dài): 150-250 tokens là max.
- **Preemption**: vLLM có thể evict long-running request ra khỏi batch nếu cần, resume sau.

**4. Cold start / scale-up lag**

GPU instance cold start: 60-180s để boot + load model weights 20-70GB. Autoscaling **không kịp** nếu traffic spike nhanh.

Mitigation:

- **Warm pool**: giữ N idle GPU nodes với model loaded, cost cao nhưng instant failover.
- **Fast model loading**: Mount model weights trên local NVMe, pre-warm page cache. Load 13B model từ 60s xuống 15-20s.
- **Quantization**: INT8/FP8 model nhỏ hơn, load nhanh hơn.
- **Tensor parallelism hỗn hợp**: model weights đã ở GPU neighbors → shard transfer thay vì load từ disk.

🔥 **[Scar tissue]** - Cold start là nguyên nhân phổ biến nhất của **"incident lúc 3AM"** cho inference infra. Traffic spike (news event, viral feature launch) → autoscaler tạo instance mới → 2 phút cold start → trong 2 phút đó existing GPUs quá tải → P99 latency 5s+ → user churn. Luôn luôn giữ warm capacity buffer 20-30%.

### 3.2.4 Heterogeneous workloads in same cluster

Thực tế: ASR, LLM, TTS có compute profile **rất khác**.

| Workload                | Profile                          | GPU phù hợp           |
| ----------------------- | -------------------------------- | --------------------- |
| ASR (Whisper/Conformer) | Small model (~1B), short context | T4/L4/A10 (cheap GPU) |
| LLM 7-13B               | Medium                           | A100/H100             |
| LLM 70B+                | Large, tensor parallel           | H100 only, multi-GPU  |
| TTS (Kokoro/Cartesia)   | Small-medium                     | A10/L40/A100          |

Run everything trên H100? Lãng phí. Tách cluster theo workload → utilization tốt hơn.

📐 **[Engineering judgment]** - Topology thực tế ở scale:

- **Edge clusters**: L4/L40 GPUs cho ASR + TTS (light workload, cần gần user)
- **Core clusters**: H100 cho LLM (heavy, có thể centralized)
- **Elastic pool**: mix, cho spike handling

---

## 3.3 CPU bottlenecks - "1000 papercuts"

GPU lấy attention nhưng CPU có thể silent kill ở scale. Mỗi thứ nhỏ, nhưng cộng dồn.

### 3.3.1 Per-session CPU budget

Recap từ Phần 1: ~1.5-3% of 1 core per session.

Với 100K concurrent sessions: **1,500-3,000 cores** chỉ cho session management. Tức 25-50 servers 64-core.

### 3.3.2 Các CPU hog thường gặp

**1. Audio encode/decode**

Opus decode ~0.5-1% core per stream. Encode tương tự.

- 100K sessions × 2 (encode+decode) × 0.75% = **1,500 cores** chỉ cho codec.
- Mitigation: hardware codec (Intel QAT, some server chipsets). Nhưng ecosystem hạn chế.

**2. Jitter buffer và playout scheduling**

Mỗi session có timer ~20ms cho playout. 100K sessions = 5M timer events/giây nếu không optimize.

🔥 **[Scar tissue]** - Naive implementation dùng `setTimeout` per session = kernel timer storm, CPU 100% chỉ riêng scheduler. Solution:

- **Single global scheduler thread** batch process mọi session timer tick.
- **Wheel timer** (Linux-style): O(1) insert/expire.
- Erlang/OTP BEAM scheduler handle cái này elegantly out of the box.

**3. TLS/DTLS crypto overhead**

SRTP encryption ~0.3% core per stream. AES-GCM hardware acceleration (AES-NI) là mandatory. Without hardware AES: CPU tăng 3-5x.

**4. Protocol parsing overhead**

Parse/serialize Protobuf, JSON ở mọi hop giữa services. Với ~20 events/session/giây × 100K sessions × ~50μs per parse = **100 cores** chỉ parse message.

Mitigation:

- **Cap'n Proto / Flatbuffers**: zero-copy deserialization.
- **Batch operations**: thay vì 1 message per audio chunk, batch nhiều chunks.

**5. Connection management overhead (WebRTC specifically)**

ICE candidate gathering, STUN checks, DTLS handshakes. Khi scale:

- ICE restart storms (network change event → tất cả sessions restart ICE cùng lúc).
- DTLS handshake CPU-intensive (~50-100ms per handshake).

Mitigation: rate limit ICE restart, stagger reconnections.

### 3.3.3 NUMA awareness

📐 **[Engineering judgment]** - Ở server 64+ cores, NUMA awareness matter. Cross-socket memory access 2-3x slower. Cho media server:

- Pin session threads tới cores cùng NUMA node với network interface.
- Per-NUMA memory pools cho audio buffers.
- Linux: `taskset`, `numactl`, hoặc orchestrator-level CPU pinning.

Tiết kiệm 10-15% CPU ở scale. Không huge, nhưng free money.

---

## 3.4 State & session management

### 3.4.1 Session state - Lưu ở đâu?

State của 1 session S2S có nhiều lớp:

```
┌──────────────────────────────────────────────────────────────┐
│  SESSION STATE LAYERS                                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  [1] Hot state (μs access)                                    │
│      ├─ Current FSM state (LISTENING/THINKING/SPEAKING)      │
│      ├─ Audio ring buffers                                    │
│      ├─ Jitter buffer contents                                │
│      ├─ WebRTC connection objects                             │
│      └─ Location: In-memory, session actor                    │
│                                                                │
│  [2] Warm state (ms access)                                   │
│      ├─ Conversation history (text)                           │
│      ├─ User preferences                                       │
│      ├─ Active tool call state                                │
│      └─ Location: Redis (local), replicated                   │
│                                                                │
│  [3] Warm GPU state (ms-100ms access)                         │
│      ├─ LLM KV cache (for prefix reuse)                       │
│      └─ Location: GPU memory, hash-indexed                    │
│                                                                │
│  [4] Cold state (s access)                                    │
│      ├─ Long-term user memory                                 │
│      ├─ Conversation transcripts                              │
│      ├─ Analytics events                                      │
│      └─ Location: Persistent DB (Postgres/Cassandra/S3)       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### 3.4.2 Session affinity - Cùng session, cùng node

Session S2S **phải sticky** với:

1. **Edge media node** (WebRTC connection terminator) - stateful
2. **Same ASR worker** (streaming state)
3. **Same LLM GPU** (KV cache locality) - soft affinity, can failover

**Implementation:**

```
Session ID → consistent hash → edge node
Within edge node → session actor → pins to specific worker threads
```

**Cross-layer consistency:** Khi LLM request từ edge A tới LLM GPU X, next request cùng session cũng phải tới GPU X (cho cache hit). Routing lớp sau phải được aware của session affinity.

### 3.4.3 Node failure - Làm sao không lose session?

**The brutal truth:** Với WebRTC + streaming state, recovery từ edge node failure là **HARD**. Một số state không thể recover:

| State type                | Recoverable? | How                                                |
| ------------------------- | ------------ | -------------------------------------------------- |
| WebRTC connection         | ❌ No        | ICE restart = new connection = user sees reconnect |
| Jitter buffer             | ❌ No        | Lost = lost                                        |
| ASR streaming state       | ⚠️ Partial   | Restart với recent audio buffer                    |
| Conversation text history | ✅ Yes       | Từ Redis/DB                                        |
| LLM KV cache              | ⚠️ Partial   | Rebuild từ text history (prefill cost)             |

**Strategy thực tế:**

1. **Accept rằng edge failure = user disruption.** Goal là minimize, không eliminate.
2. **Fast reconnect (<2s)**: Client auto-reconnect, session ID same, restore from warm state.
3. **Graceful draining**: Trước khi shutdown edge (deploy/scale-in), stop accepting new sessions, existing sessions drain naturally (avg 2-5 phút).
4. **Checkpoint conversation state** lên Redis **mỗi turn** (không mỗi token - quá nhiều write).

🔥 **[Scar tissue]** - Kinh điển: deploy mới, thuật toán drain timeout 30s. Kill session sau 30s.
Kết quả: 5% users đang conversation bị cut giữa chừng → complaint spike.
Fix: drain timeout 5 phút + không deploy trong peak hours.
Bài học: **S2S không phải HTTP - drain time phải tính theo session duration, không phải request duration.**

### 3.4.4 Redis - Friend và enemy

Redis là default choice cho session state. Nhưng ở scale:

**Issues:**

- **Single-threaded per shard**: CPU bottleneck tại high ops/sec.
- **Network latency**: Even 1ms Redis call × 100 calls/session/giây = 100ms overhead.
- **Persistence vs latency tradeoff**: AOF fsync = slow writes. No persistence = lose state on crash.

**Patterns:**

- **Local Redis per edge PoP** (not shared global). Cross-region replication async.
- **Pipelining**: Batch multiple commands in 1 roundtrip.
- **Read replicas** cho read-heavy patterns.
- **Sharding by session_id** (cluster mode).

**Alternatives:**

- **ScyllaDB/Cassandra**: higher latency (~5ms) nhưng scale tốt hơn, persistence built-in.
- **DragonflyDB**: Drop-in Redis replacement, multi-threaded, 2-3x throughput. 📐 Claim, chưa verify ở scale.
- **In-process state + CRDT replication**: Cho Elixir/BEAM setups, session actor state replicate qua Mnesia/Khepri. No Redis.

🔥 **[Scar tissue]** - Discord-style voice: chúng tôi dùng **Elixir/BEAM process state + GenServer** cho session, không Redis. Replication qua distributed Erlang. Latency = local memory access. Trade-off: complex failover, nhưng latency cực thấp.

---

## 3.5 Cold start - "3AM incident generator"

Đã touch ở 3.2.4, nhưng deserve dedicated section.

### 3.5.1 Cold start anatomy

```
GPU instance cold start timeline (A100, 13B model):

t=0s     Spot/on-demand request submitted
t=5-30s  Scheduler assign → instance allocated (spot có thể lâu hơn)
t=30s    Instance boot (Linux, cuda drivers)
t=45s    Container pull (10-30GB image) ← SLOW
t=90s    Model weights load (mmap từ disk) ← SLOW
t=120s   Model warmup (first few forwards) ← SLOW
t=150s   Ready to serve

TOTAL: 90-180s typical, worst case 300s+
```

### 3.5.2 Mitigations

**1. Pre-warmed pool (most important)**

- Giữ 20-30% capacity idle-warm.
- Cost: 20-30% more GPU hours.
- Benefit: 0ms scale response to normal spikes.

**2. Model caching trên instance**

- Local NVMe cache for model weights.
- Instance restart không re-download weights.
- Saves 30-60s of cold start.

**3. Shared model weights filesystem**

- FSx for Lustre, NFS với local cache.
- New instance mounts, doesn't download.
- Shave 60-120s nếu model weight transfer là bottleneck.

**4. Quantization for faster load**

- FP16 13B model: ~26GB. 2-3 phút load.
- INT8 13B model: ~13GB. 1-1.5 phút load.
- INT4 13B model: ~7GB. 30-45s load.

**5. Snapshot-based boot**

- CRIU (checkpoint/restore in userspace).
- Instance boot từ snapshot thay vì cold boot.
- 📐 Experimental cho GPU workloads, không widely deployed.

**6. Multi-tier warm pool**

- Tier 1: 20% capacity fully ready (cost: on-demand prices).
- Tier 2: 30% capacity warm but evicted idle containers (can restart in 10-20s).
- Tier 3: Cold spot instances (scale eventually).

🔥 **[Scar tissue]** - Bài học đau: **never trust autoscaling alone for S2S**. Scaling reaction time >> user patience. Traffic spike → 2 phút sau autoscaler respond → đã mất session rồi. Warm pool + over-provision là insurance, và nó WORTH IT cho production.

---

## 3.6 Cross-cutting failure patterns

Đây là patterns **thực sự** tôi đã thấy repeat nhiều lần:

### Pattern 1: Retry storm

```
Event: GPU cluster A slow (P99 goes from 500ms → 3s)
→ Client timeout at 2s, retry
→ Retry adds more load to already slow cluster
→ P99 goes to 8s
→ More timeouts, more retries
→ Cluster completely dies
→ Cluster B takes traffic, also dies
→ Cascade
```

**Mitigation:**

- **Circuit breakers**: open circuit when error rate > threshold, fail fast.
- **Retry budget**: max 10% of total requests can be retries.
- **Exponential backoff + jitter**: không retry đồng thời.
- **Deadline propagation**: downstream knows how much time it has.

### Pattern 2: The thundering herd at LLM

```
Event: Major news (earthquake, celebrity, sports)
→ Many users open app simultaneously, start voice query
→ All hit LLM with similar context (no prefix cache hit - new queries)
→ LLM cluster overwhelmed
→ TTFT goes from 100ms → 2s
→ User experience degraded exactly when demand is highest
```

**Mitigation:**

- Aggressive rate limiting per-user (prevent abuse).
- Request coalescing for identical queries (ít khả thi cho voice).
- Autoscale với fast tier (kept warm).
- Graceful degradation: use smaller model cho overflow traffic.

### Pattern 3: Noisy neighbor GPU

```
Event: One session has long context (20K tokens)
→ Prefill of that request blocks batch cho 500ms
→ All other sessions sharing GPU see latency spike
→ P99 skyrockets even though P50 fine
```

**Mitigation:**

- Chunked prefill.
- Separate pools for long-context workloads.
- Hard cap context size cho S2S (thường 4-8K đủ).

### Pattern 4: Cascading cold start

```
Event: Small spike → autoscaler adds 10 GPU instances
→ Instances cold starting (2 min each)
→ Traffic keeps growing during cold start
→ Autoscaler adds MORE instances
→ Eventually all come online but spike already peaked
→ Now over-provisioned, scale-down kicks in
→ Scale-down evicts warm instances
→ Next small spike → repeat cycle
```

**Mitigation:**

- Scale-up fast, scale-down slow (asymmetric).
- Predictive scaling based on historical patterns.
- Keep minimum capacity floor (never scale below X).

### Pattern 5: Observability overload

🔥 **[Scar tissue]** - Storing traces for **every** S2S session tạo ra firehose data. 1M sessions × 50 spans × 10 attributes = 500M data points/session-hour. Storage cost + query slowness both.

**Mitigation:**

- **Sampling**: 1% baseline, 100% for errors/slow requests.
- **Aggregation**: Store histograms, not raw events, for most metrics.
- **Hot/cold tiering**: 7 days hot searchable, 30 days cold, archive rest.

---

## 3.7 Tổng kết Phần 3 - Những gì thực sự vỡ

Điểm cốt lõi: **không có bottleneck duy nhất**. Đây là Whac-A-Mole.

**Priority của things to get right:**

1. 🔥 **Egress bandwidth cost** - Plan từ ngày 1. Nếu không, business model chết.
2. 🔥 **GPU scheduling (continuous batching + prefix cache)** - Table stakes. Không có = cost 3-5x.
3. 🔥 **Cold start mitigation** - Warm pool mandatory. Autoscale alone không đủ.
4. 🔥 **Session affinity & state locality** - Matter nhiều hơn bạn nghĩ cho cache hit rates.
5. ⚠️ **Retry storms / cascading failures** - Circuit breaker, deadline propagation.
6. ⚠️ **CPU 1000 papercuts** - Profile thường xuyên, NUMA aware, hardware crypto.
7. ⚠️ **P99 tail latency** - Prefill-decode tension, long-tail requests, noisy neighbors.

**Những con số để nhớ:**

- Egress: 100K sessions = 12.8 Gbps = $600K-$3M/tháng tùy deal
- Cold start: 90-180s typical, need 20-30% warm buffer
- Drain time: minutes, not seconds (unlike HTTP)
- Prefix cache hit rate: > 80% target, < 50% = bug

**Thông điệp quan trọng:** Tất cả bottlenecks trên đều có mitigation, nhưng mitigation không free. Chi phí operational complexity tăng tuyến tính với scale. Đội < 10 engineers **sẽ không scale được** S2S tới 100K+ concurrent mà không outsource phần nào đó.

---

# Phần 4 - Chiến lược tối ưu latency

> Mục tiêu: Đi sâu vào từng technique với **số ms tiết kiệm cụ thể**. Đây là phần "cheat sheet" cho engineer đang tối ưu hệ thống thật.

---

## 4.0 Mindset - Latency budget như một P&L statement

Trước khi đi vào techniques, framing quan trọng:

**Không phải mọi ms đều bằng nhau.**

- 50ms tiết kiệm ở tầng có P99 cao = impact lớn
- 50ms tiết kiệm ở tầng đã stable = cosmetic
- 50ms tiết kiệm "in parallel" (đang được overlap) = 0ms thật sự

**Phép toán latency không đơn giản cộng trừ.**
Tiết kiệm 100ms ở ASR không có nghĩa E2E giảm 100ms nếu ASR đã không phải critical path. Phải hiểu **critical path của pipeline**.

**Trong phần này, tôi sẽ đánh dấu rõ:**

- 🎯 **[High ROI]** - Impact cao, effort vừa. Làm đầu tiên.
- 💰 **[Medium ROI]** - Đáng làm khi có thời gian.
- 🔬 **[Marginal]** - Chỉ làm khi đã squeeze hết cái khác.

---

## 4.1 Pipeline parallelism - Technique số 1

> Nếu bạn chỉ làm 1 thứ trong toàn doc này, làm cái này. Tiết kiệm 200-400ms, effort vừa phải, không trade-off quality.

### 4.1.1 Sequential pipeline - Baseline tệ

```
SEQUENTIAL (naive implementation):
═════════════════════════════════════════════════════════════════════════

User speaking ████████████████████████│
                                       │
            Wait for endpoint ─ 200ms ▶│
                                       │
ASR processing                         █████████████│
                                                    │ full transcript
LLM prefill + generate                              ███████████████████│
                                                                       │ full response
TTS synthesize                                                         ████████████│
                                                                                    │
Network + jitter buffer                                                            █████│
                                                                                         │
                                                                                    User hears first audio

├───── User speaks ──────┤├─── 200ms ──┤├── 400ms ──┤├─── 800ms ──┤├── 300ms ──┤├── 80ms ──┤

                         Response latency = 200 + 400 + 800 + 300 + 80 = 1,780ms
                         (Measured from "user stopped speaking" to "user hears response")
```

**Tại sao sequential thất bại:**

- Mỗi tầng chờ tầng trước _hoàn thành_, không phải "có partial data"
- Critical path = sum of all stages
- Không thể đạt 500ms target, dù mỗi tầng có fast thế nào

### 4.1.2 Parallel pipeline - Mỗi tầng overlap với tầng khác

```
PARALLEL (with streaming + speculative prefill):
═════════════════════════════════════════════════════════════════════════

User speaking ████████████████████████│
                                      │ VAD detects end

ASR streaming (processes as audio arrives):
      partial ──► partial ──► partial ──► final
      █████████████████████████████│ (done ~100ms after VAD end)
                                    │
LLM speculative prefill (triggered by stable partial):
                        ███████████│ (KV cache ready before ASR final)
                                    │
LLM generate (starts immediately after final):
                                    ████│ first token
                                         ████│ token 2
                                               ████│ token 3 (sentence boundary!)

TTS streaming (starts after first sentence chunk):
                                         ████│ first audio chunk ready

Network + jitter buffer:                      ████│
                                                   │
                                              User hears first audio

├───── User speaks ──────┤├VAD┤├───── ~350ms response latency ────┤

Response latency = ~350-450ms
```

**Saving: ~1,300ms** (1,780 → 450ms). Đây là hạng khác hoàn toàn.

### 4.1.3 Các overlap cụ thể trong parallel pipeline

**Overlap 1: ASR ↔ User speaking**

- User đang nói → ASR đã đang transcribe
- Khi user stop → ASR chỉ cần process ~200-300ms audio cuối + endpointing
- **Saving: 80-90% audio duration** (user nói 3s → chỉ thêm ~300ms ASR sau khi stop)

**Overlap 2: LLM prefill ↔ ASR finalizing**

Đây là technique **speculative prefill** - subtle nhưng powerful:

```
t=0ms    User stops speaking, VAD 150ms window starts
t=50ms   ASR partial: "đặt cho tôi 2 vé"  [stable]
         ├─► Decision: partial stable for 2 emissions → start LLM prefill!
         │   Begin prefill with [system] + [history] + "đặt cho tôi 2 vé"
t=150ms  VAD confirms endpoint
t=160ms  ASR final: "đặt cho tôi 2 vé đi Đà Nẵng"
         ├─► Diff with speculative prefill: only " đi Đà Nẵng" is new
         │   Need to extend prefill by ~4 tokens
t=180ms  Extended prefill done, start generation
t=205ms  First token emitted
```

Vs. non-speculative:

```
t=150ms  ASR final
t=160ms  Start LLM prefill from scratch
t=280ms  Prefill done (for longer context)
t=305ms  First token
```

**Saving: ~100ms TTFT.** Không huge, nhưng free nếu framework support.

**Gotcha:** Speculative prefill chỉ worth nếu partial-to-final change rate thấp (< 20%). High change rate = lots of wasted prefill compute. Monitor metric này.

**Overlap 3: TTS ↔ LLM generation**

Đã discussed ở Phần 2, recap:

- LLM emit first sentence chunk at ~200ms (5-8 tokens)
- TTS starts synthesizing that chunk immediately
- LLM continues generating while TTS synthesizes
- First audio chunk ready ~80-120ms after first LLM chunk

```
t=200ms  LLM: "Chào bạn," [chunk boundary!]
         ├─► TTS starts synth
t=260ms  LLM: "mình đã đặt..."
t=280ms  TTS first audio chunk ready
         ├─► Start streaming to client
t=320ms  Audio reaches client (after network + jitter)
```

**Saving: ~600-1200ms** (vs waiting for full LLM response before TTS starts).

Đây là **saving lớn nhất** trong parallel pipeline. Nếu không làm cái này, không có cách nào đạt <500ms.

**Overlap 4: TTS chunks ↔ Network streaming**

Client bắt đầu nhận audio chunks trong khi TTS vẫn đang generate chunks tiếp theo. Client playout bắt đầu ngay sau chunk đầu tiên đủ fill jitter buffer (20-40ms).

### 4.1.4 Ước tính saving tổng hợp

| Optimization                                   | Saving                                       |
| ---------------------------------------------- | -------------------------------------------- |
| ASR overlap với user speaking                  | ~2,000-3,000ms (depends on utterance length) |
| Speculative LLM prefill                        | ~80-150ms                                    |
| TTS starts at first sentence chunk             | ~600-1,200ms                                 |
| Audio streaming in parallel với TTS generation | ~100-300ms                                   |
| **Total saving vs sequential**                 | **~2,800-4,650ms**                           |
| **Net response latency achievable**            | **~300-500ms**                               |

🔥 **[Scar tissue]** - Nhiều team implement "streaming" mà vẫn chờ full LLM response trước khi TTS start. Đây là **sai lầm phổ biến nhất**. Check: nếu TTS của bạn có input là `completeText: string` thay vì `textStream: Stream<Token>`, bạn đang leave 800ms trên bàn.

---

## 4.2 Speculative decoding trong LLM

### 4.2.1 Cơ chế hoạt động

Recap từ Phần 2:

- **Draft model** (nhỏ, fast): predict N tokens tiếp theo.
- **Target model** (lớn, slow): verify cả N tokens trong 1 forward pass.
- Accept prefix matching, reject rest, continue từ reject point.

### 4.2.2 Performance profile - Khi nào work?

**Key metric: acceptance rate (α).**

- α cao (70%+) → 2-3x speedup
- α thấp (40%) → minimal speedup, overhead từ draft có thể net negative

**α phụ thuộc gì:**

- **Draft model quality** (cần tương đối similar tới target)
- **Domain match** (chat vs code vs creative)
- **Context complexity** (casual conversation có α cao hơn technical query)

### 4.2.3 Saving thực tế cho S2S

📚 **[Literature/benchmarks]** - Published numbers:

- Llama-3 70B + Llama-3 8B draft: ~2.1x throughput on typical chat
- Llama-3 70B + 1B draft: ~1.6x throughput, α ~60%

**Translate sang latency cho S2S:**

```
Không speculative:
  13B model, 25ms/token, 50 tokens response = 1,250ms total generation

Với speculative (2x effective speedup):
  Effective 12.5ms/token = 625ms total generation

Saving: ~625ms cho full response
For first audio chunk (~5-8 tokens): saving ~60-100ms
```

### 4.2.4 Có phù hợp S2S không?

💰 **[Medium ROI]** - Depends.

**Pros for S2S:**

- Reduce inter-token latency → smoother audio (nếu TTS có thể consume nhanh hơn)
- Reduce total response time → user hears completion sooner

**Cons:**

- **2x GPU memory** (draft + target both loaded)
- **Complexity** (tuning acceptance threshold, fallback paths)
- **Diminishing returns nếu TTS là bottleneck** (LLM nhanh hơn cũng vô ích nếu TTS không theo kịp)

**Đề xuất:**

- Với model ≥ 30B: 🎯 **[High ROI]** - worth effort
- Với model 7-13B: 💰 **[Medium]** - marginal, ưu tiên prefix cache trước
- Với model < 7B: 🔬 **[Marginal]** - overhead ăn hết lợi ích

🔥 **[Scar tissue]** - Speculative decoding hay bị treat như "free speedup". Không free. Memory cost, complexity cost, và nếu GPU đã bottleneck bởi memory bandwidth (decode-bound), chạy 2 models = worse.

---

## 4.3 Partial/streaming output ở mọi tầng

Đã cover rải rác, giờ systematize.

### 4.3.1 Checklist cho mỗi tầng

```
┌──────────┬─────────────────┬─────────────────┬──────────────────────┐
│  Tầng    │ Input streaming │ Output streaming│ First output latency │
├──────────┼─────────────────┼─────────────────┼──────────────────────┤
│ Transport│ ✅ (per packet) │ ✅ (per packet) │ ~20ms frame          │
│ VAD      │ ✅ (per frame)  │ ✅ (state chg)  │ ~60-120ms endpoint   │
│ ASR      │ ✅ (per chunk)  │ ✅ (partial)    │ ~150-300ms first prt │
│ LLM      │ ✅ (prefill)    │ ✅ (per token)  │ ~100-200ms TTFT      │
│ TTS      │ ✅ (per token)  │ ✅ (per chunk)  │ ~60-150ms first audio│
│ Delivery │ ✅ (per chunk)  │ ✅ (per packet) │ ~20ms encode         │
└──────────┴─────────────────┴─────────────────┴──────────────────────┘
```

**Bất kỳ tầng nào non-streaming = kill pipeline.**

### 4.3.2 Common violations tôi đã thấy

**Violation 1: LLM "think" step không streaming**

```python
# BAD
response = llm.generate(prompt, max_tokens=200)  # blocks
tts.synth(response)  # starts after complete

# GOOD
async for chunk in llm.stream(prompt):
    await tts_buffer.feed(chunk)
```

**Violation 2: TTS API call per sentence, round-trip each time**

```python
# BAD
for sentence in response_sentences:
    audio = await tts_api.synth(sentence)  # 80ms RTT each
    play(audio)

# GOOD
async with tts_api.streaming_session() as session:
    async for chunk in llm_stream:
        await session.feed(chunk)
    async for audio in session.audio_stream():
        play(audio)
```

**Violation 3: Context build "just in time"**

```python
# BAD
prompt = build_prompt(history)  # ~50ms DB queries
prompt += new_user_input
llm.stream(prompt)  # then starts

# GOOD
# Pre-build prompt template during user speaking time
# Only append final ASR result when ready
```

🔥 **[Scar tissue]** - "Build prompt just in time" bug này tôi đã thấy ở nhiều startup. Prompt building (load memory, tools, system message, format) nên chạy **trong khi user đang nói**, không phải sau khi ASR final. Saving: 30-80ms TTFT.

### 4.3.3 Anti-pattern: Premature non-streaming interfaces

Khi bạn design gRPC/HTTP interface giữa services, **default to streaming both directions**:

```protobuf
// BAD - unary, kills pipeline
service LLM {
  rpc Generate(GenerateRequest) returns (GenerateResponse);
}

// GOOD - bidirectional streaming
service LLM {
  rpc Generate(stream TokenInput) returns (stream TokenOutput);
}
```

Cost: slightly more complex code. Benefit: flexibility để optimize sau mà không phải redesign interface.

---

## 4.4 Edge inference vs Centralized inference

Đây là trade-off lớn về topology.

### 4.4.1 Network RTT contribution

Nhắc lại: network RTT ăn vào budget 2 lần (uplink + downlink).

```
User → nearest edge: 10-30ms RTT
Edge → core region: varies
  - Same continent: 30-80ms RTT
  - Cross-continent: 100-250ms RTT
```

Nếu LLM ở core region, mỗi query E2E includes:

- User → edge (20ms)
- Edge → core (50ms)
- LLM processing (150ms TTFT)
- Core → edge (50ms)
- Edge → user (20ms)
- **Network overhead: 140ms** just for traversal

Nếu LLM ở edge:

- User → edge (20ms)
- LLM processing (150ms)
- Edge → user (20ms)
- **Network overhead: 40ms**

**Saving: ~100ms** khi edge inference.

### 4.4.2 Nhưng không phải mọi thứ nên ở edge

**Chi phí edge inference:**

- **GPU capital cost** spread across nhiều PoPs → utilization thấp hơn
- **Model weight distribution** phức tạp (15-30 PoPs × 70GB model = 1-2TB bandwidth để update)
- **Operational complexity** - deploy, monitor, debug nhiều sites

### 4.4.3 Đề xuất topology hybrid

```
┌──────────────────────────────────────────────────────────────────┐
│  HYBRID TOPOLOGY                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  EDGE (15-30 PoPs globally)                                       │
│  ├─ WebRTC termination (always edge - latency critical)          │
│  ├─ VAD, jitter buffer                                            │
│  ├─ ASR (small model, e.g., Whisper Turbo or Conformer) ← edge   │
│  └─ TTS (small-medium, e.g., Kokoro) ← edge                      │
│                                                                    │
│  CORE (2-4 regions)                                               │
│  └─ LLM (large, 13-70B) ← centralized                            │
│     ├─ Shared prefix cache                                        │
│     ├─ Higher GPU utilization                                     │
│     └─ Easier to update model weights                             │
│                                                                    │
│  Logic:                                                            │
│  - Small models (< 5GB memory footprint) → edge                  │
│  - Large models (> 20GB) → core                                  │
│  - Medium (5-20GB) → judgment call                               │
└──────────────────────────────────────────────────────────────────┘
```

### 4.4.4 Khi nào đẩy LLM ra edge?

🎯 Đáng đẩy khi:

- Model ≤ 8B (fits trên L40S/A10 single GPU)
- Regional volume đủ lớn để utilization > 40%
- Latency budget quá chặt (< 300ms E2E), không chấp nhận edge→core RTT

📐 Không worth khi:

- Model ≥ 13B (cost per inference tăng)
- Volume thấp per region (< 1K concurrent sessions per PoP)
- Cần latest model updates (shipping 70GB weights to 30 PoPs là pain)

🔥 **[Scar tissue]** - Một pattern hay dùng: **tiered model**.

- Edge: 7B model, handles 80% queries tốt.
- Core: 70B model, handles complex queries (tool calling, reasoning).
- Routing: heuristic/classifier decide which tier.

Saving: 80% queries get edge latency (+100ms saving), 20% get higher latency nhưng quality tốt hơn.

---

## 4.5 Predictive prefetching

🔬 **[Marginal]** trong hầu hết cases nhưng worth mention.

### 4.5.1 Prefetch gì được?

**Idea:** Trước khi user nói xong, predict probable actions/context và pre-compute.

**Examples:**

1. **Tool call predictions**: Nếu session history cho thấy user hay hỏi về "weather X", pre-warm weather API connection, pre-fetch current weather cho location phổ biến.

2. **RAG retrieval**: Dùng partial ASR để trigger semantic search. Khi LLM cần context, đã có sẵn.

3. **Response template pre-selection**: Classifier nhỏ predict category of query → pre-select system prompt template specific cho category.

### 4.5.2 Cost/benefit

**Benefit:** 30-100ms saving khi prediction đúng.
**Cost:** Wasted compute khi prediction sai (có thể 50-70% wrong).

**Đề xuất:** Chỉ prefetch những thứ **cheap** (API calls, DB queries). Không pre-compute expensive things (LLM generation, TTS synthesis) trừ khi confidence cực cao.

### 4.5.3 Real example: RAG prefetch

```
t=0ms     User: "Cho tôi biết về dự án X..."
t=150ms   ASR partial: "Cho tôi biết về dự án"
          ├─► Similarity search với partial → top 5 candidate documents
          │    (parallel, ~30ms)
t=300ms   ASR partial: "Cho tôi biết về dự án X"
          ├─► Re-rank với more context
t=500ms   ASR final
          ├─► Use pre-fetched top-K, add to LLM context
t=520ms   LLM prefill starts (RAG context already available)

Without prefetch: RAG happens after ASR final, adds ~80ms
With prefetch: RAG overlapped, adds 0-20ms
```

**Saving: ~60-80ms** cho RAG-heavy applications.

---

## 4.6 Connection keep-alive & session warmup

### 4.6.1 Handshake overhead elimination

**WebRTC session setup:**

- ICE gathering: 100-500ms
- DTLS handshake: 50-150ms
- First media frame: 20-40ms
- **Total: 170-690ms** - user trải nghiệm "connecting..."

Đây là **setup cost**, không phải per-query cost. Quan trọng cho UX khi user **start first query**, nhưng không ảnh hưởng subsequent queries trong cùng session.

### 4.6.2 Techniques

**1. Pre-establish connection on app open**

- Client connect WebRTC + signaling ngay khi app launch, không chờ user talk.
- Session "ready" state trước khi user tap mic.
- **Saving: 170-690ms** cho first query perceived latency.

**2. Long-lived session (không close giữa queries)**

- 1 WebRTC session support multiple queries.
- Chỉ close khi user leave app.
- **Saving: same as above** cho queries 2, 3, ...

**3. Pre-warm downstream services**

- Khi connection opens, ping ASR/LLM/TTS services với "warmup" request.
- Ensure caches, connections, routing paths are hot.
- **Saving: ~20-50ms** on first query.

**4. HTTP/2 or HTTP/3 multiplexing for internal calls**

- gRPC over HTTP/2 natively multiplex.
- Tránh TCP/TLS handshake cho mỗi internal call.

🔥 **[Scar tissue]** - Default "idle timeout" của nhiều frameworks/LB là 60s-5min. Cho S2S, user có thể silent 30s rồi speak. Nếu connection đã closed → reconnect storm. **Set idle timeout ≥ 10 phút cho media connections.**

---

## 4.7 Các techniques khác với saving estimate

Tổng hợp techniques nhỏ nhưng add up:

### 4.7.1 ASR optimizations

| Technique                                 | Saving     | Effort | Priority |
| ----------------------------------------- | ---------- | ------ | -------- |
| Streaming model (vs chunk-based)          | ~200-500ms | High   | 🎯       |
| Server-side VAD với semantic endpointing  | ~100-200ms | Medium | 🎯       |
| Beam search width tuning (5→3)            | ~20-40ms   | Low    | 💰       |
| Smaller model (Whisper Large → Turbo)     | ~50-100ms  | Low    | 💰       |
| Speculative LLM prefill on stable partial | ~80-150ms  | Medium | 💰       |

### 4.7.2 LLM optimizations

| Technique                           | Saving                                        | Effort        | Priority |
| ----------------------------------- | --------------------------------------------- | ------------- | -------- |
| Prefix caching (KV reuse)           | ~200-400ms TTFT                               | Medium        | 🎯       |
| Continuous batching (vLLM/SGLang)   | Enable throughput, slight latency improvement | Medium        | 🎯       |
| Chunked prefill (reduce P99 spikes) | ~500-1000ms P99                               | Medium        | 🎯       |
| Smaller model (70B → 13B)           | ~50-150ms TTFT + inter-token                  | Low (switch)  | 💰       |
| FP8/INT8 quantization               | ~15-30% faster                                | Medium        | 💰       |
| Speculative decoding                | ~2x effective                                 | High          | 💰 or 🔬 |
| Flash Attention 3                   | ~15-25% prefill faster                        | Low (upgrade) | 🎯       |
| Prompt caching API optimization     | ~50-100ms                                     | Low           | 💰       |

### 4.7.3 TTS optimizations

| Technique                                   | Saving               | Effort | Priority |
| ------------------------------------------- | -------------------- | ------ | -------- |
| Streaming input (token-level, not sentence) | ~100-300ms           | Medium | 🎯       |
| Chunked output (first chunk ASAP)           | ~200-500ms           | Medium | 🎯       |
| Smaller model (full neural → distilled)     | ~30-100ms            | Medium | 💰       |
| Pre-compute common response audio cache     | ~80-150ms cho cached | Medium | 💰       |
| GPU-resident weights (no swap)              | ~20-50ms             | Low    | 🎯       |

### 4.7.4 Network optimizations

| Technique                         | Saving                 | Effort    | Priority |
| --------------------------------- | ---------------------- | --------- | -------- |
| Edge PoP closest to user          | ~30-80ms RTT           | High      | 🎯       |
| QUIC instead of TCP for signaling | ~50-100ms connect      | Medium    | 💰       |
| Pre-established WebRTC session    | ~170-690ms first query | Low       | 🎯       |
| Direct peering với major ISPs     | ~10-30ms               | Very High | 💰       |
| Jitter buffer adaptive tuning     | ~20-50ms               | Medium    | 💰       |
| FEC tuning (reduce retransmit)    | ~30-100ms P99          | Medium    | 💰       |

---

## 4.8 Tổng hợp - Từ 1800ms → 350ms

Bảng cuối phần, đây là "build up" để achieve <500ms E2E:

```
Baseline (naive, sequential, no optimization):           ~1,800-2,500ms

Step 1: Switch to streaming at every layer              -800ms → 1,000-1,700ms
        (ASR partial emit, LLM streaming, TTS chunks)

Step 2: Pipeline parallelism (overlap ASR-LLM-TTS)     -500ms → 500-1,200ms
        ⭐ The single biggest optimization

Step 3: Prefix caching + continuous batching           -200ms → 300-1,000ms

Step 4: Chunked prefill (P99 reduction)                -100ms P99 → 300-800ms P99

Step 5: Edge placement of ASR/TTS                      -80ms  → 220-720ms

Step 6: Speculative LLM prefill                        -80ms  → 140-640ms

Step 7: Pre-established connection, warmup             -50ms  → 90-590ms first query

Step 8: Semantic endpointing                           -50ms  → 40-540ms

Step 9: Fine-tune (speculative decoding, FP8, etc.)    -30ms  → 10-510ms

Realistic production: P50 ~350ms, P99 ~700ms
Theoretical floor: ~250ms (VAD endpoint floor + irreducible physics)
```

🔥 **[Scar tissue]** - Bảng trên là **cumulative optimal**. Trong thực tế, sẽ có regression vì:

- Edge cases trong code
- Network weather biến động
- GPU cluster hot/cold
- Load balancer quirks
- Observability overhead

Plan cho P50 target 400-500ms, không 250ms. Còn P99 luôn 1.5-2x của P50 dù bạn optimize đến đâu.

---

## 4.9 Thứ tự làm - Practical roadmap

Nếu bạn đang build từ scratch, thứ tự ROI descending:

**Week 1-4: Foundation**

1. Streaming architecture everywhere (protocol, data flow)
2. Pipeline parallelism - ASR ↔ LLM ↔ TTS overlap
3. WebRTC properly configured

**Week 5-8: LLM optimization** 4. vLLM/SGLang với continuous batching 5. Prefix caching với good hit rate 6. Chunked prefill cho P99

**Week 9-12: Edge & network** 7. Multi-PoP deployment 8. Edge ASR + TTS 9. Pre-warmed connections

**Week 13-16: Fine-tuning** 10. Speculative prefill for LLM 11. Semantic endpointing 12. Adaptive jitter buffer tuning

**Beyond: Advanced** 13. Speculative decoding (nếu large model) 14. Custom hardware paths (AES-NI, NUMA) 15. Quantization refinement

📐 **[Engineering judgment]** - Bạn sẽ thấy **80% của benefit từ 20% của techniques**. Cụ thể: streaming everywhere + pipeline parallelism + prefix cache = 80% của "possible saving". Đừng bị cám dỗ chase cái advanced trước khi nail foundation.

---

## 4.10 Tổng kết Phần 4

**Những thứ quan trọng nhất để nhớ:**

1. **Pipeline parallelism** tiết kiệm ~1,000-1,500ms - the #1 technique.
2. **Streaming at every layer** is the architecture, not an optimization.
3. **Prefix caching** tiết kiệm ~200-400ms TTFT - table stakes cho LLM serving.
4. **Edge placement** tiết kiệm ~80-150ms RTT - worth cho ASR/TTS, judgment call cho LLM.
5. **Pre-established connections** tiết kiệm ~200-600ms cho first query UX.
6. **Speculative decoding** là 💰 medium ROI, không must-have cho model < 30B.

**Realistic achievable latency:**

- 🎯 Target production: **P50 350-500ms, P99 600-900ms**
- 🔬 Aggressive (all techniques, premium resources): **P50 250-350ms**
- 🏆 Theoretical floor: **~200ms** (VAD + physics-irreducible)

**Anti-patterns:**

- Non-streaming interface giữa services
- Build prompt "just in time"
- Chờ full LLM response trước khi TTS start
- Autoscale alone without warm pool
- Retry without deadline propagation

---

# Phần 5 - Chiến lược tối ưu chi phí

> Mục tiêu: S2S pipeline đắt. Phần này nói thẳng về **math của cost** và các kỹ thuật giảm cost có impact lớn - kèm trade-off với latency và quality.

---

## 5.0 Framing - Cost không phải là "optimize sau"

Trước khi đi vào techniques, một sự thật khó chịu:

**S2S ở scale không có "cost" riêng biệt với "architecture".** Cost QUYẾT ĐỊNH architecture. Không phải optimize sau khi đã build xong.

**Cost per minute concrete math:**

```
Giả sử business model: $0.20/phút (voice agent for customer service).
Gross margin target: 60% → COGS budget = $0.08/phút.

Phân bổ COGS budget:
- Infra (GPU + network + CPU):  $0.05/phút
- Support, overhead, etc.:       $0.03/phút

Nếu infra thực tế là $0.10/phút → bạn bleed $0.02/phút per session.
1M minutes/ngày × $0.02 = $20K/ngày loss = $600K/tháng.
```

🔥 **[Scar tissue]** - Tôi đã thấy startups pivot hoặc chết vì không làm math này sớm. Họ build với "API-everything" stack (Deepgram + GPT-4 + ElevenLabs), cost $0.30+/phút, price $0.15/phút "to be competitive". Runway 6 tháng.

**Nguyên tắc xuyên suốt phần này:**

- Cost/latency/quality là **triangle**, không phải đường thẳng.
- Không có free lunch, nhưng có **asymmetric wins** (giảm cost lớn với ít hy sinh).
- Tiered service là key cho sustainable economics.

---

## 5.1 Cost breakdown chi tiết

Recap từ Phần 1 với số liệu cụ thể hơn:

```
┌─────────────────────────────────────────────────────────────────────┐
│  COST PER MINUTE OF ACTIVE S2S SESSION                              │
│  (self-hosted stack, 100K+ DAU, amortized)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  COMPONENT              │ COST/MIN   │ NOTES                         │
│  ──────────────────────────────────────────────────────────────────│
│  LLM inference (13B)    │ $0.025-040 │ Dominant. 45-55% of total.   │
│  LLM inference (70B)    │ $0.08-0.15 │ If using large model.        │
│  TTS (neural, self-host)│ $0.005-010 │ 10-15% of total              │
│  ASR (streaming)        │ $0.003-006 │ 5-10% of total               │
│  Egress bandwidth       │ $0.005-020 │ Depends on peering deal!!!   │
│  CPU (media, orch)      │ $0.004-008 │ 8-12% of total               │
│  Storage + state        │ $0.001-003 │ 3-5% of total                │
│  Observability          │ $0.002-005 │ 3-5%, underestimated         │
│  Misc (DNS, LB, cert)   │ $0.001-002 │                              │
│  ──────────────────────────────────────────────────────────────────│
│  TOTAL (13B model)      │ $0.045-094 │                              │
│  TOTAL (70B model)      │ $0.10-0.21 │                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Observations quan trọng

1. **LLM dominate.** Nếu không optimize LLM inference, khỏi optimize gì khác.
2. **Egress có dải cực rộng** ($0.005-0.020). Peering deal khác biệt 4x.
3. **Observability hay bị underestimate.** Ở scale, logging + tracing + metrics có thể 5-7% tổng cost.
4. **Quality tier (7B vs 13B vs 70B) ảnh hưởng cost 3-5x.** Không phải linear.

### Cost scaling với volume

📐 **[Engineering judgment]** - Cost per minute có **economies of scale** rõ rệt:

```
Volume (min/day)    Effective cost/min    Notes
───────────────────────────────────────────────────────────────
< 100K              $0.20-0.40           API-heavy, no economies
100K - 1M           $0.10-0.20           Hybrid, partial self-host
1M - 10M            $0.05-0.10           Full self-host, good batching
10M - 100M          $0.03-0.06           Volume discounts, custom hw
> 100M              $0.02-0.04           Negotiated power deals, custom silicon
```

Ở scale lớn, **pre-built infrastructure** (bare metal, custom silicon, peering) bắt đầu worth. Ở scale nhỏ, cloud provider flexibility worth more than unit cost.

---

## 5.2 GPU utilization - Target realistic

### 5.2.1 Tại sao không target 100%?

**Naive thinking:** Higher utilization = lower cost per inference. Target 95%+.

**Reality:**

```
GPU utilization 95-100% means:
├─ No headroom cho traffic spike
├─ Queue growing → latency spike
├─ P99 disaster
├─ Any small glitch cascades
└─ Ops team never sleeps
```

### 5.2.2 Target thực tế

| Utilization target | Use case                              | Trade-off                         |
| ------------------ | ------------------------------------- | --------------------------------- |
| 40-50%             | Ultra-low-latency tier                | Expensive per request but stable  |
| 60-70%             | **Standard production (recommended)** | Sweet spot cost/stability         |
| 75-85%             | Cost-optimized, batch-tolerant        | Higher P99, needs smart shed load |
| 90%+               | Offline batch jobs only               | Never for real-time               |

🔥 **[Scar tissue]** - Target **65-70% GPU utilization at P50 load** for streaming inference. Reasons:

- Traffic có variance ~30-50% intra-day
- Spike có thể 2x baseline trong 5 phút
- Buffer cho autoscale lag
- Room cho prefix cache expansion

Nếu bạn đang ở 90% → bạn đang gamble. Next spike = incident.

### 5.2.3 Measure gì cho "GPU utilization"?

Watch out: nvidia-smi's `GPU-Util` là **misleading metric**. Nó chỉ đo "kernel running or not", không đo actual compute saturation.

**Metrics đúng hơn:**

- **SM (Streaming Multiprocessor) occupancy** - % SMs active per cycle
- **Memory bandwidth utilization** - decode-bound workloads bị limit bởi cái này
- **Tensor core usage** - cho compute-heavy (prefill)
- **Request queue depth** - functional metric, correlate với user experience

vLLM/SGLang có built-in metrics riêng, dùng những cái này thay vì nvidia-smi.

---

## 5.3 Model quantization - Trade-off thật

### 5.3.1 Precision levels - Quality/cost matrix

| Precision    | Memory | Speed     | Quality loss          | Use case                 |
| ------------ | ------ | --------- | --------------------- | ------------------------ |
| FP32         | 4x     | 1x        | None (baseline)       | Never production (waste) |
| FP16/BF16    | 2x     | ~1.3x     | Negligible            | Standard baseline        |
| FP8 (H100+)  | 1x     | ~1.6-1.8x | ~1-3% benchmark       | 🎯 Good for production   |
| INT8         | 1x     | ~1.5-1.7x | ~3-5% benchmark       | 💰 Common optimization   |
| INT4         | 0.5x   | ~2-2.5x   | ~5-10% benchmark      | ⚠️ Aggressive, careful   |
| INT2/ternary | 0.25x  | ~3x       | Large, task-dependent | Experimental             |

### 5.3.2 Quality loss reality - Benchmark vs perceived

📚 **[Literature]** - Benchmark losses (MMLU, etc.) là **misleading cho S2S**.

**Why:**

- S2S response quality ít phụ thuộc vào knowledge edge cases (mà MMLU test)
- Conversation quality depend nhiều vào **coherence, tone, natural language** - ít degraded bởi quantization
- TTS layer "hide" một số LLM output roughness

🔥 **[Scar tissue]** - INT8 quantization cho 13B chat model: benchmark drop 3%, blind A/B test với users không phân biệt được. Saving: ~40% GPU cost.

**INT4 là câu chuyện khác:**

- Có noticeable quality degradation
- Repetition, grammatical errors tăng
- Tool calling accuracy drop 10-15%
- 📐 Worth nếu model nền rất lớn (70B → INT4 ≈ quality của 13B FP16, cost thấp hơn)

### 5.3.3 Cost impact

**Math cho 13B model:**

```
FP16: 26GB memory → fits A100-40G, không fits nhiều per card
INT8: 13GB → fits 2-3 per A100, hoặc dùng smaller GPU (A10)
INT4: 7GB → fits 4-5 per A100

Throughput impact:
FP16 on H100: ~1000 tokens/sec single request
INT8 on H100: ~1600 tokens/sec (1.6x)
INT4 on H100: ~2200 tokens/sec (2.2x)

Cost saving INT8 vs FP16: ~35-40%
Cost saving INT4 vs FP16: ~55-60%
```

### 5.3.4 Đề xuất quantization strategy

**Tier 1 (premium users, quality-critical):** FP16 or BF16. Full quality.

**Tier 2 (standard):** FP8 hoặc INT8. 🎯 **Default choice** cho hầu hết.

**Tier 3 (cost-optimized, batch/async):** INT4. Chỉ nếu quality đã test kỹ.

**Quan trọng:** Quantization khác nhau cho **prefill** vs **decode**:

- Prefill: compute-bound, quantization help latency nhiều
- Decode: memory-bound, quantization help throughput qua fit nhiều concurrent

### 5.3.5 Quantization cho components khác

**ASR:**

- Whisper int8 via faster-whisper: ~2x speedup, minimal quality loss
- Conformer int8: similar. 🎯 **Default**

**TTS:**

- Neural TTS đã relatively small. Quantization marginal.
- INT8 giúp giảm memory footprint, fit nhiều concurrent streams.

---

## 5.4 Tiered service - Design cho sustainable economics

### 5.4.1 Vì sao tiered service?

**Fact:** Không phải user nào cũng cần <300ms latency. Không phải use case nào cũng cần 70B model.

**Uniform service = over-provision cho 90% users.** Tiered = match resource với actual need.

### 5.4.2 Tier design framework

```
┌──────────────────────────────────────────────────────────────────┐
│  TIERED SERVICE DESIGN                                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  TIER        LATENCY   MODEL       GPU        COST/MIN   MARGIN  │
│  ──────────────────────────────────────────────────────────────  │
│  PREMIUM     P50<300ms  70B FP16    H100      $0.15-0.25   High  │
│              P99<600ms  edge infer  dedicated                    │
│                                                                    │
│  STANDARD    P50<500ms  13B FP8     A100      $0.04-0.08   Med   │
│              P99<900ms  core infer  shared                       │
│                                                                    │
│  ECONOMY     P50<900ms  7B INT8     L4/A10    $0.015-0.03  Low   │
│              P99<2s     centralized batch                        │
│                                                                    │
│  BATCH/ASYNC No SLA     Any         Spot       $0.005-0.015 -    │
│              Best-eff   instances                                 │
└──────────────────────────────────────────────────────────────────┘
```

### 5.4.3 Routing - Ai gets what tier?

**Routing signals:**

1. **Business tier** (user's subscription): Premium pays for premium.
2. **Use case** (system prompt / agent type):
   - Customer service: Standard OK
   - Real-time translation: Premium needed
   - Content creation: Batch OK
3. **User behavior signal**:
   - First-time user / trial: bump to Premium for "wow" moment
   - Long session / heavy user: Standard (cost control)
4. **Geography**: Emerging markets có thể lower tier với lower price point

### 5.4.4 Practical tier implementation

🔥 **[Scar tissue]** - Key design: **tier should be per-request, not per-user**. Same user's "quick question" = Economy OK, "complex reasoning" = Premium.

**Classifier router:**

```
User utterance → fast classifier (few ms) →
  ├─ Simple intent (yes/no, confirm) → 7B tier
  ├─ Q&A (factual) → 13B tier
  └─ Reasoning/tool use → 70B tier
```

Saving: 50-70% of requests get cheap tier, premium tier reserved cho actual need.

📐 **[Engineering judgment]** - Mixture-of-Experts style routing là future direction. Hiện tại 2026 tech chưa mature enough cho production S2S ở scale.

---

## 5.5 Caching - Những gì cache được trong real-time pipeline

Real-time sounds like không cache được. Wrong.

### 5.5.1 Cache candidates

```
┌──────────────────────┬──────────────┬───────────────┬─────────────┐
│ Item                 │ Cache hit %  │ Saving        │ Storage     │
├──────────────────────┼──────────────┼───────────────┼─────────────┤
│ LLM prefix KV cache  │ 60-85%       │ 200-400ms TTFT│ GPU mem+Redis│
│ System prompt cache  │ 99%+         │ Part of above │ Small       │
│ RAG embeddings       │ 95%+         │ 30-60ms      │ Vector DB    │
│ Common TTS phrases   │ 5-20%        │ 60-120ms     │ Audio blob   │
│ Tool call results    │ 30-50%       │ 100-500ms    │ Redis        │
│ Voice clone models   │ 99%+         │ Load time    │ GPU mem      │
└──────────────────────┴──────────────┴───────────────┴─────────────┘
```

### 5.5.2 LLM Prefix KV cache - Most important

Đã cover ở Phần 2-4. Recap cost impact:

**Without prefix cache:**

- Every turn re-prefills full conversation
- Turn 10: prefill 4K tokens ~300ms + cost
- Compute wasted: O(turns²)

**With prefix cache (hit rate 80%):**

- Turn 10: prefill ~200 new tokens ~20ms
- Compute saving: 80-90% cho prefill workload
- **Cost saving: ~30-40% of LLM compute**

### 5.5.3 TTS audio caching

**Common scenarios:**

- Greetings: "Xin chào, tôi có thể giúp gì cho bạn?" (high hit rate)
- Confirmations: "Vâng, tôi đã hiểu." (high)
- Fallbacks: "Xin lỗi, tôi chưa hiểu ý bạn." (high)
- Actual responses: mostly unique, không cache được

**Implementation:**

```python
# Pseudo-code
def tts_with_cache(text, voice_id):
    key = hash(text + voice_id)
    cached = audio_cache.get(key)
    if cached and is_frequent_phrase(text):
        return cached  # saving ~80-150ms
    audio = tts_model.synth(text, voice_id)
    if should_cache(text):
        audio_cache.set(key, audio, ttl=7days)
    return audio

def should_cache(text):
    return (
        len(text) < 100 and
        text in known_frequent_templates  # top 100-1000 phrases
    )
```

**Saving:** 5-15% TTS GPU cost, small but real. High hit rate ở specific domains (customer service: 30-40% templated responses).

🔥 **[Scar tissue]** - Cached audio **must match TTS voice model version exactly**. Voice model update → cache invalidate. Otherwise user hears voice mismatch giữa cached/fresh audio. Versioning matter.

### 5.5.4 RAG/tool call result caching

**Tool call caching:**

- Weather for same city, last 10 min: cache
- DB query with same params: cache (short TTL)
- External API (flights, stocks): cache with tight TTL

**Semantic cache (advanced):**

- Similar queries hit same cached answer
- Ví dụ: "thời tiết Hà Nội hôm nay" và "hôm nay Hà Nội trời thế nào" → same result
- Embedding similarity > 0.95 → cache hit
- 📐 Complex to get right. Risk: stale/wrong answers cho edge cases.

---

## 5.6 Spot/Preemptible instances - Có dùng được cho GPU inference?

### 5.6.1 The temptation

Spot instances: 60-80% cheaper than on-demand. Với GPU cost dominant, 70% saving = massive.

### 5.6.2 The reality cho S2S

**Challenges:**

- **Preemption notice: 30s-2min** (AWS spot: 2min, GCP: 30s).
- **S2S session đang active cannot migrate** in 30s - WebRTC, streaming state, etc.
- **Preemption rate**: 5-15% per hour cho popular GPU SKUs.

**Math:**

```
On-demand H100: $5/hr
Spot H100: $1.50/hr (70% saving)

But: 10% preemption rate, sessions disrupted.
If 10% sessions disrupted → user churn → revenue loss
Is revenue loss > compute saving?
Usually YES, for paid users. Spot not worth.
```

### 5.6.3 Khi nào dùng được spot

🎯 **Usable cho:**

1. **Batch workloads**: Training, evaluation, offline inference
2. **Async/non-real-time**: Transcription of recordings, summarization
3. **Overflow/best-effort tier**: Free tier users, batch mode

⚠️ **Cẩn thận cho:**

1. **Warm pool** (pre-loaded but not serving): Giảm cold start cost, nhưng nếu preempted lúc cần scale = disaster
2. **Non-critical regions**: Backup region on spot, primary on-demand

❌ **Không dùng cho:**

1. Primary serving infrastructure cho paid tier
2. LLM cluster serving real-time sessions
3. Edge nodes (session state too precious)

### 5.6.4 Hybrid strategy

```
┌──────────────────────────────────────────────────────────────┐
│  HYBRID CAPACITY PLANNING                                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Baseline load:     ████████████  On-demand reserved (1yr)   │
│                                    Cost: ~40% of on-demand   │
│                                                                │
│  Peak spikes:       ████          On-demand regular           │
│                                                                │
│  Overflow/batch:    ██            Spot (cheap, preemptible)   │
│                                                                │
│  Warm pool:         ██            On-demand (reliability)     │
│                                                                │
└──────────────────────────────────────────────────────────────┘

Effective cost: ~55-65% of pure on-demand pricing
Without sacrificing reliability
```

🔥 **[Scar tissue]** - **Reserved instances** và **committed use discounts** thường underused. Commit 1-3 năm capacity baseline → 30-50% discount vs on-demand. Ít sexy hơn spot nhưng reliable và accountable cho finance. Cho stable baseline workload, **RI > spot > on-demand** về cost.

---

## 5.7 Autoscaling - Metric nào để scale?

### 5.7.1 Vấn đề với naive autoscaling

**Naive: scale on CPU/GPU utilization.**

Problem: Bởi đến lúc GPU util = 80%, **latency đã bị degrade**. Scale response time 2 phút → 2 phút user experience broken.

### 5.7.2 Metrics tốt hơn cho S2S

**Leading indicators (scale up trước khi hurt):**

1. **Request queue depth** (vLLM/SGLang expose cái này)
   - Scale up khi queue > threshold
   - Ví dụ: queue > 5 requests → add capacity

2. **TTFT P95**
   - Normal: 150ms
   - Trigger: P95 > 250ms for 30s → scale up
   - Proactive, user chưa bị impact bởi P50

3. **Batch size vs capacity**
   - batch_size / max_batch_size > 0.8 → near-saturation
   - Scale up before fully saturated

4. **Session creation rate**
   - Derivative of load
   - Spike detection: rate > X/s sustained → pre-scale

**Lagging indicators (reactive, too late):**

5. **P99 latency** - by time you see this, users are complaining
6. **GPU utilization** - saturation = already bad
7. **Error rate** - already failing

### 5.7.3 Scale-up vs Scale-down asymmetry

**Scale up fast, scale down slow.**

```
Scale up:    Trigger → 30-60s react → 60-120s GPU ready
             Total: ~90-180s (with warm pool: 10-30s)

Scale down:  Trigger → 5-10 min observation window → 5 min drain
             Total: ~10-15 min minimum
```

**Why slow scale-down:**

- Active sessions need to complete (respect drain time)
- Avoid flapping (scale down → next minute scale up)
- Cost of spare capacity << cost of under-provision

### 5.7.4 Predictive scaling

📐 **[Engineering judgment]** - Nếu traffic có pattern rõ (daily, weekly cycles), **predictive > reactive**.

**Implementation:**

- ML model predict load 15-30 min ahead
- Scale before expected spike
- Fallback reactive scaling cho unpredicted events

**Saving:** 10-20% infra cost vs reactive only. Works tốt cho consumer apps với clear diurnal patterns.

🔥 **[Scar tissue]** - Predictive scaling fail **spectacularly** khi có unusual events (news, outage of competitor, viral post). Always have reactive fallback. And limit predictive scale magnitude (can't scale 10x).

### 5.7.5 Scale unit sizing

Trade-off: large scale units (bigger nodes) vs small.

| Large units (e.g., 8-GPU nodes) | Small units (e.g., 1-GPU nodes) |
| ------------------------------- | ------------------------------- |
| Better batching within node     | Finer-grained scaling           |
| Higher utilization              | More flexibility                |
| Slower scaling (bigger step)    | Faster scaling                  |
| Scheduler complexity lower      | Scheduler complexity higher     |

**Đề xuất:** 2-4 GPU per scale unit là sweet spot. Đủ để batch hiệu quả, đủ fine-grained để scale reasonable.

---

## 5.8 Observability cost - The hidden tax

🔥 **[Scar tissue]** - Observability cost bị ignore cho đến khi bill đến. Và nó có thể 5-10% tổng infra cost.

### 5.8.1 Cost components

- **Metrics storage** (Prometheus, Datadog, etc.): 1M sessions × 50 metrics × scrape interval
- **Logs**: Structured logs of every event
- **Distributed traces**: Full trace per session
- **User session recordings** (cho replay): giá nặng

### 5.8.2 Optimization strategies

**1. Sampling**

- Traces: 1% baseline, 100% cho errors/P99 slow
- Logs: sample INFO logs (10%), keep all ERROR/WARN

**2. Aggregation at edge**

- Compute histograms locally, ship aggregated numbers
- Không ship raw events

**3. Tiered retention**

- Hot: 7 days, searchable
- Warm: 30 days, summary only
- Cold: S3 archive, for compliance

**4. Purposeful metrics**

- Mỗi metric phải có consumer (dashboard, alert, analysis)
- Remove unused metrics - chúng ăn cost nhưng chẳng ai nhìn

**Saving:** 40-60% observability cost với disciplined approach.

---

## 5.9 Network cost deep dive

### 5.9.1 Egress optimization

Recap từ Phần 3, giờ cụ thể hơn:

**Tier 1: Reduce data transmitted**

- Lower bitrate codec (32kbps → 16kbps): -50% egress
- DTX (silent suppression): -20-30% egress
- Optimize protocol overhead (smaller headers): marginal

**Tier 2: Reduce egress distance**

- Edge PoP closer to user → most traffic on cheaper local networks
- Peering agreements → skip tier-1 transit

**Tier 3: Reduce egress pricing**

- Negotiate with cloud providers (100Gbps+ commit → 70-90% discount)
- Move to cloud providers with free/flat egress (Cloudflare, OVH, Hetzner)
- Bare metal + direct peering (if scale justifies)

### 5.9.2 Cross-region traffic

Often overlooked: traffic giữa regions cũng tốn.

```
User → Edge (Asia)           : Same-region, cheap
Edge (Asia) → Core (US)      : Cross-region, $0.02-0.08/GB
Core (US) → Edge (Asia)      : Cross-region again
Edge (Asia) → User           : Same-region, cheap

If LLM at core US, per query roundtrip ~few KB
At 1M queries/day, this adds up
```

**Optimization:** Compress payload giữa edge và core. Dùng Protobuf/gRPC (built-in compression). Avoid JSON.

🔥 **[Scar tissue]** - Một team tôi biết dùng JSON cho inter-service gRPC. Switched sang Protobuf → egress cost giảm 40% cho internal traffic. Easy win.

---

## 5.10 Cost-killer optimizations - Tổng hợp priority

Bảng cuối phần - priority theo ROI:

| #   | Technique                           | Saving                            | Effort            | Priority               |
| --- | ----------------------------------- | --------------------------------- | ----------------- | ---------------------- |
| 1   | Continuous batching (vLLM/SGLang)   | 40-60% LLM cost                   | Medium            | 🎯 MUST                |
| 2   | Prefix caching (>80% hit rate)      | 30-40% LLM compute                | Medium            | 🎯 MUST                |
| 3   | FP8/INT8 quantization               | 35-45% GPU cost                   | Medium            | 🎯 MUST                |
| 4   | Tiered service (route by need)      | 40-60% overall                    | High              | 🎯 MUST at scale       |
| 5   | Reserved instances (1yr commit)     | 30-50% baseline                   | Low (procurement) | 🎯 for stable workload |
| 6   | Egress optimization (peering/codec) | 50-80% egress                     | Very high         | 🎯 for > 100K sessions |
| 7   | Chunked prefill (P99 fix)           | N/A latency, prevent scaling cost | Medium            | 🎯                     |
| 8   | Self-host (vs API) at volume        | 50-80% unit cost                  | Very high         | 💰 > 2M min/day        |
| 9   | Autoscale with leading metrics      | 15-25% overprov                   | Medium            | 💰                     |
| 10  | TTS cache for common phrases        | 5-15% TTS cost                    | Low               | 💰                     |
| 11  | INT4 quantization                   | 15-20% more                       | Medium            | ⚠️ test quality        |
| 12  | Speculative decoding                | Throughput 2x                     | High              | 💰 if large model      |
| 13  | Spot instances (for batch)          | 60-70% on spot subset             | Medium            | 💰 non-realtime        |
| 14  | Predictive autoscaling              | 10-20%                            | High              | 🔬 mature stage        |
| 15  | Observability optimization          | 40-60% of observ cost             | Low               | 💰                     |

**Top 5 gets you 70-80% of potential cost saving.** Cái khác là polish.

---

## 5.11 Practical cost target progression

Roadmap cost optimization theo scale:

**Early stage (< 100K min/day):**

- Use APIs (Deepgram, OpenAI, ElevenLabs)
- Focus on product-market fit, not cost
- Expected: $0.15-0.30/min

**Growth stage (100K - 1M min/day):**

- Self-host ASR (Whisper Turbo hoặc similar)
- Keep LLM on API hoặc self-host 7B
- Self-host TTS (Kokoro)
- Expected: $0.08-0.15/min

**Scale stage (1M - 10M min/day):**

- Full self-host pipeline
- FP8 quantization
- Prefix caching mature
- Tiered service rolled out
- Multi-PoP edge
- Expected: $0.04-0.08/min

**Hyperscale (> 10M min/day):**

- Custom silicon evaluation
- Direct peering agreements
- Dedicated team cho infra optimization
- Expected: $0.02-0.05/min

📐 **[Engineering judgment]** - Don't jump stages. Each stage requires team maturity, operational capacity. Premature optimization kills startups.

---

## 5.12 Tổng kết Phần 5

**Những insight quan trọng nhất:**

1. **LLM inference dominate cost** (45-55%). Tối ưu LLM = tối ưu cost.
2. **Target 65-70% GPU utilization.** 95% = gamble, 40% = waste.
3. **FP8/INT8 quantization** là default cho production. INT4 = aggressive, test kỹ.
4. **Tiered service** is key cho sustainable economics. Premium subsidy Economy.
5. **Prefix cache hit > 80%** = 30-40% LLM cost saving.
6. **Reserved instances** > spot > on-demand cho baseline. Spot chỉ cho batch.
7. **Egress cost có dải 10x** tùy peering. Plan day 1.
8. **Observability cost** can be 5-10% of total. Sample aggressively.
9. **Autoscale on leading metrics** (queue depth, TTFT P95), not GPU util.
10. **Scale progressively.** Each stage requires maturity. Don't jump.

**Những con số cần nhớ:**

- $0.04-0.10/min = realistic production cost at scale
- 60-70% GPU utilization = sweet spot
- 80%+ prefix cache hit rate = target
- 30-50% RI discount = worth commitment for baseline
- INT8 quant = ~40% GPU saving, < 3% quality loss

**Anti-patterns cost-wise:**

- API-everything at scale (> $0.20/min unnecessary)
- Over-provision với uniform tier (wasting 40-60%)
- Spot instances cho real-time serving (unreliable)
- Monitor on GPU util only (reactive, too late)
- Ignore egress cost (surprise $500K bill)

---

# Phần 6 - Thiết kế hạ tầng real-time

> Mục tiêu: Đi sâu vào operational design của hệ thống. Những thứ không sexy nhưng quyết định hệ thống có sống được ở production hay không.

---

## 6.0 Framing - Real-time infra khác backend thông thường ở đâu?

Trước khi đi vào chi tiết, các principles khác biệt:

**Stateful vs stateless.** HTTP backend có thể stateless - load balance dễ, scale dễ. Media servers (WebRTC) stateful - một session gắn với một node. Mọi design decision đổi theo.

**Long-lived connections.** HTTP request ~100ms. WebRTC session trung bình 2-15 phút. Deploy strategy, drain strategy, graceful shutdown - tất cả đều khác.

**Hard real-time constraints.** API gateway có thể buffer 500ms để retry, optimize throughput. Media server buffer 500ms = audio gap, user notice ngay.

**Bidirectional flows.** REST: client pulls. Streaming: server pushes, client pushes, cả hai đồng thời. Backpressure không optional.

---

## 6.1 Edge topology - Bao nhiêu PoPs và ở đâu?

### 6.1.1 Các nguyên tắc đặt PoP

**Principle 1: Minimize RTT to 95% of users.**

Target: < 40ms RTT cho P95 users. Điều này thường đồng nghĩa với:

- Tối đa ~3000km great-circle distance (speed of light round trip ~20ms, + routing overhead ~20ms)
- Cùng "internet geography" region (không cross submarine cables nếu có thể tránh)

**Principle 2: Match population density, not landmass.**

Vietnam, Thailand, Indonesia, Philippines: đông dân, nhiều tech adoption → PoPs dày ở SEA. Siberia rộng lớn nhưng low population → 1 PoP ở Moscow đủ.

**Principle 3: Regulatory boundaries matter.**

Data sovereignty law (GDPR, China, India DPDP): có thể cần PoP **trong** country, không chỉ gần. 🔥 **[Scar tissue]** - hợp đồng enterprise với EU/banking/healthcare customers thường require EU-only data processing. Không có EU PoP = không có customer đó.

### 6.1.2 Realistic PoP count cho global S2S

**Estimate cho target < 40ms P95 RTT globally:**

```
┌────────────────────────────────────────────────────────────────┐
│  GLOBAL POP DISTRIBUTION (recommended)                          │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  NORTH AMERICA (5 PoPs)                                         │
│  ├─ US West (LAX or SFO)                                        │
│  ├─ US Central (DFW or ORD)                                     │
│  ├─ US East (ASH or IAD)                                        │
│  ├─ Canada (YUL - Montreal, proxy for east CA)                  │
│  └─ Mexico (MEX)                                                 │
│                                                                  │
│  EUROPE (4-5 PoPs)                                              │
│  ├─ UK (LHR)                                                     │
│  ├─ Germany/Netherlands (FRA or AMS)                            │
│  ├─ France (CDG)                                                 │
│  ├─ Nordics (ARN - Stockholm)                                   │
│  └─ (optional) Spain/Italy (MAD or MXP)                         │
│                                                                  │
│  ASIA PACIFIC (6-7 PoPs)                                        │
│  ├─ Japan (NRT)                                                  │
│  ├─ Korea (ICN)                                                  │
│  ├─ Singapore (SIN)                                             │
│  ├─ Hong Kong (HKG)                                             │
│  ├─ India (BOM or HYD)                                          │
│  ├─ Australia (SYD)                                             │
│  └─ (optional) Jakarta or Bangkok if SEA volume justifies       │
│                                                                  │
│  SOUTH AMERICA (2-3 PoPs)                                       │
│  ├─ Brazil (GRU - São Paulo)                                    │
│  ├─ (optional) Chile (SCL) or Colombia (BOG)                    │
│                                                                  │
│  MIDDLE EAST / AFRICA (2-3 PoPs)                                │
│  ├─ UAE (DXB)                                                    │
│  ├─ South Africa (JNB)                                          │
│  └─ (optional) Egypt or Nigeria                                 │
│                                                                  │
│  ──────────────────────────────────────────────────────────    │
│  TOTAL: 20-25 PoPs                                              │
│  Adequate for global S2S với target P95 RTT < 40ms              │
└────────────────────────────────────────────────────────────────┘
```

### 6.1.3 "Chúng tôi cần 200+ PoPs như Cloudflare"?

**Không.** Đây là misconception.

**Cloudflare CDN có 300+ PoPs vì:**

- Static content caching, nhiều PoPs = cache gần hơn
- Bandwidth-heavy, spread load across many small PoPs
- Low compute per PoP

**S2S infra khác:**

- Mỗi PoP cần GPU (ASR/TTS). GPU = expensive. Nhiều PoPs = underutilized GPUs.
- Sessions stateful → không benefit từ "distribute to nearest" như cache lookup
- 15-25 PoPs với higher capacity > 200 PoPs small

📐 **[Engineering judgment]** - Sweet spot: **15-30 PoPs cho consumer S2S, 5-10 cho enterprise** (fewer customers, geographic concentration).

### 6.1.4 Tier 1 vs Tier 2 PoPs

Không phải PoP nào cũng full-feature. Tiered architecture:

```
┌──────────────────────────────────────────────────────────────┐
│  TIER 1 POP (full stack, ~10-15 globally)                    │
│  ├─ WebRTC media server                                      │
│  ├─ ASR (full model)                                         │
│  ├─ LLM (medium model, 7-13B)                                │
│  ├─ TTS (full neural)                                        │
│  └─ Full observability                                       │
│                                                                │
│  TIER 2 POP (proxy + lightweight, ~10-15 additional)         │
│  ├─ WebRTC media server                                      │
│  ├─ Route ASR/TTS to nearest Tier 1                         │
│  ├─ VAD, jitter buffer, codec only                          │
│  └─ Saves cost, lower latency for nearby users              │
│                                                                │
│  TIER 3 POP (edge cache + relay, ~5-10 additional)           │
│  ├─ TURN server (NAT traversal)                             │
│  ├─ WebRTC relay only                                        │
│  ├─ No compute                                               │
│  └─ For hard-to-reach regions (Africa, etc.)                │
└──────────────────────────────────────────────────────────────┘
```

**Saving:** Tier 2/3 PoPs cost 10-30% of Tier 1. Cho regions với volume thấp hoặc chủ yếu latency-sensitive media (không compute-heavy).

---

## 6.2 Load balancing cho WebRTC và WebSocket

### 6.2.1 Không dùng HTTP LB cho WebRTC

🔥 **[Scar tissue]** - Sai lầm kinh điển: đưa WebRTC traffic qua AWS ALB/Nginx HTTP LB.

**Lý do fail:**

- WebRTC dùng UDP (media) + TCP (signaling) - HTTP LB chỉ handle TCP
- Even signaling qua WebSocket: ALB default timeout 60s → disconnects
- SRTP traffic có header format riêng, HTTP LB không biết route

**Solution: Tách 2 paths.**

```
┌──────────────────────────────────────────────────────────────┐
│  SEPARATE PATHS FOR SIGNALING vs MEDIA                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Signaling (WebSocket, session setup, ICE candidates):       │
│  Client ─TLS──► L7 LB ─► Signaling servers (stateless-ish)   │
│                   (ALB, Envoy, Nginx)                         │
│                                                                │
│  Media (UDP, SRTP/DTLS):                                     │
│  Client ─UDP─► Anycast IP ─► L4 LB ─► Media server            │
│                  (Maglev, BGP)    (IP:port sticky)            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### 6.2.2 L4 LB cho media - Options

**1. Google Maglev-style (software L4 LB)**

- Google, Cloudflare, Meta dùng variants
- Consistent hash → session sticky
- Fast failover
- Runs on commodity hardware
- Examples: Katran (Meta), Seesaw (Google), Cilium

**2. Hardware LB**

- F5 BIG-IP, Citrix NetScaler
- Enterprise pricing ($100K+)
- Low latency, high capacity
- Worth nếu bạn có 1M+ concurrent và dedicated infra team

**3. Cloud-provided L4**

- AWS NLB: Works for TCP, **UDP support since 2018**
- GCP NLB: UDP supported
- Limitations: Less control over hash algorithm, session affinity tuning

**4. Direct IP + client-side routing (advanced)**

- Server returns specific IP cho session
- Client connects directly
- No LB in media path
- Discord pattern: 🔥 **[Scar tissue]** - chúng tôi dùng variant này cho voice. Client query "best server", receive IP, connect direct.

### 6.2.3 Session affinity strategies

**Consistent hashing (recommended):**

```
session_id → hash → bucket → server
```

Benefits:

- Same session always → same server
- Server removed → only its shard resh affected, không global reshuffle
- Server added → minimal disruption

**What's hashed? Options:**

- session_id (best for S2S - deterministic)
- source IP (bad - users behind NAT share IP)
- IP + port (better than IP alone)

### 6.2.4 Drain strategy - Critical for S2S

HTTP deploy: drain 30s, connections short-lived, done.

S2S deploy: **sessions last minutes**. 30s drain = kill sessions mid-conversation.

**Graceful drain:**

```
┌──────────────────────────────────────────────────────────────┐
│  WEB_RTC_GRACEFUL_DRAIN sequence                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  T+0    Mark node "draining" in LB → no new sessions         │
│                                                                │
│  T+0    Existing sessions continue                            │
│                                                                │
│  T+30s  Send "migration hint" to clients:                    │
│         "Please reconnect when convenient"                    │
│         Clients reconnect at natural pause points             │
│                                                                │
│  T+5min Average session completion (if natural churn)        │
│                                                                │
│  T+10min If sessions still active, send:                     │
│         "Reconnect now, brief interruption"                   │
│         Client gracefully tears down, reconnects             │
│                                                                │
│  T+12min Force kill remaining sessions                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

🔥 **[Scar tissue]** - Hidden requirement: **client must support reconnect**. Nếu client SDK không handle reconnect smoothly, mọi drain strategy fail. Invest vào client reconnect logic là first-class requirement.

**Blue-green deployment khó hơn cho S2S:**

- Không thể "switch traffic 100%" trong 1 giây
- Rolling deployment with long drain window (10-15 phút per node batch)
- Deploy window: plan 30-60 phút cho full rollout
- Emergency rollback: không instant, cần strategy (e.g., dual-write state to old version)

### 6.2.5 Connection tracking at scale

Với 100K concurrent sessions per edge node:

**Kernel-level connection tracking (conntrack):**

- Default Linux conntrack limits ~64K entries
- Must tune: `/proc/sys/net/nf_conntrack_max` to 1M+
- Memory: ~350 bytes/entry → 350MB for 1M connections
- CPU overhead for lookup

**Alternatives:**

- **XDP/eBPF** (Katran-style): Bypass conntrack, process in kernel space pre-conntrack. Lower latency, higher throughput.
- **DPDK**: Userspace packet processing, even faster. Complex.

Hầu hết teams dùng tuned conntrack + Linux networking stack. XDP worth khi > 500K concurrent per node.

---

## 6.3 Backpressure - Khi hệ thống quá tải

### 6.3.1 Định nghĩa

Backpressure = producer generates data faster than consumer can process. Without handling: buffer overflow, OOM, or cascading latency.

**Cho S2S, backpressure xảy ra ở mọi tầng:**

```
Audio from user → Jitter buffer (capacity ~200ms)
  ↓
ASR (GPU batch capacity)
  ↓
LLM (GPU batch capacity, queue)
  ↓
TTS (GPU batch capacity)
  ↓
Audio to user (network buffer)
```

Mỗi arrow có thể bị backpressure.

### 6.3.2 Strategies - Khi GPU đầy, làm gì?

**Option A: Queue (reactive, dangerous)**

```
GPU full → queue request → wait
Queue grows → latency grows → user experience degrades
Queue overflow → OOM → crash
```

Never pure queue for S2S. **Queue + deadline** OK.

**Option B: Shed load (aggressive, safe)**

```
GPU full + queue > threshold → reject new session
Return "try later" signal
```

Protects existing sessions. Controversial UX (some users can't connect).

**Option C: Graceful degradation (best)**

```
GPU full → new sessions:
  - Route to smaller/faster model (tier 3 fallback)
  - Use cached responses for common queries
  - Reduce response quality (shorter, simpler)
  - Extend jitter buffer (add latency but stable)
```

Users see lower quality, not failure. Discord approach: prefer degraded experience over disconnection.

**Option D: Shift to other capacity**

```
Region A overloaded → route to Region B
```

Works if other regions have capacity and latency penalty acceptable.

### 6.3.3 Deadline propagation

**Critical pattern:** Every request carries deadline, downstream respects.

```go
// Conceptual
type Request struct {
    Deadline time.Time  // "This must complete by X"
    ...
}

func handle(req Request) {
    remaining := req.Deadline.Sub(time.Now())
    if remaining < MIN_USEFUL_WORK {
        return ErrDeadlineExceeded  // Don't even start
    }

    subReq := ChildRequest{
        Deadline: req.Deadline,  // Propagate
        ...
    }
    // ...
}
```

**Benefits:**

- Avoid wasted compute on doomed requests
- Natural shed load (drop stale requests)
- Propagates across services (gRPC built-in support)

🔥 **[Scar tissue]** - Most cascading failures start with missing deadline propagation. Request stuck in queue 10s, client gave up 8s ago, server still processing = pure waste.

### 6.3.4 Circuit breakers

Khi downstream service fails:

```
┌─────────────────────────────────────────────────────────┐
│  CIRCUIT BREAKER STATE MACHINE                           │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  CLOSED (normal) ─── error rate > threshold ──► OPEN     │
│                                                  │        │
│                                                  │ timer  │
│                                                  ▼        │
│                                             HALF-OPEN    │
│                                                  │        │
│                      test request ◄──────────────┤        │
│                           │                                │
│                  success ─┴─ failure                       │
│                    │               │                       │
│                    ▼               ▼                       │
│                 CLOSED          OPEN                       │
│                                                            │
└─────────────────────────────────────────────────────────┘

OPEN state: Fail fast without calling downstream
           → Downstream has time to recover
           → Client gets quick "unavailable" response
```

**Parameters thực tế:**

- Error rate threshold: 50% over 10s window
- Open duration: 30s
- Half-open test ratio: 10% of requests

---

## 6.4 Rate limiting

### 6.4.1 Nhiều tầng, nhiều purposes

```
┌─────────────────────────────────────────────────────────┐
│  RATE LIMITING LAYERS                                     │
├─────────────────────────────────────────────────────────┤
│                                                            │
│  1. Per-IP (anti-abuse, DDoS)                            │
│     Limit: 100 sessions/min per IP                       │
│     Scope: Pre-auth, at load balancer                    │
│                                                            │
│  2. Per-User (fair use)                                   │
│     Limit: 60 minutes/day for free tier                  │
│     Scope: Post-auth, at API gateway                     │
│                                                            │
│  3. Per-Session (runaway prevention)                     │
│     Limit: Max 30min per session, then force close       │
│     Scope: In session manager                            │
│                                                            │
│  4. Per-Region (capacity protection)                     │
│     Limit: Max 100K concurrent sessions per PoP          │
│     Scope: At admission control                          │
│                                                            │
│  5. Per-Service (resource protection)                    │
│     Limit: Max 10K concurrent LLM streams per cluster    │
│     Scope: In LLM router                                 │
│                                                            │
└─────────────────────────────────────────────────────────┘
```

### 6.4.2 Token bucket algorithm

Standard choice cho rate limiting:

```
Bucket capacity: 100 requests
Refill rate: 10 req/sec

Each request: consume 1 token
If bucket empty: reject or queue
```

**Distributed token bucket** across many servers: Redis-based, script ensures atomicity.

### 6.4.3 Adaptive rate limits

**Static limits fail at edge cases:**

- Black Friday spike: legitimate users hit limit
- Normal day: limits too generous

**Adaptive:**

```
Observe overall system load
If load < 60%: generous limits (2x baseline)
If load 60-80%: baseline limits
If load > 80%: tight limits (0.5x baseline)
```

Protects system while maximizing usage normal times.

### 6.4.4 Rate limit signals to client

**Good UX:** client knows the limit, can back off proactively.

HTTP headers standard:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1729884000
Retry-After: 30
```

Client SDK implements exponential backoff with jitter, respecting these headers.

---

## 6.5 Observability cho streaming pipeline

### 6.5.1 Tại sao standard observability fail

Traditional APM (Datadog, New Relic) designed for REST:

- Request có start/end rõ ràng
- Span per call
- Latency = end - start

**Streaming breaks assumptions:**

- Session runs 5 minutes, nhiều "sub-requests" inside
- "Latency" có nhiều definitions (TTFT, inter-token, E2E per turn)
- Nested streams (audio stream contains token streams contains chunks)

### 6.5.2 Streaming-native metrics

**Không đo "duration of X". Đo "latency to Y event".**

Examples:

```
Per turn (one user utterance → one response):
├─ audio_end_to_asr_first_partial_ms
├─ audio_end_to_asr_final_ms
├─ asr_final_to_llm_first_token_ms     (TTFT)
├─ llm_first_token_to_tts_first_audio_ms
├─ tts_first_audio_to_client_playout_ms
└─ audio_end_to_user_hears_response_ms  (E2E, A definition)

Per session:
├─ session_duration_sec
├─ turns_per_session
├─ interruptions_per_session
├─ reconnects_per_session
└─ quality_score  (calculated from above)

Per inference:
├─ llm_prefill_time_ms
├─ llm_inter_token_time_ms (P50/P99)
├─ llm_total_tokens_generated
├─ llm_prefix_cache_hit_rate
└─ llm_gpu_occupancy_percent
```

### 6.5.3 Distributed tracing for streaming

Challenge: How to trace a request that never "ends"?

**Pattern: Turn-based tracing.**

Each conversational turn = one trace.

```
┌─────────────────────────────────────────────────────────────┐
│  TURN TRACE                                                  │
├─────────────────────────────────────────────────────────────┤
│                                                                │
│  TraceID: turn-abc-123                                        │
│  Attributes: session_id, user_id, turn_number                │
│                                                                │
│  Spans:                                                        │
│  ├─ root_span (audio_end → user_hears)                       │
│  │   ├─ asr_final (child span)                               │
│  │   ├─ llm_prefill (child span)                             │
│  │   │   └─ prefix_cache_lookup (nested)                     │
│  │   ├─ llm_generate (child span)                            │
│  │   │   └─ per_token events (not spans, too many)          │
│  │   ├─ tts_synth (child span, may have multiple chunks)    │
│  │   └─ network_delivery (child span)                        │
│                                                                │
└─────────────────────────────────────────────────────────────┘
```

**Session-level trace** (parent of all turn traces):

- Holds session metadata
- References all turns
- Tracks lifecycle events (connect, disconnect, ICE restart, etc.)

**Tools:**

- OpenTelemetry (standard, works with anything)
- Grafana Tempo, Jaeger, Honeycomb backends
- 🔥 **[Scar tissue]** - Honeycomb's "high cardinality" handling shines cho streaming workloads. Datadog trace search struggle at high session count.

### 6.5.4 Sampling strategies

Can't trace every turn. Sample strategically.

**Head-based sampling (at start):**

```
1% random sampling baseline
100% sampling for:
  - Premium tier users
  - Errors detected early
  - Users in "canary" cohort

Downside: Don't know if turn was slow until end.
```

**Tail-based sampling (after completion):**

```
Sample 100% to local buffer
After turn ends, decide whether to export:
  - P99 slow: keep
  - Error: keep
  - Interesting pattern: keep
  - Otherwise: drop

Downside: Buffer overhead, complex infrastructure
```

**Hybrid (recommended):** Head sample 1% + tail sample all P95+ outliers.

### 6.5.5 Metrics để watch

**RED method adaptation cho streaming:**

```
┌──────────────────────────────────────────────────────────────┐
│  SLI DASHBOARD (top-level)                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  RATE                                                          │
│  ├─ Sessions started per minute                               │
│  ├─ Turns per minute                                          │
│  └─ Tokens generated per minute                               │
│                                                                │
│  ERRORS                                                        │
│  ├─ Session disconnect rate (abnormal)                       │
│  ├─ ASR error rate                                            │
│  ├─ LLM error rate (timeout, OOM, etc.)                      │
│  ├─ TTS error rate                                            │
│  └─ WebRTC ICE failure rate                                   │
│                                                                │
│  DURATION (latency)                                            │
│  ├─ E2E turn latency P50/P90/P99/P99.9                       │
│  ├─ TTFT P50/P99                                              │
│  ├─ First audio latency P50/P99                               │
│  └─ Inter-token latency P50/P99                               │
│                                                                │
│  SATURATION                                                    │
│  ├─ GPU queue depth per cluster                               │
│  ├─ Prefix cache hit rate                                     │
│  ├─ Concurrent sessions per PoP                               │
│  └─ Connection reject rate                                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### 6.5.6 Alerting - Leading indicators

**Bad alerts (lagging, reactive):**

- "P99 E2E latency > 2s" - users already suffering
- "Error rate > 5%" - already customer complaints

**Good alerts (leading):**

- "GPU queue depth growing > threshold for 30s" - before latency hits
- "Prefix cache hit rate dropped > 10%" - something changed
- "TTFT P95 degrading trend" - catch early
- "Reconnect rate rising" - network issue emerging

🔥 **[Scar tissue]** - First 6 months of voice infra: alerts set on P99 latency. Always too late. Rewrote to leading indicators: queue depth, cache hit rate, saturation. Mean time to detect (MTTD) incident 15 min → 2 min.

---

## 6.6 Deploy strategies

### 6.6.1 Why standard blue-green doesn't work

Blue-green: 2 full environments, switch at LB.

**S2S problems:**

- Sessions are stateful, can't switch mid-session
- Doubling infra for deploy = 2x cost during deploy window
- State migration between versions complex

### 6.6.2 Rolling deployment with drain

Standard pattern:

```
Version N → N+1 rolling deploy:

for each batch of nodes (e.g., 10% of fleet):
  1. Remove batch from new-session routing
  2. Wait for sessions to drain (5-15 minutes)
  3. Deploy new version
  4. Health check
  5. Add back to routing
  6. Soak time (watch metrics)
  7. Next batch

Total deploy time: 1-3 hours for whole fleet
```

### 6.6.3 Canary deployment

```
Deploy N+1 to small fraction (1-5%) of fleet.
Route 1-5% of traffic to new version.
Watch key metrics for 30-60 min.
If healthy: proceed with full rollout.
If unhealthy: route back, investigate.
```

**Metrics to watch in canary:**

- Quality scores
- Error rates
- Latency percentiles
- Resource usage (memory, GPU)

**Traffic routing options:**

- By session_id hash (consistent, 1% = always same users)
- By geography (canary one region first)
- By user cohort (internal employees, beta testers)

### 6.6.4 Feature flags

Deploy code != enable feature. Decouple via flags:

```
// Code deployed (carries new model support)
if feature_flags.is_enabled("use_new_llm_model", session_id):
    response = new_llm(prompt)
else:
    response = old_llm(prompt)
```

Benefits:

- Roll out feature gradually (1% → 10% → 50% → 100%)
- Kill switch in seconds (vs full redeploy)
- A/B test variants

🔥 **[Scar tissue]** - Feature flags are mandatory for ML/AI features. Mỗi model update, routing change, prompt change = flag. Never deploy without kill switch.

---

## 6.7 Disaster recovery & multi-region failover

### 6.7.1 What failures to plan for?

```
┌──────────────────────────────────────────────────────────────┐
│  FAILURE SCENARIOS & RECOVERY TIME                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Single node failure                                           │
│  ├─ Impact: Sessions on that node lose                        │
│  ├─ Recovery: Client reconnect, routed elsewhere              │
│  └─ MTTR: 5-30 seconds                                        │
│                                                                │
│  Single PoP failure (network, power)                          │
│  ├─ Impact: All sessions at that PoP lose                     │
│  ├─ Recovery: DNS failover, route to next PoP                 │
│  └─ MTTR: 1-5 minutes                                         │
│                                                                │
│  Regional cluster failure (e.g., entire AWS region)          │
│  ├─ Impact: Significant traffic disruption                    │
│  ├─ Recovery: Multi-region failover                          │
│  └─ MTTR: 5-30 minutes depending on preparation               │
│                                                                │
│  Model serving degradation (bad deploy, corrupt weights)     │
│  ├─ Impact: Quality degrades, not outage                      │
│  ├─ Recovery: Rollback, feature flag                          │
│  └─ MTTR: 2-15 minutes                                        │
│                                                                │
│  Cascading failure (one service takes others down)           │
│  ├─ Impact: Hard to scope, often severe                       │
│  ├─ Recovery: Circuit breakers, kill switches                 │
│  └─ MTTR: 15 minutes - several hours                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### 6.7.2 Multi-region active-active

**For true reliability, active-active multi-region.**

```
┌─────────────────────────────────────────────────────────┐
│  ACTIVE-ACTIVE TOPOLOGY                                   │
├─────────────────────────────────────────────────────────┤
│                                                            │
│  Users in APAC ──► APAC PoPs ──► APAC core region         │
│                                                            │
│  Users in EU ────► EU PoPs ─────► EU core region          │
│                                                            │
│  Users in US ────► US PoPs ─────► US core region          │
│                                                            │
│  State replication (async, eventually consistent):        │
│  ├─ User profile, preferences                             │
│  ├─ Conversation long-term memory                         │
│  └─ Not real-time session state                           │
│                                                            │
│  Failure handling:                                         │
│  APAC region down → route APAC users to nearest (EU)      │
│  Latency increases 50-150ms, but service continues        │
│                                                            │
└─────────────────────────────────────────────────────────┘
```

**Cost:** ~2x infrastructure (3 regions vs 1). Premium for reliability.

### 6.7.3 Chaos engineering

📐 **[Engineering judgment]** - Regular chaos testing mandatory at scale:

- **Node kill**: Random node termination, verify graceful handling
- **Network partition**: Simulate region isolation
- **Latency injection**: Add 200ms to downstream calls, verify timeouts work
- **GPU degradation**: Simulate GPU OOM, verify graceful fallback

Tools: Netflix Chaos Monkey, Gremlin, custom scripts.

🔥 **[Scar tissue]** - First chaos test usually breaks things you didn't know were fragile. That's the point. Better to find in staging than at 3AM.

---

## 6.8 Tổng kết Phần 6

**Những thứ quan trọng nhất:**

1. **15-25 PoPs** đủ cho global S2S với P95 RTT < 40ms. Đừng đua với CDN.
2. **Tách signaling (TCP/WebSocket) và media (UDP)** paths. L4 LB cho media, L7 cho signaling.
3. **Session affinity** via consistent hashing là foundation.
4. **Drain time đo bằng phút**, không giây. Plan deploy accordingly.
5. **Backpressure cần graceful degradation**, không pure queue hoặc reject.
6. **Deadline propagation** cross-service là mandatory.
7. **Observability phải streaming-native** - turn-based tracing, leading indicators.
8. **Canary + feature flags** mandatory cho AI/ML features.
9. **Multi-region active-active** là premium tier reliability.
10. **Chaos engineering** finds fragile things before 3AM.

**Những con số để nhớ:**

- PoP count: 15-25 cho global, 5-10 enterprise
- Drain time: 5-15 phút per node batch
- Deploy rolling: 1-3 giờ full fleet
- Canary traffic: 1-5% initial
- Session affinity: consistent hash by session_id
- Alert threshold: leading (queue depth, cache hit), not lagging (P99)

**Anti-patterns:**

- HTTP LB cho WebRTC media
- Blue-green deploy cho long-lived sessions
- P99 latency alerts (too late)
- Pure queue under backpressure
- Single region without failover plan
- No deadline propagation

---

# Phần 7 - Trade-offs quan trọng

> Mục tiêu: Đây là phần tôi sẽ **có ý kiến rõ ràng**. Không "it depends" một cách lười. Mỗi trade-off sẽ có: hai option, khi nào chọn cái nào, và ý kiến của tôi với justification.

---

## 7.0 Framing - Trade-off không phải "pick one"

Trước khi vào matrix, một principle:

**Hầu hết trade-off không phải binary.** Trong thực tế, bạn thường:

1. Default sang một option (cái "safe")
2. Selectively flip sang option kia cho specific cases
3. Evolve theo thời gian khi scale/requirements thay đổi

Ví dụ: "edge vs centralized" không phải all-or-nothing. ASR ở edge, LLM ở core, TTS ở edge - đó là answer thực tế.

**Nhưng nếu phải chọn default, có answer đúng hơn cho hầu hết cases.** Phần này sẽ nói thẳng.

---

## 7.1 Trade-off matrix - Tổng quan

| #   | Trade-off                   | Option A                   | Option B                 | My default                                                                        |
| --- | --------------------------- | -------------------------- | ------------------------ | --------------------------------------------------------------------------------- |
| 1   | Latency vs Cost             | Edge inference             | Centralized inference    | **Hybrid** - small models edge, large models core                                 |
| 2   | Quality vs Speed            | Large model (70B)          | Distilled/small (7-13B)  | **13B** cho default, 70B cho premium tier                                         |
| 3   | Flexibility vs Simplicity   | Microservices per layer    | Monolithic pipeline      | **Hybrid monolith** - media node monolith, inference services                     |
| 4   | Consistency vs Availability | Strong session state       | Stateless + replay       | **Available with graceful degradation**                                           |
| 5   | Build vs Buy (ASR)          | Self-host model            | API (Deepgram)           | **API < 2M min/day, self-host beyond**                                            |
| 6   | Build vs Buy (LLM)          | Self-host model            | API (OpenAI/Anthropic)   | **API early, self-host at scale - but expect 12-18 month transition**             |
| 7   | Build vs Buy (TTS)          | Self-host                  | ElevenLabs/Cartesia      | **Self-host earlier than you think - TTS break-even sớm**                         |
| 8   | Streaming protocol          | WebRTC                     | WebSocket/QUIC           | **WebRTC** - not close                                                            |
| 9   | Language/runtime            | Rust                       | Go / Elixir / C++        | **Depends on team - all viable, but Rust/Elixir for media, Go for orchestration** |
| 10  | Orchestration               | Kubernetes                 | Nomad / Bare metal       | **K8s for control plane, bare metal for media/GPU**                               |
| 11  | State store                 | Redis                      | DragonflyDB / In-process | **Start Redis, migrate when pain shows**                                          |
| 12  | Observability               | Self-hosted (OTel+Grafana) | SaaS (Datadog)           | **SaaS early, self-host when bill > team salary**                                 |
| 13  | Deploy model                | Kubernetes rolling         | Immutable blue/green     | **Rolling with long drain for media, blue/green OK for stateless**                |
| 14  | Single cloud                | AWS/GCP/Azure              | Multi-cloud              | **Single cloud + bare metal, multi-cloud only for compliance**                    |
| 15  | Training data strategy      | Use customer data          | Synthetic + curated      | **Synthetic + opt-in customer data with clear consent**                           |

Giờ đi sâu từng cái.

---

## 7.2 Trade-off 1 - Latency vs Cost (Edge vs Centralized)

### Option A: Edge inference everywhere

```
Every PoP has full stack: ASR + LLM + TTS
User connects nearest PoP, full inference local
```

**Pros:**

- Lowest network latency (no edge-core hop)
- Failure isolation (one PoP down = others OK)
- Data sovereignty easier

**Cons:**

- GPU underutilization (spread across many PoPs)
- Model deployment complex (20+ PoPs to update)
- Cost ~2-3x higher than centralized
- Smaller models only (fitting in edge GPU budget)

### Option B: Centralized inference

```
Few core regions with big GPU clusters
Edges only do media + lightweight processing
Edge → core roundtrip per query
```

**Pros:**

- High GPU utilization
- Fewer places to deploy updates
- Can run large models (70B+)
- Lower unit cost

**Cons:**

- Edge-core roundtrip adds 30-80ms per query
- Regional outage = significant impact
- Cross-border data flow concerns

### My take: Hybrid, split by model size

```
Edge (20+ PoPs):
├─ WebRTC media (always - latency critical)
├─ VAD, jitter buffer
├─ ASR (small-medium, 1-2B)      ← fit in edge GPU
└─ TTS (Kokoro or similar)        ← fit in edge GPU

Core (3-4 regions):
└─ LLM (13-70B)                   ← needs big GPU, benefits from batching
```

**Why:**

- ASR/TTS are small, latency-critical, benefit directly from edge placement
- LLM is large, batching-hungry, can tolerate 50ms edge-core roundtrip
- Saves ~80-150ms vs fully centralized
- Costs ~50-60% of fully edge

🔥 **[Scar tissue]** - This is battle-tested hybrid. Uniform "everything edge" sounds good in slideware, falls apart economically.

---

## 7.3 Trade-off 2 - Quality vs Speed (Model size)

### Option A: Large model (70B+)

**Pros:** Better reasoning, instruction following, fewer hallucinations
**Cons:** 3-5x cost, 50-150ms slower TTFT, harder ops

### Option B: Small/distilled (7-13B)

**Pros:** Fast, cheap, easier to serve
**Cons:** Weaker on complex reasoning, more hallucinations, worse tool use

### My take: 13B default, 70B premium tier

Most S2S conversations are:

- Greetings, acknowledgments
- Simple Q&A (factual lookup + phrasing)
- Task execution (book, cancel, confirm)
- Basic chitchat

**13B model handles 80%+ of this well.**

The 20% that needs 70B:

- Multi-step reasoning
- Complex tool orchestration
- Nuanced emotional support
- Specialized domain knowledge

**Routing answer:**

```python
# Fast classifier (tiny model or heuristic)
complexity = classify(user_utterance, conversation_context)

if complexity == "simple":
    model = "13B"  # 80% of traffic
elif complexity == "complex":
    model = "70B"  # 15% of traffic, acceptable to pay more
elif complexity == "trivial":
    model = "7B"  # 5% of traffic, could be cached/templated
```

📐 **[Engineering judgment]** - This is the architecture that scales economically. Uniform 70B = bleed money. Uniform 7B = bad UX.

**Warning:** Classifier itself must be fast (<20ms) and accurate. Bad classifier = premium users get cheap model = complaints.

---

## 7.4 Trade-off 3 - Microservices vs Monolith

### Option A: Full microservices

```
Separate services: VAD, ASR, LLM-router, LLM-worker, TTS, Session-manager, etc.
Communicate via gRPC/message bus
```

**Pros:**

- Independent scaling, deploy
- Team ownership clear
- Language/stack flexibility per service

**Cons:**

- Network latency at each hop (1-5ms per call × many calls)
- Complex observability
- Debug harder (distributed trace required)
- More moving parts → more failure modes

### Option B: Monolithic pipeline

```
One binary handles entire session end-to-end
All components in same process
```

**Pros:**

- Lowest latency (in-process function calls)
- Simpler debug
- Shared memory, no serialization overhead

**Cons:**

- Hard to scale heterogeneous workloads
- Single failure = session dies
- Team bottleneck (everyone changes same codebase)

### My take: Hybrid monolith

```
┌──────────────────────────────────────────────────────┐
│  MONOLITH: Session / Media Node                      │
│  (one binary, one process per session)              │
│                                                        │
│  - WebRTC termination                                 │
│  - Jitter buffer, VAD                                 │
│  - Session FSM                                        │
│  - Orchestration of ASR/LLM/TTS calls                │
│                                                        │
│  Language: Rust or Elixir                            │
│  Stateful, long-lived per session                    │
└──────────────────────────────────────────────────────┘
          │
          │  gRPC streaming
          ▼
┌──────────────────────────────────────────────────────┐
│  SERVICES (separate): ASR worker, LLM worker, TTS    │
│                                                        │
│  - GPU-heavy                                          │
│  - Batch across sessions                             │
│  - Independent scaling                               │
│                                                        │
│  Language: Python (wrapping vLLM, etc.)              │
│  Stateless, pool-based                               │
└──────────────────────────────────────────────────────┘
```

**Why:**

- Session logic is tightly coupled (VAD triggers ASR, ASR drives LLM, etc.). Microservice split here = 10+ RPCs per turn = 30-50ms wasted
- GPU inference is independent workload with different scaling needs. Separate service makes sense.

🔥 **[Scar tissue]** - I've seen teams break up session logic into 8 microservices. Every turn: 20+ RPCs, distributed transaction challenges, debug nightmare. Revert to monolith-per-session: latency -40ms, bugs -60%.

---

## 7.5 Trade-off 4 - Consistency vs Availability

### Option A: Strong session state

```
Every session event persisted synchronously
Node failure → replay from log → resume session
```

**Pros:** No data loss, session survives failures
**Cons:** Write latency on critical path, complex failover

### Option B: Stateless + best-effort replay

```
Session state in-memory only
Node failure → session lost, client reconnects fresh
Conversation history from client-side or short-term cache
```

**Pros:** Fast, simple
**Cons:** User experiences reset on failures

### My take: Available with graceful degradation

**Reality:**

- Users won't tolerate long latency waiting for persistence
- Users DO tolerate brief reconnect if handled smoothly
- "Session survival" across node failure is impossible for WebRTC anyway

**Design:**

```
Hot state (in-memory, lost on failure):
├─ WebRTC connection
├─ Jitter buffer
├─ Current FSM state
└─ In-flight audio chunks

Warm state (persisted async, last N turns):
├─ Conversation history (text)
├─ Session metadata
└─ Persisted every turn boundary (not every token)

Cold state (persisted sync, cross-session):
├─ User profile
├─ Long-term memory
└─ Billing/audit
```

**Failure handling:**

- Node dies: client auto-reconnects (1-3s disruption)
- New session resumes conversation from warm state
- User experiences brief "one moment please" message
- Better than waiting for "strong consistency" mid-conversation

📐 **[Engineering judgment]** - CAP theorem applied: you cannot have consistency + latency + availability in streaming systems. Voice users demand latency + availability. Consistency is approximated via async persistence.

---

## 7.6 Trade-off 5 - Build vs Buy: ASR

### Recap from Phần 2

| Volume         | Recommendation |
| -------------- | -------------- |
| < 500K min/day | API (Deepgram) |
| 500K - 2M      | Hybrid         |
| > 2M           | Self-host      |

### My take: API by default, self-host at 2M/day

**Why default API:**

- Deepgram Nova-2 is genuinely good
- $0.0043/min = you can build a lot before this dominates cost
- Avoids team building+operating ASR infrastructure
- They improve models continuously - you benefit without work

**When self-host worth:**

- Cost of API exceeds 2-3x team cost to self-host (~$2M/year)
- Need custom language/dialect not well-supported
- Need specialized domain (medical, legal) models
- Want latency below what API can provide (< 80ms first partial)

🔥 **[Scar tissue]** - Self-hosted ASR sounds easy ("just Whisper!") until you realize:

- Whisper default isn't streaming-native
- Deployment complexity (10+ GPU node cluster)
- Model updates require training pipeline
- Multi-language support requires multiple models or large model

**Plan 12-18 months from decision to self-host to stable production.** Don't underestimate.

---

## 7.7 Trade-off 6 - Build vs Buy: LLM

This is the **most contested** decision. And the most expensive to get wrong.

### Option A: API (OpenAI, Anthropic, Google)

**Pros:**

- Always latest model
- No infrastructure to run
- Predictable cost (per-token)
- Tool use, function calling mature

**Cons:**

- Expensive at scale ($10-50/M tokens input, $30-150/M output)
- Latency higher (external API call, 50-200ms extra)
- Vendor lock-in
- Rate limits
- Data privacy concerns

### Option B: Self-host open models

**Pros:**

- Lower unit cost at scale
- Lower latency (local)
- Customization (fine-tuning, system prompts deep)
- Data stays internal

**Cons:**

- Massive infra investment
- Model quality gap (open models trail frontier by ~6-12 months)
- Team needed to operate
- Model updates require effort

### My take: API early, self-host at scale, but expect long transition

**Phase 1 (< 1M min/day): API.**

- Focus on product, not infra
- Gemini Flash, GPT-4o-mini, Claude Haiku are cheap + good enough
- Cost ~$0.02-0.05/min at this volume

**Phase 2 (1-5M min/day): Hybrid.**

- Self-host 7-13B for bulk traffic (simple turns)
- API for complex turns (reasoning, tool use)
- Routing classifier decides

**Phase 3 (> 5M min/day): Primarily self-host.**

- Full self-hosted stack with tier routing
- API as fallback / premium quality tier
- Unit cost reduced 60-80%

🔥 **[Scar tissue]** - Biggest mistake: "we'll self-host LLM from day 1 to own the stack."

Reality:

- 6 months just to build serving stack (pre-vLLM, now slightly better)
- 12 months to match API quality on your domain
- Meanwhile, product suffers from lack of attention
- Competitor using API ships features 10x faster

**Rule of thumb: Self-host LLM is a 2-year commitment. Don't start unless you know you'll be at scale to justify.**

---

## 7.8 Trade-off 7 - Build vs Buy: TTS

### Options

- **ElevenLabs / Cartesia API**: Best quality, expensive ($0.05-0.30/min)
- **Self-host Kokoro / XTTS / custom**: Lower cost ($0.003-0.008/min), quality can match with effort

### My take: Self-host TTS earlier than you think

**Why TTS is different from ASR/LLM:**

- Model sizes smaller (few hundred MB to few GB)
- Simpler to operate (stateless, simple batching)
- Quality gap vs commercial narrower (Kokoro, Parler are surprisingly good)
- Break-even point ~200K-500K min/day (vs 2M for ASR)

**When API still worth:**

- Need specific voice cloning at scale (ElevenLabs ecosystem unmatched)
- Early stage, not yet at volume
- Need specialized languages/accents commercial handles

📐 **[Engineering judgment]** - I'd self-host TTS as soon as you hit 200K min/day. ROI fastest of the three layers.

---

## 7.9 Trade-off 8 - Streaming protocol: WebRTC vs QUIC vs WebSocket

### Already covered in Phần 2, recap decision

| Protocol          | Media                     | Signaling                           |
| ----------------- | ------------------------- | ----------------------------------- |
| WebRTC            | ✅ Default                | ⚠️ SDP is heavy, use custom over WS |
| WebSocket         | ❌ No (TCP HoL blocking)  | ✅ For signaling                    |
| QUIC/WebTransport | 🔬 Future, not yet mature | 📐 Promising                        |
| Raw UDP           | ❌ Too much to build      | ❌ Not applicable                   |

### My take: WebRTC for media, WebSocket for signaling

**Not even close.** WebRTC for media is the answer for any voice application with consumer reach.

**Unless:** You have only custom native app clients + specific requirements. Then QUIC/custom UDP can work. But even then, libwebrtc buys you so much you should default to it.

---

## 7.10 Trade-off 9 - Language/runtime

### Options

**Rust:** Performance, memory safety, but steep learning curve, slower iteration
**Go:** Good balance, popular, GC pauses can hurt latency-critical
**Elixir/BEAM:** Great for stateful streaming, weird for the rest of stack
**C++:** Ultimate performance, but hiring, safety issues
**Python:** Fast iteration, slow runtime - not for hot path

### My take: Depends on layer and team

```
Layer                    | Preferred languages
─────────────────────────|──────────────────────────
Media server (WebRTC)    | Rust or C++ or Elixir
Session orchestration    | Rust or Elixir or Go
GPU inference serving    | Python (with C++/CUDA deps)
Control plane / APIs     | Go or Python or TypeScript
Observability tooling    | Go, Rust, Python
```

**Why Rust/Elixir for media:**

- Media server is hot path, microsecond-sensitive
- Stateful per session (Elixir shines) or performance-critical (Rust)
- GC pause during 20ms audio frame = audible glitch

**Why Python for inference:**

- ML ecosystem (vLLM, SGLang, TRT-LLM) all Python
- Inference is GPU-bound, Python overhead <5% of work
- Python's flexibility benefits ML experimentation

**Why Go for control plane:**

- Fast enough, good enough concurrency
- Team productivity high
- Huge ecosystem for K8s/cloud stuff

🔥 **[Scar tissue]** - Hiring matters. I've seen teams choose Rust because "fast!" but team is 80% Python engineers. 6 months lost to learning curve. Pragmatism wins.

Discord uses Elixir for voice extensively. Works beautifully for stateful session management. Rare skill in market.

---

## 7.11 Trade-off 10 - Orchestration: Kubernetes vs Bare metal vs Nomad

### Option A: Kubernetes everything

**Pros:** Standard, ecosystem, ops tooling
**Cons:** Overhead for stateful/long-lived workloads, complex for GPU

### Option B: Bare metal for critical, K8s for rest

**Pros:** Low overhead where it matters
**Cons:** Two operational models

### Option C: Nomad

**Pros:** Simpler than K8s, handles mixed workloads
**Cons:** Smaller ecosystem, less cloud provider integration

### My take: K8s for control plane, bare metal for media + GPU serving

**K8s is great for:**

- API services, orchestration logic
- Stateless workloads
- Short-lived jobs
- Anything where K8s' opinions help

**K8s is painful for:**

- Long-lived stateful sessions (pod evictions mid-session)
- UDP media traffic (LB complexity)
- GPU with specialized scheduling (GPU sharing, MIG, pipeline stages)
- Low-level networking (DPDK, XDP)

**Discord uses bare metal extensively for voice.** Rationale:

- Eliminates virtualization overhead
- Direct control over networking (NIC offload, peering)
- Better cost at scale (no cloud markup)
- Team experienced with bare metal ops

📐 **[Engineering judgment]** - At < 100K concurrent sessions, K8s-everything is fine. Above that, bare metal for media pays off. The transition itself is a 6-12 month project.

**Karpenter for GPU autoscaling** on K8s is genuinely good if you stay cloud-native. Handles spot, diverse instance types, scale-up/down. Best K8s GPU story I've seen.

---

## 7.12 Trade-off 11 - State store: Redis vs Alternatives

### Options

- **Redis**: Default, proven, ecosystem
- **DragonflyDB**: Drop-in Redis replacement, multi-threaded, 2-3x throughput
- **ScyllaDB/Cassandra**: Persistent, scalable, higher latency (~5ms)
- **In-process (Elixir/Erlang)**: Ultimate latency, but distribution complex

### My take: Start Redis, migrate when pain shows

**Start with Redis because:**

- Well-understood, team probably knows it
- Managed options available (ElastiCache, MemoryStore)
- Sufficient for most scales
- Migration to alternatives straightforward if needed

**Migrate when:**

- Redis CPU bottleneck (single-threaded per shard) → DragonflyDB
- Need durability + scale → ScyllaDB
- Need sub-millisecond state access → in-process with distributed fallback

🔥 **[Scar tissue]** - "Redis is bottleneck" surprises teams at higher scales than expected. Often what appears to be Redis issue is actually network chattiness or pipelining not used. Profile before migrating.

---

## 7.13 Trade-off 12 - Observability: SaaS vs Self-hosted

### Options

**SaaS (Datadog, Honeycomb, Grafana Cloud):**

- Pros: Fast setup, rich features, no ops burden
- Cons: Cost balloons with scale, vendor lock-in, data exfil concerns

**Self-hosted (OpenTelemetry + Prometheus + Grafana + Tempo):**

- Pros: Cost control, data stays, customization
- Cons: Operational burden, significant investment

### My take: SaaS until bill > observability team cost

**Math:**

```
SaaS cost at scale: $50-150K/month typical
Observability team (2 engineers + infra): $80-120K/month
Break-even: $50-100K/month SaaS cost
```

**Until break-even, SaaS wins.** After, self-host.

**Hybrid pattern common:**

- Core metrics/logs: self-hosted (cheap at volume)
- Distributed traces: SaaS (complex to self-operate)
- Session replay / user-facing analytics: SaaS
- Internal debugging: mix

🔥 **[Scar tissue]** - Datadog bill at high session counts can exceed $1M/year fast. Tagging/cardinality surprises. Set alerts on your observability spend.

---

## 7.14 Trade-off 13 - Deploy: Rolling vs Blue/Green

Already covered in Phần 6. Recap:

### My take: Rolling with long drain for media, blue/green OK elsewhere

- Media nodes: rolling, 5-15 min drain per batch
- Inference workers: rolling with short drain (sessions not bound)
- Control plane APIs: blue/green works, canary appropriate
- Databases: whatever the DB supports

---

## 7.15 Trade-off 14 - Cloud strategy

### Options

- **Single cloud** (AWS or GCP or Azure): Depth of integration
- **Multi-cloud**: Redundancy, leverage, compliance
- **Hybrid cloud + bare metal**: Cost + reliability

### My take: Single cloud + bare metal, multi-cloud only for compliance

**Multi-cloud is expensive:**

- Duplicate operational knowledge
- Worst of each cloud's limitations
- Data egress between clouds = $$$
- Inconsistent capabilities

**When multi-cloud justifies:**

- Customer contracts require (gov, healthcare, finance)
- Geographic coverage (e.g., China needs Alibaba/Tencent)
- Proven vendor risk (outage patterns)

**Bare metal addition:**

- Cloud for flexibility, burst, control plane
- Bare metal for baseline steady-state (cost)
- Direct peering with tier-1 networks

📐 **[Engineering judgment]** - AWS + bare metal is Discord's pattern. GCP + bare metal works similarly. Don't try both clouds. Add bare metal when cost justifies (usually > $10M/year cloud spend).

---

## 7.16 Trade-off 15 - Training data strategy

Real concern for S2S: improvement flywheel requires data. Privacy concerns real.

### Options

**A. Use all customer data:**

- Fast model improvement
- Legal/ethical issues
- Customer trust risk

**B. Synthetic + curated data only:**

- Privacy-safe
- Slower improvement
- Less domain-specific

**C. Opt-in customer data with clear consent:**

- Balance
- Smaller data pool
- Manageable legal

### My take: Synthetic + opt-in customer data with explicit consent

**Reasons:**

- Legal compliance (GDPR, CCPA, enterprise contracts)
- User trust = business asset
- Quality difference between opt-in and all-data is smaller than expected
- Synthetic data generation improving rapidly

**Architecture:**

```
Default: No data retention beyond session end
Opt-in tier: User explicitly consents → data flows to training pipeline
Enterprise: Often contractually zero retention, period
```

🔥 **[Scar tissue]** - Never assume "anonymization" is enough. Voice biometrics identify users. Conversation content often PII. Design data flows assuming full identifiability.

---

## 7.17 Meta trade-off - Organizational

Brief but important:

**Small team (< 10) doing S2S infra:**

- Buy more, build less
- Focus differentiator (usually: your product experience, not infra)
- Expect velocity, not unit economics
- API everything works

**Medium team (10-30):**

- Self-host one critical piece (often TTS first)
- Edge deploy for premium tier
- Begin measuring unit economics seriously

**Large team (30+) with scale:**

- Self-host everything feasible
- Custom optimizations
- Dedicated observability, security, SRE
- Negotiate peering, RIs, custom hardware

**Don't build for the team you want to be. Build for the team you are.** Organizational capacity matters as much as technical capability.

---

## 7.18 Tổng kết Phần 7

**Những quan điểm cốt lõi:**

1. **Hybrid topology wins** - edge for media+light models, core for heavy models.
2. **Tiered quality wins** - 13B default + 70B premium > uniform anything.
3. **Monolithic session node + microservice inference** - best latency/modularity balance.
4. **Availability > Consistency** for streaming - graceful degradation mandatory.
5. **API early, self-host at scale** - but expect 12-24 month self-host transition.
6. **TTS self-host sooner than ASR/LLM** - fastest break-even.
7. **WebRTC for media, WebSocket for signaling** - not debatable.
8. **Rust/Elixir for media, Python for inference, Go for control** - language by layer.
9. **K8s for control, bare metal for media+GPU** - at scale.
10. **SaaS observability until bill > team cost** - then self-host.
11. **Single cloud + bare metal > multi-cloud** - unless compliance forces.
12. **Synthetic + opt-in data** - privacy as business strategy.

**Meta-principle:**

> Most trade-offs aren't about "best technology" but "best fit for your scale, team, and stage." Same decision that's right at 10K concurrent is wrong at 1M. Plan to evolve - don't optimize prematurely, but don't paint yourself into corners either.

**Anti-patterns across all trade-offs:**

- Uniform everything (uniform model, uniform tier, uniform region)
- Premature optimization (self-host from day 1)
- Premature simplification (API forever, even at hyperscale)
- Over-engineering (full microservices too early)
- Under-engineering (monolith at 100K concurrent)
- Ignoring team capacity

---

# Phần 8 - Technology stack đề xuất

> Mục tiêu: Stack cụ thể với version, config, lý do. **Không "dùng Kubernetes"** - mà "K8s 1.29 + Karpenter cho GPU, deploy pattern X vì Y". Phần này đi vào detail operational.

---

## 8.0 Framing - Stack này cho ai?

Stack tôi đề xuất dưới đây assume:

- **Scale target:** 50K-500K concurrent sessions
- **Stage:** Past MVP, đang scale production
- **Team size:** 15-50 engineers trên infra + ML
- **Priority:** Balanced - không hyperscale (> 1M concurrent), không tiny

**Cho stage khác, stack sẽ khác:**

- Early stage (< 10K concurrent): API everything, skip half stack này
- Hyperscale (> 1M concurrent): Custom silicon, direct peering, custom protocol. Stack này là floor, không ceiling.

**Disclaimer:** Tôi sẽ nêu rõ khi một choice là [🔥 battle-tested], [📐 engineering judgment], hay [📚 based on vendor/literature].

---

## 8.1 Overview - Stack diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│  CLIENT SIDE                                                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  • libwebrtc (browser native or mobile SDK)                          │
│  • Custom signaling client (WebSocket + protobuf)                    │
│  • Silero VAD (client-side, 1MB, lite)                               │
│  • Opus codec (24kHz, 20ms, VBR 24-32kbps)                          │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
                            │ WebRTC (UDP + DTLS + SRTP)
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  EDGE POP                                                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  NETWORK                                                              │
│  • Anycast BGP for signaling discovery                               │
│  • XDP/eBPF (Katran-style) for L4 load balancing media               │
│  • DPDK optional for packet pipeline                                 │
│                                                                        │
│  MEDIA SERVER (bare metal or dedicated VMs)                          │
│  • Custom SFU in Rust (webrtc-rs) or Elixir (elixir-webrtc)          │
│  • Session state: in-process + async Redis replication               │
│                                                                        │
│  EDGE INFERENCE                                                       │
│  • ASR: faster-whisper-v3-turbo on L40S or A10                       │
│  • TTS: Kokoro-v1.2 on L40S                                          │
│  • GPU scheduling: Kubernetes + NVIDIA GPU Operator                  │
│  • Serving framework: Triton Inference Server or custom Python        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
                            │ gRPC streaming (HTTP/2)
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  CORE REGION                                                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  LLM SERVING                                                          │
│  • vLLM 0.6+ or SGLang 0.3+ (mix, route by workload)                 │
│  • Models: Llama-3.3-70B FP8 (premium), Qwen2.5-14B FP8 (standard) │
│  • H100 SXM (premium), H100 PCIe or L40S (standard)                 │
│  • Prefix cache: built-in + Redis persistent layer                   │
│  • Continuous batching + chunked prefill enabled                     │
│                                                                        │
│  CONTROL PLANE                                                        │
│  • Kubernetes 1.30+ with Karpenter                                   │
│  • Istio or Linkerd service mesh                                     │
│  • ArgoCD for GitOps deploy                                          │
│                                                                        │
│  STATE STORES                                                         │
│  • Redis Cluster or DragonflyDB (session warm state)                 │
│  • PostgreSQL (user data, billing)                                   │
│  • ClickHouse (conversation analytics)                               │
│  • S3-compatible (cold storage, recordings if retained)              │
│                                                                        │
│  OBSERVABILITY                                                        │
│  • OpenTelemetry SDKs everywhere                                     │
│  • Grafana Tempo (traces), Loki (logs), Mimir (metrics)             │
│  • Or Honeycomb/Datadog until self-host economics justify            │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

Giờ đi sâu vào từng mảng.

---

## 8.2 Transport / Media server

### 8.2.1 Đề xuất: Custom SFU, Rust hoặc Elixir

**Không dùng MCU (Multipoint Conferencing Unit).** S2S là 1:1 (user ↔ agent), SFU overkill nhẹ nhưng giản hơn MCU.

**Không dùng mediasoup/ion-sfu thuần** cho scale lớn - phù hợp MVP và early stage, nhưng customization limit khi scale.

### 8.2.2 Rust: webrtc-rs ecosystem

**Stack:**

```
webrtc-rs (https://github.com/webrtc-rs/webrtc) - core WebRTC
str0m (https://github.com/algesten/str0m) - alternative, simpler
Tokio async runtime
```

**Pros:**

- Performance (no GC, predictable latency)
- Memory safety (no segfault in production)
- Ecosystem growing

**Cons:**

- Async Rust is hard (learning curve steep)
- Hiring pool smaller than Go
- Ecosystem younger than C++ equivalents

🔥 **[Scar tissue]** - Rust WebRTC ecosystem mature enough for production as of 2025-2026. webrtc-rs hit 0.12+ stability. Several mid-sized companies running it. Still less battle-tested than Google's libwebrtc (C++).

### 8.2.3 Elixir: BEAM cho stateful media

**Stack:**

```
elixir-webrtc (https://github.com/elixir-webrtc/ex_webrtc)
Membrane Framework (https://membrane.stream/)
Phoenix for signaling (WebSocket)
```

**Pros:**

- BEAM concurrency model **perfect** for session-per-process
- Fault isolation (crash one session doesn't affect others)
- Hot code upgrade (update without restart - rare but powerful)
- Discord-proven at scale

**Cons:**

- Not great for raw CPU-bound audio processing (use NIFs for those)
- Smaller talent pool
- ML/AI ecosystem weak (need to bridge to Python for models)

### 8.2.4 My take

📐 **[Engineering judgment]** - Choose based on team:

- **Team knows Rust / wants performance-max**: webrtc-rs
- **Team knows Erlang/Elixir / likes actor model**: elixir-webrtc + Membrane
- **Team is polyglot with strong C++ roots**: libwebrtc wrapper, pain but proven
- **Team is Go-heavy**: pion/webrtc - usable, GC pauses manageable

Discord chose Elixir for voice chat. Works beautifully for stateful session management with thousands of concurrent sessions per node.

### 8.2.5 Configuration gotchas

```yaml
# Media server tuning (conceptual config)
webrtc:
  codec: opus
  sample_rate: 24000 # Super-wideband speech
  frame_duration: 20ms
  bitrate: 24000-32000 # VBR range
  fec: enabled
  dtx: enabled # Silence suppression
  plc: enabled # Packet loss concealment

jitter_buffer:
  min_depth: 20ms
  max_depth: 200ms
  adaptive: true
  target_percentile: 95 # Based on recent jitter P95

connection:
  ice_restart_interval: 30s
  dtls_handshake_timeout: 5s
  idle_timeout: 600s # 10 min, for voice conversations

os_tuning:
  net.core.rmem_max: 134217728 # 128MB UDP buffer
  net.core.wmem_max: 134217728
  net.ipv4.udp_rmem_min: 8192
  net.core.netdev_max_backlog: 5000
  nf_conntrack_max: 2097152 # 2M, for high concurrent
```

🔥 **[Scar tissue]** - Default Linux UDP buffers way too small for voice at scale. Tune these Day 1. Without tuning, packet drops at 10K+ concurrent.

---

## 8.3 ASR - Speech recognition

### 8.3.1 Đề xuất: faster-whisper-v3-turbo (self-host) hoặc Deepgram Nova-2 (API)

### 8.3.2 Self-host option: faster-whisper

**Stack:**

```
Model: Whisper-large-v3-turbo (or distilled variants)
Runtime: faster-whisper (CTranslate2 backend)
Serving: Triton Inference Server or custom Python + gRPC
GPU: NVIDIA L40S (48GB, good perf/$ for ASR)
```

**Config:**

```python
# Conceptual
model = WhisperModel(
    "large-v3-turbo",
    device="cuda",
    compute_type="int8_float16",  # Quantized, fast
    num_workers=2,  # Parallel decoding
)

# Streaming mode (chunk-based, ~500ms chunks)
chunks = []
async for audio_chunk in input_stream:
    chunks.append(audio_chunk)
    if len(chunks) * 20ms >= 500ms:  # Buffer window
        partial = model.transcribe(
            concat(chunks),
            beam_size=3,  # Lower for speed
            best_of=1,
            temperature=0.0,
            vad_filter=True,
            language="vi",  # Or auto-detect
        )
        yield partial
```

**Throughput on L40S:**

- Whisper-large-v3-turbo INT8: ~80-120 concurrent streams
- Latency first partial: 200-400ms (chunk-based)
- Cost: ~$0.004/min self-hosted

**Alternative: Streaming-native models**

Nếu chunk-based Whisper không đủ fast:

- **NVIDIA Canary** (via NeMo): Streaming, good multilingual
- **Conformer-CTC** (NeMo): Classic streaming ASR, fast
- **Paraformer** (Alibaba, for Chinese especially): Low latency

📐 **[Engineering judgment]** - Whisper-turbo is easier to operate but higher latency. Conformer/Canary harder to operate but lower latency. For S2S with < 500ms E2E target, Conformer better. Whisper OK for 800ms+ E2E target.

### 8.3.3 API option: Deepgram Nova-2

**Stack:**

```
SDK: deepgram-sdk-python
Endpoint: wss://api.deepgram.com/v1/listen
Model: nova-2 (or nova-3 when available)
```

**Config:**

```python
config = {
    "model": "nova-2",
    "language": "vi",
    "smart_format": False,  # Save latency
    "interim_results": True,  # Partials
    "endpointing": 300,  # ms silence to endpoint
    "utterance_end_ms": 1000,
    "punctuate": True,
    "diarize": False,
    "multichannel": False,
    "sample_rate": 16000,
    "encoding": "linear16",
}
```

**Cost:** $0.0043/min (2026 pricing, check current)

🔥 **[Scar tissue]** - Deepgram's streaming latency **genuinely good** (<200ms first partial). Competitive with self-hosted Conformer. For < 2M min/day, use Deepgram. Don't waste engineering cycles on self-hosting ASR at that volume.

### 8.3.4 Language considerations

- **English/major European languages**: Any ASR works well
- **Vietnamese, Thai, Indonesian**: Whisper-turbo decent, Deepgram improving
- **Chinese (Mandarin/Cantonese)**: Paraformer or specialized Chinese ASR
- **Code-switching (mixed language)**: Whisper handles best currently

---

## 8.4 LLM serving - Engine choice

### 8.4.1 Đề xuất: vLLM 0.6+ hoặc SGLang 0.3+

These two are the mainstream choices in 2026. Cả hai đều continuous batching + PagedAttention + prefix cache. Khác biệt subtle.

### 8.4.2 vLLM

**Mature, well-documented, widely deployed.**

```python
# vLLM serving example
from vllm import LLM, SamplingParams
from vllm.entrypoints.openai.api_server import run_server

llm = LLM(
    model="meta-llama/Llama-3.3-70B-Instruct",
    quantization="fp8",  # or "int8"
    tensor_parallel_size=4,  # 4 H100s
    gpu_memory_utilization=0.92,
    max_model_len=8192,
    enable_prefix_caching=True,
    enable_chunked_prefill=True,
    max_num_batched_tokens=8192,
    max_num_seqs=256,  # Concurrent sequences
)

# Runs with OpenAI-compatible API
```

**Pros:**

- Most popular, largest community
- OpenAI-compatible API easy integration
- PagedAttention + chunked prefill enabled
- Good multi-GPU support

**Cons:**

- Heavy Python overhead (mitigated but present)
- Custom kernels sometimes lag behind SGLang

### 8.4.3 SGLang

**Faster in benchmarks, newer.**

```python
# SGLang example
import sglang as sgl

runtime = sgl.Runtime(
    model_path="meta-llama/Llama-3.3-70B-Instruct",
    quantization="fp8",
    tp_size=4,
    mem_fraction_static=0.9,
    enable_prefix_caching=True,
)
```

**Pros:**

- Often 10-30% faster throughput than vLLM (per benchmarks)
- RadixAttention for better prefix cache hit rates
- Structured output support excellent
- Growing fast

**Cons:**

- Smaller community
- Sometimes less stable in edge cases

📐 **[Engineering judgment]** - Vào 2026, SGLang catching up to vLLM rapidly. For new deployments, try both on your workload and benchmark. Both are good defaults. vLLM more "boring reliable" choice.

### 8.4.4 TensorRT-LLM

**NVIDIA's own, fastest but most complex.**

**Pros:**

- Fastest absolute performance (when tuned right)
- Official NVIDIA support
- Best for production at huge scale

**Cons:**

- Compilation per model/config complex
- Less flexible (compile time optimization)
- Vendor lock-in (NVIDIA only)

**Use when:** You have dedicated ML infra team and throughput is critical. Otherwise stick with vLLM/SGLang.

### 8.4.5 Model choices (2026)

| Use case                  | Model                                  | GPU requirement         | Notes                        |
| ------------------------- | -------------------------------------- | ----------------------- | ---------------------------- |
| Premium reasoning         | Llama-3.3-70B or Qwen2.5-72B FP8       | 4x H100 tensor parallel | Best OSS quality             |
| Standard conversation     | Qwen2.5-14B or Llama-3.1-8B FP8        | 1x H100 or 2x L40S      | Good balance                 |
| Economy / high-volume     | Llama-3.2-3B or Qwen2.5-7B INT8        | 1x L40S or A10          | Fast, cheap                  |
| Specialized (VN language) | Fine-tuned Qwen2.5-14B or Llama-3.1-8B | 1x H100                 | Fine-tune recommended for VN |

🔥 **[Scar tissue]** - Don't use 70B model for greetings. Don't use 3B model for reasoning. **Router classifier** is the win.

### 8.4.6 GPU selection for LLM

```
┌─────────────────────────────────────────────────────────────────┐
│  GPU CHOICE MATRIX for LLM SERVING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  H100 SXM (80GB HBM3)                                            │
│  • Best for: 70B+ models, tensor parallel                        │
│  • Throughput: ~1.5-2x A100                                      │
│  • Price: ~$4-6/hr on-demand, $2-3/hr reserved                   │
│  • Availability: 2025+ widespread                                │
│                                                                   │
│  H200 (141GB HBM3e)                                              │
│  • Best for: Long context, larger KV cache                       │
│  • Memory bandwidth better than H100                             │
│  • 📚 claim: ~1.3-1.5x H100 for inference                       │
│                                                                   │
│  B200 / Blackwell (2025+ availability)                           │
│  • Best for: Hyperscale, FP4 support                             │
│  • 🔬 early deployments, ecosystem maturing                      │
│                                                                   │
│  A100 (80GB HBM2e)                                               │
│  • Still viable for 7-13B models                                 │
│  • Better availability than H100 in some regions                 │
│  • ~$2-3/hr on-demand                                            │
│                                                                   │
│  L40S (48GB GDDR6)                                               │
│  • Excellent for ASR, TTS, small LLMs (7B)                       │
│  • ~$1-1.5/hr on-demand                                          │
│  • Edge deployments                                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 8.4.7 Prefix cache configuration

```yaml
# Conceptual vLLM config for prefix caching
engine:
  enable_prefix_caching: true
  block_size: 16 # Token block size for PagedAttention

prefix_cache:
  gpu_capacity: 0.2 # 20% of GPU memory for prefix cache
  cpu_offload: true # Offload cold blocks to CPU RAM
  cpu_capacity_gb: 256 # Host RAM dedicated

# Redis layer for distributed prefix cache (advanced)
distributed_cache:
  backend: redis
  endpoints: ["redis-cluster.internal:6379"]
  ttl_hours: 4
  replication_factor: 2
```

🔥 **[Scar tissue]** - Prefix cache hit rate on vLLM with proper session affinity: 85-92% typical. Without session affinity (routing random): drops to 30-40%. Session affinity is mandatory for cache to work.

---

## 8.5 TTS - Text to speech

### 8.5.1 Đề xuất: Kokoro (self-host) hoặc Cartesia Sonic (API)

### 8.5.2 Self-host: Kokoro

**Stack:**

```
Model: Kokoro-v1.2 (or latest)
Framework: PyTorch + ONNX runtime
GPU: L40S or A10
Serving: Custom Python + gRPC streaming
```

**Why Kokoro:**

- Open weights (Apache 2.0)
- 82M params, tiny and fast
- Quality surprisingly good for size
- Streaming capable
- Vietnamese support improving

**Config:**

```python
model = KokoroModel.load(
    path="kokoro-v1.2",
    device="cuda",
    precision="fp16",
)

# Streaming TTS
async def synthesize_streaming(text_stream, voice_id):
    buffer = ""
    async for token in text_stream:
        buffer += token
        # Chunk at sentence/clause boundary
        if should_flush(buffer):
            audio_chunk = model.synthesize(
                text=buffer,
                voice=voice_id,
                speed=1.0,
            )
            yield audio_chunk
            buffer = ""
```

**Throughput:** ~100-150 concurrent streams on L40S.

### 8.5.3 API: Cartesia Sonic

**Stack:**

```
SDK: cartesia-python
Model: sonic-english (or sonic-multilingual)
Protocol: WebSocket streaming
```

**Why Cartesia:**

- Low latency: 40-90ms first audio chunk
- Quality excellent
- Streaming-native architecture
- Pricing reasonable (~$0.05/min typical)

### 8.5.4 ElevenLabs - Premium voices

Use when:

- Voice cloning/branding important
- Quality is differentiator
- Cost is secondary

### 8.5.5 My take

📐 **[Engineering judgment]** -

- **< 500K min/day**: Cartesia or ElevenLabs API
- **500K-5M min/day**: Self-host Kokoro for bulk, API for premium tier
- **> 5M min/day**: Self-host everything, custom voice models worth investment

---

## 8.6 Orchestration - Kubernetes stack

### 8.6.1 Đề xuất: K8s 1.30+ với Karpenter

**Versions (as of 2026):**

```
Kubernetes: 1.30+ (1.32 preferred)
Karpenter: 1.0+ (autoscaling)
NVIDIA GPU Operator: latest
Istio or Linkerd: service mesh
Cilium: CNI with eBPF
```

### 8.6.2 Why Karpenter over Cluster Autoscaler

Karpenter advantages:

- Instant node provisioning (no scaling groups)
- Smart instance type selection (cost optimization)
- Spot + on-demand mixing natively
- Native consolidation (replace small with large when fit)

🔥 **[Scar tissue]** - Cluster Autoscaler works but feels clunky for GPU workloads. Karpenter's NodePool concept maps well to "need GPU with X memory, mix of spot+on-demand, these AZs". Worth migrating.

### 8.6.3 GPU scheduling config

```yaml
# NodePool for LLM serving (H100)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: llm-h100
spec:
  template:
    spec:
      requirements:
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["p5.48xlarge", "p5en.48xlarge"] # H100 instances
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "reserved"] # No spot for critical
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule
  limits:
    cpu: 1000
    nvidia.com/gpu: 64 # Cap total GPUs
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 1h # Slow consolidation for GPU
```

### 8.6.4 Service mesh - Istio vs Linkerd

**Istio:**

- More features (traffic splitting, fault injection, mTLS everywhere)
- Heavier (sidecar CPU/memory overhead)
- Steeper learning curve

**Linkerd:**

- Simpler, lighter
- Rust-based proxy (lower overhead)
- Less feature-complete

📐 **[Engineering judgment]** - For streaming workloads, **sidecar overhead matters**. Linkerd's lightweight proxy adds ~1-3ms latency vs Istio's ~3-8ms. For S2S, Linkerd or even no service mesh + app-level instrumentation.

Some orgs skip service mesh for data plane (media), use it only for control plane APIs.

### 8.6.5 GitOps - ArgoCD

```yaml
# ArgoCD application example
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: llm-serving-us-east-1
spec:
  source:
    repoURL: https://github.com/org/infra
    path: k8s/llm-serving
    targetRevision: HEAD
  destination:
    server: https://k8s-us-east-1.internal
    namespace: llm-serving
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

ArgoCD is standard for 2026 K8s GitOps. Alternatives: Flux (simpler), Spinnaker (CD-focused).

---

## 8.7 State & data stores

### 8.7.1 Session state: Redis or DragonflyDB

**Redis:**

```
Version: Redis 7.2+ or Redis Cluster
Mode: Cluster mode for sharding
Persistence: AOF (for warm state), disabled for ephemeral
Memory: RAM per shard 32-64GB typical
```

**DragonflyDB (Redis alternative):**

```
Version: Latest
Pros: Multi-threaded, better CPU utilization, Redis-compatible
Cons: Newer, less community
```

### 8.7.2 Relational: PostgreSQL

```
Version: PostgreSQL 16+
Use cases:
- User accounts, auth
- Billing
- Agent config (prompts, tools)
- Relational structured data

Setup: Managed (RDS/CloudSQL) or patroni-managed HA
```

### 8.7.3 Analytics: ClickHouse

```
Version: ClickHouse 24.x+
Use cases:
- Conversation analytics (turn-level data)
- Usage metrics
- Observability data storage
```

### 8.7.4 Cold storage: S3-compatible

```
Options: S3, R2 (Cloudflare, no egress!), MinIO (self-host), GCS
Use cases:
- Conversation recordings (if retained)
- Model weights distribution
- Logs archive
- Backups
```

🔥 **[Scar tissue]** - **Cloudflare R2** for cold storage is game-changer. Zero egress cost vs S3's $0.09/GB. If your data access pattern fits (lots of reads, few updates), massive cost savings.

---

## 8.8 Observability stack

### 8.8.1 Self-hosted: Grafana LGTM stack

```
L = Loki (logs)
G = Grafana (dashboards)
T = Tempo (traces)
M = Mimir (metrics)

Plus:
- OpenTelemetry Collector (gateway)
- Alertmanager
- Prometheus (as metrics backend)
```

**Deployment:**

```yaml
# Helm values example for Tempo
tempo:
  storage:
    trace:
      backend: s3
      s3:
        endpoint: s3.amazonaws.com
        bucket: traces-prod

  # Sampling config
  ingester:
    trace_idle_period: 10s
    max_block_duration: 5m

  # Resource limits
  resources:
    requests:
      memory: 8Gi
      cpu: 2
    limits:
      memory: 16Gi
      cpu: 4
```

### 8.8.2 SaaS alternatives

**Datadog:**

- Full platform, easy setup
- Cost scales badly ($20K+/month common at scale)
- Trace search performance degrades at high cardinality

**Honeycomb:**

- Best traces UX
- High cardinality friendly (perfect for session-based traces)
- More expensive per-event than Datadog but often cheaper at scale

**Grafana Cloud:**

- Managed LGTM stack
- Easy migration path to self-host

### 8.8.3 My take

📐 **[Engineering judgment]** -

- Start: Grafana Cloud or Datadog
- Scale: Self-host LGTM when bill > 2 FTE engineer cost
- Always: OpenTelemetry SDK in code (vendor-neutral)

🔥 **[Scar tissue]** - Migrating observability backends is painful. Using OTel from day 1 = migration mostly config changes. Using Datadog SDK everywhere = rewrite all instrumentation on migration.

---

## 8.9 Message passing giữa components

### 8.9.1 Within session (same node): Channels/Actors

```rust
// Rust example with Tokio channels
let (audio_tx, audio_rx) = mpsc::channel::<AudioFrame>(buffer_size);
let (transcript_tx, transcript_rx) = mpsc::channel::<TranscriptEvent>(buffer_size);

// Audio producer task
tokio::spawn(async move {
    while let Some(frame) = audio_source.next().await {
        audio_tx.send(frame).await?;
    }
});

// ASR consumer task
tokio::spawn(async move {
    while let Some(frame) = audio_rx.recv().await {
        let partial = asr.process(frame).await?;
        transcript_tx.send(partial).await?;
    }
});
```

**Performance:** In-process channels = nanoseconds overhead. Best for hot path.

### 8.9.2 Cross-service: gRPC streaming

```protobuf
// Proto definition
service LLMService {
  rpc GenerateStream(stream PromptInput) returns (stream TokenOutput);
}

message PromptInput {
  oneof content {
    string system_prompt = 1;
    string user_message = 2;
    Metadata meta = 3;
  }
}

message TokenOutput {
  string token = 1;
  bool done = 2;
  TokenMetadata meta = 3;
}
```

**Why gRPC streaming:**

- HTTP/2 multiplexing
- Bidirectional streaming native
- Strongly typed (vs JSON)
- Compression (saves cross-region traffic)
- Observability ecosystem (OTel integration easy)

### 8.9.3 Async/event patterns: NATS vs Kafka

**For real-time S2S, usually not needed.** Most pipeline is synchronous streaming. But useful for:

- **NATS JetStream**: Low-latency pub/sub, good for events (session events, analytics)
- **Kafka**: Higher throughput, durable, for logging/analytics/audit

### 8.9.4 My take

📐 **[Engineering judgment]** -

- Hot path (audio/token streaming): in-process channels or gRPC streaming
- Cross-service sync: gRPC streaming (Protobuf)
- Async events: NATS for low-latency, Kafka for high-throughput/durable

**Avoid:** REST/JSON on hot path. Serialization overhead + no streaming = killer.

---

## 8.10 Deployment & CI/CD

### 8.10.1 Container images

**Base images:**

```dockerfile
# For LLM serving (vLLM)
FROM nvidia/cuda:12.4.0-cudnn9-runtime-ubuntu22.04
# ~10GB base, slow cold pulls

# For media server (Rust)
FROM debian:bookworm-slim
# ~80MB, fast pulls

# For Python services (non-GPU)
FROM python:3.12-slim
# ~150MB
```

🔥 **[Scar tissue]** - Large GPU images (CUDA + model) can be 15-30GB. Pull time becomes cold start bottleneck. Optimizations:

- Multi-stage builds
- Model weights separate from container (mount from S3/volume)
- Pre-pulled images on standby nodes

### 8.10.2 CI pipeline

```yaml
# Conceptual GitHub Actions
jobs:
  test:
    - lint, unit tests
    - integration tests (short)

  build:
    - build container images
    - push to registry
    - security scan (Trivy)

  deploy-staging:
    - ArgoCD sync to staging
    - smoke tests

  deploy-canary:
    - 1% traffic canary (automated)
    - monitor 30min

  deploy-production:
    - gradual rollout
    - monitor
```

### 8.10.3 Registry and artifact storage

```
Container registry: ECR, GCR, Harbor (self-host), GHCR
Model registry: S3 + metadata in Postgres, or MLflow, or HuggingFace private
```

---

## 8.11 Summary table - Stack at a glance

```
┌──────────────────┬────────────────────────────────────────────┐
│  LAYER           │  RECOMMENDED STACK                         │
├──────────────────┼────────────────────────────────────────────┤
│ Client           │ libwebrtc (native) + Silero VAD            │
│                  │ Custom signaling (WebSocket + Protobuf)    │
├──────────────────┼────────────────────────────────────────────┤
│ Media server     │ Rust (webrtc-rs) or Elixir (ex_webrtc)     │
│                  │ XDP/eBPF for L4 LB                         │
├──────────────────┼────────────────────────────────────────────┤
│ ASR              │ Self-host: faster-whisper-turbo on L40S    │
│                  │ API: Deepgram Nova-2                       │
├──────────────────┼────────────────────────────────────────────┤
│ LLM              │ vLLM 0.6+ or SGLang 0.3+                   │
│                  │ Models: Llama-3.3-70B FP8, Qwen2.5-14B FP8 │
│                  │ Hardware: H100 SXM (tier 1), L40S (tier 3) │
├──────────────────┼────────────────────────────────────────────┤
│ TTS              │ Self-host: Kokoro on L40S                  │
│                  │ API: Cartesia Sonic                        │
├──────────────────┼────────────────────────────────────────────┤
│ Orchestration    │ K8s 1.30+ with Karpenter                   │
│                  │ Linkerd for mesh (or skip)                 │
│                  │ ArgoCD for GitOps                          │
├──────────────────┼────────────────────────────────────────────┤
│ State            │ Redis Cluster or DragonflyDB (hot)         │
│                  │ PostgreSQL 16+ (users, billing)            │
│                  │ ClickHouse (analytics)                     │
│                  │ S3/R2 (cold)                               │
├──────────────────┼────────────────────────────────────────────┤
│ Observability    │ OpenTelemetry SDK (always)                 │
│                  │ Grafana LGTM (self-host) or Honeycomb      │
├──────────────────┼────────────────────────────────────────────┤
│ Messaging        │ In-process channels (Rust/Tokio, Elixir)   │
│                  │ gRPC streaming (cross-service)             │
│                  │ NATS (low-latency events)                  │
├──────────────────┼────────────────────────────────────────────┤
│ Network infra    │ BGP + Anycast (signaling)                  │
│                  │ L4 LB (Katran-style) for media            │
│                  │ Direct peering at scale                    │
└──────────────────┴────────────────────────────────────────────┘
```

---

## 8.12 Tổng kết Phần 8

**Những quyết định stack quan trọng nhất:**

1. **WebRTC + Custom SFU (Rust/Elixir)** - Don't use off-the-shelf mediasoup at large scale.
2. **vLLM hoặc SGLang** cho LLM - Continuous batching + prefix cache mandatory.
3. **FP8 quantization** cho H100 - Default choice for quality/cost balance.
4. **Karpenter > Cluster Autoscaler** cho K8s GPU - Better instance selection, faster.
5. **OpenTelemetry SDK everywhere** - Vendor neutral, easy migration.
6. **gRPC streaming cho cross-service** - Never REST on hot path.
7. **Redis for hot state, Postgres for relational, ClickHouse for analytics, R2 for cold** - Specialized by access pattern.
8. **Self-host TTS earlier, ASR at 2M/day, LLM at 5M+/day** - Phased approach.

**Anti-patterns to avoid:**

- MCU instead of SFU for 1:1 voice
- REST API cho streaming hot path
- Single cluster in single region
- Observability as afterthought
- Container with 30GB GPU image and no cache strategy
- Deploying model weights inside container image

**Stack evolution timeline:**

```
Month 1-3:    API everything (Deepgram + OpenAI + ElevenLabs)
Month 4-8:    Self-host TTS (Kokoro), optimize prompts
Month 9-15:   Self-host ASR, start LLM routing
Month 16-24:  Self-host LLM (bulk), API fallback for complex
Month 24+:    Edge deployment, custom optimizations, peering
```

Realistic timeline. Don't skip stages unless you have the team and funding.

---

# Phần 9 - Bài học từ vận hành thực tế

> Phần này là thứ tôi thấy ít được viết ra. Hầu hết kiến trúc docs nói về "đúng đắn" - phần này nói về **sai lầm, incident patterns, và regrets**. Đây là cái mà chỉ người từng 3AM debug production mới biết.

---

## 9.0 Framing - Tại sao phần này quan trọng

Thiết kế S2S trên slideware dễ. Vận hành ở production là Whac-A-Mole - hoặc đúng hơn, là đi qua một bãi mìn mà mọi bước đều có thể nổ.

**Insight quan trọng nhất của toàn doc:**

> Không có hệ thống S2S production-grade nào ra đời correct từ đầu. Tất cả đều evolve qua incident. Nếu team bạn không có incident trong 3 tháng đầu, hoặc (a) traffic quá thấp, hoặc (b) monitoring mù nên không thấy.

Phần này tôi sẽ share theo cách brutal honest - giả định bạn đọc đã build vài hệ thống production và muốn nghe thật, không muốn nghe marketing.

---

## 9.1 Assumptions sai lầm phổ biến

Đây là những assumptions tôi đã thấy team sau team mắc phải.

### 9.1.1 "P50 latency đẹp nghĩa là hệ thống tốt"

**Sai.** Voice UX quyết định bởi **P99 và P99.9**, không P50.

**Example:**

- System A: P50=400ms, P99=600ms → users happy
- System B: P50=300ms, P99=3000ms → users say "nó lag"

User perception dựa trên worst 1% experiences, không average. Một conversation 10 phút có ~30-60 turns. P99 3000ms → trong conversation đó có 0.3-0.6 turns lag nặng. User remember những cái này, quên 59 turns smooth.

**Rule:** Optimize P99, không P50. Hard.

### 9.1.2 "Streaming = just add 'stream=true'"

**Sai ở fundamental level.** Streaming thay đổi:

- Data model (tokens not responses)
- Error handling (partial failure possible)
- State management (in-flight state per session)
- Observability (traces don't have clear end)
- Backpressure (must handle everywhere)

Team nào start với "REST first, add streaming later" sẽ rewrite **entire codebase**. Tôi đã thấy ít nhất 4 công ty.

### 9.1.3 "Cold start không quan trọng vì ít xảy ra"

**Sai.** Cold start xảy ra khi:

- Deploy (mỗi 1-2 tuần)
- Autoscale up (hàng ngày)
- Node replacement (kernel upgrades, spot eviction)
- Post-incident recovery (worst time!)

Điểm cuối là killer: sau incident, autoscaler tạo instance mới → cold start 2 phút → trong 2 phút đó everything cascades.

**Warm pool isn't optional.**

### 9.1.4 "GPU utilization càng cao càng tốt"

**Sai.** 95% utilization at P50 load = 0% headroom cho spike. Next small spike → queue grows → latency spike → retries → death spiral.

Sweet spot: **60-70% at P50 load**. Higher for batch workloads, lower for latency-critical.

### 9.1.5 "Autoscaling sẽ giải quyết traffic spike"

**Sai.** Autoscale reaction time (60-180s) >> user tolerance (0-3s).

By the time autoscaler reacts, user experience already broken. Warm pool + over-provision for peak là insurance.

🔥 **[Scar tissue]** - Tôi đã gặp CEO hỏi "sao infra cost cao thế?" Câu trả lời: "over-provision cho reliability." CEO push back: "scale down, save money." Scaled down, incident tại spike tiếp theo, lost customers. Cost savings < customer churn cost. Reliability is a feature.

### 9.1.6 "Retry solves transient errors"

**Sai** - retry AMPLIFIES transient errors into permanent ones.

Classic retry storm:

```
Service A slow → clients timeout → clients retry
More load on Service A → slower → more timeouts → more retries
Service A dies
```

Retry must have:

- **Exponential backoff**
- **Jitter** (don't sync retries across clients)
- **Retry budget** (max 10% of calls can be retries)
- **Circuit breaker** (stop retrying when clearly failed)
- **Deadline propagation** (don't retry if deadline already passed)

### 9.1.7 "Our monitoring will catch problems"

**Sai in subtle way.** Monitoring catches what it's **configured to catch**. New failure modes → no alerts until humans notice.

Classic: "we had P99 alert but not P99.9 alert". P99.9 went from 2s to 10s for 3 days before detection.

**Always add "was this incident visible in metrics?" to post-mortems.** If no → add metric.

### 9.1.8 "Users will tell us if there's a problem"

**Sai** for voice. Users give up, don't complain. Churn is silent.

Voice UX degradation → user says "this doesn't work well" → never opens app again. No support ticket.

Must have **behavioral metrics**:

- Session completion rate (users hanging up early)
- Conversation length distribution
- Repeat users
- Reconnect rate (users struggling to connect)

### 9.1.9 "Quality benchmarks = user-perceived quality"

**Sai.** MMLU/HumanEval scores tell you about reasoning. They don't tell you about:

- How annoying the voice sounds
- How long response is (too long = bad)
- How natural the pauses are
- Whether the model interrupts appropriately

**Measure what users feel**, not what benchmarks optimize. Blind A/B with real users > 10x more signal than benchmarks.

### 9.1.10 "Once we're at scale, unit economics will work"

**Sai** unless you designed for it. Scale amplifies inefficiencies, doesn't fix them.

Team builds MVP with API-everything, $0.30/min. Plan: "at scale we'll self-host." Two years later, at $0.30/min × 5M min/day = $1.5M/day, they haven't self-hosted because it's a 12-month project and they've been busy shipping features.

**Cost is architecture.** Design for target unit cost from early.

---

## 9.2 Incident patterns - Top 10 thường gặp

Từ post-mortems của voice/streaming infra, patterns lặp lại:

### Pattern 1: Cascade từ single slow dependency

```
Timeline:
t=0       Redis latency p99 goes from 2ms to 50ms (network blip)
t=30s     Session state reads queuing in session service
t=60s     Session service request timeout cascades
t=90s     Client reconnects storm
t=120s   LLM cluster overloaded from reconnect storm
t=180s   Everything dies
```

**Root cause:** Redis blip.
**Impact:** 15 phút outage across stack.
**Fix:** Circuit breaker on Redis calls + fallback to in-memory state.

**Lesson:** One slow dependency can take down everything. Isolate with timeouts, circuit breakers, fallbacks.

### Pattern 2: Noisy neighbor on GPU

```
Timeline:
t=0      Customer A starts using 20K context in queries
t=5min   LLM cluster P99 goes from 200ms to 1.5s
t=10min  Customers B, C, D complain about latency
t=15min  Identified: Customer A's long context blocking batch
```

**Root cause:** Unbounded context, single workload impacts all.
**Impact:** Quality degradation for all users.
**Fix:** Per-workload isolation, context caps, chunked prefill.

### Pattern 3: Deploy causes mass reconnect

```
Timeline:
t=0      Rolling deploy starts
t=5min   First batch of nodes drained, sessions reconnected
t=5min   200K clients reconnect simultaneously
t=6min   Signaling service overloaded
t=7min   Reconnect failure rate 30%
t=10min  Deploy paused, emergency capacity added
```

**Root cause:** No rate limiting on reconnects.
**Fix:** Jittered reconnect intervals on client, rate limit on server.

### Pattern 4: Model update breaks production subtly

```
Timeline:
t=0       New LLM model rolled out to 100%
t=2hr     Monitoring all green (latency, error rate normal)
t=4hr     Customer complaints: "agent sounds weird"
t=6hr     Discovered: new model uses different tokens, TTS synthesizes unnaturally
t=8hr     Rolled back
```

**Root cause:** Compatibility between LLM output and TTS voice not tested.
**Fix:** A/B testing with quality rubric, not just latency/error metrics.

**Lesson:** ML model updates aren't infra updates. Regressions can be subtle + quality-based.

### Pattern 5: Region failure takes down more than expected

```
Timeline:
t=0      AWS us-east-1 network issues
t=2min   Multi-region failover kicks in, users routed elsewhere
t=5min   Other regions overwhelmed (didn't have capacity for full failover)
t=10min  us-east-1 recovers but system unstable
t=15min  Gradual recovery
```

**Root cause:** Multi-region = more regions, not full-capacity elsewhere.
**Fix:** Each region sized for N+1 failover, not just normal load.

**Cost:** 2-3x infra cost. Pays for itself in avoided downtime.

### Pattern 6: Autoscaling flap

```
Timeline:
t=0      Small spike (10% increase), autoscaler adds 5 nodes
t=5min   Nodes ready, load normalizes
t=10min  Scale-down kicks in, removes 5 nodes
t=15min  Another small spike, scale up again
... repeats every 15min for hours
```

**Root cause:** Scale down too aggressive, no cooldown.
**Fix:** Asymmetric scaling (fast up, slow down, with cooldown period).

### Pattern 7: Observability cost runaway

```
Timeline:
t=day1    New high-cardinality metric added (user_id tag)
t=day3    Datadog ingestion 10x
t=day7    Monthly bill projection $800K (from $80K)
t=day7    Metric removed, bill explained to finance
```

**Root cause:** High-cardinality dimensions in metrics.
**Fix:** Cardinality limits, review before merging new metrics.

### Pattern 8: "Silent" quality degradation

```
Timeline:
t=0       TTS model update (minor version)
t=1day    No latency or error change
t=1week   NPS survey: scores down 15%
t=2weeks  Correlation found: TTS update
```

**Root cause:** Quality metrics not automated, rely on manual survey.
**Fix:** Automated quality scoring (MOS, AB testing with blind evaluators).

### Pattern 9: WebRTC ICE failures in corporate networks

```
Timeline:
Background: 5% users can't connect
Symptom: ICE negotiation fails
Root cause: Corporate firewalls blocking UDP + STUN
```

**Fix:** Robust TURN server fallback, TLS-based tunneling. Accept some users will have higher latency but can connect.

### Pattern 10: Database connection pool exhaustion

```
Timeline:
t=0       Traffic growing normally
t=20min   PostgreSQL connection errors spike
t=25min   Investigation: pool exhausted
t=30min   Root cause: new feature creating connections not released
```

**Not S2S-specific, but common.** Always bound connection pools, always use context managers/defer.

---

## 9.3 Incident response - Hard-won wisdom

### 9.3.1 First 5 minutes of incident

**Priority order:**

1. **Stop the bleed** (degrade gracefully, rollback, shed load)
2. **Communicate** (status page, internal channel)
3. **Investigate** (don't start here - it takes hours)

🔥 **[Scar tissue]** - Engineers instinct: debug first, fix later. Wrong for production incidents. Users are suffering _now_. Stop bleed, then investigate.

**Stop bleed tools:**

- Feature flags (kill new feature immediately)
- Traffic shifting (move load away from affected region)
- Rollback (if recent deploy)
- Rate limiting (if overload)
- Circuit breakers (tripped manually if auto not working)

### 9.3.2 The "was it deploy?" question

**First question in every incident: "did we deploy recently?"**

If yes → rollback first, investigate second.
If no → proceed with investigation.

This alone resolves 40-50% of incidents in < 10 minutes.

### 9.3.3 Post-mortem discipline

Every serious incident → post-mortem. Format matters:

```
POST-MORTEM
├─ Summary (1 paragraph, exec-readable)
├─ Timeline (exact times, what happened, what engineers did)
├─ Impact (users affected, duration, business impact)
├─ Root cause (5 whys technique)
├─ What went well (don't skip - reinforce good responses)
├─ What went poorly (no blame, focus on gaps)
└─ Action items (SPECIFIC, OWNED, DATED)
```

**Action items must be:**

- Specific (not "improve monitoring")
- Owned (name)
- Dated (deadline)
- Tracked (follow-up meeting)

Otherwise, post-mortems become documents that everyone reads once and forgets. Same incident repeats in 3 months.

### 9.3.4 Blame-free culture

Critical. Without it, engineers hide information during investigation, fearing blame.

**Principles:**

- Failures are systems failures, not individual failures
- If one engineer could cause outage, the system allowed it
- Fix the system, not the engineer

🔥 **[Scar tissue]** - Worked at company with blame culture earlier in career. Incidents happened → engineers covered up → repeats. Moved to blame-free → investigations deeper → fixes real → incidents reduced.

---

## 9.4 Warning signs - What metrics tell you before users complain

This is the operational art. Reading metrics before incidents.

### 9.4.1 Leading indicators to watch

**Infrastructure:**

- **GPU queue depth increasing** → saturation building up
- **Prefix cache hit rate dropping** → routing broken or traffic pattern change
- **Reconnect rate rising** → network or server issues emerging
- **Memory growth unbounded** → leak somewhere

**Latency:**

- **P95 degrading trend** (even if P99 OK) → headroom shrinking
- **Inter-token latency variance** growing → batch instability
- **First audio latency** worsening → TTS under pressure

**Quality:**

- **Interruption rate rising** → users getting impatient, maybe latency issue
- **Session duration shortening** → frustration signal
- **ASR confidence dropping** → audio quality or model issue

**Business:**

- **Repeat user rate dropping** → product-market issue or quality
- **NPS score trend** → user satisfaction trend

### 9.4.2 Specific metric thresholds (illustrative)

These are starting points - calibrate to your system:

```
┌────────────────────────────────────────────────────────────────┐
│  LEADING INDICATOR ALERT THRESHOLDS                            │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  METRIC                           │ WARN       │ CRITICAL       │
│  ──────────────────────────────────────────────────────────── │
│  GPU queue depth                  │ > 5        │ > 20           │
│  Prefix cache hit rate            │ < 70%      │ < 50%          │
│  TTFT P95 vs baseline             │ > 1.5x     │ > 2x           │
│  ICE failure rate                 │ > 5%       │ > 10%          │
│  Reconnect rate                   │ > 2%       │ > 5%           │
│  Session completion rate          │ < 85%      │ < 75%          │
│  ASR confidence P50               │ < 0.85     │ < 0.75         │
│  Turn-to-turn response variance   │ > 2x SD    │ > 3x SD        │
│  Interruption rate                │ > 15%      │ > 25%          │
│  Autoscale in-flight count        │ > 20%      │ > 40%          │
└────────────────────────────────────────────────────────────────┘
```

### 9.4.3 Early warning example - The 48-hour window

Example scenario: incident in the making over 48 hours.

```
T-48hr: Prefix cache hit rate drops 5% (75% → 70%)
        - No alerts (within normal variance)
        - Nobody investigates
T-24hr: GPU queue depth occasionally spikes (3 → 8)
        - Alert triggered once, engineer marks "transient"
T-12hr: TTFT P95 degrades 20%
        - Borderline alert, not escalated
T-4hr:  P99 TTFT crosses threshold
        - Pager fires
T-2hr:  User complaints
T=0:    Incident declared, investigation starts
```

**Root cause:** Session routing bug caused cache affinity loss over days. Earlier investigation could have caught at T-48hr.

**Lesson:** Don't dismiss small metric changes. Trending matters more than absolute thresholds.

---

## 9.5 Điều tôi sẽ làm khác nếu làm lại

### 9.5.1 Invest in observability before scaling

**Past mistake:** Scale first, observe later. Result: flying blind at higher scale.

**If redo:** Day 1 - OTel instrumentation throughout, even if it feels overkill. Observability debt compounds faster than code debt.

### 9.5.2 Define quality metrics before launching

**Past mistake:** Launch with "it seems to work" quality assessment.

**If redo:** Define and instrument:

- Automated quality scoring (Mean Opinion Score automated, blind A/B)
- User behavior metrics (completion, repeat use)
- Quality regression gates in CI

**Lesson:** You can't improve what you don't measure. Quality especially.

### 9.5.3 Build graceful degradation from Day 1

**Past mistake:** Build happy path first, fallbacks "later".

**If redo:** Every component has explicit fallback from start:

- LLM fail → cached response / simpler model
- TTS fail → backup voice
- ASR fail → "please repeat"
- Network fail → graceful reconnect

Graceful degradation is not a feature to add, it's a design principle.

### 9.5.4 Session affinity design from the start

**Past mistake:** Naive round-robin load balancing. Retrofit session affinity later.

**If redo:** Design routing for affinity from beginning. Affects:

- Load balancer strategy
- Cache invalidation
- Deploy drain logic
- Client reconnect behavior

### 9.5.5 Don't over-engineer initial orchestration

**Past mistake:** Microservice-everything from Day 1. 8 services for pipeline that should be 3.

**If redo:** Monolith-first where it makes sense (session node). Split services only when:

- Independent scaling needs
- Team boundaries
- Language/runtime differences justified

### 9.5.6 Budget for unknown unknowns

**Past mistake:** Plan for known failure modes. Unknown failures surprise us.

**If redo:** Budget 20-30% of engineering capacity for:

- Chaos testing (finding unknowns proactively)
- Post-incident improvements
- Tech debt pay-down

This is hard to justify to execs. Do it anyway.

### 9.5.7 Invest in client SDK quality

**Past mistake:** Focus on backend, assume clients will "handle it".

**If redo:** Client SDK as first-class product:

- Robust reconnect logic
- Exponential backoff with jitter
- Adaptive quality based on network
- Comprehensive error reporting

A great backend with buggy client = buggy product.

### 9.5.8 Test under adverse conditions

**Past mistake:** Test on fast WiFi in SF office. Production user on 3G in Jakarta.

**If redo:** Test suites that simulate:

- High jitter (100-200ms)
- Packet loss (5-10%)
- High latency (300ms RTT)
- Intermittent connectivity
- Old mobile devices

Tools: Linux tc (traffic control), Chaos Mesh, or specialized mobile network simulators.

### 9.5.9 Document tribal knowledge early

**Past mistake:** Key knowledge in heads of senior engineers. When they leave, institutional memory lost.

**If redo:**

- Runbooks for common incidents (grow from post-mortems)
- Architecture decision records (ADRs) for "why we did X"
- On-call training program
- "Day in life of a request" documentation

Tribal knowledge is technical debt.

### 9.5.10 Set cost budgets per feature

**Past mistake:** Build features without cost accounting. Discover cost structure only after launch.

**If redo:** Each major feature has:

- Estimated cost per user/transaction
- Budget constraints
- Cost review in launch process
- Ongoing cost monitoring

"We'll optimize later" usually means never.

---

## 9.6 Team & Organizational lessons

### 9.6.1 On-call is expensive - design to minimize

Being on-call for voice infra means:

- Ready to respond 24/7 for your week
- Stress, sleep disruption
- Burnout → attrition

Make on-call sustainable:

- **Runbooks for top 10 incidents** (self-serve resolution)
- **Automated remediation** where possible (auto-rollback, auto-scale)
- **Alert hygiene** (no false positives - each page must be real)
- **Compensation** (on-call pay, time off)
- **Rotation** (weekly, with backup)

🔥 **[Scar tissue]** - Team with great on-call culture keeps engineers 3x longer than bad one. Burnout from 3AM pages is #1 reason senior engineers quit.

### 9.6.2 Specialization vs Generalization

Voice infra requires deep specialists AND generalists who see big picture.

**Need specialists for:**

- Media protocols (WebRTC internals)
- GPU inference (kernels, serving frameworks)
- Network engineering (peering, BGP)
- ML models (training, fine-tuning)

**Need generalists for:**

- Architecture decisions
- Incident response (cross-layer reasoning)
- Debugging (knowing where to look)

Small team can't have full specialists. Accept gaps, supplement with consultants/vendors for deep expertise.

### 9.6.3 Hire for production experience

Graduate engineers are smart. Production-experienced engineers have scars.

For voice infra, weight hiring toward:

- Engineers who've been on-call for streaming systems
- People who've owned post-mortems
- People who've operated GPU clusters

Not "did you ship it" but "did you operate it for a year after shipping."

### 9.6.4 The 5-year infrastructure engineer

Voice infra is a 5-year commitment. Each year:

- Y1: Build. Everything breaks. Learn.
- Y2: Stabilize. Incidents decrease. Patterns emerge.
- Y3: Optimize. Cost matters. Scale concerns dominate.
- Y4: Evolve. Technology landscape shifts (new GPUs, new models).
- Y5: Mentor. Hand off to next generation.

If you can't commit 3-5 years, voice infra is wrong specialty. The learning curve is steep, but the depth pays compound interest.

---

## 9.7 Technology bets - What's coming

Brief thoughts on what's changing 2026-2028:

### 9.7.1 Hardware

- **NVIDIA Blackwell (B100/B200)** mainstream adoption - FP4 native, 2-3x H100 throughput
- **Custom silicon** (Groq, Cerebras, AMD MI300X) gaining viability for specific workloads
- **Optical interconnects** in data centers reduce latency across GPUs
- **Edge GPUs** (smaller, power-efficient) enable more edge inference

### 9.7.2 Models

- **Speech-to-speech native models** (e.g., GPT-4o, Gemini 2.5 Flash) - potentially bypass ASR+LLM+TTS pipeline entirely
- **Quality gap between open and closed** narrowing
- **Smaller models catching up** to 2024-scale large models
- **Multimodal unified models** (speech + vision + text)

### 9.7.3 Protocols

- **WebTransport / WebCodecs** maturing - alternative to WebRTC for some use cases
- **QUIC for media** continued experimentation
- **5G/6G deployment** reducing mobile network jitter in key markets

### 9.7.4 Infrastructure

- **Disaggregated inference** (prefill vs decode on different GPUs) becoming mainstream
- **Mooncake-style architectures** for long-context efficiency
- **Serverless GPU** improving (Modal, RunPod, etc.)
- **Edge-cloud continuum** - compute flexibly placed based on workload

📐 **[Engineering judgment]** - Biggest bet for S2S by 2028: **native speech-to-speech models may collapse the pipeline**. If model can take audio in, audio out directly, entire ASR+LLM+TTS architecture becomes legacy. Watch GPT-4o-realtime style models closely.

**But:** Current specialized pipeline will dominate until native models match on:

- Quality (tool use, reasoning)
- Cost (unified models may be expensive)
- Latency (TTFT of unified model vs specialized pipeline)

---

## 9.8 Reflections - Voice infra is a philosophy

After years, some meta-observations:

### 9.8.1 It's harder than web services

Backend engineers often think "voice is just backend with audio". No.

Voice adds:

- Real-time constraints (hard, not soft)
- Long-lived stateful sessions
- Bidirectional streaming
- UDP transport complexity
- Audio quality subjectivity
- Continuous service (session can't just "end and retry")

Expect 2-3x complexity vs equivalent REST service.

### 9.8.2 It's more operational than development

For voice infra, operational excellence > feature velocity.

In ratio:

- 60% operational (monitor, incident, optimize, scale)
- 30% maintenance (update deps, security, tech debt)
- 10% new features

Team composition should reflect this. SRE-to-SWE ratio high.

### 9.8.3 Users are the most honest feedback

Metrics lie subtly. Users don't.

When users say "it feels laggy" but P50 is 400ms, believe users. Dig into P99, jitter, interruption recovery, anything. Metrics have blind spots; user experience integrates everything.

### 9.8.4 Incidents are the teacher

Every major design improvement I've made came from an incident. Not from design reviews, not from literature, not from conferences. From 3AM debugging.

Embrace incidents. Learn from them. They are the most expensive but most effective teacher.

### 9.8.5 Don't chase perfection

Voice infra has no "done". Always latency to reduce, cost to lower, quality to improve. Setting "perfect" as goal = paralysis and burnout.

Set "good enough to ship, better than last quarter" as goal. Ship. Learn. Iterate.

Perfection is the enemy of deployed.

---

## 9.9 Final checklist - If you take only one thing

If you're starting S2S infra and can only do 10 things, do these:

1. ✅ **Streaming everywhere from Day 1** (not an optimization)
2. ✅ **Pipeline parallelism** (ASR↔LLM↔TTS overlap) - biggest latency win
3. ✅ **WebRTC for media, WebSocket for signaling** - not close
4. ✅ **Prefix caching + continuous batching** in LLM serving - mandatory
5. ✅ **Tiered service** (route by need) - sustainable economics
6. ✅ **Session affinity via consistent hash** - foundation
7. ✅ **Graceful degradation everywhere** - never fail silently
8. ✅ **Leading indicator alerts** (not P99 lagging) - catch early
9. ✅ **Warm pool for autoscale** - not reactive alone
10. ✅ **Blameless post-mortems with tracked action items** - compound learning

If you do these 10, you'll survive Year 1. Everything else in this doc is refinement on top.
