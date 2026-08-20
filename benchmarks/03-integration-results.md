# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.8 | 3225.1 | 3226.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2829.9 | 2829.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 3015.2 | 3015.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.3** ·
llm **3023.4** · total **3023.8**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece | Real hay stub? | Ghi chú |
|---|---|---|---|
| N16 Cloud/IaC | Stub | "localhost only" — chạy trên máy cá nhân, không có k8s/Compose |
| N17 Data pipeline | Stub | `TOY_DOCS` — list in-memory có sẵn, không có Airflow DAG |
| N18 Lakehouse | Stub | vẫn là `TOY_DOCS` dict, không có Delta/Iceberg table |
| N19 Vector + features | Stub | retrieve bằng keyword overlap, chưa nối vector index thật (không chạy `serve-embed`) |
| N20 Serving | **Real** | `llama-server` thật, chạy ở `:8095`, trả lời qua `/v1/chat/completions` |

Dominant stage là `llm` (100% của total) — **đúng như kỳ vọng**, vì N17-N19 đang stub
nên `embed` và `retrieve` gần như miễn phí (0.0ms và 0.3ms trung bình); toàn bộ thời gian
đổ vào gọi model thật.

**Quan sát bất ngờ đáng nói hơn nằm bên trong chính `llm`:** thời gian wall-clock đo được
ở client (`llm_ms`, trung bình 3023.4ms) lớn hơn nhiều so với tổng thời gian compute mà
chính `llama-server` tự báo cáo (`prefill` + `decode` cộng lại chỉ ~470-670ms mỗi query).
Khoảng cách này (~2.3-2.6s, khá đều nhau ở cả 3 query) không giải thích được bằng số token
xử lý — nó là overhead nằm ngoài phần model compute, nhiều khả năng từ phía client
(`httpx.post()` trong `pipeline.py` mở kết nối mới mỗi lần gọi, không giữ keep-alive) hoặc
một bước xử lý phía server chưa được đo (tokenize, áp chat template, hoặc chờ slot). Tôi
chưa xác định được chính xác nguyên nhân, nhưng đây là bằng chứng cho thấy **server
timings không tự động phản ánh đúng latency mà client thực sự nhận được** — muốn đo
goodput đúng phải đo ở phía client, không chỉ tin số server tự báo.

**Nếu phải giảm 2× latency của pipeline này**, việc đầu tiên tôi sẽ làm **không phải**
đổi quantization hay thread — mà là đi tìm và loại bỏ khoảng overhead ~2.3-2.6s kể trên,
vì bản thân phần compute thật của model chỉ chiếm ~15-20% tổng thời gian đo được. Nếu
đóng được gap đó, latency đã giảm hơn 4× mà chưa cần đụng tới model. Sau đó mới tới các
đòn bẩy quen thuộc: dùng `UD-Q2_K_XL` (nhanh hơn 1.13× ở §2) hoặc giảm `max_tokens`.
