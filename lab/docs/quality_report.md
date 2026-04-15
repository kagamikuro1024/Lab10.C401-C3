# Quality report — Lab Day 10 (nhóm)

**run_id:** sprint3-clean (pipeline chuẩn) / inject-bad (pipeline inject)  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | inject-bad | sprint3-clean | Ghi chú |
|--------|-----------|---------------|---------|
| raw_records | 10 | 10 | Cùng CSV đầu vào `data/raw/policy_export_dirty.csv` |
| cleaned_records | 6 | 6 | inject: giữ chunk stale "14 ngày" thay vì fix thành "7 ngày" |
| quarantine_records | 4 | 4 | Cùng 4 lý do: duplicate, missing_effective_date, stale_hr, unknown_doc_id |
| Expectation halt? | **FAIL** (E3: `refund_no_stale_14d_window` violations=1) — bypass bởi `--skip-validate` | Không (tất cả 8 expectations OK) | inject dùng `--skip-validate` để vẫn embed dữ liệu xấu |

---

## 2. Before / after retrieval (bắt buộc)

> Artifact: `artifacts/eval/after_inject_bad.csv` vs `artifacts/eval/after_sprint3_clean.csv`

**Câu hỏi then chốt:** refund window (`q_refund_window`)

**Trước (inject-bad):**
```
q_refund_window, contains_expected=yes, hits_forbidden=yes
```
→ Chunk stale "14 ngày làm việc" vẫn nằm trong top-k do `--no-refund-fix` giữ nguyên text gốc từ CSV migration lỗi (policy v3). Expectation E3 `refund_no_stale_14d_window` FAIL (halt) nhưng bị bypass bởi `--skip-validate`.

**Sau (sprint3-clean):**
```
q_refund_window, contains_expected=yes, hits_forbidden=no
```
→ Pipeline chuẩn fix "14 ngày" → "7 ngày làm việc", prune vector cũ (`embed_prune_removed=1`), chỉ còn chunk đúng policy v4 trong Chroma.

**Merit: versioning HR — `q_leave_version`**

**Trước (inject-bad):**
```
q_leave_version, contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes
```
→ HR leave không bị ảnh hưởng bởi inject-bad (chỉ tắt refund fix, HR stale quarantine vẫn hoạt động).

**Sau (sprint3-clean):**
```
q_leave_version, contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes
```
→ Đúng: "12 ngày phép năm" (bản 2026), không chứa "10 ngày" (bản 2025 đã bị quarantine). Top-1 doc = `hr_leave_policy` ✅

---

## 3. Freshness & monitor

Kết quả freshness_check: **FAIL**
- `age_hours ≈ 119.5h` (CSV exported `2026-04-10T08:00:00`, chạy ngày `2026-04-15`)
- `sla_hours = 24.0h`
- **Giải thích:** CSV mẫu có `exported_at` cũ 5 ngày → FAIL là hành vi đúng và mong đợi
- SLA 24h phù hợp cho batch export hàng ngày từ DB nội bộ
- Lệnh kiểm tra: `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint3-clean.json`

---

## 4. Corruption inject (Sprint 3)

**Kịch bản:** `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`

- `--no-refund-fix`: giữ nguyên chunk stale "14 ngày làm việc" (policy v3 migration error) — không apply rule fix refund window
- `--skip-validate`: bỏ qua expectation halt → vẫn embed dữ liệu xấu vào Chroma dù E3 FAIL

**Cách phát hiện:**
1. Expectation E3 `refund_no_stale_14d_window` FAIL (halt) :: violations=1 — pipeline tự dừng nếu không `--skip-validate`
2. Eval `q_refund_window`: `hits_forbidden=yes` — chunk stale "14 ngày" vẫn trong top-k
3. Log: `embed_prune_removed=1` khi chạy lại sprint3-clean → vector cũ bị xóa

**Chứng cứ log inject-bad (trích):**
```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1
WARN: expectation failed but --skip-validate -> tiep tuc embed (chi dung cho demo Sprint 3).
embed_prune_removed=1
```

---

## 5. Hạn chế & việc chưa làm

- Chưa xử lý OCR confidence cho PDF scan (chỉ đọc plaintext `.txt`)
- Chưa có CDC real-time cho database changes — pipeline chỉ batch
- Embedding model `all-MiniLM-L6-v2` chưa tối ưu cho tiếng Việt dài (có thể thay `bkai-foundation-models/vietnamese-bi-encoder`)
- Freshness chỉ đo 1 boundary (`exported_at` = publish); chưa đo ingest boundary riêng
- Rule versioning HR hard-code ngày `2026-01-01` trong code — nên đọc từ `contracts/data_contract.yaml` (`hr_leave_min_effective_date`)
