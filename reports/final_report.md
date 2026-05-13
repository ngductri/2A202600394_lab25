# Day 10 Reliability Final Report

## Metrics Summary

| Metric | Value |
|---|---:|
| total_requests | 400 |
| availability | 0.9975 |
| error_rate | 0.0025 |
| latency_p50_ms | 470.25 |
| latency_p95_ms | 823.02 |
| latency_p99_ms | 858.49 |
| fallback_success_rate | 0.9773 |
| cache_hit_rate | 0.75 |
| circuit_open_count | 5 |
| recovery_time_ms | None |
| estimated_cost | 0.05048 |
| estimated_cost_saved | 0.3 |

## Chaos Scenarios

| Scenario | Status |
|---|---|
| primary_timeout_100 | pass |
| primary_flaky_50 | pass |
| all_healthy | pass |
| cache_stale_candidate | pass |

## 1. Architecture Summary
**Luồng xử lý (Architecture):** 
User -> Gateway -> Cache (In-memory/Redis) -> Circuit Breaker -> Primary Provider -> Fallback Provider -> Static Fallback.
Hệ thống sử dụng Circuit Breaker với 3 trạng thái (CLOSED, OPEN, HALF_OPEN) để bảo vệ các provider. Nếu provider lỗi, yêu cầu sẽ tự động nhảy sang fallback provider. Ngoài ra, Redis Cache được tích hợp để chia sẻ dữ liệu giữa nhiều instances.

## 2. Configuration Table

| Setting | Value | Why this value |
|---|---:|---|
| failure_threshold | 3 | Vừa đủ để phát hiện lỗi nhanh mà không bị đóng nhầm do mạng chập chờn. |
| reset_timeout_seconds | 2 | Khớp với thời gian trung bình để một provider phục hồi. |
| cache TTL | 300 | 5 phút là đủ độ tươi mới cho các câu hỏi phổ biến mà vẫn giữ được tỷ lệ cache hit cao. |
| similarity_threshold | 0.92 | Ngưỡng này giúp tránh nhầm lẫn giữa các câu hỏi gần giống nhau (như năm 2024 và 2026). |

## 3. Cache Comparison
| Metric | Without cache | With cache |
|---|---:|---:|
| latency_p50_ms | ~522 | ~470 |
| estimated_cost | 0.182 | 0.050 |
| cache_hit_rate | 0.0 | 0.75 |

- **Guardrails:** Đã cài đặt kiểm tra bảo mật `_is_uncacheable` (ẩn số thẻ tín dụng, thông tin cá nhân) và `_looks_like_false_hit` (giúp tránh nhầm lẫn "năm 2024" và "năm 2026"). Nếu không có guardrail này, hệ thống sẽ trả về false-hit khi đổi các con số mang ý nghĩa khác biệt.

## 4. Redis Shared Cache
- **Lý do cần Shared Cache:** Trên môi trường production thực tế, chúng ta sẽ có nhiều container chạy Gateway. Redis giúp các container này chia sẻ chung một bộ nhớ cache, tối ưu chi phí và tốc độ. Bằng chứng là các container khác nhau có thể truy cập cùng một key (như `rl:cache:9e413fd814eb`).

## 5. Failure Analysis & Next Steps
- **Điểm yếu (Failure analysis):** Hiện tại `recovery_time_ms` bị None/sai trong một số trường hợp do cách tính thời gian trong log state chưa bao phủ hết lúc mô phỏng kết thúc sớm. Hơn nữa, Gateway chưa có cơ chế giới hạn số lượng request từ một user (Rate Limiting).
- **Next steps (Hướng khắc phục):** 
  1. Thêm Rate Limiter (dựa trên Redis) để chống spam/DDoS.
  2. Bổ sung cơ chế tự động chuyển hướng theo chi phí (nếu xài quá 80% ngân sách thì chỉ dùng model rẻ tiền).