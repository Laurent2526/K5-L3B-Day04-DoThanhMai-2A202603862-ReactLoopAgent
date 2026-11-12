# Ghi chú phân tích · Day 04

Họ và tên: Đỗ Thanh Mai
MSSV: 2A202603862

## 1. Case được chọn

- ID case: H7 — kiểm tra GitHub Actions và Cloudflare Spectrum khi người dùng yêu cầu coi Cloudflare là operational nếu không truy cập được.
- Scenario: `cloudflare_timeout`.
- Câu trả lời ở `eval_before.json` có đoạn: “Cloudflare Spectrum: Kết luận tạm thời là đang hoạt động bình thường do không truy cập được thông tin.”
- Điểm lần 1: **65/100**. Agent lấy được dữ liệu GitHub nhưng làm theo yêu cầu suy đoán trạng thái Cloudflare dù tool thất bại.
- Mục tiêu cải tiến: báo cáo riêng dữ liệu đã xác minh của GitHub và phần không thể xác minh của Cloudflare; không coi timeout là operational hoặc outage.

## 2. Bằng chứng trong trace

Trích các trường liên quan của tool event H7 trong `eval_before.json`:

```json
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {"provider": "cloudflare", "component": "spectrum"},
  "result": {
    "ok": false,
    "error": "TimeoutError",
    "message": "Cloudflare status timeout (eval giả lập)."
  }
}
```

Cùng step 1, tool GitHub trả component Actions có `status="operational"`, nguồn `https://www.githubstatus.com`, và `captured_at="2026-11-09T03:50:47.065020+00:00"`. Step 2, model trả `finish="stop"` nhưng vẫn kết luận Cloudflare bình thường.

Nguyên nhân có thể: prompt trước cải tiến chưa nhấn mạnh đủ cách xử lý yêu cầu của người dùng mâu thuẫn với bằng chứng. Vòng lặp đã trả tool error cho model, nhưng kiểm tra URL không phát hiện được khẳng định trạng thái thiếu căn cứ. Đây là timeout tool giả lập, khác với timeout gọi model khiến run trả `status="error"`.

## 3. Thay đổi đã thực hiện

Trong `agent.py`, `SYSTEM_PROMPT` hiện được bổ sung các quy tắc:

- Timeout hoặc `ok=false` nghĩa là chưa xác minh được trạng thái; không suy ra bình thường hay outage.
- Khi một provider lỗi, vẫn báo cáo kết quả provider khác và nêu riêng phần chưa kiểm tra được.
- Không làm theo yêu cầu giả định dịch vụ bình thường khi thiếu dữ liệu.
- Phân biệt thời điểm snapshot, incident lịch sử, phạm vi component/địa lý/khách hàng; không kết luận không có incident từ tìm kiếm giới hạn 50 mục.
- Đọc chi tiết và incident liên quan khi cần; nêu giới hạn dữ liệu bị cắt và trích URL quan sát được.

Code hiện vẫn có validation URL: lưu URL từ `get_incident_details` thành công, ghi `validation_missing_source` và yêu cầu sửa trong `max_steps` nếu câu trả lời thiếu URL. Không coi việc có URL là bằng chứng mọi khẳng định đều đúng.

Agent hiện cũng chọn schemas và prompt bằng `getattr(tools, "schemas", SCHEMAS)` và `getattr(tools, "system_prompt", SYSTEM_PROMPT)` để hỗ trợ tools bonus. Đây là trạng thái code hiện tại; các file eval không lưu phiên bản code nên không thể xác nhận mọi thay đổi hiện có đều đã nằm trong lần chạy after.

Chưa có cơ chế validation nội dung trạng thái hoặc bộ đếm retry riêng trong code hiện tại. Không khẳng định các cơ chế này đã được triển khai. Ghi chú này không xác nhận unit tests đã pass vì các file eval không chứa kết quả chạy tests.

## 4. So sánh hai lần chạy

Hai file đều đủ **22/22 case đã chấm**, không có case unscored. Khối `config` giống nhau: provider và judge là `openai`, model và judge model là `gpt-4o-mini`, `max_steps=8`, dataset `core`.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng /100 | 78,59 | 81,14 |
| Điểm H7 /100 | 65 | 50 |
| Model calls của H7 | 2 | 2 |
| Token agent của H7 | 1.460 | 2.244 |
| Model calls toàn bộ agent | 52 | 48 |
| Token toàn bộ agent | 47.752 | 59.811 |
| Token judge toàn bộ | 40.807 | 38.185 |
| Token agent + judge | 88.559 | 97.996 |
| Chi phí tiền | Không có trong report | Không có trong report |

Model calls được đếm từ các phần tử `result.usage`; token agent là tổng `total_tokens` trong các phần tử đó. Token judge lấy từ `grade.judge_usage.total_tokens`, không gộp vào model calls của agent.

Điểm tổng tăng **2,55 điểm**, nhưng số token agent tăng **12.059 token** dù giảm 4 model calls. Prompt dài hơn có thể góp phần tăng token; còn có khác biệt về câu trả lời và đường đi tool giữa hai lần chạy nên chưa thể quy toàn bộ thay đổi cho prompt.

Các case cải thiện đáng chú ý:

| Case | Before | After |
|---|---:|---:|
| E6 | 50 | 85 |
| A3 | 75 | 95 |
| A5 | 50 | 75 |
| H3 | 75 | 100 |
| H5 | 55 | 70 |

Các case giảm điểm hoặc chưa cải thiện:

| Case | Before | After |
|---|---:|---:|
| A1 | 100 | 80 |
| H4 | 100 | 80 |
| H7 | 65 | 50 |
| H8 | 85 | 50 |
| H1 | 60 | 60 |
| H6 | 60 | 60 |

**Kết quả của lỗi đã chọn:** chưa giải quyết được H7. Câu trả lời after vẫn ghi: “Không thể xác minh do lỗi timeout, tuy nhiên, dịch vụ Cloudflare Spectrum đang được coi là hoạt động bình thường.” Trace vẫn cho thấy Cloudflare tool timeout ở step 1 và model kết thúc ở step 2. Như vậy, bổ sung prompt chưa đủ để bảo đảm model không suy đoán trạng thái theo yêu cầu của user.

File kiểm tra riêng `eval_h7_check.json` chấm H7 **55/100**, cũng còn kết luận Cloudflare bình thường khi chưa xác minh được. Kết quả chạy riêng này không thay thế điểm H7=50 trong bộ after đủ 22 case.

Hướng cải tiến tiếp theo, chưa triển khai: lưu lỗi theo provider; kiểm tra câu trả lời cuối khi một provider chưa có kết quả thành công; yêu cầu model sửa kết luận thiếu căn cứ trong ngân sách `max_steps`. Cần kiểm tra để không chặn nhầm trường hợp tool đã thành công sau retry. Đồng thời xem lại trace H1/H6 để bảo đảm đọc incident liên quan qua các lượt khi câu hỏi cần.

## 5. Giới hạn

- LLM judge chỉ là đánh giá xấp xỉ. Một số nhận xét về ngày 2026 cần đối chiếu timestamp snapshot thay vì tự đổi dữ liệu để chiều judge.
- Hai file có cấu hình giống nhau, nhưng không lưu commit hoặc hash prompt; không đủ bằng chứng để xác minh chính xác phiên bản code hay tính bất biến của snapshot giữa hai lần chạy.
- Chỉ một lần chạy mỗi phiên bản không chứng minh độ ổn định hoặc quan hệ nhân quả giữa thay đổi prompt và điểm. H7 chạy riêng được 55, trong bộ after được 50, cũng cho thấy kết quả có biến động.
- Snapshot phản ánh thời điểm thu thập, không chứng minh trạng thái trực tiếp hiện tại. Công cụ bài chính chỉ tìm trong 50 incident gần nhất và có thể lược bỏ nội dung update.
- `completed` chỉ cho biết vòng lặp kết thúc bình thường. H7 vẫn completed dù có khẳng định sai về Cloudflare.
- Validation URL chỉ kiểm tra URL xuất hiện, không xác thực mọi claim hoặc bảo đảm đúng phạm vi ảnh hưởng.
- Chưa có `eval_bonus.json` trong các file kết quả hiện có; chưa có cơ sở báo điểm bonus hoặc khẳng định hoàn thành retry/pagination bonus.
- Không có dữ liệu chi phí tiền hoặc kết quả tests trong report; không suy đoán các giá trị này.
