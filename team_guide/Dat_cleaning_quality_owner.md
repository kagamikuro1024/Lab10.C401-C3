# 📋 Kế Hoạch Cá Nhân — Đạt (Cleaning / Quality Owner)

## Lab Day 10 — Data Pipeline & Data Observability

**Vai trò:** Cleaning & Quality Owner  
**Nhánh Git:** `feature/cleaning`  
**File chịu trách nhiệm chính:**
- `transform/cleaning_rules.py` — thêm ≥3 rule mới
- `quality/expectations.py` — thêm ≥2 expectation mới
- `docs/quality_report.md` — quality report (từ template)

---

## 🌿 Tạo Nhánh

```bash
git checkout main
git pull origin main
git checkout -b feature/cleaning
```

---

## Sprint 1 (60 phút) — Đọc hiểu baseline

### Nhiệm vụ cụ thể

#### 1. Setup + chạy pipeline lần đầu (10 phút)
```bash
cd lab
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
python etl_pipeline.py run --run-id dat-test
```

#### 2. Đọc kỹ baseline cleaning_rules.py (25 phút)
**Baseline đã có 6 rules:**
1. Quarantine `doc_id` không thuộc allowlist → `legacy_catalog_xyz_zzz` bị loại (row 9)
2. Chuẩn hoá `effective_date` sang ISO `YYYY-MM-DD` → row 10 `01/02/2026` → `2026-02-01`
3. Quarantine HR cũ `effective_date < 2026-01-01` → row 7 (HR 2025, 10 ngày phép) bị loại
4. Quarantine `chunk_text` rỗng → row 5 bị loại
5. Dedupe `chunk_text` (normalize + lowercase) → row 2 (trùng row 1) bị loại
6. Fix stale refund `14 ngày → 7 ngày` → row 3 được sửa

**Kết quả baseline:** 10 raw → 6 cleaned + 4 quarantine

#### 3. Đọc kỹ baseline expectations.py (15 phút)
**Baseline đã có 6 expectations:**
- E1: `min_one_row` (halt) — có ít nhất 1 dòng
- E2: `no_empty_doc_id` (halt) — không doc_id rỗng
- E3: `refund_no_stale_14d_window` (halt) — không còn "14 ngày làm việc"
- E4: `chunk_min_length_8` (warn) — chunk >= 8 chars
- E5: `effective_date_iso_yyyy_mm_dd` (halt) — ngày đúng ISO
- E6: `hr_leave_no_stale_10d_annual` (halt) — không còn "10 ngày phép năm"

#### 4. Phân tích dirty CSV (10 phút)
Mở `data/raw/policy_export_dirty.csv` — ghi chú từng lỗi:

| Row | doc_id | Lỗi | Rule baseline xử lý |
|-----|--------|------|---------------------|
| 2 | policy_refund_v4 | Duplicate row 1 | Rule 5 (dedupe) |
| 3 | policy_refund_v4 | Stale "14 ngày" | Rule 6 (refund fix) |
| 5 | policy_refund_v4 | Empty chunk_text + missing date | Rule 4 |
| 7 | hr_leave_policy | Bản HR cũ 2025 | Rule 3 |
| 9 | legacy_catalog | Doc_id lạ | Rule 1 (allowlist) |
| 10 | it_helpdesk_faq | Date DD/MM/YYYY | Rule 2 (normalize) |

### Commit Sprint 1

```bash
git commit -m "[Sprint 1] cleaning: analyze baseline rules and dirty CSV structure

- Documented 6 baseline rules and 6 baseline expectations
- Mapped each dirty row to its handling rule
- Prepared plan for 3 new rules + 2 new expectations"
git push origin feature/cleaning
```

---

## Sprint 2 (60 phút) — ⭐ Sprint CHÍNH của Đạt

### Nhiệm vụ cụ thể

#### 1. Thêm 3 Cleaning Rules mới (35 phút)

**Mở `transform/cleaning_rules.py`**, thêm vào hàm `clean_rows()`:

##### Rule 7: BOM / encoding strip
```python
# === Rule mới 7: BOM strip ===
# Loại BOM character (\ufeff) đầu chunk_text — metric_impact: quarantine nếu inject BOM
# Owner: Đạt
if fixed_text.startswith('\ufeff'):
    fixed_text = fixed_text.lstrip('\ufeff')
    fixed_text += " [cleaned: bom_stripped]"
```

##### Rule 8: Normalize unicode (NFKC)
```python
import unicodedata

# === Rule mới 8: Unicode NFKC normalize ===
# Chuẩn hoá unicode NFKC để tránh cùng nội dung nhưng encoding khác → metric_impact: content_hash thay đổi
# Owner: Đạt
fixed_text = unicodedata.normalize("NFKC", fixed_text)
```

##### Rule 9: Đánh dấu chunk có ký tự replacement (U+FFFD)
```python
# === Rule mới 9: Flag replacement character (U+FFFD) ===
# Quarantine chunk chứa ký tự thay thế (lỗi encoding nghiêm trọng) — metric_impact: quarantine_records tăng
# Owner: Đạt
if '\ufffd' in text:
    quarantine.append({**raw, "reason": "contains_replacement_char_encoding_error"})
    continue
```

> **QUAN TRỌNG:** Mỗi rule phải có comment rõ: tên, mục đích, metric_impact

#### 2. Thêm 2 Expectations mới (15 phút)

**Mở `quality/expectations.py`**, thêm vào hàm `run_expectations()`:

##### E7: chunk_id uniqueness

```python
# E7: chunk_id phải unique — phát hiện hash collision hoặc logic sai
# Owner: Đạt
# Severity: halt — trùng chunk_id sẽ overwrite khi upsert
seen_ids = set()
dup_ids = []
for r in cleaned_rows:
    cid = r.get("chunk_id", "")
    if cid in seen_ids:
        dup_ids.append(cid)
    seen_ids.add(cid)
ok7 = len(dup_ids) == 0
results.append(
    ExpectationResult(
        "chunk_id_unique",
        ok7,
        "halt",
        f"duplicate_chunk_ids={len(dup_ids)}",
    )
)
```

##### E8: exported_at not future

```python
# E8: exported_at không được ở tương lai — phát hiện clock drift hoặc inject sai timestamp
# Owner: Đạt
# Severity: warn — có thể chấp nhận nhưng cần ghi nhận
import re as _re
from datetime import datetime as _dt, timezone as _tz

future_rows = []
now_str = _dt.now(_tz.utc).isoformat()
for r in cleaned_rows:
    exp = (r.get("exported_at") or "").strip()
    if exp and exp > now_str:
        future_rows.append(exp)
ok8 = len(future_rows) == 0
results.append(
    ExpectationResult(
        "exported_at_not_future",
        ok8,
        "warn",
        f"future_exported_at={len(future_rows)}",
    )
)
```

#### 3. Test + verify metrics (10 phút)

```bash
# Chạy pipeline sau khi thêm rules
python etl_pipeline.py run --run-id sprint2-dat

# Verify expectations pass
# Kiểm tra log: tất cả E1-E8 phải OK
```

**Ghi lại metric_impact:**

| Rule / Expectation | Trước (baseline) | Sau (thêm rule) | Chứng cứ |
|---|---|---|---|
| Rule 9: replacement char | quarantine=4 | quarantine=4 (CSV mẫu không có U+FFFD, nhưng inject sẽ tăng) | log sprint2-dat |
| Rule 8: NFKC normalize | N/A | chunk_text chuẩn hoá | diff trên cleaned CSV |
| Rule 7: BOM strip | N/A | BOM sẽ bị strip nếu inject | log khi inject BOM |
| E7: chunk_id_unique | N/A | OK (0 duplicate) | expectation log |
| E8: exported_at_not_future | N/A | OK (0 future) | expectation log |

### Commit Sprint 2

```bash
git add transform/cleaning_rules.py quality/expectations.py
git commit -m "[Sprint 2] cleaning: add 3 new rules + 2 new expectations

New rules in cleaning_rules.py:
- Rule 7: BOM strip (metric: quarantine if BOM injected)
- Rule 8: Unicode NFKC normalize (metric: content_hash changes)
- Rule 9: Replacement char quarantine (metric: quarantine_records +1 per injection)

New expectations in expectations.py:
- E7: chunk_id_unique (halt) — detect hash collision
- E8: exported_at_not_future (warn) — detect clock drift

All expectations pass on sprint2-dat run"
git push origin feature/cleaning
```

> **Sau đó:** Báo Trung để merge vào main.

---

## Sprint 3 (60 phút) — Quality Report

### Nhiệm vụ cụ thể

#### 1. Viết quality report (40 phút)
Copy `docs/quality_report_template.md` → `docs/quality_report.md`

Điền đầy đủ:
- **Mục 1** — Tóm tắt số liệu: dùng run_id từ pipeline chạy trước/sau
- **Mục 2** — Before/after: dùng dữ liệu từ `artifacts/eval/after_inject_bad.csv` vs `after_clean_eval.csv`
- **Mục 3** — Freshness: sẽ có kết quả từ Minh
- **Mục 4** — Corruption inject: mô tả `--no-refund-fix --skip-validate`
- **Mục 5** — Hạn chế: ví dụ chưa xử lý OCR confidence, chưa có CDC real-time

#### 2. Hỗ trợ Vinh so sánh eval (20 phút)
- Kiểm tra dòng `q_refund_window` trong 2 file eval
- Kiểm tra dòng `q_leave_version` (Merit)

### Commit Sprint 3

```bash
git add docs/quality_report.md
git commit -m "[Sprint 3] quality: complete quality report with before/after evidence

- run_id=inject-bad vs sprint3-clean comparison
- q_refund_window: hits_forbidden yes→no
- Metric impact table filled with 5 rules/expectations"
git push origin feature/cleaning
```

---

## Sprint 4 (60 phút) — Individual Report

### Nhiệm vụ cụ thể

1. **Viết individual report** `reports/individual/Dat.md` (40 phút):
   - **Mục 1** (80-120 từ): Cleaning & Quality Owner — `cleaning_rules.py`, `expectations.py`
   - **Mục 2** (100-150 từ): Quyết định KT — ví dụ: tại sao chọn halt cho chunk_id_unique (vì overwrite khi upsert) vs warn cho exported_at_not_future
   - **Mục 3** (100-150 từ): Anomaly — ví dụ: phát hiện row 3 chứa "14 ngày" là bản sync cũ policy-v3
   - **Mục 4** (80-120 từ): Dán 2 dòng eval CSV before/after + run_id
   - **Mục 5** (40-80 từ): Cải tiến — ví dụ: tích hợp pydantic model validation, Great Expectations

2. **Đóng góp vào group report mục metric_impact** (20 phút)

### Commit Sprint 4

```bash
git add reports/individual/Dat.md
git commit -m "[Sprint 4] individual: Dat cleaning/quality owner report (400-650 words)

- Phần phụ trách: cleaning_rules.py (3 rules), expectations.py (2 expectations)
- Quyết định KT: halt vs warn severity choice
- Anomaly: stale refund 14→7 ngày detection
- Before/after: q_refund_window + q_leave_version evidence"
git push origin feature/cleaning
```

---

## ✅ Checklist deliverables của Đạt

| Deliverable | Sprint | Trạng thái |
|---|---|---|
| Đọc hiểu baseline: 6 rules + 6 expectations | 1 | ☐ |
| Thêm 3 rules mới vào `cleaning_rules.py` (mỗi rule có comment + metric_impact) | 2 | ☐ |
| Thêm 2 expectations mới vào `expectations.py` (phân biệt halt/warn) | 2 | ☐ |
| Pipeline exit 0 sau khi thêm rules | 2 | ☐ |
| `docs/quality_report.md` hoàn chỉnh | 3 | ☐ |
| Bảng metric_impact cho group report | 3 | ☐ |
| `reports/individual/Dat.md` (400-650 từ) | 4 | ☐ |
