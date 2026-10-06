# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Thọ Đạt
**MSSV:** 2A202602484
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** 12th Gen Intel(R) Core(TM) i7-1250U
- **Cores:** 10 physical / 12 logical
- **CPU extensions:** AVX2
- **RAM:** 15.6 GB
- **Accelerator:** Intel(R) Iris(R) Xe Graphics (Vulkan backend)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Setup diễn ra thuận lợi sau khi set PYTHONUTF8=1 để hỗ trợ hiển thị ký tự UTF-8 trên Windows PowerShell. Máy tự nhận diện backend Vulkan trên iGPU Intel Iris Xe và tải thành công 2 bản quant Gemma 4 E2B.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 26588 | 883 / 4195 | 59.1 / 60.7 | 4580 / 8017 / 8017 | 16.9 |
| UD-Q2_K_XL | 2.24 | 15184 | 1349 / 12015 | 111.3 / 140.1 | 8730 / 19002 / 19002 | 9.0 |

**Quan sát** (≤ 60 chữ): 2-bit chậm hơn 4-bit 1.88x (9.0 vs 16.9 tok/s) và TTFT chậm hơn 1.53x vì GPU bị compute-limited với giải thuật dequantize phức tạp. Chất lượng trả lời kém hơn rõ rệt. Không đáng để đánh đổi.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.28 | 29000 | 41000 | 41000 | 7.4 | 0.0% |
| 50 | 0.44 | 38000 | 55000 | 58000 | 14.6 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.60×
- **P95 tăng:** 1.34×
- **Effective concurrency ở 50 users:** 14.6 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.88 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà rõ rệt: tải tăng 5x nhưng RPS chỉ tăng 1.60x. Concurrency đạt 14.6 vượt xa 4 slots, 46 request bị deferred, chứng minh P95 tăng chủ yếu do queue time. Cần tăng --parallel kèm admission control trước tiên để cải thiện goodput@SLO.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local Windows 11 | stub |
| N17 Data pipeline | In-memory toy docs | stub |
| N18 Lakehouse | Memory corpus | stub |
| N19 Vector + features | Keyword overlap retrieval | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.7 ms
- llm: 4827.8 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck 100% nằm ở stage LLM sinh text, đúng như kỳ vọng. Để giảm latency 2x cho pipeline, giải pháp hiệu quả nhất là tối ưu LLM: bật Prefix Caching cho system prompt, hoặc dùng Speculative Decoding.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giữ quantization UD-Q4_K_XL thay vì UD-Q2_K_XL

```
before:  9.0 tok/s
after:   16.9 tok/s
speedup: 1.88×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả cho thấy tốc độ sinh token tăng 1.88x (từ 9.0 lên 16.9 tok/s) khi sử dụng định dạng 4-bit (UD-Q4_K_XL) so với 2-bit (UD-Q2_K_XL), trái với trực giác thông thường rằng model nhẹ hơn thì decode sẽ nhanh hơn.

Cơ chế nằm ở chỗ máy sử dụng iGPU Intel Iris Xe (Vulkan offload). Khi chạy decode, mô hình 2-bit nén sâu đòi hỏi các phép tính giải nén (dequantization) toán học phức tạp cho từng block trước khi nhân ma trận. Năng lực tính toán ALU của iGPU bị quá tải bởi chi phí dequantize này (compute-bound) vượt xa lợi ích tiết kiệm băng thông bộ nhớ RAM. Bản 4-bit có kernel tính toán tối ưu hoá trực tiếp trên phần cứng hơn rất nhiều, giúp tận dụng tối đa năng lực iGPU và đem lại speedup vượt trội cùng chất lượng vượt trội.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** 

**Numbers:**

```
before:  
after:   
speedup: 
```

**Điều này nói lên gì mà deck chưa nói:**



---

## 7. Khảo sát & feedback  *(optional)*

1. Bạn mất bao nhiêu thời gian cho lab này (không tính thời gian tải model)?
   ~1.5 giờ.

2. Khái niệm nào trong bài lab này làm bạn bất ngờ nhất hoặc vỡ lẽ ra điều mới?
   Hiện tượng 2-bit quantization lại chạy chậm hơn 4-bit trên GPU tích hợp do compute overhead của dequantization.

---

## 8. Checklist nộp bài  *(tự kiểm tra trước khi push)*

- [x] Đã chạy `make verify` (Windows: `.\lab.ps1 verify`) và lệnh **exit 0**.
- [x] Đã kiểm tra repo trên GitHub ở chế độ **Public**.
- [x] Tên repo đúng quy ước `K4-L3-DAY20-HoVaTen-MSSV-ModelServing`.
- [x] Đã đủ 5 screenshot trong `submission/screenshots/`.
- [x] Không còn placeholder hay section "required -- replace this line".
- [x] Số liệu trong REFLECTION khớp với `benchmarks/*.md`.
- [x] Đã paste URL repo vào VinUni LMS trước deadline.

---

## 9. Khai báo công cụ AI  *(rubric quy định)*

- **Công cụ đã dùng:** Antigravity / Gemini
- **Mục đích sử dụng:** Hỗ trợ chạy các lệnh benchmark, debug định dạng ký tự UTF-8 trên Windows PowerShell, phân tích cơ chế Little's Law và giải thích sự khác biệt hiệu năng giữa các quantization.