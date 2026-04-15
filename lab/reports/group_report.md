# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** C401-C3  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nghĩa | Ingestion / Raw Owner | — |
| Đạt | Cleaning & Quality Owner | — |
| Vinh | Embed & Idempotency Owner | — |
| Minh | Monitoring / Docs Owner | — |
| Trung | End-to-end Lead | — |

**Ngày nộp:** 2026-04-15  
**Repo:** Lab Day 10 — `day10/lab`  
**Độ dài khuyến nghị:** 600–1000 từ

---

## 1. Pipeline tổng quan (150–200 từ)

Pipeline Day 10 xử lý dữ liệu từ file CSV raw export (`data/raw/policy_export_dirty.csv`) với 10 record mô phỏng batch export từ DB nội bộ. Chuỗi xử lý end-to-end gồm 5 bước: (1) **Ingest** — `load_raw_csv()` đọc CSV UTF-8, (2) **Clean** — `clean_rows()` áp dụng 9 cleaning rules (6 baseline + 3 mới) để loại stale data, normalize encoding, fix policy version, và quarantine record lỗi, (3) **Validate** — `run_expectations()` chạy 8 expectations (6 baseline + 2 mới) với cơ chế halt/warn kiểm soát chất lượng, (4) **Embed** — upsert idempotent vào Chroma collection `day10_kb` với prune vector cũ, (5) **Serving** — `eval_retrieval.py` query top-k để verify retrieval quality.

Kết quả chuẩn: `raw_records=10`, `cleaned_records=6`, `quarantine_records=4`. Pipeline exit 0 khi tất cả halt expectations pass.

**Lệnh chạy một dòng:**
```bash
python etl_pipeline.py run --run-id sprint3-clean
```

`run_id` ghi trong: `artifacts/logs/run_sprint3-clean.log`, `artifacts/manifests/manifest_sprint3-clean.json`, và Chroma metadata.

---

## 2. Cleaning & expectation (150–200 từ)

Baseline đã có 6 cleaning rules: allowlist `doc_id`, chuẩn hoá `effective_date` ISO, quarantine HR cũ (<2026), quarantine chunk rỗng, dedupe normalize+lowercase, và fix stale refund 14→7 ngày. Nhóm thêm 3 rule mới: **Rule 7** (Replacement character quarantine — halt nếu chứa U+FFFD), **Rule 8** (BOM strip — loại BOM character đầu chunk), **Rule 9** (Unicode NFKC normalize — chuẩn hoá full-width/half-width).

Baseline có 6 expectations (E1-E6). Nhóm thêm: **E7** `chunk_id_unique` (halt — phát hiện hash collision khi upsert) và **E8** `exported_at_not_future` (warn — phát hiện clock drift hoặc timestamp sai).

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| Rule 7: replacement_char quarantine | quarantine=4 | quarantine=4 (CSV mẫu không có U+FFFD; inject sẽ +1) | `run_sprint2.log` |
| Rule 8: BOM strip | text unchanged | text stripped nếu có BOM (\ufeff) | `cleaned_sprint2.csv` |
| Rule 9: NFKC normalize | text unchanged | text normalized (full-width → half-width) | `cleaned_sprint2.csv` |
| E7: chunk_id_unique (halt) | N/A | OK (duplicate_chunk_ids=0) | `run_sprint2.log` dòng 13 |
| E8: exported_at_not_future (warn) | N/A | OK (future_exported_at=0) | `run_sprint2.log` dòng 14 |

**Ví dụ expectation fail (inject-bad):**
Khi chạy `--no-refund-fix --skip-validate`, E3 `refund_no_stale_14d_window` FAIL (halt) :: violations=1 — phát hiện chunk "14 ngày làm việc" chưa được fix. Pipeline tiếp tục embed do `--skip-validate`.

---

## 3. Before / after ảnh hưởng retrieval (200–250 từ)

**Kịch bản inject:** `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`
- `--no-refund-fix`: giữ nguyên chunk stale "14 ngày làm việc" (lỗi migration từ policy v3)
- `--skip-validate`: bỏ qua expectation halt → embed dữ liệu xấu vào Chroma

**Kết quả định lượng:**

| Question | Metric | inject-bad | sprint3-clean | Kết luận |
|----------|--------|-----------|---------------|----------|
| q_refund_window | contains_expected | yes | yes | Cả 2 đều chứa "7 ngày" |
| q_refund_window | **hits_forbidden** | **yes** ❌ | **no** ✅ | Prune xóa chunk stale |
| q_leave_version | contains_expected | yes | yes | "12 ngày phép năm" đúng |
| q_leave_version | hits_forbidden | no | no | HR cũ đã quarantine |
| q_leave_version | top1_doc_expected | yes | yes | Top-1 = hr_leave_policy |

Key evidence: `q_refund_window` — trong inject-bad, `hits_forbidden=yes` vì chunk "14 ngày làm việc" vẫn nằm trong top-k (không bị fix). Sau khi chạy sprint3-clean, prune xóa vector cũ (`embed_prune_removed=1`), chỉ còn chunk đúng "7 ngày làm việc".

Artifact: `artifacts/eval/after_inject_bad.csv` vs `artifacts/eval/after_sprint3_clean.csv`

---

## 4. Freshness & monitoring (100–150 từ)

SLA freshness: **24 giờ** — phù hợp cho batch CSV export hàng ngày từ DB nội bộ.

Kết quả: **FAIL** — `age_hours=119.7 > sla_hours=24.0`
- `latest_exported_at=2026-04-10T08:00:00` (CSV mẫu exported 5 ngày trước)
- FAIL là hành vi đúng và mong đợi; trong production, CSV sẽ được export daily

Lệnh kiểm tra: `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint3-clean.json`

Freshness đo tại boundary **publish** (dùng `exported_at` trong manifest). Chưa đo boundary ingest riêng (improvement note trong runbook).

---

## 5. Liên hệ Day 09 (50–100 từ)

Pipeline Day 10 feed collection `day10_kb` riêng, tách khỏi collection Day 09. Cả hai dùng chung 5 docs trong `data/docs/` (policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy, access_control_sop). Tích hợp có thể thực hiện bằng cách đổi `CHROMA_COLLECTION` trong `.env` hoặc merge collection, nhưng tách riêng để tránh side-effect khi pipeline Day 10 prune vector.

---

## 6. Rủi ro còn lại & việc chưa làm

- Freshness FAIL trên CSV mẫu (exported_at cũ 5 ngày) — expected behavior cho lab
- Chưa có CDC real-time cho database changes — pipeline chỉ batch
- Embedding model `all-MiniLM-L6-v2` chưa tối ưu cho tiếng Việt dài
- Rule versioning HR hard-code ngày `2026-01-01` — nên đọc từ `contracts/data_contract.yaml`
- Chưa xử lý OCR confidence cho PDF scan (chỉ đọc plaintext `.txt`)
