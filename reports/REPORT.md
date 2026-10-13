# Lab 25B — Report

**Họ tên:** Đỗ Trung Tuyến · MSSV 2A202602427 · GitHub: TrungTuyenDo02  **Commit:** nhánh `main` (mỗi cụm Part một commit, xem `git log`)  **Điểm tự động (`make grade`):** 84 / 88 (4đ còn lại là dữ liệu SLO ẩn do giảng viên chấm)

> Số liệu lấy từ `reports/gameday/*.json` (seed 25), `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json`.
> Ngoài băng công khai, mình còn chạy `make grade --seeds 1..8` và 3 bộ băng biến thể (lệch thời điểm +37 s / −20 s / +53 s, cường độ ×1.0 / ×1.2 / ×1.3): vẫn 6/6 băng + C1 PASS, tức không overfit băng công khai.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.16 | 7.74 | 6.74 | 2.163 | 4 | ✗ (availability, 36 027 vi phạm retry-after, amplification) |
| G1 | yours | 1.0000 | 4.78 | 1.06 | 0.001 | 2.104 | 0 | ✓ |
| G2 | naive | 0.4560 | 15.03 | 9.90 | 8.90 | 4.627 | 155 | ✗ (availability, amplification, recovery) |
| G2 | yours | 1.0000 | 10.36 | 1.04 | 0.005 | 2.112 | 0 | ✓ (alpha share sau restore = 1.0) |
| G3 | naive | 1.0000 | 4.77 | 1.16 | 0.164 | 2.328 | 0 | ✗ (retry_ratio 0.164 > 0.10) |
| G3 | yours | 1.0000 | 4.62 | 1.06 | 0.030 | 2.140 | 0 | ✓ |
| G4 | naive | 1.0000 | 13.76 | 1.00 | 0.004 | 2.008 | 0 | ✗ (p95 13.76 > 10) |
| G4 | yours | 1.0000 | 5.63 | 1.01 | 0.002 | 2.072 | 0 | ✓ (alpha share sau restore = 1.0) |
| G5 | naive | 0.3699 | 5.36 | 17.03 | 16.03 | 2.124 | — | ✗ (9 149 retry trên lỗi 400, availability) |
| G5 | yours | 0.8179 | 5.48 | 1.00 | 0.002 | 2.504 | — | ✓ (0 retry trên 400, 0 failover thừa) |
| G6 | naive | 0.5170 | 4.84 | 12.34 | 11.34 | 2.009 | 0 | ✗ (availability) |
| G6 | yours | 1.0000 | 5.96 | 1.06 | 0.018 | 2.196 | 0 | ✓ (0 `incompatible_routes`) |

Kết quả: naive 0/6, yours 6/6. Chi tiết bất ngờ: G5 của mình chỉ đạt 0.8179 vì 413 turn bị lỗi 400 **do chính bug của mình** — `GiveUp("client_error")` là đúng, không có provider nào cứu được request sai (ngưỡng đề bài 0.78).

## 2. Cấu hình client và lý do

Khối `client:` mình giữ nguyên giá trị khởi điểm của starter vì nó đã cho 6/6 PASS trên seed 25, trên 8 seed và trên các băng biến thể ở trên; mình không chỉnh thêm chỉ để "đẹp số". Lý do từng tham số (và thí nghiệm ở mục 3 chứng minh chúng không tuỳ tiện):

| Tham số | Giá trị | Vì sao (băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Đủ dài hơn p95 bình thường (~5 s) nhưng đủ ngắn để failover trong deadline 30 s. Với 30 s, G6 tụt availability 1.000 → **0.9488** (< 0.95, FAIL) — xem thí nghiệm 2. `_send` còn tự cắt timeout theo deadline. |
| max_attempts_per_provider | 2 | 1 retry cùng provider rồi failover. G2: amplification chỉ 1.04. Khi thử 5 lần (kèm breaker hỏng, thí nghiệm 1) amplification lên 1.38–1.58 và retry_ratio lên 0.378 nếu không có budget. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter `uniform(0, min(cap, base·2^(n−1)))`: các client không đồng bộ lại thành đợt sóng thứ hai. Retry đầu ≤ 0.25 s nên không phá deadline; cap 4 s để lần retry thứ n vẫn vừa trong 30 s. `retry_after` được tôn trọng tuyệt đối: G1 0 vi phạm (naive 36 027). |
| retry_budget_ratio | 0.1 (+ 0.5/s, max 20 token) | Chặn retry ≲ 10% lưu lượng + chút token hồi. G3 retry_ratio 0.030 (≤ 0.10). Khi tắt budget (thí nghiệm 1) G3 retry_ratio = 0.158 → FAIL. |
| breaker.window_size / min_calls | 20 / 10 | Cửa sổ 20 kết quả, cần ≥ 10 mới xét: không trip vì 2 lỗi lúc rảnh, nhưng ở ~8 call/s cửa sổ đầy trong ~2.5 s nên G2/G6 trip gần như tức thì (recovery 0 s, G2 alpha chỉ nhận 3 525 attempt so với 34 475 của naive). |
| breaker.failure_rate_threshold | 0.5 | G3 chỉ lỗi 30% nên **không** trip (đúng: 70% request vẫn thành công, retry + budget xử lý đủ, availability 1.0) trong khi G2/G6 lỗi ≫ 50% thì trip. |
| breaker.slow_call_s / slow_rate_threshold | 5.0 / 0.6 | G4 không có lỗi nào, alpha chỉ chậm thêm 5 s (latency ≈ 6–7 s). Không có slow-call detection thì breaker mù: naive p95 = 13.76 s. Với ngưỡng 5 s breaker trip theo `slow_call_rate`, p95 còn 5.63 s. |
| breaker.open_duration_s | 10 | Đủ ngắn để quay lại alpha sớm (G2/G4 `primary_share_after_restore` = 1.0, ≥ 0.8) và đủ dài để không dò provider đang chết liên tục. |
| breaker.half_open_max_probes / successes | 2 / 2 | Chỉ 2 request đang bay trong HALF-OPEN → không thundering herd lên provider đang hồi phục; 2 lần thành công thì đóng. |
| quota_cooldown_s | 300 | Hết quota không tự hết sau vài giây; G5 failover ngay sang beta/alpha-eu thay vì đập vào alpha (alpha chỉ nhận ~2 232 attempt, retry_ratio 0.002). |

Sơ đồ quyết định của `on_error`:

```
loại lỗi          ghi breaker?        retry cùng provider?        failover?          cooldown?
CLIENT_ERROR      release (không)     KHÔNG                       KHÔNG -> GiveUp    không   (bug của mình, sai ở mọi provider)
QUOTA_EXHAUSTED   release             KHÔNG                       CÓ, ngay           300 s   (vô ích retry, billing)
RATE_LIMITED      release             KHÔNG (sai lúc)             CÓ, ngay           retry_after (spillover sang alpha-eu/...)
OVERLOADED        on_failure          CÓ nếu tries<2 & budget     sau khi hết lượt   không   (breaker lo)
SERVER_ERROR      on_failure          & vừa deadline              hoặc budget cạn
TIMEOUT           on_failure          (delay = full jitter)
```

Chỉ **một** tầng retry (NaiveClient có 3 tầng = 36 attempt/call). Mỗi provider có breaker riêng; mỗi call luôn bắt đầu từ đầu ladder (`alpha` trước) nên khi breaker alpha đóng lại traffic tự quay về.

## 3. Ba thí nghiệm

Cách chạy: sinh bản sao `gameday.yaml` với tham số đổi, chạy `run_gameday.py --config`. Bảng gốc: seed 25.

1. **Retry budget là lớp phòng thủ cuối.** Breaker hỏng (`min_calls: 1e9`) + `max_attempts_per_provider: 5`:

   | Băng | Có budget (ratio 0.1) | Tắt budget (ratio 100) |
   |---|---|---|
   | G2 | availability 0.9983, amplification 1.378, retry_ratio 0.058, recovery 0 s, p95 19.3 s | availability **0.9754**, amplification **1.580**, retry_ratio **0.378**, recovery **24 s**, p95 21.5 s |
   | G3 | retry_ratio 0.068 ✓ | retry_ratio **0.158** ✗ (> 0.10) |

   Khi breaker bị cấu hình sai, chỉ budget còn chặn được khuếch đại: retry không bao giờ vượt ~10% lưu lượng dù sự cố nặng đến đâu. Khi breaker *đúng* thì ít retry tới được provider chết nên budget gần như không kích hoạt (retry_ratio 0.005 ở G2). Vẫn cần cả hai vì khác lớp: breaker chặn theo **provider đã chết** (nhưng cần `min_calls` kết quả và `open_duration` mới phản ứng, và lỗi 30% như G3 không trip nó); budget chặn theo **tổng lưu lượng** bất kể nguyên nhân (lỗi một phần, config sai, bug). Một cái là mạch ngắt theo mục tiêu, cái kia là cầu chì tổng.

2. **`attempt_timeout_s: 30`.** G4: không đổi (availability 1.0, p95 5.61 s, vì alpha chậm chứ không chết, timeout dài không hại gì). G6 (outage hoàn toàn): availability **1.000 → 0.9488** (FAIL < 0.95). Mỗi attempt vào provider chết treo cả 30 s = đúng bằng deadline của user, nên failover không còn thời gian: user bỏ đi trước khi thử provider kế tiếp. Timeout "đúng" nên dựa vào phân phối latency của provider khoẻ (≈ p99 × hệ số, ở đây p95 ~5 s → 8 s) và vào deadline còn lại để còn dư chỗ cho ≥ 2 attempt, không dựa vào "đủ rộng cho mọi trường hợp".

3. **Bỏ lọc tool calling khỏi `ladder()`** (ladder = cả 4 provider cho mọi task). G6: `incompatible_routes` = **257**, availability **1.000 → 0.8922**. Nói với PM: "257 lần agent gọi công cụ (tra đơn, hoàn tiền) mà lại giao cho model rẻ không biết gọi công cụ. Bước hành động không xảy ra, user nhận câu trả lời nghe có vẻ ổn nhưng việc không được làm — còn tệ hơn lỗi rõ ràng, vì không ai biết để thử lại." Failover phải kiểm tra **tương thích**, không chỉ "còn sống".

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid (cent) | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.000 | 81 | 128 | 81 | 0 | 1 574 200 | 1.123 |
| yours | 1.000 | 66 | **0** | **0** | 0 | 0 | 0.303 |

**Vì sao chỉ idempotency key là chưa đủ?** Key chặn được *lặp hiệu ứng ngoài* (khi `ServiceTimeout` retry cùng key, payments trả lại kết quả đã lưu) nhưng worker chết thì mất toàn bộ bộ nhớ: worker mới không biết step nào xong, phải chạy lại cả classify/lookup (LLM tốn tiền: naive 1.123 LLM lãng phí/crash) và không có `refund_id` để tiếp. **Vì sao chỉ checkpoint là chưa đủ?** Có một khe không thể loại bỏ: tiền đã đi xong, worker chết *trước khi* kịp ghi checkpoint → resume sẽ gọi `issue_refund` lần nữa; không có key ổn định thì hoàn tiền lần hai (naive: 128 hoàn tiền trùng). Hai cơ chế bổ sung nhau: checkpoint cho tiến độ, key làm khe đó an toàn.

**Nếu ghi checkpoint *trước* khi gọi payment?** Checkpoint nói "đã xong" khi chưa xong: worker chết giữa chừng thì lần resume bỏ qua step, tiền không bao giờ được hoàn (**lost refund**) và email còn báo "đã hoàn". Vì vậy phải ghi **sau** khi step thành công (at-least-once execution + idempotency = exactly-once effect). `WorkerCrash` không được bắt: nếu nuốt nó, process "sống tiếp" với state sai thay vì để runner dựng worker mới resume từ store.

Key = `refund:{session_id}:{step}`, checkpoint = `checkpoint:{session_id}:{step}`: ổn định theo (session, step), khác nhau giữa session và giữa step, không dùng `uuid4()`.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter (5 phút > 5%) | Burn-rate alerter | Độ trễ phát hiện (burn-rate) |
|---|---|---|---|---:|
| checkout_bug | page | page (trễ 1 phút) | page ✓ | 27 phút (≤ 35) |
| provider_outage | page | page (trễ 0) | page ✓ | 4 phút (≤ 8) |
| deploy_blip | none | **page sai** | không báo ✓ | — |
| slow_leak | ticket | **không phát hiện** | ticket ✓ (không page) | 1 337 phút (≤ 1 800) |
| degraded_evening | page | page (trễ 214) | page ✓ | 291 phút (≤ 330) |
| night_spike | none | **page sai** | không báo ✓ | — |
| small_bump | none | **page sai** | không báo ✓ | — |

Naive: 7 page, 3 false page, bỏ sót rò rỉ chậm (FAIL). Burn-rate: 3 page, 1 ticket, 0 false page (PASS).

**Error budget 30 ngày** (`error_budget_report`): 25 901 755 request, cho phép 129 508.8 bad, thực tế 55 618 bad → availability 0.997853, **đã tiêu 42.95%, còn 57.05%**. Chưa hết budget nên **không đóng băng release** (nhưng 3 sự cố page + rò rỉ 2 ngày là tín hiệu nên ưu tiên độ tin cậy trong sprint kế).

**Vì sao `small_bump` (20 phút ở 12% lỗi lúc 11 giờ trưa) không nên đánh thức ai?** Burn rate 5 phút là 24× nhưng cửa sổ dài thì thấp: 1h = 20×12%/60 = 4% lỗi → 8× (< 14.4×), 6h = 0.67% → 1.3× (< 6×). Luật yêu cầu *cả hai* cửa sổ vượt nên không có page nào. Nó đáng xem lại vào sáng hôm sau (ticket), không đáng làm ai mất ngủ. Ngưỡng 5 phút đơn lẻ (naive) cho ra đúng 3 false page loại này.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình: `reports/gameday/G2_overload_storm.png` (`make plot`); số liệu: `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt:** Lúc t=120 s alpha mất 75% GPU (trigger). Client v1 chồng 3 tầng retry không jitter, không budget, không breaker nên tải thực tế lên alpha tăng từ ~7.5 lên 254 attempt/s (×34), lấp đầy hàng đợi bằng request mà user đã bỏ đi; kể cả khi GPU trở lại lúc t=300, tải tự duy trì khiến hệ thống tiếp tục sập thêm 152 s (đến t=452). 54.4% lượt user (1 285/2 362) bị lỗi.

**Ảnh hưởng:** 2 362 turn, chỉ 1 077 thành công → availability 0.456 (SLO 99.5%); p95 15.0 s; 28 381 vi phạm `retry-after`; 34 475 attempt cho ~2 362 turn (amplification 9.9, chi phí/turn 4.63 so với 2.11 của client mới). Nếu tháng chỉ có sự cố này, với ~4 turn/s (10.37 triệu turn/tháng) ngân sách là 51 840 turn lỗi; 1 285 turn = **2.5% budget tháng** (trong khoảng 5 phút ảo — burn rate trung bình ≈ 109×, chạm ngưỡng page 14.4× ngay phút đầu).

**Timeline** (giây ảo — từ CSV; "attempt/s" = tổng attempts mọi provider):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 0–119 | Bình thường | ~7.5 attempt/s trung bình, gần như mọi call thành công (t=119: 7 ok / 7 done / 9 attempt) |
| 120 | alpha mất 75% năng lực (trigger) | t=121: chỉ còn 2 ok, t=122: 0 ok; capacity 25% nhưng tải đến vẫn ~4 turn/s |
| 120–136 | Suy giảm, retry bắt đầu chồng | thành công 2–5/s; attempt/s tăng 5 → 53 (t=136) |
| 137 | **Sụp đổ**: không còn call nào thành công | t=137: 0 ok, 82 attempt/s; t=140: 216; **đỉnh t=143: 254 attempt/s (×34 so với bình thường)** |
| 140–300 | Bão retry: toàn bộ attempt vào alpha | trung bình **150.5 attempt/s**, 0 thành công trong cả 163 s |
| 300 | alpha hồi phục hoàn toàn (GPU về) | vẫn 115 attempt/s tại t=300, 0 ok → hệ thống KHÔNG tự lành |
| 300–452 | Metastable: provider khoẻ mà user vẫn lỗi | attempt/s giảm dần 115 → 11 (t=310) nhưng vẫn 50 trung bình trong 300–450; 0 ok suốt 140→452 |
| 453 | Hồi phục (backlog cạn, deadline user hết) | t=453: ≥3 ok/s; `recovery_s` = 155 s sau restore |

**Trigger vs root cause:** *Trigger* là mất 75% GPU của alpha — một sự kiện bình thường mà hệ thống phải chịu được (nhiều provider, có dự phòng). *Root cause* là thiết kế client: (1) **3 tầng retry nhân nhau** (SDK × HTTP × orchestration = tới 36 attempt/call), (2) **không jitter** nên các client đồng bộ lại thành sóng, (3) **bỏ qua `retry-after`**, (4) **không retry budget và không circuit breaker** nên không có thứ gì dừng khuếch đại, (5) **không failover** — mọi attempt vẫn đổ vào alpha dù có alpha-eu/beta rảnh.

**Vì sao không tự hồi phục sau t=300 (metastable failure):** có vòng lặp tự duy trì: alpha chậm → client timeout → client *bỏ* nhưng server **vẫn xử lý** request đã timeout (hàng đợi đầy request mồ côi, vô ích) → request mới phải chờ sau hàng đợi đó nên lại timeout → mỗi timeout sinh thêm retry (×9.9) → hàng đợi lại đầy. Tải hiệu dụng (≈ 150 attempt/s) cao hơn nhiều năng lực hồi phục của alpha (khoảng 18 server / ~1.2 s ≈ 15 req/s), nên dù GPU đã về 100%, hệ thống ở trạng thái "ổn định nhưng hỏng" cho đến khi backlog tự cạn khi deadline 30 s của các turn cũ hết hạn. Hồi phục bằng cách *dừng cấp thêm tải*, không phải bằng cách thêm năng lực.

**Điều gì giúp:** server có `max_queue_wait_s` 10 s nên không nghẽn mãi mãi (đó là lý do cuối cùng nó cũng tự hồi phục sau ~150 s); deadline user 30 s cắt backlog; telemetry theo giây (CSV) cho phép dựng lại timeline. **Điều gì làm tệ hơn:** retry không jitter đồng bộ thành sóng; bỏ qua `retry-after`; không có kill-switch để người trực tắt retry; không có alert theo amplification/attempt-per-call nên không ai thấy tải ×34.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay NaiveClient bằng ResilientClient: **một** tầng retry duy nhất, full jitter, tôn trọng `retry-after` (đã làm trong repo) | prevent | Nhóm Platform/Client SDK (owner: Tuyến) | G2: amplification ≤ 1.2 (hiện 1.04), 0 vi phạm retry-after, chạy trong CI mỗi PR |
| 2 | Retry budget (ratio 0.1) + circuit breaker từng provider có slow-call detection; failover có lọc tương thích tool | mitigate | Platform | G2/G3 PASS khi tắt breaker (thí nghiệm 1: G3 retry_ratio ≤ 0.10); `incompatible_routes = 0` ở G6 |
| 3 | Alert burn-rate đa cửa sổ (14.4× 1h&5m …) cộng alert riêng **attempts/call > 1.5** và tỷ lệ retry | detect | SRE on-call | Diễn tập G2 bằng client cũ: page trong < 5 phút kể từ t=120, 0 false page trên `sli_30d.csv` |
| 4 | Server/gateway: huỷ request khi client đã ngắt (propagate deadline, load-shed), nhận `Retry-After` thật | mitigate | Nhóm Provider gateway | Game day: sau restore hàng đợi về 0 trong ≤ 30 s (hiện `recovery_s` naive = 155 s) |
| 5 | Runbook + kill-switch: tắt retry/giảm tải ở tầng gateway khi amplification > 3 | mitigate | SRE on-call | Diễn tập hàng quý, thời gian từ page đến giảm tải ≤ 5 phút |

Tinh thần blameless: không ai "viết sai retry"; thiết kế mặc định của 3 lớp SDK (mỗi lớp hợp lý riêng lẻ) tạo ra tương tác nguy hiểm mà quy trình review/test tải chưa phát hiện — action items nhắm vào hệ thống, test và vận hành.

## 7. Một điểm yếu còn lại

**Cả bốn provider cùng chậm hoặc cùng lỗi / beta gánh tải dài.** Breaker mỗi provider độc lập nên khi mọi provider chậm cùng lúc, mọi breaker mở thì `GiveUp("no_provider_available")`; user nhận câu trả lời tĩnh. Ngoài ra khi alpha chết kéo dài, beta (cost 1.4) gánh phần lớn: ở G6 beta nhận 2 124 attempt, cost/turn 2.196 so với 2.104 ở G1. G5 chỉ đạt 0.8179 vì 413 turn bị lỗi 400 không thể cứu. G2 p95 = 10.36 s vì trước khi breaker mở, vài request đầu đợi cả `max_queue_wait_s` = 10 s. Cách sửa: (a) hedged request / giảm timeout theo p99 đo thực tế cho từng provider; (b) cấp ngân sách chi phí và alert khi tỷ lệ traffic sang provider đắt vượt ngưỡng; (c) bậc cuối trong ladder là câu trả lời cache/tĩnh có nhãn rõ thay vì lỗi; (d) load-shed ưu tiên (giữ chat nhẹ, giảm tool-call phụ).

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`, seed 25, băng C1).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | ✗ |
| cached, starter (0.80 / 600 / 0) | 0.8469 | 0.8587 | 0.742 | 0 | **181** (`other_question`) | **80** | 0.921 | ✗ |
| cached, của mình (0.90 / 60 / 300) | **0.9124** | **0.9645** | **0.652** | 157 | **0** | **3** | **0.915** | ✓ |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.90 | Paraphrase cùng ý có cosine ~0.94–0.98; ý "gần mà khác" (refund policy vs refund time, phí ship nội thành vs ra tỉnh…) ~0.80–0.86. 0.90 nằm giữa hai cụm. Ở 0.80: `wrong_answers` = 175–181 (toàn `other_question`). Ở 0.90 và 0.97: 0 sai, nhưng 0.97 giảm hit ratio 0.652 → 0.579 và cost 0.915 → 0.965 (bỏ lỡ cả paraphrase hợp lệ). |
| ttl_s | 60 | `ttl_s` là thứ duy nhất giới hạn `outdated_answers` (webhook không báo gì về thay đổi bảng phí ship lúc t=480). TTL 30: outdated 2, hit 0.592; **60: outdated 3, hit 0.652**; 120: outdated 24 (FAIL > 15); 300: outdated **95**, 600: 80. 60 s là điểm cân bằng hit ratio ≥ 0.5 và outdated ≤ 15. |
| max_stale_s | 300 | Phải dài hơn thời gian outage (t=240–360 = 120 s) cộng phần TTL đã hết: một entry lưu ở t=150 tuổi 210 s lúc t=360. Với 300 s (ttl+stale = 360 s) đủ phủ. Với 0: faq_availability 0.9645 → **0.8733** (stale_served 157 → 0). |
| max_entries | 500 | Không đổi: catalog FAQ nhỏ hơn nhiều, LRU không bao giờ phải đuổi trong băng này. |

Thí nghiệm:

1. **Ngưỡng 0.80 / 0.90 / 0.97:** hit ratio 0.659 / 0.652 / 0.579; wrong_answers 175 / 0 / 0; cost/turn 0.978 / 0.915 / 0.965; faq_avail 0.8605 / 0.9645 / 0.964. Ở 0.80 các cặp "gần mà khác" bị trả nhầm: ví dụ hỏi "Bao lâu thì nhận được tiền hoàn?" (refund_time) lại nhận câu trả lời của "Chính sách hoàn tiền thế nào?" (refund_policy) — cosine ~0.86 ≥ 0.80. Đây là lỗi tệ nhất vì tự tin, trôi chảy và sai.
2. **TTL 30 / 60 / 300:** outdated_answers 2 / 3 / 95; hit ratio 0.592 / 0.652 / 0.731. Webhook chỉ invalidate khi hệ thống nguồn *có báo* (policy refund lúc t=150 → wrong/outdated = 0 nhờ `on_content_update`); thay đổi bảng phí ship không báo trước (t=480) thì không sự kiện nào gọi được `invalidate`, nên chỉ hết hạn TTL mới thoát ra khỏi cache. TTL càng dài thì càng nhiều câu cũ.
3. **`max_stale_s: 0`:** faq_availability 0.9645 → 0.8733, availability 0.9124 → 0.8565, và C1 FAIL. Khi mọi provider sập 120 s, 157 câu FAQ được phục vụ từ bản cũ thay vì lỗi.

**Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả theo từng user?** (1) Embedding gần như y hệt giữa hai user ("đơn #123" ≈ "đơn #124", cosine ~0.99) nên một cache ngữ nghĩa sẽ trộn user này với user khác — rò rỉ dữ liệu cá nhân; (2) ngay cả khi key theo user, câu trả lời phụ thuộc trạng thái đơn hàng *thay đổi liên tục* (đã giao chưa, hoàn tiền xong chưa), không có tài liệu nguồn nào để webhook invalidate, nên TTL nào cũng có nguy cơ sai kiểu "đơn của bạn vẫn đang giao" khi đã giao. Sai ở đây vừa là lỗi nghiệp vụ vừa là lỗi riêng tư.

**Khi mọi provider sập mà dashboard vẫn xanh nhờ cache, alert theo số nào?** Alert trên **sức khoẻ provider và mức độ phụ thuộc cache**, không chỉ SLI user-facing: (a) tỷ lệ lỗi/`GiveUp("no_provider_available")` ở tầng provider và số breaker đang OPEN; (b) `stale_served / FAQ turns` (burn rate riêng); (c) cache_hit_ratio cao đột biến + tỷ lệ miss thất bại; (d) tuổi của câu trả lời stale lớn nhất đang phục vụ so với `ttl_s + max_stale_s` (sắp hết đệm). Page khi stale_served > ngưỡng trong cả cửa sổ dài và ngắn — vì cache chỉ mua thời gian (ở đây 300 s), không sửa được sự cố.
