# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Đào Đức Hải
- **MSSV:** 2A202602752
- **Lớp:** K4-L3A
- **Repository URL:**
- **Commit SHA cuối:**
- **Challenge ID:**
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602752`

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
| `validate_logs.py` | 30/100 | | Thiếu correlation_id và enrichment context (chưa làm CP1) |
| `validate_dashboard.py` | 6/6 panel | | Contract schema 6/6 panel hợp lệ |
| `pytest` | 22 passed | | Toàn bộ 22 unit tests đều pass |
| Số traces hợp lệ | 0 | | Chưa tạo child observations (retriever/generation) |
| Số PII leak | 0 | | Chưa phát hiện leak trong bộ sample query hiện tại |
| Latency P95 / TTFT P95 | 3776 ms / 50 ms | | P95 cao do request khởi tạo cold-start ban đầu |
| Retrieval success rate | 100% | | Hệ thống hoạt động bình thường, chưa inject incident |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `CorrelationIdMiddleware`, trước mỗi request gọi `clear_contextvars()` để tránh rò rỉ context giữa các request. Kiểm tra header `x-request-id`: nếu có thì sử dụng, nếu không thì sinh mới theo mẫu `req-<8-hex>` (`req-{uuid.uuid4().hex[:8]}`). Sau đó bind vào structlog context qua `bind_contextvars(correlation_id=correlation_id)`, lưu vào `request.state.correlation_id`, đồng thời trả về client qua header `x-request-id` và `x-response-time-ms`.
- **Các metadata được ghi vào structured log:** Gồm các trường định danh và ngữ cảnh: `correlation_id`, `user_id_hash` (hash SHA256 12 ký tự hex đầu), `session_id`, `feature`, `model`, `env`. Ở log phản hồi (`response_sent`) bổ sung thêm: `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Đăng ký processor `scrub_event` đứng trước `JsonlFileProcessor` và `JSONRenderer` trong pipeline structlog. Hàm `scrub_event` duyệt đệ quy qua các trường dữ liệu và gọi `scrub_text` để che giấu các mẫu PII (email, phone Việt Nam, CCCD 12 số, thẻ thanh toán 16 số) thành nhãn `[REDACTED_<TYPE>]` trước khi chuỗi JSON được tạo và ghi ra file `data/logs.jsonl`.
- **Cách kiểm chứng kết quả:** Chạy `python scripts/validate_logs.py` đạt 100/100 (0 missing required, 0 missing context, 0 PII leaks, 10 unique correlation IDs). Chạy `pytest` đạt 24/24 tests pass (bao gồm các test case cho email, đa dạng format số điện thoại VN, CCCD và số thẻ tín dụng).

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:**
- **Cấu trúc root/retrieval/generation observations:**
- **Cách nối trace với log:**
- **Prompt name:**
- **Version/label baseline:**
- **Version/label candidate:**
- **Trace ID của mỗi version:**
- **Cách promote và rollback `production`:**

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
- **SLO và lý do chọn:**
- **Cách tính error budget:**
- **Ba alert và runbook tương ứng:**

## 7. Điều tra challenge

- **Challenge ID:**
- **Khoảng thời gian điều tra:**
- **Triệu chứng từ metrics:**
- **Log line và correlation ID liên quan:**
- **Trace ID và span gây ảnh hưởng:**
- **Root cause:**
- **Fix action:**
- **Preventive measure:**

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:**
- **Một lỗi/blocker đã gặp:**
- **Cách tìm nguyên nhân và xử lý:**
- **Cách hiểu luồng Metrics → Logs → Traces:**
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
- **Điều quan trọng nhất đã học:**
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:**

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
