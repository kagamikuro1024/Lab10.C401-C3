# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lê Văn Quang Trung  
**Vai trò:** End-to-end Lead — tổng hợp pipeline, review code, merge, verify toàn bộ  
**Ngày nộp:** 2026-04-15  
**Độ dài yêu cầu:** **400–650 từ**

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `etl_pipeline.py` — entrypoint chính: tôi review toàn bộ flow ingest → clean → validate → embed → manifest → freshness, đảm bảo các module kết nối đúng
- `transform/cleaning_rules.py` — review và sửa lỗi vị trí 3 cleaning rules mới (7, 8, 9) mà Đạt thêm
- `quality/expectations.py` — verify 2 expectations mới (E7, E8) hoạt động đúng trong log
- `docs/quality_report.md`, `reports/group_report.md` — tổng hợp chứng cứ từ các thành viên

**Kết nối với thành viên khác:**

Tôi nhận code từ Đạt (cleaning + expectations), Vinh (embed + eval), Nghĩa (data contract), Minh (docs + runbook), chạy end-to-end verify rồi merge vào main. Khi phát hiện lỗi, tôi fix và rerun pipeline để tạo log evidence mới.

**Bằng chứng:** Comment `Owner: Đạt` trong `cleaning_rules.py`; log files `run_sprint2.log`, `run_sprint3-clean.log`, `run_inject-bad.log` do tôi chạy verify.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Khi review code Đạt, tôi phát hiện 3 cleaning rules mới (Rule 7: replacement char quarantine, Rule 8: BOM strip, Rule 9: NFKC normalize) được đặt **sau** `cleaned.append()` — tức là sau khi row đã được thêm vào danh sách cleaned. Điều này khiến:

- Rule 8 và 9 sửa biến `fixed_text` nhưng bản đã append là bản cũ → **không có tác dụng** trên output
- Rule 7 (replacement char) append row vào quarantine, nhưng row đã nằm trong cleaned → **row xuất hiện ở cả hai list**

Tôi quyết định di chuyển tất cả rules về **trước** `cleaned.append()`, theo thứ tự: dedupe check → replacement char quarantine → BOM strip → NFKC normalize → refund fix → append. Thứ tự này đảm bảo quarantine check chạy trước text transformation, và text được chuẩn hoá trước khi tính `chunk_id` hash — quan trọng vì hash phụ thuộc nội dung text.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** Khi chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`, pipeline crash với `UnicodeEncodeError: 'charmap' codec can't encode character '\u2192'` — ký tự mũi tên `→` trong message WARN không hiển thị được trên Windows console (cp1252 encoding).

**Phát hiện:** Exit code 1 thay vì 0 — pipeline dừng trước khi embed dữ liệu xấu, khiến Sprint 3 inject scenario không thực hiện được.

**Fix:** Thay ký tự Unicode `→` bằng ASCII `->` trong dòng log WARN (file `etl_pipeline.py` dòng 91). Sau fix, pipeline inject-bad chạy thành công: expectation E3 `refund_no_stale_14d_window` FAIL (halt) :: violations=1, nhưng `--skip-validate` cho phép embed tiếp → log ghi `embed_prune_removed=1` khi vector cũ bị thay thế. Run_id: `inject-bad`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**Trước (run_id=inject-bad)** — `artifacts/eval/after_inject_bad.csv`:
```
q_refund_window, contains_expected=yes, hits_forbidden=yes
```
→ Chunk stale "14 ngày làm việc" vẫn trong top-k do `--no-refund-fix`.

**Sau (run_id=sprint3-clean)** — `artifacts/eval/after_sprint3_clean.csv`:
```
q_refund_window, contains_expected=yes, hits_forbidden=no
```
→ Pipeline chuẩn fix "14→7 ngày", prune xóa vector cũ. Chỉ còn chunk đúng policy v4.

**Merit evidence (run_id=sprint3-clean):**
```
q_leave_version, contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes
```
→ HR bản 2026 (12 ngày phép), HR bản 2025 (10 ngày) đã bị quarantine. Top-1 doc đúng = `hr_leave_policy`.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ đọc `hr_leave_min_effective_date` từ `contracts/data_contract.yaml` thay vì hard-code `"2026-01-01"` trong `cleaning_rules.py` dòng 105. Điều này cho phép thay đổi cutoff versioning qua config mà không cần sửa code — đáp ứng tiêu chí Distinction (d) trong SCORING.md. Tôi sẽ inject test với cutoff khác để chứng minh quyết định clean thay đổi theo config.
