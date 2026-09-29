# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: HighLatency
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack
- SLI/SLO liên quan: fast_successful_requests
- Điều kiện và thời gian duy trì: p95_latency_ms > 3000ms duy trì trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ lâu để nhận được phản hồi, gây trải nghiệm kém.
- Ba bước kiểm tra đầu tiên: Xem dashboard độ trễ, kiểm tra logs lỗi, xem traces để tìm công đoạn chậm.
- Mitigation tạm thời: Khởi động lại dịch vụ hoặc fallback về mô hình nhẹ hơn.
- Owner: SRE Team

## Alert 2

- Tên: HighErrorRate
- Severity: critical
- Duration: 5m
- Kênh thông báo: Slack
- SLI/SLO liên quan: error_rate_pct_max
- Điều kiện và thời gian duy trì: Lỗi vượt mức 2% duy trì trong 5 phút
- Ảnh hưởng tới người dùng: Nhiều yêu cầu bị từ chối hoặc lỗi, không thể sử dụng tính năng.
- Ba bước kiểm tra đầu tiên: Xem logs lỗi, kiểm tra trạng thái API LLM, kiểm tra mạng.
- Mitigation tạm thời: Rollback phiên bản mới hoặc tắt tính năng gây lỗi.
- Owner: SRE Team

## Alert 3

- Tên: HighCost
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack
- SLI/SLO liên quan: daily_cost_usd_max
- Điều kiện và thời gian duy trì: Chi phí vượt quá $2.5 trong khoảng thời gian ngắn
- Ảnh hưởng tới người dùng: Không ảnh hưởng trực tiếp người dùng, nhưng lãng phí chi phí.
- Ba bước kiểm tra đầu tiên: Kiểm tra token usage, kiểm tra truy vấn rác, xác nhận cấu hình LLM.
- Mitigation tạm thời: Đặt giới hạn token, block IP spam.
- Owner: FinOps Team
