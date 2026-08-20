# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 223 | 3.82 | 1600 | 2800 | 4400 | 6.6 | 0.0% |
| 50 | 212 | 3.65 | 13000 | 14000 | 14000 | 42.5 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.96x** (19% of linear) |
| P95 latency | **5.00x** |
| Effective concurrency at 50 users | 42.5 vs `--parallel 4` slots (occupancy/slot ratio 10.62) |

**Saturated.** Throughput delivered only 0.96x for 5x the offered load, and effective concurrency (42.5) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.96x while P95 moved 5.00x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server bão hoà ở khoảng 10 user trở xuống, không phải đợi tới 50. Bằng chứng rõ nhất:
tải tăng gấp 5 lần (10 → 50 user) nhưng throughput gần như không đổi (RPS chỉ 0.96×,
tức không tăng gì). Nếu server còn rảnh, RPS phải tăng theo tải chứ không đứng im. Thêm
bằng chứng từ chính server: lúc chạy `make metrics` cùng lúc `load-50`, `n_busy_slots`
lên tới 3.98/4 -- tức 4 slot decode gần như lúc nào cũng bận hết công suất, và có lúc
tới 45 request phải xếp hàng chờ.

Vậy phần latency tăng thêm (P95 tăng 5×) là **thời gian chờ tới lượt (queue time)**,
không phải do mỗi token tính chậm hơn. Vì tốc độ decode mỗi token (TPOT) đo ở phần 2
gần như không đổi dù tải cao hay thấp -- chỉ là số request phải đợi lâu hơn để được
vào 1 trong 4 slot.

Nếu cần tăng throughput, việc đầu tiên tôi sẽ thử là tăng `--parallel` (ví dụ 4 lên 8)
thay vì đổi số thread (`-t`). Lý do: GPU đang rảnh dung lượng (còn dư ~7 GB VRAM), nên
tăng số slot cho phép server xử lý gộp nhiều request hơn mỗi lượt, giảm hàng đợi. Đổi
`-t` thì không ích gì vì phần 1.2 đã đo được threads gần như không ảnh hưởng tốc độ khi
model đã chạy trên GPU.
