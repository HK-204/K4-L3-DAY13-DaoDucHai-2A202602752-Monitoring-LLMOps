# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Đào Đức Hải
- **MSSV:** 2A202602752
- **Lớp:** K4-L3A
- **Repository URL:**
- **Commit SHA cuối:**
- **Challenge ID:** practice-rag_slow
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
| `validate_logs.py` | 30/100 | 100/100 | Đạt chuẩn schema, đủ correlation ID, enrichment context, scrub sạch PII |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Contract schema 6/6 panel đầy đủ và hợp lệ |
| `pytest` | 22 passed | 24 passed | Toàn bộ unit tests đều pass (bổ sung test CCCD, credit card) |
| Số traces hợp lệ | 0 | 25+ traces | Đầy đủ span tree: root (agent), retrieval (retriever), llm-generate (generation) |
| Số PII leak | 0 | 0 leak | Không phát hiện rò rỉ dữ liệu nhạy cảm trong log và trace |
| Latency P95 / TTFT P95 | 3776 ms / 50 ms | 157 ms / 50 ms (bình thường), 2653 ms (incident) | Phản ánh chính xác triệu chứng sự cố và khôi phục tốt |
| Retrieval success rate | 100% | 100% | Hệ thống tra cứu tài liệu ổn định |

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

- **Challenge ID:** `practice-rag_slow` (Thực hiện điều tra practice scenario theo đúng quy trình chuẩn của `docs/DASHBOARD_SETUP.md` và `README.md`).
- **Khoảng thời gian điều tra:** `2026-09-29 09:37:00 UTC` - `2026-09-29 09:40:00 UTC` (16:37 - 16:40 giờ Việt Nam).
- **Triệu chứng từ metrics:** Độ trễ tăng đột biến diện rộng: `latency_p50 = 2651.0 ms`, `latency_p95 = 2653.0 ms`, `latency_p99 = 12272.0 ms` (vi phạm ngưỡng SLO 3000ms và Alert 2500ms), trong khi `ttft_p95` vẫn giữ nguyên ở mức 50ms và không có lỗi crash HTTP 5xx.
- **Log line và correlation ID liên quan:**
  - `correlation_id`: `req-002b3ffb`
  - Log `request_received`: `{"service": "api", "payload": {"message_preview": "What is your refund policy? My email is [REDACTED_EMAIL]"}, "event": "request_received", "correlation_id": "req-002b3ffb", "ts": "2026-09-29T09:37:11.159093Z"}`
  - Log `response_sent`: `{"service": "api", "latency_ms": 2652, "ttft_ms": 50, "tokens_in": 45, "tokens_out": 115, "cost_usd": 0.00186, "quality_score": 0.9, "tool_name": "retrieval", "tool_success": true, "correlation_id": "req-002b3ffb", "event": "response_sent", "ts": "2026-09-29T09:37:13.812891Z"}`
- **Trace ID và span gây ảnh hưởng:**
  - Trace ID: `4195abd30d6346ca44f06384f2e38602`
  - Span gây ảnh hưởng: `retrieval` (type `RETRIEVER`) mất **2.501 giây** trên tổng số **2.652 giây** của toàn bộ request (chiếm ~94.3% độ trễ). Trong khi đó span `llm-generate` (type `GENERATION`) chỉ mất **0.150 giây** (150ms).
- **Root cause:** Tầng tra cứu vector store / RAG backend (`mock_rag.retrieve`) bị nghẽn làm phát sinh thêm 2.5s độ trễ cho mỗi truy vấn, không phải do mô hình LLM sinh từ chậm.
- **Fix action:**
  - Tắt kịch bản incident thông qua endpoint `/incidents/rag_slow/disable`.
  - Kiểm tra hiệu năng và tối ưu chỉ mục (indexing) của Vector Database, scale thêm read replica cho cụm search service.
- **Preventive measure:**
  - Thiết lập timeout cho bước retrieval (ví dụ 1.5s). Nếu quá thời gian, tự động fallback trả lời ngữ cảnh cơ bản để không chặn request của người dùng.
  - Triển khai Semantic Caching cho các câu hỏi tra cứu phổ biến nhằm giảm tải trực tiếp cho Vector DB.
  - Sử dụng Alert `HighLatencyP95` đã cấu hình để phát hiện và cảnh báo sớm về kênh Slack trong vòng 3 phút khi triệu chứng bắt đầu xuất hiện.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Đặt `scrub_event` processor đứng trước `JsonlFileProcessor` và `JSONRenderer` trong pipeline structlog. Lý do: Bảo đảm thông tin cá nhân (PII) được khử hoàn toàn ngay trong bộ nhớ trước khi dữ liệu được ghi xuống file log trên đĩa hay gửi ra ngoài qua network.
- **Một lỗi/blocker đã gặp:** Khi kiểm tra prompt version ban đầu, app trả về `local-v1` thay vì `version 1` từ Langfuse Cloud.
- **Cách tìm nguyên nhân và xử lý:** Qua điều tra trace metadata thấy `prompt_source=local-fallback` do prompt `day13-chat` chưa tồn tại trên project Langfuse cá nhân khi server khởi động. Sau khi tạo prompt `day13-chat` trên Langfuse và restart uvicorn server, app đã fetch thành công prompt từ Langfuse với `version=1` và `prompt_source=langfuse`.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  - **Metrics:** Cho biết *hệ thống có chuyện gì và khi nào* (triệu chứng diện rộng: P95 tăng vọt lên ~2.6s).
  - **Logs:** Giúp xác định *request cụ thể nào bị ảnh hưởng* thông qua việc lọc dòng log chậm và lấy `correlation_id` (ví dụ `req-002b3ffb`).
  - **Traces:** Giúp chỉ ra *bước nào là nguyên nhân gốc rễ* bằng cách mở biểu đồ thác nước (waterfall), thấy ngay span `retrieval` tốn 2.501s trong khi `llm-generate` chỉ tốn 0.150s.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** LLM là hệ thống tốn kém tài nguyên và phi tất định. Prompt versioning cho phép quản trị phiên bản và rollback tức thì khi prompt mới gây suy giảm chất lượng hoặc tiêu tốn token. Giám sát token/cost và SLO giúp bảo vệ ngân sách dự án và cam kết chất lượng phản hồi đối với người dùng.
- **Điều quan trọng nhất đã học:** Nắm vững quy trình Observability thực chiến cho hệ thống LLM: Biến một AI API từ "hộp đen" thành hệ thống có thể giải trình minh bạch mọi request thông qua Metrics → Logs → Traces.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Chưa có file release `config/challenge.json` riêng của lớp từ Lab Coach nên đã thực hành hoàn chỉnh quy trình trên kịch bản `rag_slow` chuẩn theo tài liệu hướng dẫn. Sẵn sàng nạp file challenge chính thức bất cứ khi nào Lab Coach cung cấp.

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
