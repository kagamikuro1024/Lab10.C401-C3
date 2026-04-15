# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Trọng Minh  
**Vai trò:** Ingestion / Cleaning / Embed / Monitoring — Docs Owner (group_report + docs)  
**Ngày nộp:** 2026-04-15  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)


---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- Tôi phụ trách mảng tài liệu, trực tiếp biên soạn `reports/group_report.md` và các file trong thư mục docs gồm `docs/quality_report.md`, `docs/runbook.md`, `docs/pipeline_architecture.md`, `docs/data_contract.md`.
- Công việc chính của tôi là chuẩn hóa narrative kỹ thuật từ artifact thật (manifest, eval CSV, log), bảo đảm số liệu trong báo cáo nhóm thống nhất giữa các phần pipeline, quality và monitoring.

**Kết nối với thành viên khác:**

Tôi nhận kết quả chạy từ Trung (end-to-end), số liệu clean/expectation từ Đạt, và bằng chứng embed/prune từ Vinh để tổng hợp thành một báo cáo nhóm liền mạch. Tôi cũng đối chiếu owner table trong tài liệu kiến trúc để mô tả đúng ranh giới trách nhiệm từng người.

**Bằng chứng (commit / comment trong code):**

Bằng chứng nằm trong bộ tài liệu tôi viết: `reports/group_report.md` (bản nộp nhóm) và các tài liệu hỗ trợ trong docs. Tôi dùng trực tiếp hai run_id `inject-bad` và `sprint3-clean`, cùng artifact `artifacts/eval/after_inject_bad.csv`, `artifacts/eval/after_sprint3_clean.csv`, `artifacts/manifests/manifest_sprint3-clean.json` để điền số liệu.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật tôi chịu trách nhiệm là thiết kế bộ tài liệu theo chuỗi điều tra sự cố, thay vì viết rời rạc từng file. Cụ thể, `pipeline_architecture.md` mô tả luồng và owner, `quality_report.md` ghi định lượng before/after, `runbook.md` quy định thao tác detection-diagnosis-mitigation, còn `group_report.md` là bản tổng hợp để nộp. Cách này giúp một metric được truy vết xuyên suốt: ví dụ `q_refund_window hits_forbidden=yes/no` xuất hiện nhất quán từ báo cáo chất lượng sang runbook xử lý incident. Tôi cũng cố định naming theo `run_id` để tránh nhầm artifact khi nhóm rerun nhiều lần. Nhờ đó, người chấm có thể lần ngược từ kết luận trong group report về đúng file evidence mà không bị lệch số.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Anomaly tôi xử lý ở vai trò docs là lỗi lệch narrative giữa các tài liệu: ban đầu phần freshness FAIL dễ bị hiểu nhầm là nguyên nhân trực tiếp gây sai policy “14 ngày”. Tôi đã chỉnh lại diễn giải trong `docs/quality_report.md` và `docs/runbook.md` để tách bạch nguyên nhân: freshness chỉ phản ánh độ cũ dữ liệu, còn lỗi nội dung được chốt bởi expectation E3 và retrieval evidence. Tôi đưa cùng một bộ chứng cứ vào cả hai tài liệu: `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1` và dòng eval `q_refund_window, contains_expected=yes, hits_forbidden=yes`. Sau đó, ở run sạch `sprint3-clean`, tôi cập nhật trạng thái `hits_forbidden=no` và log `embed_prune_removed=1` để đóng vòng incident rõ ràng.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Tôi đối chiếu trực tiếp hai artifact `artifacts/eval/after_inject_bad.csv` và `artifacts/eval/after_sprint3_clean.csv`, rồi phản ánh nguyên văn vào `docs/quality_report.md` và `reports/group_report.md`.

- Trước (`run_id=inject-bad`): `q_refund_window, contains_expected=yes, hits_forbidden=yes`
- Sau (`run_id=sprint3-clean`): `q_refund_window, contains_expected=yes, hits_forbidden=no`

Tôi kiểm tra thêm câu hỏi versioning HR để tránh kết luận một chiều:

- Trước (`run_id=inject-bad`): `q_leave_version, contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes`
- Sau (`run_id=sprint3-clean`): `q_leave_version, contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes`

Điều này cho thấy inject chỉ làm hỏng nhánh refund, còn nhánh HR vẫn ổn định; kết luận này được tôi giữ nhất quán giữa báo cáo nhóm và tài liệu docs.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ bổ sung một bảng traceability trong `group_report.md` để map từng claim với artifact cụ thể (manifest, eval, log) và đồng bộ phần ngưỡng HR stale theo `contracts/data_contract.yaml` để tránh tài liệu/code lệch nhau theo thời gian. Cải tiến này giúp phần docs kiểm chứng nhanh hơn khi nhóm rerun nhiều run_id.
