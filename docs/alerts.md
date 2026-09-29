# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: HighLatencyP95
- Severity: warning
- Duration: 3m
- Kênh thông báo: Slack (#alerts-latency)
- SLI/SLO liên quan: `fast_successful_requests` (SLO latency <= 3000ms target 99.5%)
- Điều kiện và thời gian duy trì: Độ trễ P95 của các request vượt quá 2500ms liên tục trong 3 phút.
- Ảnh hưởng tới người dùng: Người dùng phải chờ đợi lâu khi gửi câu hỏi, trải nghiệm chat bị gián đoạn, nguy cơ chạm timeout.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel Latency và TTFT trên Dashboard xem tail latency tăng ở bước nào.
  2. Lọc `data/logs.jsonl` tìm các request có `latency_ms > 2500` và trích xuất `correlation_id`.
  3. Mở trace có `correlation_id` tương ứng trên Langfuse để xem waterfall: kiểm tra span `retrieval` hay span `llm-generate` bị nghẽn.
- Mitigation tạm thời: Bật fallback sang mô hình nhẹ hơn hoặc giảm số lượng documents vector search, ngắt các incident giả lập nếu có (`/incidents/rag_slow/disable`).
- Owner: platform-oncall

## Alert 2

- Tên: HighErrorRate
- Severity: critical
- Duration: 2m
- Kênh thông báo: Slack (#alerts-critical)
- SLI/SLO liên quan: Guardrail `error_rate_pct_max <= 2%` và `retrieval_success_rate_pct_min >= 90%`
- Điều kiện và thời gian duy trì: Tỷ lệ request lỗi (HTTP 5xx / `request_failed`) vượt quá 2% trong 2 phút liên tục.
- Ảnh hưởng tới người dùng: Người dùng nhận thông báo lỗi không có câu trả lời, dịch vụ AI bị gián đoạn diện rộng.
- Ba bước kiểm tra đầu tiên:
  1. Mở panel Errors trên Dashboard xem tỷ lệ lỗi và breakdown theo `error_type` (e.g. `RuntimeError`, `TimeoutError`).
  2. Tra cứu log `request_failed` trong `data/logs.jsonl` lấy error message và `correlation_id`.
  3. Xem trace trên Langfuse để xác định thành phần thất bại (ví dụ: vector database timeout hoặc fake llm crash).
- Mitigation tạm thời: Chuyển hướng traffic sang cụm database phụ trợ, tắt incident `tool_fail` (`/incidents/tool_fail/disable`), kích hoạt chế độ trả lời không dùng RAG (fallback context).
- Owner: llmops-oncall

## Alert 3

- Tên: CostSpikeAnomaly
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#alerts-finops)
- SLI/SLO liên quan: Guardrail `daily_cost_usd_max <= 2.5$`
- Điều kiện và thời gian duy trì: Tốc độ tiêu tốn chi phí vượt quá $0.05/phút hoặc token đầu ra tăng đột biến liên tục trong 5 phút.
- Ảnh hưởng tới người dùng: Gián tiếp làm cạn kiệt budget dự án, có thể khiến API bị ngắt do vượt ngưỡng chi phí ngày.
- Ba bước kiểm tra đầu tiên:
  1. Mở panel Cost và Tokens trên Dashboard để kiểm tra mức tăng của `tokens_out` so với `tokens_in`.
  2. Lọc logs có `tokens_out` bất thường và lấy `correlation_id`.
  3. Kiểm tra trace trên Langfuse xem prompt version nào đang chạy và token generation của LLM có bị lặp vô tận (loop generation) hay không.
- Mitigation tạm thời: Giảm `max_tokens` trong config LLM, tắt incident `cost_spike` (`/incidents/cost_spike/disable`), rollback prompt nếu do prompt mới sinh văn bản quá dài.
- Owner: finops-oncall
