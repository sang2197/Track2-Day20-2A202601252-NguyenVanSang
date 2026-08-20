# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 107.5 | 95% |
| 4 | 108.3 | 96% |
| 8 | 111.7 | 99% |
| 16 | 113.1 | 100% |
| 32 | 112.3 | 99% |

**Best**: `-t 16` at 113.1 tok/s
**Slowest tested**: `-t 1` at 107.5 tok/s (1.05x spread)
**Against the physical-core default** (`-t 8`, 111.7 tok/s): 1.01x

Use this in your run:

```bash
LAB_N_THREADS=16 make bench
```

## Your explanation

Chỉ 1.05x chênh lệch giữa `-t 1` (107.5 tok/s) và `-t 16`(113.1 tok/s), không có knee rõ rệt ở 8 physical core hay drop mạnh sau 16 logical core như deck mô tả cho CPU-only decode. `-t 32` (oversubscribe 2x logical core) chỉ giảm nhẹ (112.3, -0.7%) chứ không sập.

Lý do: `ngl=99` nghĩa là gần như toàn bộ layer được offload lên GPU. Phần compute nặng của decode -- nhân ma trận mỗi token -- chạy trên CUDA, bị giới hạn bởi băng thông đọc trọng số từ VRAM, không phải bởi CPU threads. `-t` ở đây chỉ điều khiển số thread CPU lo việc điều phối nhẹ (dispatch kernel, sampling, quản lý KV cache), nên tăng threads gần như không đổi được throughput -- đúng như quan sát. Mức tăng nhỏ từ `-t 1` lên `-t 16` (~5%) khớp với việc có thêm vài thread giúp overlap tốt hơn phần orchestration đó, còn `-t 32` giảm nhẹ vì oversubscription (32 thread tranh nhau 16 logical core) gây overhead scheduling không đáng kể nhưng đo được.

Đây là kết quả **khác kỳ vọng từ deck** lý do là setup của tôi dùng GPU offload đầy đủ, nên bottleneck đã chuyển từ CPU compute/threading sang GPU memory bandwidth trước khi thread count kịp có ảnh hưởng đáng kể.
