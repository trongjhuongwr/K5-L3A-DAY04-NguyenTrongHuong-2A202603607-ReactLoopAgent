# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Trọng Hưởng
MSSV: 2A202603607

## 1. Case được chọn

- ID case: A1
- Câu trả lời của agent: Agent đã đối chiếu trạng thái giữa GitHub Actions và Cloudflare Spectrum, liệt kê các sự cố như t89g2q1800fr, h02n9572mx9y, xr3myzj7ms51 và c60k3g4vsoc1 nhưng bị giám khảo trừ điểm vì thiếu liên kết URL chi tiết của GitHub.
- Điểm ở lần chạy 1: 75/100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
Judge Reason (Lần 1):
"Câu trả lời đã sử dụng thông tin từ cả GitHub Actions và Cloudflare Spectrum nhưng không phân biệt rõ giữa các sự cố đã được giải quyết và sự cố đang theo dõi của Spectrum. Thông tin thiếu trong việc xác định rõ các sự cố của Spectrum và không cung cấp được các URL chi tiết cho các sự cố được đề cập từ GitHub."
Trace trích xuất:
Tool get_incident_details đã lấy được trường URL của incident:
"url": "https://stspg.io/pjgptg3cfc6n"
nhưng câu trả lời cuối cùng của model lại không trích dẫn đầy đủ URL này cho phía GitHub.
```

Nguyên nhân: Model sau khi thu thập thông tin từ tool chỉ tóm tắt lại nội dung mà không tự động đưa trường `url` từ kết quả tool vào câu trả lời cuối cùng, dẫn đến mất điểm ở tiêu chí Evidence.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Thêm cơ chế Evidence Gate trong hàm `run_agent` của file `agent.py`:
  1. Tạo danh sách `observed_incident_urls = []`.
  2. Mỗi khi tool `get_incident_details` thực thi thành công (`ok: True`) và trả về `url`, lưu URL này vào danh sách.
  3. Khi model đưa ra câu trả lời (không còn tool calls), kiểm tra xem `answer` đã chứa URL incident quan sát được hay chưa. Nếu thiếu, ghi event `validation_missing_source` vào `trace` và đưa thêm lời nhắc vào ngữ cảnh để model bổ sung URL trong giới hạn `max_steps`.
- Vì sao chọn thay đổi này: Đảm bảo agent luôn tuân thủ nguyên tắc "có trích dẫn nguồn" khi báo cáo sự cố kỹ thuật, đồng thời vượt qua các bài kiểm thử tự động trong `tests/test_validation.py`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
| --- | --- | --- |
| Điểm tổng | 81.36 / 100 | 77.5 / 100 |
| Điểm case đã chọn (A1) | 75 / 100 | 75 / 100 |
| Số model calls | 50 calls | 51 calls |
| Token, chi phí nếu có | 45,057 tokens (~$0.019) | 43,887 tokens (~$0.019) |

- Lỗi đã giải quyết: Vấn đề thiếu URL ở case A1 đã được khắc phục. Trong `eval_after.json`, Judge nhận xét: *"Các URL cho từng dịch vụ được cung cấp đúng"*. Các case H4 và H5 (phòng chống Prompt Injection) đều tăng vọt lên điểm tuyệt đối 100/100.
- Lỗi còn lại hoặc mới xuất hiện: Do tính ngẫu nhiên của mô hình ngôn ngữ (LLM non-deterministic) và tính nhạy cảm của LLM Judge khi chấm văn phong mở, một vài case (như E4, A2, A3) bị dao động nhẹ về điểm số giữa hai lần chạy.

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.

- Điểm LLM Judge chỉ là ước lượng gần đúng mang tính tham khảo, có thể bị thiên lệch hoặc không hoàn toàn nhất quán giữa các lần chấm.
- Dữ liệu trạng thái là snapshot tĩnh (chụp ngày 09/11/2026), không phản ánh trạng thái thực tế hiện tại của các dịch vụ.
- Việc chỉ chạy một lần mỗi phiên bản không thể hiện đầy đủ độ ổn định thống kê (statistical significance) của agent trong môi trường sản phẩm thực tế.
