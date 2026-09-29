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

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Cấu hình API key riêng trên Langfuse Cloud (`day13-k4-l3a-2A202602752`), mỗi trace mang mã băm user id của học viên (`2055254ee30a`), tags `["lab", feature, "claude-sonnet-4-5"]`, và `environment="dev"`.
- **Cấu trúc root/retrieval/generation observations:** Root observation `lab-agent-run` (type `AGENT`) bao bọc toàn bộ request. Phía trong gồm 2 child observations: `retrieval` (type `RETRIEVER`) đo thời gian tra cứu vector/mock RAG, và `llm-generate` (type `GENERATION`) đo thời gian sinh câu trả lời của mô hình, ghi nhận model, prompt version, input/output tokens và chi phí USD.
- **Cách nối trace với log:** Thông qua `correlation_id` (ví dụ `req-7054d41e`). Middleware tạo và bind correlation ID vào contextvars log, đồng thời chuyển vào metadata của Langfuse trace (`metadata={"correlation_id": correlation_id}`). Nhờ đó có thể từ một dòng log bất kỳ nhảy sang trace tương ứng trên Langfuse để xem span waterfall.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version `1`, mang labels `baseline` và `production`.
- **Version/label candidate:** Version `2`, mang label `candidate` (chỉnh sửa template thêm yêu cầu trả lời ngắn gọn theo gạch đầu dòng).
- **Trace ID của mỗi version:**
  - Version 1 (baseline/production): Trace ID `7293ca0d6e5ca95add13ff71392b26ff` (correlation ID: `req-7054d41e`).
  - Version 2 (candidate): Trace ID `380cb6620569c2a8dd38e345db9ca260` (correlation ID: `req-f0ab17c6`).
- **Cách promote và rollback `production`:**
  - Promote: Dùng Langfuse SDK hoặc giao diện UI chuyển label `production` sang Version 2 (`client.update_prompt(name='day13-chat', version=2, new_labels=['candidate', 'production'])`).
  - Rollback: Khi cần hoàn nguyên về Version 1, cập nhật lại label `production` cho Version 1 (`client.update_prompt(name='day13-chat', version=1, new_labels=['baseline', 'production'])`). Hệ thống tự động fetch prompt theo label `production` nên lập tức quay về phiên bản cũ mà không cần restart hay sửa code.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Dựng đúng contract `config/dashboard.yaml` lấy nguồn từ `data/logs.jsonl`:
  1. `latency`: P50, P95, P99 và TTFT P95 (ngưỡng P95 <= 3000ms).
  2. `traffic`: Số lượng request và request rate theo phút (ngưỡng >= 1 req/min).
  3. `errors`: Error rate %, breakdown theo error_type và retrieval success rate % (ngưỡng error <= 2%).
  4. `cost`: Chi phí USD theo phút và tổng lũy kế (ngưỡng <= $2.5).
  5. `tokens`: Tổng tokens_in và tokens_out (ngưỡng <= 50,000 tokens).
  6. `quality`: Điểm chất lượng trung bình của câu trả lời proxy (ngưỡng mean >= 0.75).
- **SLO và lý do chọn:** Primary SLO là `fast_successful_requests` với target 99.5% trong cửa sổ 28 ngày (`latency_ms <= 3000` và `response_sent`). Lý do: Đáp ứng tương tác người dùng thời gian thực, đảm bảo dịch vụ phản hồi ổn định và không làm người dùng chờ đợi quá 3 giây.
- **Cách tính error budget:** Error budget = 100% - 99.5% = 0.5% tổng số request trong cửa sổ 28 ngày. Với 100,000 requests, ngân sách lỗi cho phép là 500 requests thất bại hoặc vượt quá 3000ms.
- **Ba alert và runbook tương ứng:**
  1. `HighLatencyP95`: P95 > 2500ms trong 3 phút (warning, Slack #alerts-latency, Runbook: `docs/alerts.md#alert-1`).
  2. `HighErrorRate`: Error rate > 2% trong 2 phút (critical, Slack #alerts-critical, Runbook: `docs/alerts.md#alert-2`).
  3. `CostSpikeAnomaly`: Chi phí > $0.05/phút trong 5 phút (warning, Slack #alerts-finops, Runbook: `docs/alerts.md#alert-3`).

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
