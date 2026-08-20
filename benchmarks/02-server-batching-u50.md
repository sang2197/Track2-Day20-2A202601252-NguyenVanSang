# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.98 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 38147 |

Highest sampled value was **3.98 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width: **3.98 of 4 slots (99%)** -- decode engine gần như luôn full trong
suốt 60s load-50, xác nhận continuous batching thật sự đang gộp nhiều request vào chung
decode step, không phải serve tuần tự.

Effective concurrency (Little's Law = RPS x avg latency, tính từ `locust-50_stats.csv`:
3.65 req/s x 11.63s) ~= **42.5** -- cao hơn nhiều so với 3.98 busy slots. Hai con số này
**không mâu thuẫn**, vì chúng đo hai thứ khác nhau: `n_busy_slots_per_decode` bị chặn trần
ở `--parallel` (4) theo định nghĩa -- nó chỉ đếm request đang thực sự được GPU decode tại
một thời điểm. Effective concurrency đếm **toàn bộ** request đang "sống" trong hệ thống,
kể cả đang nằm hàng đợi. `requests_deferred` lên tới 45 trong log `make metrics` là bằng
chứng trực tiếp: phần lớn trong số ~42.5 request "concurrent" đó đang chờ, không đang chạy.

**Tôi tin cả hai**, nhưng cho hai câu hỏi khác nhau: busy-slots trả lời "server đang tính
bao nhiêu request cùng lúc" (compute-bound, trần cứng = 4), còn effective concurrency trả
lời "hệ thống đang giữ bao nhiêu request tổng cộng" (phần lớn là hàng đợi). Khoảng cách
4 vs 42.5 chính là dấu hiệu server đã bão hoà: decode engine đã chạm trần từ trước khi tải
lên tới 50 user, mọi request thêm vào chỉ xếp hàng chứ không được xử lý song song thêm.
