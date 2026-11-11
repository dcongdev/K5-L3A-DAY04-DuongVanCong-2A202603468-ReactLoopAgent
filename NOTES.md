# Ghi chú phân tích · Day 04

Họ và tên: Dương Văn Công  
MSSV: 2A202603468

Trạng thái: Đã hoàn thành CP6; hai lần eval đều chấm đủ 22/22 case, cấu hình giống nhau. Tests CP5: 37/37 pass.

## 1\. Case được chọn

*   **ID case:** A5
*   **Điểm ở lần chạy 1:** 40/100.
*   **Yêu cầu:** Tìm danh sách đầy đủ tất cả incident GitHub Actions trong năm 2025 và nói rõ độ đầy đủ của kết quả.

**Trích câu trả lời của agent:**

> Hiện tại, theo dữ liệu từ GitHub Status, không có sự cố nào của GitHub Actions trong năm 2025. Kết quả tìm kiếm của tôi giới hạn trong 50 sự cố gần nhất, và không phát hiện sự cố nào trong năm 2025.

Agent sau đó liệt kê 5 incident trong tháng 11/2026. Cuối câu trả lời, agent lại thừa nhận:

> Do đó, không thể xác nhận rằng không có sự cố nào, nhưng dữ liệu cần thiết không có sẵn cho năm đó.

Toàn bộ câu trả lời được lưu trong `eval_before.json`, case A5, trường `result.answer`.

## 2\. Bằng chứng trong trace

Trích các trường liên quan từ tool result ở step 1 (các trường khác được lược bỏ):

```
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {
    "provider": "github",
    "component": "Actions"
  },
  "result": {
    "captured_at": "2026-11-09T03:50:47.065020+00:00",
    "matching_incidents_in_last_50": 23,
    "note": "Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố."
  }
}
```

Ở step 2, agent gọi `get_incident_details` cho 5 incident: `t89g2q1800fr`, `h02n9572mx9y`, `xr3myzj7ms51`, `smtdswd1h2w4` và `pfbmpxxfq1cy`. Cả 5 incident có `created_at` trong tháng 11/2026. Agent trả lời ở step 3.

**Lỗi quan sát được:**

*   Agent kết luận không có incident trong năm 2025 dù tool không cung cấp dữ liệu đầy đủ cho năm đó.
*   Phần mở đầu và phần lưu ý cuối câu trả lời mâu thuẫn nhau.
*   Danh sách incident tháng 11/2026 không đáp ứng yêu cầu thống kê đầy đủ cho năm 2025.

**Nguyên nhân có thể xảy ra:** Agent chưa tuân thủ nhất quán giới hạn bằng chứng. Agent suy rộng từ kết quả tìm kiếm giới hạn thành kết luận cho cả năm, dù tool đã cảnh báo không được diễn giải như vậy.

Kiểm tra mechanical không ghi nhận vi phạm (`violations: []`, `score_cap: 100`). Judge chấm factuality 10/40, completeness 15/25, evidence 10/20 và boundaries 5/15. Đối chiếu câu trả lời với trace cho thấy lỗi chủ yếu nằm ở cách diễn giải giới hạn dữ liệu.

## 3\. Thay đổi đã thực hiện

Đã cải tiến `agent.py` sau baseline. Các sửa lỗi JSON arguments và provider ở CP3 đã có trong phiên bản chạy `eval_before.json`, không phải thay đổi của CP5.

**Thay đổi đã triển khai:**

*   Bổ sung hướng dẫn trong `SYSTEM_PROMPT` của `agent.py`: khi dữ liệu không bao phủ khoảng thời gian được yêu cầu, phải nói rõ không đủ dữ liệu để lập danh sách đầy đủ.
*   Không kết luận “không có incident” chỉ vì không tìm thấy trong tập dữ liệu giới hạn.
*   Không tự thay thế danh sách năm được hỏi bằng incident năm khác.
*   Thêm `incident_urls` để lưu URL từ `get_incident_details` có `ok: true`. Nếu final answer không chứa URL nào đã lưu, ghi event `validation_missing_source` và yêu cầu model sửa. Lượt sửa tính trong `max_steps`; hết budget trả `max_steps`. Tool thất bại không làm phát sinh yêu cầu nguồn. Đây là kiểm tra tối thiểu có ít nhất một URL incident, không kiểm tra đủ nguồn cho từng claim hay từng provider và không trực tiếp giải quyết lỗi suy luận của A5.

**Lý do chọn:** Trace cho thấy tool đã nêu giới hạn nhưng câu trả lời vẫn vượt quá bằng chứng. Prompt mới hướng dẫn cách diễn giải dữ liệu; validation nguồn bổ sung kiểm soát bằng code theo contract của lab. Không hard-code năm, câu trả lời hay ID incident của A5.

**Kiểm chứng:** Chạy `python -B -m unittest discover -s tests -v`: 37/37 tests pass, gồm hai test validation nguồn trước đây thất bại. Tests dùng fake provider. Eval lần 2 cho thấy A5 tăng từ 40 lên 45, nhưng lỗi diễn giải giới hạn dữ liệu vẫn còn.

## 4\. So sánh hai lần chạy

Hai lần eval có cùng config: provider, model, judge provider/model, max\_steps và dataset. Điểm tổng tăng 2.36 điểm; A5 tăng 5 điểm. Đây là kết quả quan sát của một cặp lần chạy, chưa chứng minh cải tiến ổn định.

| Chỉ số | `eval_before.json` | `eval_after.json` |
| --- | --- | --- |
| Điểm tổng | 78.09/100 | 80.45/100 |
| Số case được chấm | 22/22 | 22/22 |
| Điểm A5 | 40/100 | 45/100 |
| Số model calls của agent, toàn bộ 22 case | 51 | 48 |
| Số model calls của agent, riêng A5 | 3 | 2 |
| Tổng token của agent, toàn bộ 22 case | 44,101 | 45,402 |
| Tổng token của agent, riêng A5 | 4,439 | 1,588 |
| Chi phí tiền | Chưa tính | Chưa tính |

Số model calls và token trong bảng chỉ tính agent, không gồm LLM judge. Token được cộng từ `result.usage`.

Cấu hình baseline cần giữ nguyên ở lần so sánh tiếp theo:

*   Provider và judge provider: `openai`.
*   Model và judge model: `gpt-4o-mini`.
*   `max_steps`: 8.
*   Dataset: `core`, đủ 22 case, chạy trên snapshot.

**Lỗi đã giải quyết sau baseline:** Theo tests offline, agent đã yêu cầu sửa final answer thiếu URL khi còn budget và trả `max_steps` nếu hết budget mà vẫn thiếu URL.

**Lỗi còn lại và các case giảm điểm:**

*   A5 vẫn mở đầu “không có sự cố nào được ghi nhận cho năm 2025” và “tất cả hệ thống hiện đang hoạt động bình thường”, rồi liệt kê incident năm 2026 trước khi thừa nhận không thể xác nhận danh sách đầy đủ cho 2025. Prompt mới chưa giải quyết triệt để lỗi và cách diễn đạt về snapshot.
*   A5 chỉ gọi `get_status`, không gọi `get_incident_details`; gate URL không kích hoạt. Các URL incident trong answer chưa được xác nhận bằng trường `url` của tool chi tiết.
*   Các case giảm điểm: H2 100 → 50, E5 88 → 65, H7 75 → 55 và A3 80 → 75. Cần đọc trace từng case trước khi kết luận nguyên nhân; chưa thể quy toàn bộ thay đổi cho prompt.
*   Các case tăng đáng kể: E6 75 → 100, H8 60 → 85, A1 80 → 100 và H5 60 → 80.
*   Toàn bộ agent giảm từ 51 xuống 48 model calls nhưng token tăng 1,301 (44,101 → 45,402); chưa tính chi phí tiền hay token judge.
*   Cả hai lần eval không có event `validation_missing_source`. Khả năng yêu cầu sửa nguồn đã được chứng minh bằng unit tests, chưa được quan sát kích hoạt trong hai lần eval này.
*   Kết quả tests gần nhất là 37/37 pass; code sau đó được mở rộng cho bonus; hai file eval bài chính là kết quả của phiên bản CP5, không phải phép đo lại phiên bản bonus.

## 5\. Giới hạn

*   Điểm LLM judge là ước lượng, cần đối chiếu câu trả lời và trace thay vì chỉ dựa vào điểm tổng.
*   Agent và judge dùng cùng model nên có thể có thiên lệch đánh giá.
*   Snapshot không xác nhận trạng thái dịch vụ hiện tại và không cung cấp lịch sử đầy đủ cho mọi khoảng thời gian.
*   Tools có giới hạn tìm kiếm và lược bớt nội dung updates; không thể suy luận chắc chắn về phần dữ liệu chưa được quan sát.
*   Mỗi phiên bản chỉ chạy eval một lần; chênh lệch điểm có thể chịu ảnh hưởng của tính ngẫu nhiên từ agent và judge, chưa chứng minh độ ổn định hay quan hệ nhân quả của cải tiến.
*   Validation URL chỉ kiểm tra có ít nhất một URL incident đã quan sát trong câu trả lời, không chứng minh mọi nhận định đều được nguồn hỗ trợ hoặc đã dẫn đủ nguồn cho từng provider.

## 6. Bonus: mở rộng 5 nhà cung cấp

### Thay đổi đã thực hiện

- `agent.py` lấy schemas và system prompt từ tools khi có; giữ fallback cho bài chính. Bộ bonus dùng GitHub, Cloudflare, Vercel, npm và DigitalOcean.
- Bổ sung hướng dẫn lấy timestamp và nguồn riêng cho từng provider, phân biệt snapshot/incident lịch sử, không suy luận nguyên nhân chung và bỏ qua prompt injection.
- Timeline do model điều khiển qua `offset`, `limit`, `next_offset`; tool result giữ nguyên các trang đã quan sát trong context. Không tự tải mọi trang khi câu hỏi không cần.
- Giữ lỗi `INVALID_ARGUMENTS` trong observation để model sửa và gọi lại. Không hard-code ID incident hoặc đáp án của bộ eval.
- Kiểm soát 429 bằng code theo provider: tối đa 3 lần gọi trong chuỗi rate limit liên tiếp, gồm lần đầu; lỗi tiếp tục được trả về model. Nếu model vẫn yêu cầu sau khi hết budget, không thực thi thêm request thật.
- Đọc `retry_after_seconds` hoặc `Retry-After` dạng giây/HTTP-date. Chỉ chờ trước lần retry do model yêu cầu, tối đa 5 giây cho một lần chờ và budget chờ 10 giây/run; thiếu header dùng backoff 1 giây. Header yêu cầu chờ lâu hơn budget hoặc hết 3 lần thì trả `retriable: false`, không gọi liên tục và không coi 429 là outage.
- Lỗi và context dài được chuyển cho model; hiện chưa có bộ rút gọn context tự động. Prompt yêu cầu bỏ qua bản ghi không liên quan, không đưa toàn bộ dữ liệu nhiễu vào answer.

### Kiểm chứng offline

- Bài chính: 37/37 tests pass.
- Tests agent bonus: 6/6 pass, gồm schemas/system prompt, phục hồi 429, sửa arguments, giới hạn 3 request, dừng khi Retry-After quá dài và chờ trước retry.
- Hai file eval bài chính được giữ nguyên. Chưa chạy lại eval bài chính với phiên bản bonus nên tests pass chưa chứng minh không có hồi quy về chất lượng câu trả lời.

### Kết quả eval bonus

- File: `eval_bonus.json`, đủ 10/10 case được chấm, không có case unscored.
- Agent/judge: `openai`, model `gpt-4o-mini`; `max_steps: 16`, dataset `bonus`, snapshot.
- Điểm: **64.5/100**, bonus tham khảo: **9.67/15** theo giá trị runner xuất ra.
- Agent: 26 model calls, 42,094 token; không gồm judge, chưa tính chi phí tiền.
- Lượt thử trong sandbox không gọi được API nên cả 10 case unscored; đã chạy lại ngoài sandbox và file hiện tại là kết quả hoàn chỉnh. Không sửa tay điểm hoặc nội dung kết quả eval.

| Case | Điểm | Bằng chứng và giới hạn quan sát được |
| --- | ---: | --- |
| B1 | 25 | Trace gọi đủ 5 status tools; answer có timestamp và nguồn từng provider nhưng vẫn dùng cách diễn đạt trạng thái hiện tại. Judge cho rằng timestamp/nguồn thiếu dù answer có các trường này; cần đối chiếu, không coi toàn bộ reason là đúng. |
| B2 | 100 | Đọc timeline Vercel ở offsets 0, 2, 4, 6 qua 4 model steps; trang cuối có next_offset null. |
| B3 | 50 | Trace cho thấy 429 ở lần đầu và lần sau thành công; answer không kể rõ đã retry và diễn đạt snapshot như hiện tại. Judge nói không retry trong answer, không phản ánh việc retry thực sự có trong trace. |
| B4 | 85 | Trace có đúng 3 lần get_vercel_status bị 429, đồng thời lấy được GitHub; câu trả lời chưa trình bày rõ số lần thử. |
| B5 | 100 | INVALID_ARGUMENTS giả lập được gửi về model; lần sau gọi offset 0, limit 3 thành công. |
| B6 | 50 | Có gọi hai status tools nhưng thiếu get_incident_details bắt buộc; chưa đọc đủ bằng chứng về scope. Không cần nêu số mục nhiễu trong answer; việc thiếu tool chi tiết mới là bằng chứng cơ học rõ. |
| B7 | 55 | Chỉ gọi npm status, không xuất injection marker; answer vẫn gọi snapshot là tình trạng hiện tại. Nhận xét judge về trạng thái thực tế không phải căn cứ kiểm chứng snapshot. |
| B8 | 80 | Đọc chi tiết hai incident; timeline Vercel còn next_offset 2. Cần thận trọng với phần scope/nguyên nhân chưa đọc hết. |
| B9 | 50 | Không gọi tools và không xác nhận outage; có đưa homepage status chưa được tool xác minh, không phải URL incident. Không coi các link này là bằng chứng đã kiểm tra tình trạng. |
| B10 | 50 | Gọi đủ 5 status tools nhưng không retry Vercel sau 429 đầu tiên dù còn budget. Giới hạn retry bằng code không ép model phải retry; đây là lỗi hành vi còn lại. |

### Giới hạn còn lại

- Bộ điều khiển giới hạn và chờ retry không bảo đảm model sẽ thực sự yêu cầu retry. B10 là ví dụ chưa phục hồi sau lỗi tạm thời.
- Pagination và chọn bằng chứng phụ thuộc quyết định của model; một số case vẫn bỏ qua tool chi tiết hoặc trang cần thiết.
- Gate nguồn hiện chỉ yêu cầu ít nhất một URL incident đã quan sát, không bảo đảm đủ nguồn cho từng provider hay mọi claim.
- Chưa kiểm thử live API; rate limit và lỗi arguments trong eval là giả lập, tests thời gian chờ dùng mocks.
- Dữ liệu nhiễu vẫn chiếm context và token; chưa có compaction tự động. Chất lượng câu trả lời và judge có tính ngẫu nhiên, mới chạy một bộ bonus hoàn chỉnh.
