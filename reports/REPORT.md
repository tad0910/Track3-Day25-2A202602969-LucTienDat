# Lab 25B — Report

**Họ tên / nhóm:** Lục Tiến Đạt (MSV: 2A202602969)  **Commit:** 4af2dcc  **Điểm tự động (`make grade`):** 84.0 / 88 (100% điểm public, + 4đ dữ liệu ẩn)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0s | Ở G4 & G6, provider bị treo/chậm 5-8s. Đặt 8.0s (lớn hơn p99 lúc khỏe ~4s) giúp kịp ngắt và failover sang provider khác trước khi cạn ngân sách 30s của user. |
| max_attempts_per_provider | 2 | Chỉ cho phép 1 retry duy nhất trên cùng provider. Nếu lỗi quá tải/sập vẫn còn thì chuyển ngay sang provider dự phòng, tránh khuếch đại tải như naive (tới 36 attempts trong G2). |
| backoff_base_s / cap_s | 0.25s / 4.0s | Full jitter rải đều request tránh thundering herd; cap 4.0s đảm bảo một lần retry không ngốn quá nhiều thời gian trong tổng hạn mức 30s. |
| retry_budget_ratio | 0.1 | Ở G3 (brown-out 30% lỗi), budget ratio 0.1 giới hạn lượng retry chỉ chiếm tối đa 10% traffic, đưa `retry_ratio` thực tế xuống 0.034 (đạt chuẩn <= 0.10). |
| breaker.window_size / min_calls | 20 / 10 | Kích thước cửa sổ trượt 20 mẫu đủ để đại diện cho xu hướng tải gần nhất; min_calls = 10 giúp tránh trip oan khi lưu lượng thấp vào ban đêm. |
| breaker.failure_rate_threshold | 0.5 | Ở G2, khi alpha mất 75% GPU và tỷ lệ lỗi vọt lên >50%, breaker ngắt mạch ngay lập tức (trip to OPEN), bảo vệ hàng đợi của alpha và cứu hệ thống khỏi metastable failure. |
| breaker.slow_call_s | 5.0s | Ở G4, alpha không trả mã lỗi nhưng chậm thêm 5s. Đặt ngưỡng 5.0s giúp breaker nhận diện các cuộc gọi chậm là tín hiệu suy thoái nghiêm trọng. |
| breaker.slow_rate_threshold | 0.6 | Khi 60% cuộc gọi bị chậm quá 5.0s, breaker trip sang OPEN, chuyển traffic sang `alpha-eu`, hạ latency p95 từ 13.76s (naive) xuống còn 5.56s (yours). |
| breaker.open_duration_s | 10.0s | Thời gian 10s đủ dài để hàng đợi 18 slot của alpha xả hết các request cũ bị nghẽn trước khi cho phép probe thăm dò lại. |
| breaker.half_open_max_probes | 2 | Ở trạng thái HALF-OPEN chỉ cho đúng 2 request bay thăm dò; ngăn ngừa làn sóng thundering herd đánh sập lại provider vừa mới hồi phục. |

Sơ đồ quyết định của `on_error` (ASCII hoặc bảng: loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```text
[Provider Error Received]
          │
          ├──► CLIENT_ERROR (400, 401, 422)
          │         ├─ Breaker: Release permit (không tính lỗi)
          │         └─ Hành động: GiveUp("client_error") (không retry, không failover)
          │
          ├──► QUOTA_EXHAUSTED (429 insufficient_quota)
          │         ├─ Breaker: Release permit
          │         ├─ Cooldown: Khóa provider 300s (quota_cooldown_s)
          │         └─ Hành động: Failover ngay sang provider kế tiếp trong ladder
          │
          ├──► RATE_LIMITED (429 rate_limit_exceeded)
          │         ├─ Breaker: Release permit
          │         ├─ Cooldown: Khóa provider tới now + retry_after
          │         └─ Hành động: Failover ngay sang provider kế tiếp trong ladder
          │
          └──► OVERLOADED (503, 529) / SERVER_ERROR (500, 502) / TIMEOUT (0, 408)
                    ├─ Breaker: Ghi nhận lỗi on_failure(permit)
                    │
                    ├──► Nếu tries < 2 VÀ còn Retry Budget VÀ còn kịp deadline:
                    │         └─ Hành động: Retry trên CÙNG provider sau FullJitterBackoff
                    │
                    └──► Ngược lại (hết tries / hết budget / breaker OPEN / sát deadline):
                              └─ Hành động: Failover sang provider kế tiếp trong ladder
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Khi cố tình làm hỏng Circuit Breaker (`breaker.min_calls: 1000000000`) và tăng `max_attempts_per_provider: 5`:
   - Ở kịch bản giữ retry budget: trên G3, `retry_ratio` vẫn bị chặn ở mức ~0.10 và `amplification` giữ ở 1.11 nhờ cơ chế token-bucket siết chặt tỷ lệ thử lại.
   - Khi tắt budget (`retry_budget_ratio: 100`, max_tokens: $10^9$): trên G3, `retry_ratio` vọt lên 0.42, và trên G2 `amplification` tăng vọt lên >4.5 khiến thời gian phục hồi `recovery_s` kéo dài trên 60s.
   - Kết luận: Khi breaker hoạt động đúng, nó bảo vệ provider khỏi sập diện rộng; nhưng khi breaker bị cấu hình sai hoặc gặp lỗi chập chờn phân tán (chưa đủ tỷ lệ trip), Retry Budget chính là chốt chặn cuối cùng kiểm soát tổng số request phát sinh toàn hệ thống. Cần có cả hai vì Breaker bảo vệ theo từng node/endpoint, còn Budget bảo vệ tài nguyên trên toàn hệ sinh thái.

2. **`attempt_timeout_s: 30`.** Khi tăng timeout lên 30s:
   - Trên G6 (provider A sập và treo): `availability` tụt từ 1.00 xuống còn 0.48, các lượt người dùng bị `GiveUp("deadline")` hàng loạt vì request đầu tiên treo trọn 30s khiến hết hạn phiên của user trước khi kịp failover sang `beta`.
   - Nghịch lý Survivorship Bias: chỉ số `latency_p95_s` trên các lượt *thành công còn sót lại* có vẻ thấp (~3-4s), nhưng thực tế 52% người dùng đã bị hủy bỏ hoàn toàn.
   - Trên G4: chỉ số gần như không đổi vì provider alpha chỉ chậm thêm 5s (phản hồi lúc ~6.5s < 30s), nhưng người dùng phải chịu độ trễ cao suốt phiên.
   - Kết luận: Timeout đúng không được đặt theo deadline tối đa của user (30s) mà phải dựa vào: $p99 \text{ bình thường} + \text{buffer}$ (khoảng 6-8s) để chừa đủ thời gian cho ít nhất 1 lần failover trước khi user bỏ đi.

3. **Bỏ lọc tool calling khỏi `ladder()`.** Khi bỏ bộ lọc capability trong `ladder()` và chạy băng G6:
   - Chỉ số `incompatible_routes` tăng vọt từ 0 lên 446 lượt vi phạm, do các tool call bị điều hướng sai vào `gamma-mini` (model giá rẻ không có khả năng tool calling).
   - Dưới góc nhìn Product Manager: Đây là trải nghiệm khách hàng tồi tệ nhất — khách hàng bấm "Hoàn tiền", tiền không được hoàn, hệ thống trả về lỗi ngớ ngẩn hoặc bị treo do model không hiểu format schema của tool. Tiết kiệm chi phí bằng model rẻ nhưng không tương thích chức năng tương đương với việc tung ra tính năng hỏng cho người dùng.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---|---:|---:|---:|---:|---:|---:|
| naive | 1.00 | 81 | 128 | 81 | 0 | 14,967.00 $ | 1.123 |
| yours | 1.00 | 66 | 0 | 0 | 0 | 0.00 $ | 0.303 |

Trả lời: 
- Chỉ có **Idempotency Key là chưa đủ**: Nếu worker bị crash sau bước thanh toán và không có checkpoint, worker mới khởi động lại sẽ phải chạy lại từ đầu toàn bộ các bước tính toán trước đó (tốn tiền và thời gian gọi LLM vô ích, `wasted_llm_per_crash > 1.0`).
- Chỉ có **Checkpoint là chưa đủ**: Khi gặp `ServiceTimeout` (lệnh thanh toán đã thành công trên cổng thanh toán nhưng gói tin mạng phản hồi bị rơi), nếu không có idempotency key, worker retry sẽ tạo ra một giao dịch trừ tiền thứ hai (`duplicate_refunds`).
- **Nếu ghi checkpoint TRƯỚC khi gọi payment**: Nếu worker ghi nhận bước "đã hoàn tiền" vào DB rồi crash ngay trước khi gửi lệnh HTTP đến cổng thanh toán, worker mới khi khôi phục sẽ thấy checkpoint đã tồn tại và bỏ qua bước này $\rightarrow$ Khách hàng không bao giờ nhận được tiền (**Lost Refund** thảm họa).

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (FALSE) | none (Đúng) | N/A |
| slow_leak | ticket | none (BỎ SÓT) | ticket | 1337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page (FALSE) | none (Đúng) | N/A |
| small_bump | none | page (FALSE) | none (Đúng) | N/A |

Error budget 30 ngày: đã tiêu **42.95%**, còn **57.05%**. **Không đóng băng release** (`freeze_releases: False`) vì ngân sách lỗi còn hơn một nửa. 

Vì sao `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức ai lúc 11 giờ trưa? 
Bởi vì trong 20 phút đó, lượng lỗi chỉ tiêu tốn một phần cực nhỏ (~0.2%) trong tổng error budget của cả tháng. Nó tự khắc phục nhanh chóng và không đe dọa cam kết SLO 99.5%. Nếu rung chuông (page) on-call cho những biến động ngắn như vậy, kỹ sư sẽ bị kiệt sức (alert fatigue) và mất tập trung vào các vấn đề thực sự nghiêm trọng.

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm](reports/gameday/G2_overload_storm.png)

**Tóm tắt (3 câu):** Vào thời điểm $t=120s$, provider chính `alpha` bị suy giảm đột ngột 75% năng lực GPU trong vòng 3 phút. Client v1 (`NaiveClient`) phản ứng bằng cơ chế retry 3 tầng không có jitter và không có circuit breaker, bơm tới 254 attempts/s vào hàng đợi của alpha. Hậu quả là hệ thống rơi vào trạng thái thất bại giả bền vững (metastable failure), khiến dịch vụ sập hoàn toàn thêm 155 giây sau khi GPU đã hồi phục, làm tỷ lệ khả dụng giảm còn 45.6%.

**Ảnh hưởng:** 1.285 lượt người dùng thất bại (chỉ có 1.077 / 2.362 lượt thành công), kéo dài trong 335 giây ảo (từ $t=120$ đến $t=455$). Sự cố này tiêu tốn hơn 120% error budget của tháng (vượt quá toàn bộ ngân sách lỗi hàng tháng chỉ trong một sự cố duy nhất).

**Timeline** (giây ảo — lấy từ CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha mất 75% năng lực GPU | `attempts_alpha = 5`, `calls_ok = 7`, hàng đợi bắt đầu dồn ứ |
| 122 | Tỷ lệ thành công sụp đổ hoàn toàn | `calls_ok = 0`, lỗi 529 và timeout bắt đầu xuất hiện |
| 136 | Bắt đầu bão retry (Retry Storm) | `attempts_alpha` tăng vọt lên 53 attempts/s do retry 3 tầng kích hoạt |
| 143 | Đỉnh điểm quá tải hàng đợi | `attempts_alpha` đạt đỉnh 254 attempts/s; `calls_ok = 0` |
| 300 | alpha hồi phục 100% phần cứng GPU | Provider sẵn sàng 18 slot nhưng `calls_ok` vẫn bằng 0 |
| 305 | Hệ thống vẫn chưa hồi phục (Metastable Failure) | Hàng đợi alpha vẫn nghẹt cứng request cũ đã quá hạn; attempts vẫn ở mức 10-30 req/s |
| 455 | Hệ thống bắt đầu phục hồi | `calls_ok` đạt 4 req/s, hàng đợi dần thông thoáng sau 155s phục hồi trễ |
| 471 | Trở lại trạng thái bình thường | `calls_ok = 16`, `attempts_alpha = 7`, đạt 100% throughput ban đầu |

**Trigger vs root cause:** 
- **Trigger (Ngòi nổ)**: Provider alpha bị mất 75% GPU do lỗi hạ tầng phần cứng phía đối tác trong 180 giây.
- **Root cause (Nguyên nhân gốc rễ)**: `NaiveClient` sử dụng 3 tầng retry lồng nhau (lên tới 36 lần/call) với khoảng trễ cố định, không có randomized jitter, không có retry budget và không có circuit breaker. Khi xảy ra nghẽn, client khuếch đại lưu lượng lên gấp 10 lần (`amplification = 9.904`), biến một sự cố cục bộ 3 phút thành thảm họa sập toàn diện.
- **Cơ chế Metastable Failure**: Server xử lý theo hàng đợi FIFO và vẫn tốn tài nguyên xử lý các request mà client đã timeout bỏ cuộc. Khi GPU phục hồi ở $t=300$, lượng request retry rác tồn đọng tiếp tục chiếm dụng 100% năng lực GPU, khiến các request mới tiếp tục bị timeout, tự nuôi dưỡng vòng lặp quá tải suốt 155 giây tiếp theo.

**Điều gì giúp / điều gì làm tệ hơn:** 
- *Làm tệ hơn*: Client tiếp tục spam request với delay cố định; không có cơ chế failover sang `alpha-eu` hay `beta`.
- *Giúp khắc phục*: Sau $t=300$, lượng request mới từ user có khoảng giảm nhẹ ngẫu nhiên, giúp hàng đợi dần xử lý hết lượng request tồn đọng.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai `ResilientClient` có Circuit Breaker sliding-window và Full Jitter Backoff | Prevent | Core Agent Team | Chạy test CI băng G2 đạt amplification <= 1.1 và recovery_s = 0 |
| 2 | Áp dụng `RetryBudget` (Token Bucket) giới hạn retry <= 10% tổng lưu lượng | Prevent | Infra Team | Chạy stress-test đảm bảo retry_ratio <= 0.10 khi provider trả lỗi 500/529 |
| 3 | Tích hợp Burn-rate Alerting đa cửa sổ theo chuẩn Google SRE | Detect | SRE On-call | Phát hiện sự cố mất năng lực GPU trong <= 5 phút, 0 false alarm |
| 4 | Cấu hình tự động failover sang `alpha-eu` và `beta` khi primary gặp sự cố | Mitigate | Platform Team | Băng G2 đạt availability >= 99% trong suốt thời gian diễn ra sự cố |

## 7. Một điểm yếu còn lại

**Tình huống**: Chi phí tăng vọt khi failover dồn tải sang `beta`.
- Trong sự cố G6 (toàn bộ provider A sập), `beta` gánh toàn bộ tải chính. Model của `beta` có chi phí 1.4/call (đắt hơn 40% so với alpha 1.0/call) và tốc độ chậm hơn (1.8s so với 1.2s), đồng thời rate limit của beta chỉ có 16 rps. Nếu sự cố kéo dài nhiều ngày, chi phí vận hành sẽ tăng đột biến và `beta` có nguy cơ bị nghẽn thứ cấp.
- **Cách khắc phục**: Triển khai cơ chế **Semantic Cache** (như Part G) để phục vụ 60% FAQ trực tiếp, kết hợp với bộ điều tiết tải phân bổ: chỉ gửi các request nghiệp vụ phức tạp sang `beta`, còn các tác vụ chat thông thường chuyển tối đa sang model siêu rẻ `gamma-mini` (0.2/call).

## 8. Cache (Part G)

Số từ `reports/gameday/cached.json` (`make cache`) và `reports/gameday/resilient.json`:

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9120 | 0.9645 | 0.635 | 174 | 0 | 6 | 0.927 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Khi để 0.80, hệ thống trả lời sai 181 câu (`wrong_answers = 181`) do nhầm giữa các câu hỏi gần nghĩa. Đặt 0.92 đưa `wrong_answers` về đúng 0 tuyệt đối. |
| ttl_s | 45s | Sự cố carrier đổi bảng giá vận chuyển âm thầm ở t=480s không có webhook. TTL 45s giúp hạn chế số câu trả lời lỗi thời chỉ còn 6 câu (`outdated_answers = 6 <= 15`). |
| max_stale_s | 300s | Khoảng thời gian toàn bộ provider sập kéo dài 120s (t=240-360). max_stale 300s cho phép phục vụ 174 câu trả lời stale, đưa `faq_availability` lên 96.45% (vượt chuẩn 93%). |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.** Khi chạy C1 với các ngưỡng khác nhau:
   - Ở `0.80`: Cache hit ratio cao (74.2%), chi phí thấp (0.921), nhưng có **181 câu trả lời sai** (`wrong_answers_by_reason: other_question = 181`). Cặp câu hỏi bị nhầm nhiều nhất là *"Chính sách hoàn tiền"* và *"Thời gian nhận tiền hoàn"*.
   - Ở `0.92`: `wrong_answers = 0`, hit ratio đạt 63.5%, chi phí 0.927.
   - Ở `0.97`: Ngưỡng quá khắt khe khiến hit ratio giảm sâu xuống <35%, chi phí vọt lên 1.25/turn và không tận dụng được sức mạnh của cache.

2. **TTL.** Khi chạy với `ttl_s = 30s / 45s / 600s`:
   - Ở `600s`: Khi carrier đổi bảng giá ở t=480s không có thông báo, cache giữ giá cũ tới hết bài test, gây ra **80 câu trả lời sai giá** (`outdated_answers = 80 > 15`).
   - Ở `45s`: Số câu trả lời cũ giảm xuống chỉ còn 6 câu (`outdated = 6`). Webhook không giúp gì được vì carrier thay đổi âm thầm không bắn sự kiện; do đó TTL ngắn là biện pháp bảo vệ duy nhất đối với các thay đổi không báo trước.

3. **Stale-if-error.** Khi đặt `max_stale_s: 0` (tắt stale-if-error):
   - Trong 120s toàn bộ provider sập (t=240-360s), cache từ chối phục vụ câu trả lời cũ đã hết TTL $\rightarrow$ `stale_served = 0`, và `faq_availability` tụt từ 96.45% xuống còn **85.87%** (FAIL chuẩn 93%). Stale-if-error đã cứu sống 174 lượt khách hàng trong cơn khủng hoảng.

Trả lời:
- Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") **không** được cache, kể cả khi cache theo từng user?
  - Vì câu hỏi cá nhân liên quan đến dữ liệu động thay đổi theo thời gian thực (trạng thái đơn hàng, số dư ví). Nếu cache, khách hàng sẽ nhận được thông tin trạng thái cũ (ví dụ: đơn đã giao nhưng vẫn báo đang lấy hàng). Hơn nữa, việc cache dữ liệu định danh cá nhân (PII) mang rủi ro rò rỉ dữ liệu chéo giữa các phiên đăng nhập hoặc vi phạm quy định bảo mật dữ liệu.
- Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?
  - Tuyệt đối không chỉ nhìn vào tỷ lệ thành công tổng thể (User-Facing Availability). Phải thiết lập cảnh báo dựa trên:
    1. **Tỷ lệ phục vụ dữ liệu cũ (`Stale Served Ratio`)**: Khi `stale_served > 0` tăng đột biến, chứng tỏ backend đang hỏng và cache đang phải gánh đỡ.
    2. **Tỷ lệ lỗi của Provider Backend (`Provider Error Rate / 5xx / Circuit Breaker State`)**: Khi toàn bộ breaker chuyển sang OPEN, phải bắn Page khẩn cấp ngay lập tức bất kể người dùng có nhận được câu trả lời từ cache hay không.
