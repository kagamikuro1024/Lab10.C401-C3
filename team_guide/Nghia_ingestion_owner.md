# 📋 Kế Hoạch Cá Nhân — Nghĩa (Ingestion Owner)

## Lab Day 10 — Data Pipeline & Data Observability

**Vai trò:** Ingestion Owner  
**Nhánh Git:** `feature/ingestion`  
**File chịu trách nhiệm chính:**
- `etl_pipeline.py` (phần logging, manifest)
- `contracts/data_contract.yaml` (owner, SLA)
- `docs/data_contract.md` (source map)

---

## 🌿 Tạo Nhánh

```bash
git checkout main
git pull origin main
git checkout -b feature/ingestion
```

---

## Sprint 1 (60 phút) — ⭐ Sprint CHÍNH của Nghĩa

### Nhiệm vụ cụ thể

#### 1. Setup + chạy pipeline lần đầu (15 phút)
```bash
cd lab
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
python etl_pipeline.py run --run-id nghia-test
```

#### 2. Đọc hiểu flow ingestion (10 phút)
- Đọc kỹ `etl_pipeline.py` → hiểu flow: `load_raw_csv()` → `clean_rows()` → `run_expectations()` → `embed`
- Đọc `data/raw/policy_export_dirty.csv` → nhận diện 10 record, các lỗi có sẵn
- Đọc `transform/cleaning_rules.py` → hiểu baseline rules

#### 3. Điền contracts/data_contract.yaml (15 phút)
Thay các `__TODO__` bằng thông tin thật:
```yaml
owner_team: "Nhóm [tên nhóm]"
freshness:
  alert_channel: "#data-quality-alert"
```

#### 4. Điền docs/data_contract.md — Source Map (15 phút)
Điền bảng source map ít nhất 2 nguồn:

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_export_dirty.csv` (batch export) | CSV load qua `load_raw_csv()` | encoding error, empty rows, duplicate | `raw_records` count, quarantine rate |
| `data/docs/*.txt` (5 policy docs) | File read + embed | file missing, version conflict | freshness SLA, version metadata |

#### 5. Verify logs đúng format (5 phút)
Chạy lại pipeline, kiểm tra log có đủ:
```
run_id=sprint1
raw_records=10
cleaned_records=6
quarantine_records=4
```

### Commit Sprint 1

```bash
git add contracts/data_contract.yaml docs/data_contract.md
git add artifacts/logs/ artifacts/manifests/
git commit -m "[Sprint 1] ingestion: fill data_contract + source map + verify raw logging

- data_contract.yaml: owner_team, alert_channel filled
- data_contract.md: source map 2 sources (CSV export + docs files)
- Verified: raw_records=10, cleaned_records=6, quarantine_records=4"
git push origin feature/ingestion
```

> **Sau đó:** Báo Trung để merge vào main.

---

## Sprint 2 (60 phút) — Hỗ trợ Đạt

### Nhiệm vụ cụ thể

1. **Hỗ trợ Đạt test cleaning rules** (30 phút):
   - Tạo thêm test case bẩn cho dirty CSV (nếu cần)
   - Verify quarantine output khi thêm rule mới
   - Chạy pipeline sau khi Đạt modify code, kiểm tra số liệu

2. **Đồng bộ data_contract.yaml** (15 phút):
   - Nếu Đạt thêm rule mới → cập nhật `quality_rules` trong YAML
   - Nếu thêm doc_id mới → cập nhật `allowed_doc_ids`

3. **Review pipeline output** (15 phút):
   - So sánh log Sprint 1 vs Sprint 2
   - Verify quarantine reasons khớp với rule descriptions

### Commit Sprint 2

```bash
git add contracts/data_contract.yaml
git commit -m "[Sprint 2] ingestion: sync data_contract with new rules from Đạt

- quality_rules updated to reflect 3 new cleaning rules  
- Verified quarantine output after rule changes"
git push origin feature/ingestion
```

---

## Sprint 3 (60 phút) — Hỗ trợ Vinh inject

### Nhiệm vụ cụ thể

1. **Hỗ trợ Vinh chạy inject corruption** (30 phút):
   ```bash
   # Chạy pipeline XẤU
   python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
   python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
   
   # Chạy pipeline TỐT
   python etl_pipeline.py run --run-id sprint3-clean
   python eval_retrieval.py --out artifacts/eval/after_clean_eval.csv
   ```

2. **So sánh 2 file eval** (20 phút):
   - Mở `after_inject_bad.csv` — check `q_refund_window` có `hits_forbidden=yes`
   - Mở `after_clean_eval.csv` — check `q_refund_window` có `hits_forbidden=no`
   - Ghi lại evidence cho quality report

3. **Chuẩn bị chứng cứ cho group report** (10 phút):
   - Screenshot hoặc copy dòng CSV
   - Ghi run_id của cả 2 run

### Commit Sprint 3

```bash
git commit -m "[Sprint 3] ingestion: support inject evidence collection

- Verified inject-bad vs sprint3-clean eval comparison
- Evidence: q_refund_window before/after documented"
git push origin feature/ingestion
```

---

## Sprint 4 (60 phút) — Individual report

### Nhiệm vụ cụ thể

1. **Viết individual report** `reports/individual/Nghia.md` (40 phút):
   - **Mục 1** (80-120 từ): Mô tả vai trò Ingestion Owner — file nào, function nào
   - **Mục 2** (100-150 từ): Quyết định kỹ thuật — ví dụ: tại sao dùng `run_id` format UTC timestamp, cách đặt SLA freshness
   - **Mục 3** (100-150 từ): Anomaly phát hiện — ví dụ: `legacy_catalog_xyz_zzz` bị quarantine vì không thuộc allowlist
   - **Mục 4** (80-120 từ): Dán 2 dòng before/after CSV + ghi run_id
   - **Mục 5** (40-80 từ): Cải tiến — ví dụ: thêm checkpoint cho API ingestion, CDC monitoring

2. **Hỗ trợ Minh hoàn thiện docs** (20 phút)

### Commit Sprint 4

```bash
git add reports/individual/Nghia.md
git commit -m "[Sprint 4] individual: Nghia ingestion owner report (400-650 words)

- Phần phụ trách: data_contract, source map, logging
- Quyết định KT: SLA freshness + run_id format
- Anomaly: legacy_catalog quarantine
- Before/after: q_refund_window evidence"
git push origin feature/ingestion
```

> **Sau đó:** Báo Trung để merge lần cuối vào main.

---

## ✅ Checklist deliverables của Nghĩa

| Deliverable | Sprint | Trạng thái |
|---|---|---|
| `contracts/data_contract.yaml` — fill owner, SLA | 1 | ☐ |
| `docs/data_contract.md` — source map ≥2 nguồn | 1 | ☐ |
| Verify pipeline log format (run_id, raw, clean, quar) | 1 | ☐ |
| Sync data_contract.yaml với rules mới | 2 | ☐ |
| Evidence inject before/after | 3 | ☐ |
| `reports/individual/Nghia.md` (400-650 từ) | 4 | ☐ |
