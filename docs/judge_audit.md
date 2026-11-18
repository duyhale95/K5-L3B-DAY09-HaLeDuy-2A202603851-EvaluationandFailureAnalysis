# Judge audit

Chạy:
- `.\.venv\Scripts\python.exe -m lab9.judge --csv data/eval/judge_tone_public.csv --audit 5`
- `.\.venv\Scripts\python.exe -m lab9.judge --csv data/eval/judge_helpful_public.csv --audit 10`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| tone_ok | 100 | 0.757 | 0.857 | 0.800 | Có (PASS) | dùng |
| helpful | 100 | 0.375 | 1.000 | 0.300 | Không (FAIL: κ < 0.6, R < 0.7) | loại |

- `tone_ok`: `{'tp': 24, 'fp': 4, 'fn': 6, 'tn': 66}`, observed fail rate `0.28` → Rogan-Gladen corrected `0.3`.
- `helpful`: `{'tp': 9, 'fp': 0, 'fn': 21, 'tn': 70}`, observed fail rate `0.09` → Rogan-Gladen corrected `0.3`.

## ≥5 case audit thủ công
| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú |
|---|---|---|---|---|---|
| 1 | `HL-015` (`helpful`): `"Dạ, LogiX đã hoàn 55.000đ cho đơn O-112 ạ."` | PASS | FAIL | Người đúng | Sai tiền hoàn: reply ghi `55.000đ`, nhưng ledger ghi `40.000đ`. Judge không có dữ liệu ledger để đối chiếu số tiền nên cho PASS nhầm. |
| 2 | `HL-016` (`helpful`): `"Dạ, LogiX thành thật xin lỗi... LogiX đã hoàn 55.000đ cho đơn O-404..."` | PASS | FAIL | Người đúng | Sai tiền hoàn: reply ghi `55.000đ`, ledger ghi `35.000đ`. Judge bị đánh lừa bởi câu xin lỗi dài, lịch sự nhưng sai thông tin số học. |
| 3 | `HL-020` (`helpful`): `"Dạ, LogiX đã hoàn 90.000đ cho đơn O-303 ạ."` | PASS | FAIL | Người đúng | Sai tiền hoàn: reply ghi `90.000đ`, ledger ghi `120.000đ`. |
| 4 | `HL-028` (`helpful`): `"Dạ, LogiX thành thật xin lỗi... LogiX đã hoàn 50.000đ cho đơn O-677..."` | PASS | FAIL | Người đúng | Sai tiền hoàn: reply ghi `50.000đ`, ledger ghi `35.000đ`. |
| 5 | `H-008` (`tone_ok`): `"Em đã hoàn tiền cho anh rồi nha, anh check giúp em."` | PASS | FAIL | Người đúng | Xưng `"em"` + `"nha"`, quá suồng sã theo chuẩn CS LogiX; judge nhầm sang thân thiện. |
| 6 | `H-029` (`tone_ok`): `"Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm."` | PASS | FAIL | Người đúng | Giọng ra lệnh (`"không cần nhắn thêm"`), thiếu lịch sự; judge chỉ bắt từ khóa `"Dạ, vui lòng"`. |
| 7 | `H-042` (`tone_ok`): `"Dạ, đơn O-808 đã hoàn 120.000đ. Tiền về trong 3–5 ngày ạ."` | FAIL | PASS | Người đúng | Câu trả lời ngắn gọn, lịch sự, đủ thông tin; judge phạt nhầm vì đòi hỏi phải có lời xin lỗi. |
| 8 | `H-051` (`tone_ok`): `"Dạ, LogiX đã refund 90.000đ cho đơn O-202 ạ."` | FAIL | PASS | Người đúng | Dùng từ `"refund"` được CS LogiX cho phép; judge phạt nhầm vì lẫn tiếng Anh. |

## Cách sửa rubric & tách kiểm tra số tiền sang code
- **Tách kiểm tra số tiền và mã đơn sang code (deterministic verifier):** Không dùng LLM judge để chấm tính đúng/sai của số tiền hoàn hay mã đơn vì judge không có ground-truth ledger và dễ bị nhiễu bởi văn phong lịch sự. Cần viết assertion bằng code (regex/parser + đối chiếu trực tiếp với ledger/tool output của đơn hàng) để bắt buộc `amount_in_reply == ledger_amount` và `order_id_in_reply == expected_order_id`.
- **Sửa rubric cho `helpful` và `tone_ok`:**
  - `helpful`: Chỉ đánh giá sau khi đã qua deterministic check về số tiền/mã đơn; tập trung vào việc trả lời có đúng trọng tâm câu hỏi và nêu rõ bước tiếp theo hay không.
  - `tone_ok`: Bổ sung quy tắc rõ ràng cho ngữ cảnh CS LogiX (cấm giọng ra lệnh như `"không cần nhắn thêm"`, cấm từ ngữ quá suồng sã như `"nha"`/`"bạn ơi"`, không phạt các câu ngắn gọn đủ ý hoặc từ mượn chuyên ngành được phép như `"refund"`).

## Kết luận
- **Tiêu chí bị loại:** `helpful` bị loại vì không đạt ngưỡng FLOORS (`κ = 0.375 < 0.6`, `Recall = 0.300 < 0.7`, dù `Precision = 1.000`). Dựa trên ma trận nhầm lẫn `{'tp': 9, 'fp': 0, 'fn': 21, 'tn': 70}`, judge bỏ sót 21/30 ca FAIL (`FN = 21`), chủ yếu là các ca hoàn sai số tiền so với ledger (làm observed fail rate chỉ là `0.09` trong khi thực tế sau hiệu chỉnh Rogan-Gladen là `0.30`).
- **Tiêu chí được giữ lại:** `tone_ok` đạt FLOORS (`κ = 0.757 ≥ 0.6`, `Precision = 0.857 ≥ 0.8`, `Recall = 0.800 ≥ 0.7`, ma trận `{'tp': 24, 'fp': 4, 'fn': 6, 'tn': 66}`) nên có thể dùng tạm thời kèm theo việc tinh chỉnh rubric cho các edge case về xưng hô và từ vựng cho phép.
- **Rủi ro còn lại:** Judge chỉ được kiểm trên dữ liệu replay đã công khai (`judge_tone_public.csv`, `judge_helpful_public.csv`); cần nhãn người mới trên tập held-out thật để xác nhận hiệu năng trước khi đưa vào CI/production.
