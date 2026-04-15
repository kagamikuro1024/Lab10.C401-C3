# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Hoàng Đức Nghĩa
**Vai trò:** Ingestion
**Ngày nộp:** 15/04/2026
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `lab/etl_pipeline.py` (kiểm tra end-to-end run, logs, manifest)
- `contracts/data_contract.yaml` (owner team, freshness SLA, alert channel)
- `lab/docs/data_contract.md` (source map, canonical source mô tả ingest)

**Kết nối với thành viên khác:**

Tôi phối hợp với nhóm để đồng bộ `allowed_doc_ids` và `quality_rules` giữa hợp đồng dữ liệu và rule code, đảm bảo team hiểu rõ nguồn `policy_export_dirty.csv` và 5 file docs canonical.

**Bằng chứng (commit / comment trong code):**

Commit "[Sprint 1] ingestion: fill data_contract + source map + verify raw logging- data_contract.yaml: owner_team, alert_channel filled- data_contract.md: source map 2 sources (CSV export + docs files)- Verified: raw_records=10, cleaned_records=6, quarantine_records=4" trong nhánh `feature/ingestion`
Hoạt động của tôi thể hiện qua chỉnh sửa `contracts/data_contract.yaml` và `lab/docs/data_contract.md`; tôi cũng xác nhận log pipeline từ lệnh `python lab\etl_pipeline.py run --run-id sprint1-test --skip-validate`.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Tôi chọn nhấn mạnh freshness và traceability trong ingestion layer. Cụ thể, tôi hoàn thiện `contracts/data_contract.yaml` với `freshness.sla_hours: 24` và `alert_channel: "#data-quality-alert"`, đồng thời đưa các source canonical vào `lab/docs/data_contract.md`. Điều này giúp pipeline không chỉ kiểm tra số lượng record (`raw_records`, `cleaned_records`, `quarantine_records`) mà còn có thể truy nguyên nguồn dữ liệu khi gặp anomaly. Tôi cũng giữ định dạng log đơn giản theo key=value để dễ parse, như `run_id=sprint1-test` và `raw_records=10`, điều này phù hợp với yêu cầu monitor/alert của nhóm.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Trong Sprint 1, tôi tập trung xử lý anomaly từ raw export và định nghĩa rõ failure mode. Tôi xác nhận rằng `policy_export_dirty.csv` chứa 10 record và 4 record bị quarantine, gồm các trường hợp `missing_effective_date`, `invalid_effective_date_format`, `duplicate_chunk_text`, và `stale_refund_window`. Metric `quarantine_records=4` đã được dùng để kiểm tra trực tiếp kết quả clean. Việc này đảm bảo không có record bị giữ lại trong cleaned export nếu dữ liệu nguồn không hợp lệ hoặc `doc_id` không khớp hợp đồng.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Đây là bằng chứng từ Sprint 3 về việc inject corruption và clean của nhóm tôi:

**Trước (inject bad, file `after_inject_bad.csv`, run_id=inject-bad):**
```
q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,yes,,3
```

**Sau (sprint3 clean, file `after_sprint3_clean.csv`,run_id=sprint3-clean):**
```
q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,no,,3
```

Theo như tôi thấy, sự khác biệt rõ ràng ở cột `hits_forbidden`: từ `yes` (trước) xuống `no` (sau), chứng minh rằng việc fix stale refund window đã loại bỏ forbidden content khỏi retrieval.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ bổ sung một kiểm tra tự động cho `contracts/data_contract.yaml` và `transform/cleaning_rules.py` để so sánh `allowed_doc_ids` và `quality_rules` giữa code và contract, tránh mismatch khi nhóm thêm doc mới hoặc rule mới trong tương lai.
