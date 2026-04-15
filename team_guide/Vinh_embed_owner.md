# 📋 Kế Hoạch Cá Nhân — Vinh (Embed Owner)

## Lab Day 10 — Data Pipeline & Data Observability

**Vai trò:** Embed & Idempotency Owner  
**Nhánh Git:** `feature/embed`  
**File chịu trách nhiệm chính:**
- `etl_pipeline.py` (hàm `cmd_embed_internal()` — upsert + prune)
- `eval_retrieval.py` (chạy + phân tích before/after)
- `artifacts/eval/` (output eval CSV)

---

## 🌿 Tạo Nhánh

```bash
git checkout main
git pull origin main
git checkout -b feature/embed
```

---

## Sprint 1 (60 phút) — Setup + đọc hiểu embed flow

### Nhiệm vụ cụ thể

#### 1. Setup project (10 phút)
```bash
cd lab
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

> **Lưu ý:** Lần đầu chạy sẽ tải model `all-MiniLM-L6-v2` (~90MB) — cần mạng.

#### 2. Đọc kỹ embed logic (25 phút)

**File `etl_pipeline.py`, hàm `cmd_embed_internal()` (line 131-177):**
- Dùng `chromadb.PersistentClient` → lưu tại `./chroma_db`
- Collection: `day10_kb` (configurable qua `.env`)
- **Idempotent**: `col.upsert(ids=ids, documents=documents, metadatas=metadatas)`
- **Prune**: xóa id cũ không còn trong cleaned (`col.delete(ids=drop)`)
- Log: `embed_upsert count=...` và `embed_prune_removed=...`

**Hiểu metadata embed:**
```python
metadatas = [
    {
        "doc_id": r.get("doc_id", ""),
        "effective_date": r.get("effective_date", ""),
        "run_id": run_id,
    }
    for r in rows
]
```

#### 3. Chạy pipeline lần đầu + verify embed (15 phút)
```bash
python etl_pipeline.py run --run-id vinh-test

# Verify collection có data
python -c "
import chromadb
c = chromadb.PersistentClient(path='./chroma_db')
col = c.get_collection('day10_kb')
print(f'Collection count: {col.count()}')
print(f'Sample IDs: {col.get(limit=3)[\"ids\"][:3]}')
"
```

#### 4. Test idempotency (10 phút)
```bash
# Chạy lần 2 — count PHẢI KHÔNG ĐỔI
python etl_pipeline.py run --run-id vinh-test-2

python -c "
import chromadb
c = chromadb.PersistentClient(path='./chroma_db')
col = c.get_collection('day10_kb')
print(f'Count after rerun: {col.count()}')
"
```

> ✅ Nếu count giống lần 1 → idempotent hoạt động đúng.

### Commit Sprint 1

```bash
git commit -m "[Sprint 1] embed: verify idempotent upsert + prune mechanism

- First run: collection count = 6 chunks
- Rerun: collection count unchanged (upsert working)
- embed_prune_removed logged correctly"
git push origin feature/embed
```

---

## Sprint 2 (60 phút) — ⭐ Sprint CHÍNH của Vinh (phần 1)

### Nhiệm vụ cụ thể

#### 1. Đọc hiểu eval_retrieval.py (15 phút)

**File `eval_retrieval.py`:**
- Load câu hỏi từ `data/test_questions.json` (4 câu golden)
- Query Chroma top-k (mặc định k=3)
- So sánh keyword: `must_contain_any` vs `must_not_contain`
- Output CSV: `question_id, contains_expected, hits_forbidden, top1_doc_expected`

**4 câu test:**
1. `q_refund_window` — phải có "7 ngày", KHÔNG có "14 ngày"
2. `q_p1_sla` — phải có "15 phút"
3. `q_lockout` — phải có "5 lần"
4. `q_leave_version` — phải có "12 ngày", KHÔNG có "10 ngày phép năm", top1 = `hr_leave_policy`

#### 2. Chạy eval lần đầu (pipeline chuẩn) (10 phút)
```bash
# Đảm bảo pipeline chuẩn đã chạy
python etl_pipeline.py run --run-id sprint2-embed
python eval_retrieval.py --out artifacts/eval/after_clean_eval.csv
```

#### 3. Kiểm tra kết quả eval (15 phút)
Mở `artifacts/eval/after_clean_eval.csv`:
- `q_refund_window`: `contains_expected=yes`, `hits_forbidden=no` ✅
- `q_p1_sla`: `contains_expected=yes` ✅
- `q_lockout`: `contains_expected=yes` ✅
- `q_leave_version`: `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` ✅

#### 4. Đảm bảo prune hoạt động đúng (20 phút)

Kiểm tra: sau pipeline, không còn vector "rác" (từ run cũ):
```bash
python -c "
import chromadb
c = chromadb.PersistentClient(path='./chroma_db')
col = c.get_collection('day10_kb')
data = col.get(include=['metadatas'])
for i, (id, meta) in enumerate(zip(data['ids'], data['metadatas'])):
    print(f'{id}: doc_id={meta.get(\"doc_id\")}, run_id={meta.get(\"run_id\")}')
"
```

> Tất cả vector phải có `run_id=sprint2-embed` (cùng run).

### Commit Sprint 2

```bash
git add artifacts/eval/
git commit -m "[Sprint 2] embed: eval retrieval baseline - all 4 questions pass

- after_clean_eval.csv: 4/4 questions pass
- q_refund_window: contains_expected=yes, hits_forbidden=no
- q_leave_version: top1_doc_expected=yes (Merit)
- Collection count=6, no stale vectors"
git push origin feature/embed
```

---

## Sprint 3 (60 phút) — ⭐ Sprint CHÍNH của Vinh (phần 2: Inject)

### Nhiệm vụ cụ thể

#### 1. Chạy pipeline INJECT (corruption có chủ đích) (15 phút)
```bash
# Bước 1: Embed dữ liệu XẤU (bỏ fix refund + skip validate)
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```

**Giải thích flags:**
- `--no-refund-fix`: KHÔNG sửa "14 ngày" → "7 ngày" → chunk stale ở lại
- `--skip-validate`: bỏ qua expectation halt → vẫn embed (dùng cho demo)

#### 2. Eval trên dữ liệu xấu (10 phút)
```bash
python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
```

**Kết quả kỳ vọng:**
- `q_refund_window`: `contains_expected=???`, `hits_forbidden=yes` ❌ (còn "14 ngày")
- `q_leave_version`: có thể vẫn pass hoặc fail tuỳ quá trình

#### 3. Chạy lại pipeline CHUẨN (15 phút)
```bash
# Chạy lại pipeline tốt
python etl_pipeline.py run --run-id sprint3-clean

# Eval lại
python eval_retrieval.py --out artifacts/eval/after_clean_eval.csv
```

**Kết quả kỳ vọng:**
- `q_refund_window`: `contains_expected=yes`, `hits_forbidden=no` ✅

#### 4. So sánh + ghi chứng cứ (20 phút)

Tạo bảng so sánh:

| Question | Metric | inject-bad | sprint3-clean | Kết luận |
|---|---|---|---|---|
| q_refund_window | contains_expected | ? | yes | Pipeline fix "14→7" critical |
| q_refund_window | hits_forbidden | yes | no | Prune xóa chunk stale |
| q_leave_version | contains_expected | ? | yes | HR cũ quarantined |
| q_leave_version | hits_forbidden | ? | no | No stale 10d |
| q_leave_version | top1_doc_expected | ? | yes | Top-1 đúng doc |

> **Lưu file evidence** → cung cấp cho Đạt (quality report) và Trung (group report).

### Commit Sprint 3

```bash
git add artifacts/eval/
git add artifacts/logs/ artifacts/manifests/
git commit -m "[Sprint 3] embed: inject corruption + before/after evidence

- inject-bad: --no-refund-fix --skip-validate
- q_refund_window: hits_forbidden=yes (inject) → no (clean)
- sprint3-clean: all eval pass
- Evidence files: after_inject_bad.csv, after_clean_eval.csv"
git push origin feature/embed
```

> **Sau đó:** Báo Trung để merge vào main.

---

## Sprint 4 (60 phút) — Individual Report + Grading

### Nhiệm vụ cụ thể

#### 1. Chạy grading (nếu public) (10 phút)
```bash
# Chỉ chạy khi grading_questions.json được publish
python grading_run.py --out artifacts/eval/grading_run.jsonl
```

#### 2. Viết individual report `reports/individual/Vinh.md` (40 phút)
- **Mục 1** (80-120 từ): Embed Owner — `cmd_embed_internal()`, upsert/prune logic, `eval_retrieval.py`
- **Mục 2** (100-150 từ): Quyết định KT — idempotency: tại sao upsert theo `chunk_id` + prune stale vectors; trade-off: mất data nếu prune sai vs giữ rác
- **Mục 3** (100-150 từ): Anomaly — inject-bad: "14 ngày" xuất hiện trong top-k → `hits_forbidden=yes` → fix bằng chạy lại pipeline chuẩn + prune
- **Mục 4** (80-120 từ): 2 dòng eval: `q_refund_window` before (`hits_forbidden=yes`) vs after (`hits_forbidden=no`); run_id
- **Mục 5** (40-80 từ): Cải tiến — incremental embed (chỉ embed chunk thay đổi thay vì full upsert), hoặc versioned collection

#### 3. Hỗ trợ Minh nếu cần (10 phút)

### Commit Sprint 4

```bash
git add reports/individual/Vinh.md
git add artifacts/eval/grading_run.jsonl  # nếu có
git commit -m "[Sprint 4] individual: Vinh embed owner report (400-650 words)

- Phần phụ trách: embed idempotent (upsert+prune), eval_retrieval.py
- Quyết định KT: upsert vs replace strategy
- Anomaly: inject '14 ngày' detected via hits_forbidden
- Before/after: q_refund_window inject-bad vs sprint3-clean"
git push origin feature/embed
```

---

## ✅ Checklist deliverables của Vinh

| Deliverable | Sprint | Trạng thái |
|---|---|---|
| Verify embed idempotent (rerun 2 lần, count không đổi) | 1-2 | ☐ |
| `after_clean_eval.csv` — pipeline chuẩn | 2 | ☐ |
| `after_inject_bad.csv` — inject corruption | 3 | ☐ |
| Bảng so sánh before/after (evidence cho quality report) | 3 | ☐ |
| `grading_run.jsonl` (nếu public) | 4 | ☐ |
| `reports/individual/Vinh.md` (400-650 từ) | 4 | ☐ |
