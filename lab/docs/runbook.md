# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

Agent trả lời "khách hàng có thể yêu cầu hoàn tiền trong vòng **14 ngày làm việc**" — sai so với chính sách v4 hiện hành (7 ngày).
User báo: "chính sách đã cập nhật từ tháng 2, sao agent vẫn nói cũ?"

---

## Detection

- **freshness_check**: `FAIL` — `age_hours=119.7 > sla_hours=24.0`
  - Chạy: `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run_id>.json`
- **expectation[refund_no_stale_14d_window]**: `FAIL (halt)` — phát hiện chunk vẫn chứa "14 ngày làm việc"
- **eval_retrieval.py**: `q_refund_window` → `hits_forbidden=yes` — chunk stale trong top-k

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/manifest_<run_id>.json` | `latest_exported_at` quá SLA → root cause: data chưa refresh |
| 2 | Mở `artifacts/quarantine/quarantine_<run_id>.csv` | Kiểm tra lý do quarantine — nếu thiếu `stale_refund` → rule chưa chạy |
| 3 | Chạy `python eval_retrieval.py` | `q_refund_window`: nếu `hits_forbidden=yes` → Chroma còn vector cũ |
| 4 | Kiểm tra Chroma collection | `col.get(where={"doc_id":"policy_refund_v4"})` → tìm chunk "14 ngày" |

---

## Mitigation

1. Chạy lại pipeline chuẩn (KHÔNG dùng `--no-refund-fix`):
   ```bash
   python etl_pipeline.py run --run-id hotfix-20260415
   ```
2. Verify eval:
   ```bash
   python eval_retrieval.py --out artifacts/eval/hotfix_eval.csv
   ```
3. Nếu `q_refund_window` có `hits_forbidden=no` → pipeline đã fix + prune thành công
4. Nếu vẫn FAIL → kiểm tra raw CSV có được cập nhật chưa, hoặc thêm rule mới

---

## Prevention

- Schedule pipeline chạy mỗi 12h (SLA buffer = 50% của 24h)
- Alert khi `freshness_check=WARN` (50% SLA) trước khi FAIL
- Không bao giờ dùng `--skip-validate` trong production
- Expectation `refund_no_stale_14d_window` = halt → pipeline tự dừng nếu data sai
- Ghi `run_id` vào Chroma metadata → trace ngược từ agent answer đến pipeline run
- Review quarantine CSV hàng tuần — flag pattern mới
