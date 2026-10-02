# 📋 CHECKLIST HOÀN THÀNH BÀI LAB — AGENT ARENA (DAY 16)

> **Mục tiêu:** Bọc quanh tác tử ReAct yếu bằng **5 lớp middleware** trong thư mục `harness/layers/` để nâng điểm từ mức sàn **~24/100 lên ~81.7+/100**, tối ưu trên cả 3 tiêu chí: **Grounding (55đ)**, **Safety (30đ)**, và **Efficiency (15đ)**.

---

## ⏱️ PHÂN BỔ THỜI GIAN (120 PHÚT)

- [ ] **0 – 15m: Khởi động & Làm quen**
  - Chạy môi trường, kiểm tra `verify.py`, đo baseline (`~24.27/100`).
  - Đọc `README.md`, `GUIDE.md`, `RUBRIC.md` và `arena/scorer.py`.
- [ ] **15 – 95m: Triển khai 5 Layer (Trọng tâm bài lab)**
  - Viết code cho 5 file trong `harness/layers/` theo thứ tự ưu tiên điểm số.
  - Lặp lại quy trình: Viết layer → Chạy thử (`run_practice.py`) → Xem chẩn đoán (`selfeval.py`) → Tinh chỉnh.
- [ ] **95 – 105m: Kiểm chứng Leave-One-Out & Đóng băng nộp bài**
  - Chạy full stack và rút thử từng layer kiểm tra hiệu quả thực tế.
  - `git commit` và `git push` trước khi hết giờ.
- [ ] **105 – 120m: Vòng chấm điểm**
  - Giảng viên chấm trên bộ brief riêng + mô hình thật.

---

## 🛡️ CHI TIẾT 5 LỚP BẢO VỆ CẦN HOÀN THÀNH

Các file cần sửa nằm tại: [`harness/layers/`](file:///d:/Hoc/VinAI/K4-DAY16-Track3A-PhanHoangVu-2A202602450-AgentArena/harness/layers/)  
*(Mỗi file chỉ cần viết khoảng 10–25 dòng code theo đúng hướng dẫn TODO).*

### 1. `critic` — Phản biện & Tự đánh giá ([`harness/layers/critic.py`](file:///d:/Hoc/VinAI/K4-DAY16-Track3A-PhanHoangVu-2A202602450-AgentArena/harness/layers/critic.py))
*Trọng số điểm cao nhất: Cứu điểm Honesty (15đ) và Precision.*
- [x] Cài đặt hook `after_agent(self, ctx, report)`:
  - [x] Lấy danh sách `report["claims"]` (kiểm tra kiểu dữ liệu list).
  - [x] **Lọc claim:** Nếu `claim["text"]` có trong `ctx.observed_text` (hoặc `ctx.saw(...)`) → **Giữ nguyên**.
  - [x] **Xử lý mâu thuẫn (bẫy pub-04):** Nếu không có trong quan sát, thử tách câu ghép qua liên từ `" và "`. Nếu 2 nửa câu đều xuất hiện nguyên văn trong `ctx.observed_text` và thuộc về 2 tài liệu khác nhau:
    - Giữ cả hai nửa thành 2 claims riêng (gán đúng `doc_id` của tài liệu chứa nó).
    - Đặt `report["abstain"] = True`.
  - [x] **Loại bỏ claim ảo giác:** Nếu không tách được → Claim do mô hình bịa đặt → **Xóa bỏ claim**.
  - [x] **Xử lý thiếu dữ liệu (bẫy pub-05):** Nếu không còn claim nào hợp lệ:
    - Đặt `report["abstain"] = True`.
    - Gán `report["claims"] = []`, `report["citations"] = []`.
    - Viết lại `report["answer"]` thông báo rõ không đủ căn cứ.
  - [x] Cập nhật lại `report["citations"] = sorted(set(c["doc_id"] for c in report["claims"]))`.

---

### 2. `citation_checker` — Kiểm tra & Gắn lại trích dẫn ([`harness/layers/citation_checker.py`](file:///d:/Hoc/VinAI/K4-DAY16-Track3A-PhanHoangVu-2A202602450-AgentArena/harness/layers/citation_checker.py))
*Mục tiêu: Sửa lỗi trích nhầm tài liệu giao diện/giả mạo (giữ vững 55đ Grounding).*
- [x] Cài đặt hook `after_agent(self, ctx, report)`:
  - [x] Bỏ qua nếu `report["claims"]` rỗng hoặc `ctx.corpus is None`.
  - [x] Với mỗi claim, lấy `doc = ctx.corpus.get(claim["doc_id"])`.
  - [x] **Kiểm tra theo dòng:** Nếu `doc` tồn tại VÀ `claim["text"]` khớp **nguyên văn một dòng** (`claim["text"] in doc.body.splitlines()`) → Trích dẫn đúng, giữ nguyên.
  - [x] **Gắn lại nguồn thật (Misattribution):** Nếu không khớp dòng, tìm tài liệu `target_doc` trong `ctx.corpus.docs` thỏa mãn:
    1. Đã được đọc nguyên vẹn: `target_doc.body in ctx.observed_text` (tránh lỗi `UNRETRIEVED`).
    2. Khớp nguyên văn một dòng: `claim["text"] in target_doc.body.splitlines()`.
    - Đổi `claim["doc_id"] = target_doc.doc_id` (**TUYỆT ĐỐI GIỮ NGUYÊN `claim["text"]`**).
  - [x] Nếu không tìm thấy trong bất kỳ tài liệu đã đọc nào: Giữ nguyên để `critic` loại bỏ.
  - [x] Cập nhật `report["citations"] = sorted(set(c["doc_id"] for c in report["claims"]))`.

---

### 3. `injection_guard` — Phòng vệ chèn mã độc ([`harness/layers/injection_guard.py`](file:///d:/Hoc/VinAI/K4-DAY16-Track3A-PhanHoangVu-2A202602450-AgentArena/harness/layers/injection_guard.py))
*Mục tiêu: Bảo vệ 15đ Safety chống Prompt Injection, loại bỏ Canary string.*
- [x] Cài đặt hook `wrap_tool_call(self, ctx, call, name, args)`:
  - [x] Gọi `result = call(name, args)`.
  - [x] Kiểm tra nếu `BLOCK_START` (`"--- LƯU Ý HỆ THỐNG"`) có trong `result.content`:
    - Dùng vòng lặp cắt bỏ đoạn giữa `BLOCK_START` và `BLOCK_END` (`"--- HẾT LƯU Ý HỆ THỐNG ---"`), thay thế bằng `PLACEHOLDER`.
    - **Xử lý bẫy TRUNCATED:** Nếu gặp `BLOCK_START` nhưng không thấy `BLOCK_END` (do nội dung bị cắt cụt) → Cắt từ `BLOCK_START` đến hết chuỗi rồi thoát lặp.
  - [x] Trả về `ToolResult(ok=result.ok, content=clean_content, error=result.error)`.
- [x] Cài đặt hook `after_agent(self, ctx, report)`:
  - [x] Nếu `INJECTION_CANARY` còn tồn tại trong `report["answer"]` → Xóa bỏ canary khỏi `report["answer"]`.
  - [x] **LƯU Ý:** Không được xóa hay sửa chữ trong `claim["text"]`.

---

### 4. `budget_policy` — Kiểm soát ngân sách công cụ ([`harness/layers/budget_policy.py`](file:///d:/Hoc/VinAI/K4-DAY16-Track3A-PhanHoangVu-2A202602450-AgentArena/harness/layers/budget_policy.py))
*Mục tiêu: Đạt tối đa 15đ Efficiency (không gọi dư thừa 4 bước rác cuối).*
- [x] Cài đặt hàm helper `_spent(self, ctx) -> bool`:
  - [x] Kiểm tra: `ctx.max_tool_calls is not None` và `ctx.tools.calls >= ctx.max_tool_calls - self.reserve` (mặc định `reserve = 1` dành riêng cho hành động `submit`).
- [x] Cài đặt hook `before_model(self, ctx, messages)`:
  - [x] Nếu chưa hết ngân sách: Trả về `messages` nguyên vẹn.
  - [x] Nếu đã hết ngân sách: Trả về bản sao `messages + [{"role": "user", "content": NUDGE}]` (sử dụng phép cộng danh sách, **không dùng `.append()`**, câu `NUDGE` bắt buộc chứa `FINALIZE_SENTINEL`).
- [x] Cài đặt hook `wrap_tool_call(self, ctx, call, name, args)`:
  - [x] Nếu chưa hết ngân sách: `return call(name, args)`.
  - [x] Nếu đã chạm trần ngân sách: Chặn gọi tool, trả về `ToolResult(ok=False, content="", error="Hết ngân sách công cụ")`.

---

### 5. `retry` — Thử lại công cụ khi lỗi ([`harness/layers/retry.py`](file:///d:/Hoc/VinAI/K4-DAY16-Track3A-PhanHoangVu-2A202602450-AgentArena/harness/layers/retry.py))
*Mục tiêu: Giảm phương sai (từ 24.21 xuống 11.43), vượt qua 15% lỗi ngẫu nhiên / suy giảm của công cụ.*
- [x] Cài đặt hook `wrap_tool_call(self, ctx, call, name, args)`:
  - [x] Gọi lần đầu: `result = call(name, args)`.
  - [x] Lặp lại khi `attempts < self.max_attempts` VÀ `((not result.ok) or is_degraded(result.content))`:
    - **Tự kiểm tra ngân sách:** Nếu `ctx.max_tool_calls is not None and ctx.tools.calls >= ctx.max_tool_calls - self.reserve` → `break` ngay để không lạm tiêu vào lượt `submit`.
    - Gọi lại: `result = call(name, args)`.
    - Tăng `attempts += 1`.
  - [x] Ghi nhận số lượt thử vào `ctx.state["retry_attempts"]`.
  - [x] Trả về `result` cuối cùng (kể cả khi vẫn hỏng, không tự ý bịa dữ liệu).

---

## ⚠️ CÁC NGUYÊN TẮC CẤM KỴ (ĐÃ ĐO THỰC TẾ)

| Hành vi cấm | Hậu quả đo được | Quy tắc đúng |
|---|---|---|
| **Sửa đổi file trong `arena/`** | **0 điểm toàn bài** (trượt kiểm tra mã băm MD5) | Chỉ đọc `arena/scorer.py`, không sửa bất kỳ file nào trong `arena/`. |
| **Sửa chữ trong `claim["text"]`** (thêm dấu chấm, sửa chính tả, đổi nháy, chuẩn hóa dấu cách) | **Mất 40 đến 47.16 điểm** (bị phạt `NOT_FROM_MODEL` 1.5đ/claim) | **Chỉ được:** Giữ nguyên, xóa hẳn, cắt bớt (substring), hoặc đổi `doc_id`. |
| **Gắn `doc_id` của tài liệu chưa đọc** | Bị phạt `UNRETRIEVED` (0.75đ/claim) | Chỉ gán `doc_id` khi `doc.body in ctx.observed_text`. |
| **Bỏ quên `FINALIZE_SENTINEL` trong `NUDGE`** | Agent đem chính câu nhắc đi search tài liệu, mất điểm | Dùng hằng số `NUDGE` có sẵn trong `budget_policy.py`. |
| **Nén hoặc tóm tắt `messages` trong `before_model`** | **Mất 47.16 điểm** do mô hình không trích dẫn được | Giữ nguyên lịch sử message, không tóm tắt. |
| **Bọc `try/except` nuốt lỗi trong hook** | Nuốt lỗi khiến bài bị chấm 0 âm thầm | Để exception nổi lên để phát hiện và sửa ngay lúc luyện tập. |
| **Hard-code theo `brief_id` hoặc đọc `Doc.tags`** | Điểm luyện tập ảo, về 0 khi chấm thi thật | `Doc.tags` luôn rỗng khi runner dựng corpus. |
| **Hạ `MAX_STEPS = 40` trong `agent.py`** | Agent hết bước trước khi ra `FINAL`, nhận 0 điểm | Giữ nguyên `MAX_STEPS = 40`. |

---

## 🛠️ CÁC LỆNH CHẠY & KIỂM TRA ĐỊNH KỲ

*Lưu ý trên môi trường Windows PowerShell, luôn chạy kèm `$env:PYTHONUTF8=1` để tránh lỗi mã hóa ký tự tiếng Việt.*

```powershell
# 1. Kiểm tra 21 điều kiện hệ thống
$env:PYTHONUTF8=1; python scripts/verify.py

# 2. Chạy baseline (chưa có layer nào, ~24.27đ)
$env:PYTHONUTF8=1; python scripts/run_practice.py --layers none

# 3. Chạy từng layer độc lập để kiểm tra logic
$env:PYTHONUTF8=1; python scripts/run_practice.py --layers critic
$env:PYTHONUTF8=1; python scripts/run_practice.py --layers critic,citation_checker

# 4. Xem chẩn đoán chi tiết điểm số và bẫy
$env:PYTHONUTF8=1; python scripts/selfeval.py --summary
$env:PYTHONUTF8=1; python scripts/selfeval.py --brief pub-05-chi-so-kho-lanh

# 5. Chạy toàn bộ 5 layer
$env:PYTHONUTF8=1; python scripts/run_practice.py --entry doi-cua-ban --out runs/doi-cua-ban.json

# 6. Kiểm tra Leave-One-Out (rút thử từng layer xem điểm có tụt không)
$env:PYTHONUTF8=1; python scripts/run_practice.py --layers injection_guard,critic,citation_checker,budget_policy  # bỏ retry
$env:PYTHONUTF8=1; python scripts/leaderboard.py runs/*.json

# 7. Chạy bộ unit tests cho các layer
python -m pytest tests/test_layers_stubs.py -q
```

---

## 🏁 CHECKLIST TRƯỚC KHI FREEZE & NỘP BÀI (PHÚT 95)

- [x] `$env:PYTHONUTF8=1; python scripts/verify.py` đạt **21/21 mục** (màu xanh).
- [x] `python -m pytest tests/test_layers_stubs.py -q` đạt 18/18 passed.
- [x] `runs/practice.json` có `gate_passed = true` cho cả 9 brief.
- [x] Không có cảnh báo `⚠ Không có FINAL đọc được ở: ...`.
- [x] Không có gán giá trị mới cho `claim["text"]` trong bất kỳ file nào.
- [x] Thư mục `arena/` hoàn toàn nguyên vẹn (`git status` không có file sửa đổi trong `arena/`).
- [x] Điểm luyện tập full stack đạt chuẩn mẫu: **81.71 / 100** (5 brief đạt tuyệt đối 100/100).
- [ ] Thực hiện nộp bài:
  ```powershell
  git add harness/
  git commit -m "Agent Arena — Hoàn thành 5 middleware layers"
  git push
  ```
