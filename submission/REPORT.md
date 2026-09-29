# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** K4-L3-DAY13-NguyenHoNam
- **MSSV:** 2A202602788
- **Lớp:** K4-L3A
- **Repository URL:** <Your Repo URL>
- **Commit SHA cuối:** <Your Commit SHA>
- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602788`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | <Chưa rõ> | 100/100 | Validator chạy mượt, pass toàn bộ PII và schema |
| `validate_dashboard.py` | <Chưa rõ> | HỢP LỆ 6/6 panel | Đã setup đủ panel theo contract |
| `pytest` | Fail | 22/22 passed | Không còn lỗi nào trong test |
| Số traces hợp lệ | 0 | >10 | Load test đã tạo nhiều request hợp lệ |
| Số PII leak | <Chưa rõ> | 0 | Scrubbing hoạt động hiệu quả |
| Latency P95 / TTFT P95 | <Chưa rõ> | ~2650ms / ~50ms | Bị tác động tăng vọt do incident `rag_slow` timeout |
| Retrieval success rate | <Chưa rõ> | 100% | RAG trả về kết quả |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `CorrelationIdMiddleware`, ưu tiên lấy header `x-request-id`, nếu thiếu thì tự sinh ra ID qua UUID (format `req-<8-hex>`). Dùng `bind_contextvars` của structlog để lưu correlation ID vào context chung.
- **Các metadata được ghi vào structured log:** `user_id_hash`, `session_id`, `feature`, `model`, `env`, `correlation_id`, `latency_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Cấu hình processor `scrub_event` trong `app/logging_config.py` trước khi serialize dict ra dạng JSON. Regex tự động filter thông tin thẻ/email/sdt thành `[REDACTED]`.
- **Cách kiểm chứng kết quả:** Chạy script `scripts/validate_logs.py` để đảm bảo score đạt 100/100 điểm.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Cấu hình `LANGFUSE_SECRET_KEY` trong `.env`, tạo prompts v1, v2 trên cloud và xem Trace list.
- **Cấu trúc root/retrieval/generation observations:** Decorator `@observe(as_type="span")` trên hàm `_do_retrieve` và `@observe(as_type="generation")` trên hàm `_do_generate`. Cả 2 được gọi bên dưới scope cha `run()` mang `@observe(as_type="agent")`.
- **Cách nối trace với log:** Sử dụng chung một `correlation_id` (được đưa vào metadata của trace bằng block `propagate_attributes`).
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 (label: `baseline`, `production`)
- **Version/label candidate:** Version 2 (label: `candidate`)
- **Trace ID của mỗi version:** (Sau khi chạy, xem Trace trong Langfuse UI và lấy ID ghi vào đây)
- **Cách promote và rollback `production`:** Sử dụng dashboard UI Langfuse, xóa label `production` khỏi v2 và gán lại label `production` cho v1.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Tuân theo config `dashboard.yaml`, tạo đủ 6 panel với các metrics Latency, Error, Cost, Tokens.
- **SLO và lý do chọn:** Latency `P95 < 3000ms`, vì Time-To-First-Token của LLM model thường rất nhanh (~150ms), và RAG bình thường dưới 500ms, nên request mất quá 3s là không đạt chuẩn.
- **Cách tính error budget:** `100% - Target (99.5%) = 0.5%` budget.
- **Ba alert và runbook tương ứng:** HighLatency (P95 > 3s), HighErrorRate (Rate > 2%), HighCost (Cost > 2.5$). Runbook được cập nhật trong `docs/alerts.md`.

## 7. Điều tra challenge

- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Khoảng thời gian điều tra:** Ghi giờ hiện tại lúc chạy sự cố.
- **Triệu chứng từ metrics:** P95 Latency của feature `monitoring` tăng vọt lên trên 2600ms, vi phạm ngưỡng SLO threshold (2000ms theo challenge) kéo theo Error budget sụt giảm.
- **Log line và correlation ID liên quan:** Filter event `response_sent` thấy request mất ~2653ms với `correlation_id` như `req-e8852ec9`.
- **Trace ID và span gây ảnh hưởng:** Tra cứu Trace có `correlation_id` = `req-e8852ec9`, mở waterfall xem thì thấy rõ span `retrieval` bị kẹt đúng `2.5s`.
- **Root cause:** Trong logic RAG, STATE incident `"rag_slow"` bị kích hoạt, gọi `time.sleep(2.5)` mô phỏng lỗi DB quá tải chậm truy vấn.
- **Fix action:** Reset trạng thái lỗi (Tối ưu truy vấn vector hoặc scale db) để RAG nhanh lại như cũ.
- **Preventive measure:** Thêm circuit breaker/timeout cực hạn (ví dụ `1.0s`) ngay ở `_do_retrieve` để ném lỗi thay vì treo app quá lâu, kết hợp fallback.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Decorate chia nhỏ `_do_generate` và `_do_retrieve` thành các child span trong Langfuse. Nhờ bóc tách như thế nên khi xảy ra incident em mới dễ dàng chứng minh `retrieval` là thủ phạm gây slow thay vì đổ lỗi cho mô hình LLM.
- **Một lỗi/blocker đã gặp:** App báo lỗi `No connection could be made` khi chạy `load_test.py`
- **Cách tìm nguyên nhân và xử lý:** Lỗi kết nối HTTP đồng nghĩa API Backend chưa chạy, em cần chạy ngầm `uvicorn app.main:app` bằng terminal phụ rồi mới start HTTP requests.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics để phát hiện triệu chứng diện rộng nhanh nhất -> Logs để khoanh vùng chi tiết và tìm `correlation_id` của request gặp bệnh -> Traces dựa vào ID để xem luồng gọi API và tìm thủ phạm.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Prompt version hỗ trợ A/B test hiệu năng Prompt. Alert Cost/Token chống spam. Rollback kịp thời cứu vãn khi đưa Candidate prompt lỗi lên Production.
- **Điều quan trọng nhất đã học:** Kỹ thuật Observability áp dụng structlog, context variables và Langfuse tracing vào 1 Request liên kết mạch lạc với nhau.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Các metric alert hiện đang dừng lại ở mockup file markdown chưa bắn về slack bot thật.

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
