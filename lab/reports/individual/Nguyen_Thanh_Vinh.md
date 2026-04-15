# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Thành Vinh  
**Vai trò:** Embed
**Ngày nộp:** 15/04/2026  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `etl_pipeline.py` và `eval_retrival.py`

**Kết nối với thành viên khác:**

Đánh giá kết quả của Đạt.

**Bằng chứng (commit / comment trong code):**

```
git commit -m "[Sprint 2] embed: eval retrieval baseline - all 4 questions pass
- after_clean_eval.csv: 4/4 questions pass
- q_refund_window: contains_expected=yes, hits_forbidden=no
- q_leave_version: top1_doc_expected=yes (Merit)
- Collection count=6, no stale vectors"

git commit -m "[Sprint 3] embed: inject corruption + before/after evidence
- inject-bad: --no-refund-fix --skip-validate
- q_refund_window: hits_forbidden=yes (inject) → no (clean)
- sprint3-clean: all eval pass
- Evidence files: after_inject_bad.csv, after_clean_eval.csv"

```

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Upsert theo `chunk_id` + prune stale vectors. Khi thực hiện lệnh Upsert, nếu `chunk_id` đã tồn tại, nó sẽ ghi đè bản mới nhất lên bản cũ. Nếu chưa có, nó sẽ tạo mới. Điều này đảm bảo dù bạn có nhấn "Run" 10 lần, số lượng vector trong DB vẫn không đổi. Sau khi cập nhật các chunk mới, hệ thống phải tìm và xóa tất cả các vector thuộc tài liệu đó nhưng không nằm trong danh sách `chunk_id` vừa cập nhật.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)
Injected-bad: "14 ngày" xuất hiện trong top-k, nhận biết qua file `artifacts/eval/after_inject_bad.csv`, cột `hits_forbidden=yes`. 
Sửa: Chạy lại Data pipeline chuẩn. Sau khi chạy lại, cột `hits_forbidden=no` trong file `artifacts/eval/after_sprint3_clean.csv`

---

## 4. Bằng chứng trước / sau (80–120 từ)

Trong file `artifacts/eval/after_inject_bad.csv`, trước Data pipeline chuẩn:
```
q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,yes,,3
```

Sau Data pipeline chuẩn trong file ``:
```
q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,no,yes,,3
```

---

## 5. Cải tiến tiếp theo (40–80 từ)

Tôi muốn cải tiến embed: chỉ chunk phần thay đổi thay vì full upsert.

