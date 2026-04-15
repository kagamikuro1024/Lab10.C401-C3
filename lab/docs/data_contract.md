# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `data/raw/policy_export_dirty.csv` | Batch CSV export load qua `load_raw_csv()` | encoding error, missing effective_date, duplicate chunk_text, invalid effective_date format, stale refund window | `raw_records`, `quarantine_records`, `manifest` freshness check |
| `data/docs/*.txt` | Canonical docs read + embed metadata | file missing, doc_id mismatch, version conflict, stale policy | freshness SLA, `latest_exported_at`, canonical source coverage |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | ID ổn định sau clean, hash `doc_id` + seq để tránh trùng |
| doc_id | string | Có | Khóa logic của nguồn policy/docs |
| chunk_text | string | Có | Nội dung chunk đã chuẩn hoá và fix stale rule |
| effective_date | date | Có | Chuẩn hoá ISO `YYYY-MM-DD` hoặc reject nếu format sai |
| exported_at | datetime | Có | Thời điểm export raw, dùng cho tracking và freshness |

---

## 3. Quy tắc quarantine vs drop

- `unknown_doc_id`: quarantine khi doc_id không nằm trong `allowed_doc_ids`.
- `missing_effective_date` / `invalid_effective_date_format`: quarantine khi effective_date không parse được.
- `stale_hr_policy_effective_date`: quarantine khi `hr_leave_policy` có effective_date cũ hơn 2026-01-01.
- `missing_chunk_text`: quarantine khi nội dung rỗng.
- `duplicate_chunk_text`: quarantine khi cùng nội dung chunk đã xuất hiện trước đó.
- `missing_chunk_text` và các lỗi định danh phải được review trước khi merge lại.

---

## 4. Phiên bản & canonical

Source of truth cho policy refund là `data/docs/policy_refund_v4.txt` với effective date 2026-02-01. Phiên bản HR stale check được đồng bộ với `contracts/data_contract.yaml` và `transform/cleaning_rules.py` để đảm bảo `hr_leave_policy` chỉ chấp nhận effective_date >= 2026-01-01.
