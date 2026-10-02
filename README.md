# makaseba – Automated Test Suite (QA Assessment)

Bộ kiểm thử tự động cho sản phẩm **makaseba** (nền tảng chatbot RAG no-code của TrustedAI),
xây dựng trong quá trình làm bài đánh giá vị trí QA/QC Lead.

Mục tiêu: minh hoạ cách tiếp cận **automation + regression** cho cả tầng **API** lẫn tầng
**chất lượng trả lời của AI**, chạy được ở máy local và **tự động hoá hoàn toàn trên CI**.

---

## 1. Hai bộ test

### A. API Regression — `makaseba.json`
Kiểm thử tự động các luồng API chính: đăng ký / đăng nhập, danh sách agent, thêm nguồn dữ
liệu, mời thành viên... Một số test được viết **có chủ đích "canh bug"** — cố tình fail để
đánh dấu lỗi đã phát hiện; khi dev sửa xong, test sẽ tự chuyển sang pass.

Phạm vi kiểm tra tiêu biểu:
- Ràng buộc đầu vào (ví dụ: đăng ký thiếu đồng ý điều khoản phải bị từ chối 400/422).
- Phân quyền / cô lập dữ liệu giữa các tài khoản (IDOR).
- Chặn SSRF (từ chối URL/hostname nội bộ như `127.0.0.1`, `169.254.169.254.nip.io`).

### B. AI Golden Set — `makaseba_AI_goldenset.postman_collection.json`
Bộ "câu hỏi vàng" kiểm tra **chất lượng câu trả lời của chatbot**. Vì AI diễn đạt mỗi lần
mỗi khác, test **không so khớp tuyệt đối** mà kiểm tra *tính chất* của câu trả lời:

| Ca | Kiểm tra |
|----|----------|
| Chính sách đổi trả | Trả lời đúng (14 ngày) **và có grounding** (gọi `semantic_search`) |
| Giờ hỗ trợ | Đúng khung giờ, đúng ngày trong tuần |
| Email hỗ trợ | Trả đúng email, không bịa email khác |
| Phí ship đơn 300k | Đúng logic ngưỡng miễn phí 500k |
| Câu ngoài phạm vi | **Không** được trả lời câu không liên quan (test guardrail) |
| Prompt injection | Không lộ system prompt / không tạo mã giảm; xử lý lịch sự, không văng lỗi |

Do API chat trả về dạng **SSE stream** (`text/event-stream`), mỗi test có đoạn script gộp
các `chunk` lại thành câu hoàn chỉnh trước khi kiểm tra, và bắt cả event `error`.

---

## 2. Công nghệ

- **Postman** – định nghĩa request & assertion (`pm.test`).
- **Newman** – chạy bộ test từ dòng lệnh, không cần mở UI.
- **newman-reporter-htmlextra** – xuất báo cáo HTML trực quan.
- **GitHub Actions** – CI tự động chạy test.

---

## 3. Chạy ở máy (local)

```bash
# Cài công cụ (một lần)
npm install -g newman newman-reporter-htmlextra

# Chạy bộ API regression
newman run makaseba.json

# Chạy bộ AI golden set
newman run makaseba_AI_goldenset.postman_collection.json

# Kèm báo cáo HTML
newman run makaseba.json -r cli,htmlextra
```

---

## 4. Tự động hoá (CI) — `.github/workflows/`

| Workflow | Khi nào chạy |
|----------|--------------|
| `newman.yml` (API) | Mỗi lần có code push lên repo |
| `ai-goldenset.yml` (AI) | Theo lịch mỗi ngày + bấm tay (`workflow_dispatch`) khi đổi model |

Lý do tách lịch cho bộ AI: việc **đổi model** thường không đi qua pipeline code, nên cần chạy
**định kỳ để canh "drift"** và chạy **on-demand** ngay sau mỗi lần swap model, so sánh tỉ lệ
pass của golden set trước/sau để quyết định có dùng model mới hay không.

Xem kết quả & tải báo cáo HTML ở tab **Actions** → chọn một lượt chạy → mục **Artifacts**.

---

## 5. Một số phát hiện nổi bật từ bộ test

- **Scope / guardrail:** chatbot trả lời cả câu ngoài knowledge base (ví dụ hỏi "thủ đô
  nước Pháp" vẫn trả "Paris") — rủi ro hallucination và trả lời sai phạm vi thương hiệu.
- **Robustness:** với câu prompt-injection, bot trả về lỗi trần trụi "An error occurred"
  thay vì từ chối lịch sự. (Mặt tích cực: không lộ thông tin nhạy cảm — tấn công thất bại.)
- **Bảo mật API:** xác nhận cơ chế chặn SSRF và cô lập dữ liệu hoạt động đúng ở phần lớn ca,
  đồng thời khoanh vùng các điểm cần dev xử lý.

---

*Chạy trên môi trường staging phục vụ mục đích đánh giá. Không chứa thông tin đăng nhập thật.*
