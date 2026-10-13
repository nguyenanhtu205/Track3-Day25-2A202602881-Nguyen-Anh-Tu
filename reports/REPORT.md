# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Anh Tú — MSSV: 2A202602881  **Commit:** 463e623fd8b4d41d2259f2ef381cdfd7e2a45866  **Điểm tự động (`make grade`):** 84 / 88 (4 điểm E dữ liệu ẩn không có trong repo)

> Mọi số dưới đây chạy lại với `seed: 25`; artifact nguồn nằm trong `reports/gameday/`, `reports/cache/`, `reports/workflows.json` và `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | ✗ |
| G1 | yours | 1.0000 | 4.780 | 1.063 | 0.001 | 2.104 | 0.0 | ✓ |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | ✗ |
| G2 | yours | 1.0000 | 10.362 | 1.036 | 0.005 | 2.108 | 0.0 | ✓ |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | ✗ |
| G3 | yours | 1.0000 | 4.668 | 1.059 | 0.030 | 2.143 | 0.0 | ✓ |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | ✗ |
| G4 | yours | 1.0000 | 5.626 | 1.014 | 0.002 | 2.072 | 0.0 | ✓ |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | — | ✗ |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | — | ✓ |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | ✗ |
| G6 | yours | 1.0000 | 6.025 | 1.056 | 0.019 | 2.188 | 0.0 | ✓ |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 s | Đủ vượt qua độ trễ bình thường nhưng còn thời gian failover trong deadline 30 s. G6 đạt availability 1.0000; thử 30 s chỉ còn 0.9488 và 122 lượt `deadline`. |
| `max_attempts_per_provider` | 2 | Một retry cùng provider rồi chuyển vùng/provider; G2 amplification chỉ 1.036 thay vì naive 9.904. |
| `backoff_base_s` / `cap_s` | 0.25 / 4.0 s | Full jitter phân tán retry; G3 retry ratio 0.030, dưới giới hạn 0.10. |
| `retry_budget_ratio` | 0.10 | Chặn storm khi breaker chưa kịp hoạt động: thí nghiệm G2/G3 ở mục 3. |
| `breaker.window_size` / `min_calls` | 20 / 10 | Cần đủ mẫu trước khi trip, nhưng vẫn phản ứng với 50% lỗi gần đây. |
| `breaker.failure_rate_threshold` | 0.50 | Tránh bỏ primary vì lỗi nền; G2 vẫn hồi 0 s và primary quay lại sau restore. |
| `breaker.slow_call_s` | 5.0 s | Phát hiện slow success, nên G4 p95 5.626 s thay vì 13.764 s của naive. |
| `breaker.open_duration_s` | 10.0 s | Cooldown ngắn để probe phục hồi mà không dồn herd; G2 recovery 0 s. |
| `breaker.half_open_max_probes` | 2 | Tối đa hai probe đang bay, tránh bùng request khi mở lại. |

Quyết định `on_error`:

| Loại lỗi | Breaker | Retry cùng provider | Failover / cooldown |
|---|---|---|---|
| `CLIENT_ERROR` (400…) | release | không | `GiveUp("client_error")` |
| `QUOTA_EXHAUSTED` | release | không | cooldown 300 s, failover ngay |
| `RATE_LIMITED` | release | không | tôn trọng `retry_after`, cooldown chung, spillover |
| overload / 5xx / timeout | record failure | tối đa một lần, jitter + budget + deadline | khi hết retry/budget hoặc breaker mở |

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối.** Tôi cố ý vô hiệu breaker (`min_calls=1e9`) và tăng `max_attempts_per_provider=5`. Với budget 0.1, G2 có availability 0.9987, amplification 1.379, retry ratio 0.057 và recovery 0 s; G3 có retry ratio 0.068. Khi tắt budget (`ratio=100`, refill 1e6/s, 1e9 token), G2 giảm còn 0.9750 availability, amplification tăng 1.584, retry ratio 0.384 và recovery 16 s; G3 retry ratio 0.158 nên FAIL. Breaker là phản ứng theo health của một provider; budget là giới hạn tổng tải từ retry khi breaker cấu hình sai, phản ứng chậm, hoặc nhiều provider cùng lỗi — vì vậy cần cả hai.

2. **`attempt_timeout_s: 30` tạo survivorship bias.** Với G6, timeout 30 s có availability 0.9488 và 122 lượt `deadline`, dù p95 của các lượt thành công chỉ 5.314 s; cấu hình 8 s đạt availability 1.0000, p95 6.025 s. G4 là đối chứng: timeout 30 s vẫn availability 1.0000 và p95 5.608 s vì provider chậm nhưng vẫn trả lời, không bị hang. Do đó phải xem availability và `give_up_reasons` trước p95; timeout chọn từ tail latency bình thường cộng thời gian cần để failover, nhưng phải nhỏ hơn deadline người dùng.

3. **Bỏ lọc tool capability là lỗi sản phẩm, không phải chỉ lỗi kỹ thuật.** Khi cố ý cho tool call đi cả `gamma-mini`, G6 có 253 `incompatible_routes`, availability 0.8939 và FAIL; lọc capability cho kết quả 0 và availability 1.0000. Với product manager: hệ thống đã chọn một người trả lời rẻ/nhanh nhưng người đó không biết gọi công cụ để hoàn tiền, nên câu trả lời trông như hoàn tất mà workflow không thể thực hiện yêu cầu.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,496,700 cents | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 cents | 0.303 |

Chỉ idempotency key chưa đủ: sau crash, worker vẫn có thể lặp lại LLM/read steps, tốn chi phí và không biết đã đi đến đâu. Chỉ checkpoint cũng chưa đủ: crash sau payment commit nhưng trước checkpoint khiến worker gọi lại effect; key ổn định mới cho payment/email trả lại kết quả cũ thay vì chạy lại. Nếu checkpoint được ghi *trước* payment, crash trước khi gọi payment sẽ làm worker tin refund đã hoàn tất và tạo lost refund.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện (phút) |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 |
| provider_outage | page | page | page | 4 |
| deploy_blip | none | page (sai) | none | — |
| slow_leak | ticket | none (sai) | ticket | 1337 |
| degraded_evening | page | page | page | 291 |
| night_spike | none | page (sai) | none | — |
| small_bump | none | page (sai) | none | — |

Error budget 30 ngày: 25,901,755 requests, cho phép 129,508.8 lỗi; thực tế 55,618 lỗi. Đã tiêu 42.95%, còn 57.05%, nên **không** đóng băng release. `small_bump` chỉ kéo dài 20 phút ở 12% và không thỏa cả cửa sổ dài + ngắn của page rule; đánh thức on-call lúc 11 giờ trưa sẽ tạo alert fatigue trong khi chưa tiêu ngân sách đáng kể.

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 overload-storm timeseries](gameday/G2_overload_storm.png)

**Tóm tắt:** Trong G2, alpha mất 75% GPU trong 180 giây. NaiveClient có ba tầng retry với delay cố định, không jitter/budget/breaker/fallback, biến suy giảm capacity thành retry storm. Kết quả chỉ 1,077/2,362 turns thành công (45.60%) và hệ thống mất thêm 155 s sau restore mới phục hồi.

**Ảnh hưởng:** 1,285 lượt user thất bại; p95 của lượt thành công là 15.034 s. Có 34,475 attempts cho 3,481 logical calls (amplification 9.904), gồm 26,298 rate-limit, 3,194 overload và 2,737 client timeout. Trong phạm vi 2,362 turns của incident, SLO 99.5% chỉ cho phép 11.8 failure; 1,285 failures tiêu khoảng 108.8 lần (10,880%) error budget tương ứng.

**Timeline** (giây ảo, từ `naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | Alpha mất 75% capacity | 7/7 call hoàn tất, nhưng alpha đã nhận 5 attempts trong giây mở đầu. |
| 150 | Hàng đợi/retry storm tăng mạnh | 0/3 calls thành công; 163 attempts alpha/s. |
| 180 | Failure lặp lại duy trì tải | 0/5 thành công; 138 attempts alpha/s. |
| 300 | Provider được restore hoàn toàn | 0/5 thành công, vẫn 115 attempts alpha/s: restore không xóa hàng đợi cũ. |
| 360 | Request đã hết hạn vẫn làm nghẽn server | 0/1 thành công; 50 attempts alpha/s. |
| 420 | Backlog còn tiếp diễn | 0/2 thành công; 53 attempts alpha/s. |
| 455 | Hệ thống mới đạt recovery window | 4/5 thành công; 13 attempts alpha/s; recovery = 155 s sau t=300. |

**Trigger vs root cause:** Trigger là việc alpha mất GPU/capacity. Root cause không phải cá nhân hay một lỗi retry đơn lẻ: kiến trúc có 3 tầng retry chồng nhau (tối đa 36 attempts), fixed delay, không retry budget/circuit breaker và không fallback. Điều này khuếch đại lỗi và rate-limit, làm attempts đến nhanh hơn capacity. Sau t=300, server vẫn FIFO xử lý queue request mà client đã bỏ; retry mới tiếp tục giữ tải trên capacity, kéo dài vòng lặp metastable failure dù provider đã hồi phục.

**Điều gì giúp / điều gì làm tệ hơn:** Provider có queue shedding (529) ngăn chặn một phần request chờ vô hạn, và deadline người dùng giới hạn một phần thiệt hại. Ngược lại, client timeout không hủy work phía server; retry fixed-delay đồng pha, cùng với 26,298 429, tạo thêm load đúng lúc hệ thống cần giảm tải.

**Action items**:

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Giữ một retry layer, full jitter và retry budget 10% | prevent | Agent platform | CI G2: amplification ≤ 1.6 và retry ratio trong ngưỡng |
| 2 | Circuit breaker theo failure/slow rate, HALF_OPEN tối đa 2 probe | mitigate | Reliability team | Chaos G2/G4: recovery ≤ 30 s, G4 p95 ≤ 10 s |
| 3 | Dashboard/alert theo amplification, 429 và queue/recovery lag | detect | SRE on-call | Alert khi amplification > 1.6 trong 5 phút; drill có runbook fallback |
| 4 | Capability-aware fallback alpha → alpha-eu → beta | mitigate | LLM integration | G6 CI: availability ≥ 95%, incompatible tool routes = 0 |

## 7. Một điểm yếu còn lại

Khi cả bốn provider cùng chậm nhưng vẫn trả lời, fallback không tạo thêm capacity; beta cũng tăng chi phí (1.4/call so với 1.0 ở alpha). Phiên bản tiếp theo sẽ thêm admission control theo deadline và per-provider concurrency/queue signals, đồng thời đo cost budget để chỉ spillover những request có giá trị cao.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | ✗ |
| cached (của bạn) | 0.9120 | 0.9640 | 0.652 | 156 | 0 | 3 | 0.916 | ✓ |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.95 | 0.80 tạo 166 `other_question` và FAIL; 0.95 có 0 wrong, hit 0.652, cost/turn 0.916. 0.97 cũng đúng nhưng hit chỉ 0.578 và cost 0.965. |
| `ttl_s` | 60 s | 30 s: hit 0.591, outdated 2; 60 s: hit 0.652, outdated 3; 300 s: hit 0.731 nhưng 95 outdated và FAIL. 60 giữ hit cao mà dưới ngưỡng outdated 15. |
| `max_stale_s` | 180 s | Che outage 120 s: stale=156, FAQ availability 0.964. Đặt 0 khiến stale=0 và FAQ availability 0.8733, FAIL. |

Thí nghiệm:

1. **Ngưỡng.** Ở 0.80, cache hit 0.659 và cost 0.975 nhưng nhầm 166 câu (`other_question`), nên availability FAQ còn 0.8651. Giá trị 0.95 loại nhầm hoàn toàn và vẫn có hit 0.652. Ở 0.97, hit giảm xuống 0.578, cost tăng 0.965; độ chặt thêm không mang lợi ích an toàn so với 0.95.
2. **TTL.** TTL 30/60/300 cho outdated lần lượt 2/3/95 và hit 0.591/0.652/0.731. Webhook chỉ invalidates nguồn refund được công bố; thay đổi shipping không báo không thể biết được. Vì vậy TTL là giới hạn duy nhất đối với câu trả lời cũ của nguồn silent-update.
3. **Stale-if-error.** Với `max_stale_s: 0`, FAQ availability là 0.8733 và stale served là 0. Với 180 s, FAQ availability lên 0.9640 và phục vụ 156 stale answer, nhưng vẫn không dùng entry quá TTL+stale và vẫn invalidates ngay khi có webhook.

Câu hỏi cá nhân không được cache, kể cả theo từng user, vì nội dung có thể thay đổi theo order/refund/points, và một lỗi scope/key hay biến thể semantic có thể làm lộ dữ liệu hoặc trả stale state. Trong outage, dashboard availability có thể xanh nhờ cache; on-call vẫn cần alert theo provider attempt success/error, `GiveUp`/stale-served rate và tỉ lệ FAQ được cache, thay vì chỉ nhìn end-user availability.
