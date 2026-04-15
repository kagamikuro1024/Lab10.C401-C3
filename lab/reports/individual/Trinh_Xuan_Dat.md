# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Trịnh Xuân Đạt 
**Vai trò:** Cleaning
**Ngày nộp:** 15/04/2026 
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `transform/cleaning_rules.py` — thêm 3 rule mới để chuẩn hoá nội dung và chống lỗi encoding.
- `quality/expectations.py` — bổ sung 2 expectation mới để kiểm soát `chunk_id` và `exported_at`.
- `lab/artifacts/logs/run_sprint2-dat.log` — dùng log pipeline để chứng minh số liệu `quarantine_records` và expectation.

**Kết nối với thành viên khác:**

Tôi phối hợp với team trên đầu vào/đầu ra: đồng bộ với Nghĩa để giữ `run_id` nhất quán, với Minh để đảm bảo metric obserability, và với Vinh để không làm hỏng tài liệu embed downstream.

_________________

**Bằng chứng (commit / comment trong code):**
# === Rule mới 7: BOM strip ===
# Loại BOM character (\ufeff) đầu chunk_text — metric_impact: quarantine nếu inject BOM
# Owner: Đạt

# === Rule mới 8: Unicode NFKC normalize ===
# Chuẩn hoá unicode NFKC để tránh cùng nội dung nhưng encoding khác → metric_impact: content_hash thay đổi
# Owner: Đạt

# === Rule mới 9: Flag replacement character (U+FFFD) ===
# Quarantine chunk chứa ký tự thay thế (lỗi encoding nghiêm trọng) — metric_impact: quarantine_records tăng
# Owner: Đạt

# E7: chunk_id phải unique — phát hiện hash collision hoặc logic sai
# Owner: Đạt
# Severity: halt — trùng chunk_id sẽ overwrite khi upsert

# E8: exported_at không được ở tương lai — phát hiện clock drift hoặc inject sai timestamp
# Owner: Đạt
# Severity: warn — có thể chấp nhận nhưng cần ghi nhận
_________________

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Tôi quyết định rõ ràng giữa `halt` và `warn` để tránh mất dữ liệu nghiêm trọng. Khi E7 phát hiện `chunk_id` duplicate, tôi đặt severity là `halt` vì trùng ID sẽ overwrite khi upsert vào KB và có thể làm mất nội dung đúng. Còn E8 được đặt là `warn` vì `exported_at` ở tương lai thường phản ánh clock drift hoặc ingest metadata sai, vẫn cần ghi nhận nhưng không nhất thiết dừng toàn bộ pipeline.

Đồng thời, tôi dùng idempotency bằng cách đưa `Unicode NFKC normalize` và `BOM strip` vào pipeline, để cùng một nội dung không bị phân tách thành nhiều chunk khác nhau chỉ vì encoding. Quyết định này giúp giảm rủi ro false duplicate, giữ `content_hash` ổn định và phù hợp với metric `top1_doc_expected` khi truy vấn sau này.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Triệu chứng: một số chunk text có thể chứa BOM hoặc ký tự thay thế U+FFFD, dẫn đến cleanup không nhất quán và có thể làm nội dung cùng nghĩa bị coi là khác biệt. Metric tôi theo dõi là `quarantine_records` và expectation `chunk_id_unique`.

Check pipeline `run_id=sprint2-dat` trong `lab/artifacts/logs/run_sprint2-dat.log` cho thấy `quarantine_records=4` và tất cả expectation E1-E6 pass. Tôi đóng góp thêm Rule 7/8/9 để strip BOM, normalize NFKC và quarantine chunk chứa U+FFFD. Khi E7 được thêm, nếu có duplicate `chunk_id` pipeline sẽ halt và báo rõ `duplicate_chunk_ids` để tránh overwrite dữ liệu.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Dưới đây là hai dòng thực tế từ artifact eval trước và sau, cho thấy lỗi forbidden content đã được thay thế bằng kết quả clean hơn. Tôi sử dụng `after_inject_bad.csv` làm trước và `after_sprint3_clean.csv` làm sau.

`inject-bad` / before:
q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,yes,,3

`sprint3-clean` / after:
q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,no,,3

Điều này thể hiện `hits_forbidden` chuyển từ `yes` sang `no`, phù hợp với việc tôi chịu trách nhiệm thêm rule clean và expectation để giảm lỗi truy vấn nội dung cấm.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ xây thêm test CI cho encoding fuzzing: tự động inject BOM + U+FFFD + các biến thể NFKC vào mẫu, kiểm tra `quarantine_records` và `chunk_id_unique`. Việc này giúp đảm bảo rule mới không bị regress và các `run_id` tiếp theo giữ tính nhất quán.

_________________
