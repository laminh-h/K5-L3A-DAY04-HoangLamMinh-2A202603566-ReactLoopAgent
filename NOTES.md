# Ghi chú phân tích · Day 04

Họ và tên: Hoàng Lam Minh
MSSV: 2A202603566

## 1. Case được chọn

- ID case: A1
- Câu trả lời của agent: Đối chiếu GitHub Actions với Cloudflare Spectrum, nêu incident, trạng thái, phạm vi và nguồn của từng bên; không suy luận hai bên có cùng nguyên nhân.
- Điểm ở lần chạy 1: 100/100.

## 2. Bằng chứng trong trace

Ở bản sau cải tiến, trace ghi nhận bước kiểm tra nguồn sau khi agent hoàn tất điều tra:

```text
validation_missing_source, step=3
urls=https://stspg.io/lv4q3xm9r0rk
     https://stspg.io/18blsgwnbyy6
```

Hai URL đều đến từ các kết quả `get_incident_details` thành công. Agent được yêu cầu sửa câu trả lời; câu trả lời cuối có các URL incident, cùng nguồn Cloudflare. Số lần gọi model của A1 tăng từ 3 lên 4. Baseline không có bước kiểm tra này; judge chấm A1 100 ở cả hai lần.

## 3. Thay đổi đã thực hiện

- Trong `agent.py`, lưu URL từ mỗi `get_incident_details` thành công. Nếu câu trả lời cuối thiếu một URL đã quan sát, ghi `validation_missing_source` vào trace và yêu cầu model sửa trong giới hạn `max_steps`.
- Bổ sung giới hạn tối đa ba lần thực thi cho cùng một yêu cầu bị rate limit; đọc `Retry-After`, chỉ chờ tối đa 2 giây và dừng retry nếu thời gian chờ dài hơn.
- Làm rõ trong system prompt rằng timeout/rate limit không chứng minh trạng thái dịch vụ, và dữ liệu status không chứng minh lịch sử ngoài phạm vi trả về. Sau tool error, thêm nhắc nhở có cấu trúc để model nói rõ phần chưa xác minh, hoặc retry trong giới hạn.
- Với câu hỏi có năm không xuất hiện trong timestamp của status evidence, ghi `validation_limited_history` và yêu cầu model nêu rõ không thể xác định đầy đủ lịch sử của năm đó.
- Vì sao chọn thay đổi này: yêu cầu bài lab đòi hỏi nguồn incident cụ thể. Kiểm tra sự hiện diện URL chỉ đảm bảo link được dẫn, không xác minh mọi claim.

## 4. So sánh hai lần chạy

Cả hai lần dùng `openai/gpt-4o-mini` cho agent và judge, bộ `core`, snapshot và `max_steps=8`. Baseline được chạy không có bước kiểm tra nguồn; bản sau có bước kiểm tra nguồn.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 81.59/100 | 78.86/100 |
| Điểm A1 | 100/100 | 100/100 |
| Model calls của agent | 50 | 49 |
| Judge calls | 22 | 22 |
| Token agent (input + output) | 42,817 | 44,807 |
| Token judge (input + output) | 41,595 | 38,439 |
| Chi phí | Không được provider trả về trong kết quả | Không được provider trả về trong kết quả |

- Kết quả `eval_after.json` sau khi cập nhật: A1 đạt 100 ở cả hai lần; điểm tổng giảm 2.73 điểm so với baseline. Một lần chạy không đủ tách ảnh hưởng của code khỏi dao động của model/judge.
- H7: trace có `TimeoutError` cho Cloudflare; câu trả lời cuối nêu rõ không thể xác minh Cloudflare và không kết luận Operational. Điểm judge là 65; đây là ước lượng, cần đối chiếu expected và trace.
- A5: trace có `validation_limited_history` cho năm 2025. Câu trả lời cuối nói rõ dữ liệu không bao phủ đủ lịch sử và không thể kết luận không có incident trong năm đó; điểm judge là 75.
- `eval_after.json` được chạy lại trên phiên bản có kiểm tra lịch sử; cả 22 case đều được chấm và cấu hình khớp với `eval_before.json`. Kiểm tra nguồn và kiểm tra năm chỉ xử lý các dạng thiếu bằng chứng đã phát hiện, không chứng minh mọi claim đều đúng.

## 5. Giới hạn

Eval dùng snapshot cố định nên không phản ánh trạng thái hiện tại của các dịch vụ. Điểm judge là ước lượng; cùng một model được dùng để chạy agent và chấm, và kết quả có thể dao động giữa các lần. Mỗi phiên bản chỉ chạy một lần nên chưa chứng minh độ ổn định. Kiểm tra nguồn xác nhận câu trả lời có chứa URL đã quan sát, không xác nhận mọi nhận định đều được nguồn hỗ trợ.
