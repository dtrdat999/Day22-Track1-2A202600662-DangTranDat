# Case 1 - Support Ticket Triage - Bản trả lời

## 1. Unit of Work

Tôi chọn lát cắt: **một ticket support mới đi vào → AI gán category, urgency, route_to, requires_human, reason_codes và summary ngắn để hiển thị trong inbox nội bộ**. Output cuối cùng được dùng bởi nhân viên support / support lead để quyết định ticket nằm ở hàng đợi nào và có cần xử lý ngay không.

Đây là đơn vị đủ nhỏ để eval vì nó chỉ đo **một quyết định vận hành cụ thể** thay vì toàn bộ hệ thống chăm sóc khách hàng. Tuy nhiên nó vẫn đủ quan trọng vì nếu AI route sai hoặc bỏ sót escalation, ticket enterprise có thể bị trễ, khách mất niềm tin, và team support phải tốn thời gian sửa lỗi ở downstream.

## 2. Quality Question

Câu hỏi chất lượng chính: **AI có phân loại đúng loại ticket, đánh đúng mức độ khẩn và route/escalate đúng để ticket không bị đi sai hàng xử lý không?**

Nếu fail ở đây, hậu quả không chỉ là nhãn hiển thị sai, mà là ticket có thể bị đưa vào queue bình thường trong khi khách enterprise đang bị khóa tài khoản hoặc chặn công việc. Behavior bắt buộc là phải phát hiện tín hiệu high-risk như `locked out`, `account disabled`, `blocking work`, và nếu là enterprise + high/critical thì phải bật `requires_human = true`. Behavior bị cấm là route billing sang product team, đánh low cho case đang chặn công việc, hoặc bịa reason không có trong ticket.

## 3. Output Contract tối thiểu

Tôi đề xuất output contract tối thiểu như sau:

```json
{
  "ticket_id": "T-002",
  "category": "billing | technical | feature_request | account_access | unknown",
  "urgency": "low | medium | high | critical",
  "route_to": "support_l1 | technical_support | billing_ops | product_team | human_escalation | needs_review",
  "requires_human": true,
  "queue": "normal | high_priority | escalation",
  "confidence": 0.0,
  "reason_codes": ["enterprise_customer", "account_disabled", "blocking_work"],
  "summary_reason": "Khách enterprise bị khóa tài khoản do lỗi thanh toán, đang chặn công việc.",
  "evidence_spans": ["payment failed", "account disabled", "locked out"],
  "unknown_or_ambiguous": false
}
```

- `ticket_id`: cần để gắn output với đúng ticket và chống bịa mã ticket trong eval.
- `category`: cần cho UI và routing; phải là enum chuẩn để downstream xử lý được.
- `urgency`: cần để quyết định hàng đợi bình thường hay ưu tiên cao.
- `route_to`: quyết định vận hành quan trọng nhất; sai route là team nhận ticket không xử lý được.
- `requires_human`: cần cho escalation; đặc biệt với enterprise + high/critical.
- `queue`: giúp UI hiển thị hàng đợi hiện tại; có thể suy ra từ urgency + requires_human nhưng nên lưu để audit.
- `confidence`: cần cho gate human review; confidence thấp phải vào `needs_review`.
- `reason_codes`: giúp eval kiểm tra AI có dựa vào tín hiệu thật hay không.
- `summary_reason`: giúp nhân viên hiểu lý do gợi ý, không chỉ nhìn nhãn.
- `evidence_spans`: cần để human reviewer kiểm tra AI không bịa lý do.
- `unknown_or_ambiguous`: giúp xử lý case thiếu thông tin thay vì ép AI đoán.

## 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema và enum của output | Có | Không | Không | Không | Deterministic: có thể validate bằng JSON schema, allowed enum và required fields. |
| Logic enterprise + high/critical → requires_human=true | Có | Không | Không | Không | Đây là business rule rõ ràng; code bắt chính xác và ổn định hơn LLM. |
| Billing không route sang product_team | Có | Không | Không | Không | Rule vận hành rõ; nếu category=billing và route_to=product_team thì fail ngay. |
| Phát hiện tín hiệu blocking/locked/account disabled không bị đánh low | Có | Có | Có | Không | Có thể bắt keyword bằng code, nhưng mức độ diễn đạt đa dạng nên cần LLM/human sample để chấm semantic. |
| Category/urgency có hợp lý với nội dung ticket | Không | Có | Có | Không | Cần đọc hiểu nội dung; code không bắt tốt các câu mơ hồ hoặc diễn đạt khác nhau. |
| Summary/reason_codes có bịa thông tin không | Có | Có | Có | Không | Code so evidence span/ticket_id; LLM chấm xem reason có grounded vào nội dung thật không. |
| Ambiguous/low-info case có bị overconfident không | Có | Có | Có | Không | Code kiểm confidence threshold; LLM/human đánh giá hành vi “nên hỏi lại/needs_review” có hợp lý không. |
| Release decision cuối trước pilot | Không | Không | Có | Không | Case này không cần domain expert sâu; support lead/human ops đủ xác nhận ranh giới vận hành. |

## 5. Kiểm tra tự động bằng code

- Kiểm tra: output parse được JSON hợp lệ.  
  Vì sao nên giao cho code: đây là điều kiện deterministic, không cần judgment.

- Kiểm tra: output có đủ required fields: `ticket_id`, `category`, `urgency`, `route_to`, `requires_human`, `queue`, `confidence`, `reason_codes`, `summary_reason`, `evidence_spans`.  
  Vì sao nên giao cho code: thiếu field sẽ làm UI, routing hoặc eval downstream bị gãy.

- Kiểm tra: `category` thuộc allowed enum.  
  Vì sao nên giao cho code: nếu AI trả “payment problem” thay vì `billing`, hệ thống route có thể không hiểu.

- Kiểm tra: `urgency` thuộc allowed enum và không rỗng.  
  Vì sao nên giao cho code: urgency quyết định queue và escalation.

- Kiểm tra: `route_to` thuộc allowed enum.  
  Vì sao nên giao cho code: route sai enum sẽ không map được vào team thật.

- Kiểm tra: `confidence` là số trong khoảng 0-1.  
  Vì sao nên giao cho code: đây là numeric threshold rõ ràng.

- Kiểm tra: nếu `customer_tier=enterprise` và `urgency in {high, critical}` thì `requires_human=true`.  
  Vì sao nên giao cho code: business rule rõ, fail là lỗi nghiêm trọng.

- Kiểm tra: nếu `category=billing` thì `route_to != product_team`.  
  Vì sao nên giao cho code: routing rule rõ, không cần LLM.

- Kiểm tra: nếu subject/message chứa “locked out”, “account disabled”, “blocking work”, “cannot access” thì `urgency != low`.  
  Vì sao nên giao cho code: có thể bắt pattern tối thiểu để không bỏ sót tín hiệu nghiêm trọng.

- Kiểm tra: nếu input thiếu category signal rõ và AI trả `confidence > 0.75` mà không đặt `unknown_or_ambiguous=true` hoặc `route_to=needs_review`, thì fail.  
  Vì sao nên giao cho code: confidence threshold có thể kiểm được bằng rule, giúp tránh overconfidence.

- Kiểm tra: `reason_codes` chỉ chứa allowed reason code và ít nhất một reason code phải có evidence trong subject/message hoặc metadata.  
  Vì sao nên giao cho code: giảm hallucination vận hành.

- Kiểm tra: `evidence_spans` phải là substring hoặc gần substring của input.  
  Vì sao nên giao cho code: nếu evidence không tồn tại trong input, có dấu hiệu bịa.

- Kiểm tra: `summary_reason` không được chứa ticket ID hoặc account name khác với input.  
  Vì sao nên giao cho code: chống bịa thông tin định danh.

- Kiểm tra: latency p95 của triage output dưới 2 giây trong pilot.  
  Vì sao nên giao cho code: đúng nhưng quá chậm vẫn làm inbox vận hành kém.

- Kiểm tra: cost per triage dưới ngưỡng pilot đã đặt.  
  Vì sao nên giao cho code: tránh prompt/judge quá tốn chi phí khi scale.

## 6. Tiêu chí chấm bằng LLM

- Tiêu chí: category có phản ánh đúng intent chính của ticket không.  
  Vì sao code không bắt tốt: cùng một vấn đề có thể diễn đạt theo nhiều cách, không chỉ keyword.

- Tiêu chí: urgency có hợp lý với mức ảnh hưởng vận hành trong ticket không.  
  Vì sao code không bắt tốt: “gấp”, “chặn việc”, “team không vào được” có mức nghiêm trọng khác nhau theo context.

- Tiêu chí: route_to có phù hợp với loại vấn đề và khả năng xử lý của team không.  
  Vì sao code không bắt tốt: route đôi khi phụ thuộc kết hợp category + urgency + mô tả vấn đề.

- Tiêu chí: summary_reason có đủ ngắn, đúng trọng tâm và không làm nhẹ mức độ nghiêm trọng không.  
  Vì sao code không bắt tốt: cần hiểu ý nghĩa câu tóm tắt, không chỉ độ dài.

- Tiêu chí: reason_codes có grounded vào nội dung ticket không.  
  Vì sao code không bắt tốt: có những tín hiệu không trùng exact string nhưng vẫn có nghĩa tương đương.

- Tiêu chí: AI có xử lý đúng case mơ hồ bằng `unknown`/`needs_review` thay vì đoán quá tự tin không.  
  Vì sao code không bắt tốt: cần đánh giá mức độ thiếu thông tin trong nội dung tự nhiên.

- Tiêu chí: nếu ticket có nhiều intent, AI có chọn intent vận hành quan trọng nhất không.  
  Vì sao code không bắt tốt: multi-intent cần semantic ranking theo rủi ro.

## 7. Dataset Edge Cases đề xuất

1. Happy path: ticket enterprise báo “Cannot login after password reset” và “blocking my work”; kỳ vọng `category=technical`, `urgency=high`, `requires_human=true`, route về `technical_support`.  
   Case này bắt failure bỏ sót tín hiệu blocking work hoặc route nhầm login issue sang support L1 bình thường.

2. Ambiguous input: subject `Help`, message `Please help asap`, không có mô tả vấn đề hoặc metadata rõ.  
   Case này bắt failure AI đoán category quá tự tin thay vì dùng `unknown`, `needs_review` hoặc confidence thấp.

3. Missing information: subject `Invoice issue`, message `Can someone check this?`, customer tier `standard`, không có invoice ID hay mô tả lỗi.  
   Case này bắt failure AI bịa chi tiết thanh toán, bịa account bị khóa, hoặc route/escalate quá mạnh khi thiếu evidence.

4. High-risk / escalation: ticket enterprise báo `URGENT: payment failed and account disabled`, message có `locked out` và `cannot access account`.  
   Case này bắt failure giống mock outcome: đánh medium/low, route sai sang support L1/product team, hoặc quên `requires_human=true`.

5. Regression case: ticket có nhiều intent “I was charged twice, and I also want a feature for invoices”.  
   Case này bắt failure model ưu tiên nhầm feature request thay vì billing issue đang gây hại trực tiếp, đặc biệt không được route billing sang `product_team`.

## 8. Human / Expert Review

Người cần review là **Support Lead / Operations Lead** và một số **support agents có kinh nghiệm**. Họ review các case có confidence thấp, case enterprise + high/critical, case LLM judge và code disagree, case bị user/team phản hồi sai route, và top/bottom output samples trong mỗi batch eval.

Case này **không cần domain expert chuyên sâu** vì taxonomy support như billing/technical/feature request và rule escalation là nghiệp vụ vận hành nội bộ, không phải domain y tế/pháp lý. Human review vận hành là đủ để xác nhận “ticket này nên vào queue nào” và “team nào xử lý được”. Tuy nhiên trước khi pilot nên có support lead approve taxonomy và release gate.

### 8A. Màn hình cho Domain Expert (ASCII)

Không áp dụng. Không cần domain expert chuyên sâu; dùng Support Lead làm human reviewer vận hành.

### 8B. Tiêu chí review của Domain Expert

Không áp dụng. Tiêu chí cho Support Lead review:

1. Category có đúng với vấn đề chính của ticket không?
2. Urgency có phản ánh đúng rủi ro vận hành không?
3. Route_to có đưa ticket tới team có khả năng xử lý không?
4. Case enterprise/high-risk có được escalation không?
5. Summary/reason có đủ căn cứ từ input, không bịa thêm thông tin không?

## 9. Release Gate

Tôi đề xuất release gate cho pilot như sau:

- Block release nếu schema pass rate < 100%.
- Block release nếu có bất kỳ case nào vi phạm rule enterprise + high/critical nhưng `requires_human=false`.
- Block release nếu có bất kỳ case billing route sang `product_team`.
- Category accuracy trên reference dataset tối thiểu 90%.
- Urgency accuracy tối thiểu 88%, riêng high/critical recall tối thiểu 95%.
- Requires_human precision tối thiểu 90%, recall với escalation cases tối thiểu 95%.
- LLM judge/human sample xác nhận summary grounded tối thiểu 90%.
- p95 latency < 2 giây, chi phí trung bình mỗi ticket dưới ngưỡng pilot.
- Các case confidence < 0.7, ambiguous, hoặc code/LLM disagree phải vào human review trước khi dùng để route tự động.

Quyết định rollout: nếu pass toàn bộ gate thì limited pilot trên 10-20% ticket nội bộ. Nếu chỉ fail ở semantic summary nhưng route đúng, có thể pilot ở chế độ “suggestion only”. Nếu fail escalation/high-risk thì hold.

## 10. Kế hoạch chạy thử và dự toán chi phí

Giả định API: dùng **gpt-5.4-mini** cho generation/judge, giá tham khảo từ OpenAI API pricing kiểm tra ngày 26/06/2026: **$0.75 / 1M input tokens** và **$4.50 / 1M output tokens**. Tôi chọn model này vì đủ rẻ cho pilot eval, có thể chạy nhiều vòng và vẫn đủ tốt cho semantic judge sơ bộ.

Quy mô pilot: 80 cases reference dataset, gồm 40 normal, 15 ambiguous, 15 enterprise/high-risk, 10 regression cases. Chạy 40 lần/lặp gồm các vòng sửa prompt, sửa output contract, sửa threshold và calibration judge. Tổng lượt chạy ước tính: 80 × 40 = 3,200 output generations; semantic judge chạy trên khoảng 3,200 outputs.

Giả định token: mỗi generation trung bình 800 input tokens + 250 output tokens; mỗi LLM judge trung bình 1,200 input tokens + 200 output tokens. Chi phí generation khoảng 3,200 × [(800/1M×0.75) + (250/1M×4.50)] ≈ $5.52. Chi phí judge khoảng 3,200 × [(1,200/1M×0.75) + (200/1M×4.50)] ≈ $5.76. Cộng buffer retry/logging 30%, API budget nên xin khoảng **$15-20**.

Giờ công dự kiến: PM/eval design 8 giờ để chốt unit, output contract, decision map và release gate; engineer/ops 8 giờ để mock runner, schema checks và export report; support lead 4 giờ để label calibration set và review high-risk cases; support agents 4 giờ để review sample và feedback taxonomy. Nếu quy đổi chi phí nội bộ thử nghiệm ở mức PM 150,000 VND/giờ, engineer 200,000 VND/giờ, support reviewer 100,000 VND/giờ thì phần người khoảng 1.2M + 1.6M + 0.8M = **3.6M VND**. API khoảng 500k VND nếu xin buffer rộng. Tổng pilot khoảng **4.1M VND** trong 3-5 ngày làm việc.

Với quy mô này, team chưa chứng minh được production-ready, nhưng đủ chứng minh: taxonomy có dùng được không, high-risk escalation có bắt được không, model có đáng tin ở chế độ suggestion-only không, và cần thêm dữ liệu ở failure mode nào trước khi rollout rộng hơn.
