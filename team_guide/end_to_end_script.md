# 🎯 End-to-End Script — Tech Lead (Trung)

## Lab Day 10 — Data Pipeline & Data Observability

**Vai trò:** Tech Lead + cầm repo main  
**Nhóm:** Trung (Lead), Nghĩa (Ingestion), Đạt (Cleaning/Quality), Vinh (Embed), Minh (Monitoring/Docs)  
**Tổng thời gian:** 4 sprints × ~60 phút  
**Tổng điểm:** 100 (Nhóm 60 + Cá nhân 40)

---

## 📋 Tổng Quan Lab

### Cấu trúc bài lab

Lab mô phỏng pipeline: `raw CSV (dirty)` → `clean` → `validate (expectations)` → `embed (Chroma)` → `eval retrieval`.

**Dữ liệu đầu vào:** `data/raw/policy_export_dirty.csv` — 11 dòng (10 records), chứa:
- Duplicate chunk (row 2 = row 1)
- Chunk stale refund "14 ngày" (row 3) — cần fix sang "7 ngày"
- Chunk rỗng, thiếu effective_date (row 5)
- HR bản cũ 2025 "10 ngày phép" (row 7) — cần quarantine
- Doc_id lạ `legacy_catalog_xyz_zzz` (row 9) — cần quarantine
- Ngày format DD/MM/YYYY thay vì ISO (row 10)

**5 tài liệu docs:** `policy_refund_v4.txt`, `sla_p1_2026.txt`, `it_helpdesk_faq.txt`, `hr_leave_policy.txt`, `access_control_sop.txt`

### Scoring tóm tắt

| Phần | Điểm | Trọng tâm |
|------|------|-----------|
| ETL & pipeline | 27 | Pipeline chạy, log đúng, ≥3 rule mới, embed idempotent |
| Documentation | 15 | 3 file md (architecture, contract, runbook) |
| Quality evidence | 18 | ≥2 expectation mới, before/after eval, quality report |
| Grading JSONL | 0-12 | 3 câu grading (public muộn) |
| Individual report | 30 | Mỗi người 400-650 từ |
| Code contribution | 10 | Commit khớp vai trò |

---

## 🌿 Git Workflow

### Cấu trúc nhánh

```
main                    ← Trung cầm, merge từ các nhánh feature
├── feature/ingestion   ← Nghĩa
├── feature/cleaning    ← Đạt
├── feature/embed       ← Vinh
└── feature/monitoring  ← Minh
```

### Quy tắc commit

```
[Sprint X] <vai_trò>: <mô tả ngắn>

Ví dụ:
[Sprint 1] ingestion: add raw logging and manifest generation
[Sprint 2] cleaning: add 3 new rules (BOM, phone_mask, content_hash)
[Sprint 2] quality: add 2 new expectations (chunk_uniqueness, version_conflict)
[Sprint 3] embed: inject corruption and capture before/after eval
[Sprint 4] monitoring: complete freshness check and runbook
```

---

## 🏃 Sprint 1 (60 phút) — Ingest & Schema

### Mục tiêu
Pipeline chạy được, log ra `run_id`, `raw_records`, `cleaned_records`, `quarantine_records`.

### Lead Action (Trung)

**Trước khi bắt đầu (15 phút):**
1. Đảm bảo repo sạch trên main, mọi người đã clone
2. Hướng dẫn team setup:
   ```bash
   cd lab
   python -m venv .venv
   .venv\Scripts\activate
   pip install -r requirements.txt
   cp .env.example .env
   ```
3. Phân công rõ: ai tạo nhánh gì, code file nào
4. Chạy thử pipeline lần đầu trên main:
   ```bash
   python etl_pipeline.py run --run-id sprint1-test
   ```

**Trong sprint (30 phút):**
5. Review code của Nghĩa: kiểm tra log format, path handling
6. Hỗ trợ Đạt đọc hiểu baseline `cleaning_rules.py` để chuẩn bị Sprint 2
7. Nhắc Minh bắt đầu đọc templates trong `docs/`

**Cuối sprint (15 phút):**
8. Nhận PR/code từ Nghĩa, merge vào main
9. Chạy pipeline trên main, verify log:
   ```bash
   python etl_pipeline.py run --run-id sprint1
   ```
10. **Kiểm tra DoD:**
    - [ ] Log có `run_id=sprint1`
    - [ ] Log có `raw_records=10`
    - [ ] Log có `cleaned_records` (expected: 6)
    - [ ] Log có `quarantine_records` (expected: 4)
    - [ ] File `artifacts/manifests/manifest_sprint1.json` tồn tại
    - [ ] File `artifacts/quarantine/quarantine_sprint1.csv` tồn tại

### Commit trên main sau Sprint 1

```bash
git add artifacts/logs/ artifacts/manifests/ artifacts/quarantine/ artifacts/cleaned/
git add contracts/data_contract.yaml docs/data_contract.md
git commit -m "[Sprint 1] tech-lead: initial pipeline run, log + manifest + quarantine verified

run_id=sprint1
raw_records=10, cleaned_records=6, quarantine_records=4
Nghĩa: ingestion logging, manifest generation
data_contract.md: source map filled (≥2 sources)"
```

> ⚠️ **DỪNG LẠI** — Check kỹ output trước khi qua Sprint 2.

---

## 🏃 Sprint 2 (60 phút) — Clean + Validate + Embed

### Mục tiêu
- Thêm ≥3 cleaning rules mới (có metric_impact)
- Thêm ≥2 expectations mới (phân biệt warn/halt)
- Embed idempotent (upsert + prune)
- Pipeline `exit 0` với expectations không halt

### Lead Action (Trung)

**Đầu sprint (10 phút):**
1. Briefing: nhắc team yêu cầu **chống trivial** — mỗi rule/expectation phải có tác động đo được
2. Phân công rõ:
   - **Đạt**: thêm 3 rule mới vào `transform/cleaning_rules.py` + 2 expectation mới vào `quality/expectations.py`
   - **Vinh**: đảm bảo embed idempotent, test rerun 2 lần xem collection count có phình không
   - **Nghĩa**: hỗ trợ Đạt test rule trên dữ liệu
   - **Minh**: bắt đầu điền `docs/data_contract.md`

**Trong sprint (35 phút):**
3. Review PR của Đạt: kiểm tra mỗi rule/expectation có docstring + comment + tên rõ
4. Review PR của Vinh: test embed count sau 2 lần rerun
5. **Gợi ý 3 rule mới cho Đạt** (không trivial):
   - **Rule: BOM/encoding strip** — loại BOM character `\ufeff` khỏi chunk_text → metric: quarantine_records tăng nếu inject BOM
   - **Rule: phone/email PII masking** — mask số điện thoại/email nội bộ trong chunk_text trước embed → metric: số chunk bị modify
   - **Rule: content hash dedup** — sha256 hash nội dung chunk sau normalize, loại duplicate tinh vi (khác whitespace) → metric: quarantine_records tăng
6. **Gợi ý 2 expectation mới cho Đạt**:
   - **E_new_1: chunk_id uniqueness** (halt) — kiểm tra không có chunk_id trùng sau clean
   - **E_new_2: exported_at not empty** (warn) — kiểm tra tất cả row đều có exported_at

**Cuối sprint (15 phút):**
7. Merge Đạt → main, merge Vinh → main
8. Chạy full pipeline:
   ```bash
   python etl_pipeline.py run --run-id sprint2
   python eval_retrieval.py --out artifacts/eval/after_clean_eval.csv
   ```
9. **Kiểm tra DoD:**
    - [ ] Pipeline exit 0 (không halt)
    - [ ] `cleaning_rules.py` có ≥3 rule mới (comment + docstring)
    - [ ] `expectations.py` có ≥2 expectation mới
    - [ ] Log có tất cả expectations PASS (hoặc warn OK)
    - [ ] Rerun 2 lần: Chroma collection count không đổi
    - [ ] `embed_prune_removed` (nếu có) ghi trong log

### Commit trên main sau Sprint 2

```bash
git add transform/cleaning_rules.py quality/expectations.py
git add artifacts/logs/ artifacts/manifests/ artifacts/cleaned/ artifacts/eval/
git commit -m "[Sprint 2] tech-lead: merge cleaning+quality+embed

Đạt: 3 new rules (bom_strip, pii_mask, content_hash_dedup)
Đạt: 2 new expectations (chunk_id_unique:halt, exported_at_present:warn)
Vinh: embed idempotent verified, upsert+prune working
run_id=sprint2, pipeline exit 0, all expectations OK"
```

> ⚠️ **DỪNG LẠI** — Check kỹ trước khi qua Sprint 3.

---

## 🏃 Sprint 3 (60 phút) — Inject Corruption & Before/After

### Mục tiêu
- Inject corruption có chủ đích (embed dữ liệu xấu)
- So sánh before/after retrieval
- Hoàn thành quality report

### Lead Action (Trung)

**Đầu sprint (10 phút):**
1. Briefing: Sprint này cần **2 lần chạy pipeline** — 1 xấu + 1 tốt — để so sánh
2. Phân công:
   - **Vinh**: chạy inject xấu → eval → lưu file
   - **Đạt**: viết quality report từ template
   - **Nghĩa**: hỗ trợ Vinh, chuẩn bị evidence
   - **Minh**: bắt đầu `docs/runbook.md`

**Trong sprint (35 phút):**
3. **Kịch bản inject corruption:**
   ```bash
   # Bước 1: Chạy pipeline XẤU (bỏ fix refund, skip validate)
   python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
   
   # Bước 2: Eval retrieval trên dữ liệu xấu
   python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
   
   # Bước 3: Chạy lại pipeline TỐT (pipeline chuẩn)
   python etl_pipeline.py run --run-id sprint3-clean
   
   # Bước 4: Eval retrieval trên dữ liệu tốt
   python eval_retrieval.py --out artifacts/eval/after_clean_eval.csv
   ```
4. So sánh 2 file eval: `after_inject_bad.csv` vs `after_clean_eval.csv`
   - `q_refund_window`: trước: `contains_expected=no, hits_forbidden=yes` → sau: `contains_expected=yes, hits_forbidden=no`
   - `q_leave_version` (Merit): trước: `hits_forbidden=yes` (vì HR cũ 10 ngày) → sau: pass
5. Hoàn thành `docs/quality_report_template.md` → save as `docs/quality_report.md`

**Cuối sprint (15 phút):**
6. Merge code + artifacts từ Vinh, Đạt
7. **Kiểm tra DoD:**
    - [ ] Có ≥2 file eval (inject vs clean) hoặc 1 file có cột scenario
    - [ ] `q_refund_window`: before `hits_forbidden=yes`, after `hits_forbidden=no`
    - [ ] Quality report có run_id + số liệu + interpret
    - [ ] (Merit) Evidence cho `q_leave_version`

### Commit trên main sau Sprint 3

```bash
git add artifacts/eval/ docs/quality_report.md
git add artifacts/logs/ artifacts/manifests/
git commit -m "[Sprint 3] tech-lead: inject corruption + before/after evidence

Vinh: inject-bad run (--no-refund-fix --skip-validate)
q_refund_window: before hits_forbidden=yes → after hits_forbidden=no
q_leave_version: before hits_forbidden=yes → after pass (Merit evidence)
Đạt: quality report completed with run_id and metrics"
```

> ⚠️ **DỪNG LẠI** — Check kỹ trước khi qua Sprint 4.

---

## 🏃 Sprint 4 (60 phút) — Monitoring + Docs + Báo Cáo

### Mục tiêu
- Freshness check + runbook
- Hoàn thiện 3 docs (`pipeline_architecture.md`, `data_contract.md`, `runbook.md`)
- Group report + 4 individual reports
- Grading JSONL (nếu được public)

### Lead Action (Trung)

**Đầu sprint (10 phút):**
1. Briefing: Sprint cuối — tập trung documentation + báo cáo
2. Phân công:
   - **Minh**: hoàn thiện 3 docs + freshness check
   - **Mỗi người**: viết individual report 400-650 từ
   - **Trung**: tổng hợp group report

**Trong sprint (35 phút):**
3. Chạy freshness check:
   ```bash
   python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint3-clean.json
   ```
4. Giải thích PASS/WARN/FAIL trong runbook
5. Nhắc team: **freshness FAIL là hợp lý** vì CSV mẫu có `exported_at` cũ — ghi giải thích trong runbook
6. Nếu `grading_questions.json` được public:
   ```bash
   python grading_run.py --out artifacts/eval/grading_run.jsonl
   ```
7. Hoàn thành `reports/group_report.md`:
   - Mục 1: Pipeline tổng quan (150-200 từ)
   - Mục 2: Cleaning & expectation + bảng metric_impact (bắt buộc!)
   - Mục 3: Before/after evidence (200-250 từ)
   - Mục 4: Freshness & monitoring (100-150 từ)
   - Mục 5: Liên hệ Day 09 (50-100 từ)
   - Mục 6: Rủi ro còn lại

**Cuối sprint (15 phút):**
8. Thu individual reports từ 4 người
9. Merge tất cả vào main
10. **Kiểm tra DoD cuối cùng:**
    - [ ] `docs/pipeline_architecture.md` có sơ đồ + bảng ranh giới
    - [ ] `docs/data_contract.md` có source map (≥2 nguồn)
    - [ ] `docs/runbook.md` đủ 5 mục (Symptom→Prevention)
    - [ ] `reports/group_report.md` hoàn chỉnh, có bảng metric_impact
    - [ ] 4 file `reports/individual/[ten].md`
    - [ ] `artifacts/eval/grading_run.jsonl` (nếu có)
    - [ ] README nhóm có "một lệnh chạy cả pipeline"

### Commit cuối cùng trên main

```bash
# Commit docs
git add docs/ contracts/
git commit -m "[Sprint 4] monitoring: complete docs (architecture, contract, runbook)
Minh: pipeline_architecture.md, data_contract.md, runbook.md
Freshness check: FAIL (expected — CSV exported_at cũ, giải thích trong runbook)"

# Commit reports
git add reports/
git commit -m "[Sprint 4] reports: group report + 4 individual reports
Bảng metric_impact: 3 rules x impact verified
Individual: Trung, Nghĩa, Đạt, Vinh, Minh"

# Commit grading (nếu có)
git add artifacts/eval/grading_run.jsonl
git commit -m "[Sprint 4] grading: run grading_run.py, 3 câu gq_d10_01..03"
```

---

## ✅ Checklist Nộp Bài Cuối Cùng

| File/Thư mục | Trạng thái |
|---|---|
| `etl_pipeline.py` + `transform/` + `quality/` + `monitoring/` | ☐ |
| `contracts/data_contract.yaml` | ☐ |
| `artifacts/logs/`, `manifests/`, `quarantine/`, `eval/` | ☐ |
| `docs/pipeline_architecture.md` | ☐ |
| `docs/data_contract.md` | ☐ |
| `docs/runbook.md` | ☐ |
| `docs/quality_report.md` (hoặc template) | ☐ |
| `reports/group_report.md` | ☐ |
| `reports/individual/Trung.md` | ☐ |
| `reports/individual/Nghia.md` | ☐ |
| `reports/individual/Dat.md` | ☐ |
| `reports/individual/Vinh.md` | ☐ |
| `reports/individual/Minh.md` | ☐ |
| `artifacts/eval/grading_run.jsonl` | ☐ |

---

## 📌 Lưu ý quan trọng cho Tech Lead

1. **Không commit `chroma_db/`** — đã có trong `.gitignore`
2. **Bảng metric_impact trong group_report.md là BẮT BUỘC** — thiếu sẽ bị trừ điểm
3. **Mỗi rule/expectation mới phải có tác động đo được** — "strip space" mà không đổi metric = trivial = trừ điểm
4. **Commit phải khớp vai trò** — nếu ghi Đạt làm cleaning thì phải có commit từ feature/cleaning
5. **freshness FAIL là bình thường** — CSV mẫu `exported_at=2026-04-10T08:00:00` sẽ FAIL nếu SLA 24h. Giải thích trong runbook.
6. **After embed, rerun 2 lần** — verify idempotent: collection count không đổi
