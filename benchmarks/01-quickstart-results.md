# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 6208 | 525 / 978 | 10.6 / 11.6 | 1178 / 1649 / 1649 | 94.4 |
| UD-Q2_K_XL | 2.24 | 6168 | 502 / 894 | 9.4 / 10.5 | 1086 / 1483 / 1483 | 106.7 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.13x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

`UD-Q2_K_XL` là 1.13x nhanh hơn ở decode (106.7 vs 94.4 tok/s), TTFT P50 nhanh hơn ~23ms, và nhỏ hơn 0.73 GB (2.24 GB so với 2.97 GB).

Để đánh giá chất lượng tôi chạy song song hai server (`serve.py --port 8080` cho Q4_K_XL, `serve.py --compare --port 8090` cho Q2_K_XL) rồi gửi cùng hai câu hỏi:

1. **Câu hỏi giải thích khái niệm** ("vì sao quantize hạ chất lượng model"): cả hai trả lời
   mạch lạc, đúng ý, không khác biệt đáng kể về nội dung hay văn phong -- chỉ lệch vài từ.
2. **Câu hỏi suy luận có tính toán** (tốc độ trung bình 2 chặng đường): cả hai đều trình bày
   đúng từng bước (tổng quãng đường 300 km, tổng thời gian 4.0 giờ), không có lỗi số học hay
   ảo giác (hallucination) ở bước nào tôi quan sát được, kể cả ở bản 2-bit.

**Kết luận:** với prompt ngắn/trung bình như trong lab này, `UD-Q2_K_XL` đáng dùng mặc dù có thể mất rất ít, đổi lại nhanh hơn 13% và nhẹ hơn 25%. Tuy nhiên 2-bit chưa chắc an toàn cho mọi tác vụ -- Q2_K_XL nén mạnh hơn có thể lộ rõ hơn ở context dài hoặc câu hỏi cần kiến thức chuyên sâu/chính xác cao.
