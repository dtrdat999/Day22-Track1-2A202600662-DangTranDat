# Day 22 - Eval Decision Workbook - Bản hoàn thành

Học viên: Đặng Trần Đạt  
Mã HV: 2A202600662  
Track: Track 1 - AI Product  

> Ghi chú: Đây là bản PM proposal ở mức workbook. Mục tiêu không phải viết eval runner thật, mà là ra quyết định: unit of work nào cần đo, đo bằng code / LLM / human / expert ở đâu, gate release thế nào, và pilot cần bao nhiêu nguồn lực.

---
## Tóm tắt quyết định làm bài

Repo này hoàn thành 3 case của Day 22 - Eval Decision Workbook:

1. `01-ticket-triage.md` - Support Ticket Triage.
2. `02-sales-copilot.md` - Sales Chat Copilot.
3. `03-medical-routing.md` - Medical Routing.

## Cách tiếp cận chung

Tôi dùng cùng một logic cho cả 3 case:

```text
Unit of Work
    ↓
Quality Question
    ↓
Output Contract
    ↓
Eval Decision Map: Code / LLM / Human / Expert
    ↓
Code-based checks
    ↓
LLM judge criteria
    ↓
Dataset Edge Cases
    ↓
Human / Expert checkpoint
    ↓
Release Gate
    ↓
Pilot plan + budget
```

## Nguyên tắc chính

- Nếu rule kiểm được bằng schema, enum, regex, DB/tool result, latency hoặc cost → ưu tiên `code-based eval`.
- Nếu tiêu chí cần hiểu ngữ nghĩa, mức độ hữu ích, tone, intent hoặc reasonableness → dùng `LLM-as-Judge`, nhưng phải có human calibration.
- Nếu case high-stakes hoặc cần chuyên môn domain → dùng `domain expert` làm chuẩn kiểm chứng.
- Release gate phải có điều kiện block rõ, không dùng kiểu “có vẻ ổn”.
- Pilot plan chỉ là ước lượng nhỏ để xin budget thử nghiệm, không phải dự toán production.

## Ghi chú về giá API

Các dự toán API trong bài dùng `gpt-5.4-mini` với giá tham khảo từ OpenAI API Pricing kiểm tra ngày 26/06/2026: `$0.75 / 1M input tokens` và `$4.50 / 1M output tokens`. Đây vẫn là giả định để tính pilot sơ bộ; khi nộp thật cần kiểm tra lại trang pricing chính thức nếu giá thay đổi.
