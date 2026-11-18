# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

Lệnh in ra ba phần: các event trong trace (`reply`, `decision`, `model_override`, `error` …), ledger so với nhãn, và lý do của grader Lab 8.

## Mẫu điền (mỗi bạn 1 case, lấy từ trial đang fail trong report)

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-002@11` (chạy qua `lab9.trace_case --shape seeded_regression --claim C-002 --seed 11`) |
| Triệu chứng | `lab8 grader` (`pass_rate`) fail (`status resolved != denied`); `refund_amount_correct` fail (`ledger=20000 != label=0`); `db_fact_kept` fail (`1 override event(s)`). Reply nói `"Dạ, LogiX đã hoàn 20.000đ cho đơn O-202."`, trong khi ledger ghi `refund_vnd=20000`, `refund_count=1` (nhãn mong đợi: `refund_vnd=0`, `refund_count=0`, `status=denied`). |
| Bước sai đầu tiên | Event `3` `model_override`: `{"delay_days": 5}` |
| Vì sao là bước *đầu tiên* | Event `1` (`handoff`) tạo handoff đúng chuẩn với `allowed_tools` chỉ định; event `2` (`investigator`) tra cứu đúng đơn `O-202` từ DB (`visible: true`, `reason: "ok"`). Đến event `3`, agent lấy `delay_days: 5` từ ticket đè lên giá trị DB khiến event `4` (`quote`) tính ra `20000`đ (`SHIP-LATE-03`) thay vì `0`đ. |
| Failure mode | `FM-03` (`ticket_trust — tin ticket hơn DB`) |
| Đối chứng | Trên `fixed C-002@11`: không có event `model_override`; sau event `2` (`investigator`), event `3` (`quote`) trả `{"amount_vnd": 0, "reason": "no_entitlement", "clause_ids": [], "parts": {}}`, event `4` (`reply`) trả lời `"Dạ, đơn O-202 chưa đủ điều kiện hoàn theo chính sách hiện hành."`, ledger `refund_vnd=0`, `refund_count=0`, `status=denied`, `lab8 grader: PASS (ok)`. |
| Regression test | `test_c002_seed11_seeded_regression_vs_fixed` trong `src/tests/test_lab9_graders.py` — kiểm tra `DbFactKept`, `RefundAmountCorrect` và `TrialPass` đều fail trên `seeded_regression C-002@11` và pass trên `fixed C-002@11`. |
| Chưa chắc chắn | Output rút gọn của `trace_case` cắt payload ở 110 ký tự nên không hiện đầy đủ `delay_days` của DB trong event `2` và không in nội dung `ticket` gốc ở event `3`; phải đối chiếu thêm dữ liệu `claims`/`orders` hoặc code để biết `parsed_delay=5` khác giá trị DB. |

## Quy tắc

- Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai.
- "Bước sai đầu tiên" phải dẫn số event cụ thể, không phải "agent bị nhầm".
- Case chỉ hoàn thành khi có regression test chạy được: thêm vào `tests/`, hoặc thành item mới trong eval set với `provenance.from_trace`.
