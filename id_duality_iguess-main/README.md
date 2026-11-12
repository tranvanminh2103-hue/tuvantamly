# DUAL // SELF — Hệ thống tư vấn tâm lý học đường AI

Mở rộng từ giao diện **"thì thầm"** (`main.html`). Giao diện cũ được giữ nguyên; chỉ thêm 7 dòng tích hợp.

## Các phân hệ

| Phân hệ | Mô tả | Mã nguồn |
|---|---|---|
| AI Chat | Tư vấn theo ngữ cảnh: vai trò, tâm trạng, chủ đề, lịch sử hội thoại | `server/chat.py`, `server/prompts.py`, `assets/ai-chat.js` |
| Multi-AI | Gemini, Groq, OpenRouter, Claude, OpenAI, Ollama (local), mock. Tự chuyển khi lỗi/hết hạn mức; admin đổi thứ tự/mô hình không cần sửa mã | `server/providers/` |
| Knowledge Base | Tài liệu riêng theo đối tượng (học sinh/phụ huynh/giáo viên), tải `.txt/.md/.pdf` | `server/rag.py`, `knowledge_seed/` |
| RAG | Truy xuất BM25 (SQLite FTS5, không dấu) trước khi trả lời, kèm nguồn tham khảo | `server/rag.py` |
| User Management | 4 vai trò: học sinh (ẩn danh hoặc tài khoản), phụ huynh, giáo viên (cần mã mời), quản trị | `server/auth.py` |
| Conversation | Lưu, xem lại, đổi tên, xoá; chỉ chủ sở hữu truy cập | `server/api.py` |
| Admin Dashboard | Thống kê, cảnh báo an toàn, kho kiến thức, người dùng, cấu hình AI/hotline | `admin.html`, `server/admin_api.py` |
| Safety Layer | Phát hiện nguy cơ trước & sau AI (chi tiết bên dưới) | `server/safety.py` |

## Chạy thử

```bash
pip install -r requirements.txt
cp .env.example .env          # điền ADMIN_PASSWORD và ít nhất 1 API key (hoặc để trống: chạy chế độ mock)
python -m server.cli seed     # nạp 6 tài liệu mẫu
python gemini_ai_server.py    # http://127.0.0.1:5000  (admin: /admin)
python -m unittest discover -s server/tests -t .    # 36 test
```

Khóa AI miễn phí/chi phí thấp: Gemini (Google AI Studio), Groq, OpenRouter (mô hình `:free`), hoặc Ollama chạy local (`OLLAMA_MODEL=...`).
Tên mô hình thay đổi theo thời gian: kiểm tra lại trong tài liệu nhà cung cấp rồi sửa `*_MODEL` hoặc ngay trong `/admin`.

## Phân quyền

| | Học sinh | Phụ huynh | Giáo viên | Admin |
|---|:-:|:-:|:-:|:-:|
| Chat AI + lịch sử của chính mình | ✔ | ✔ | ✔ | ✔ |
| Tài liệu RAG xem được | all + student | all + parent | all + teacher | tất cả |
| Xem cảnh báo an toàn (không nội dung) | | | ✔ | ✔ |
| Xem trích đoạn tin nhắn nguy cơ | | | | ✔ |
| Quản lý kho kiến thức, người dùng, AI, hotline | | | | ✔ |

Phụ huynh và giáo viên **không** đọc được hội thoại của học sinh.

## Safety Layer

1. **Trước AI** — quy tắc nhận diện tiếng Việt (có/không dấu, teencode, loại trừ cách nói cường điệu như "cười chết mất"). Bốn mức: 0 bình thường, 1 lưu ý, 2 nguy cơ cao, 3 khủng hoảng.
2. **Mức 3 không gọi AI**: trả mẫu cố định kèm số khẩn cấp. Mức 2: AI chỉ phản hồi theo kịch bản an toàn, luôn kèm nguồn hỗ trợ. Mức nguy cơ được giữ ("sticky") trong vài tin nhắn kế tiếp.
3. **Sau AI** — chặn chẩn đoán ("bạn bị trầm cảm"), chặn thông tin về phương tiện/liều lượng; nếu vi phạm, thay bằng phản hồi an toàn.
4. **Prompt hệ thống**: AI nói rõ mình không phải chuyên gia, không chẩn đoán, không kê thuốc; tài liệu RAG chỉ là dữ liệu, không phải mệnh lệnh.
5. **Bài đăng/bình luận/tin nhắn riêng ở giao diện cũ** cũng được kiểm tra; nếu nguy cơ, hiện thẻ hỗ trợ (không chặn đăng, không lưu nội dung, không gắn danh tính).
6. Nếu mọi nhà cung cấp AI lỗi, hệ thống trả phản hồi an toàn thay vì im lặng.

## Bảo mật & quyền riêng tư

- API key chỉ đọc từ biến môi trường ở máy chủ; không có trong frontend, không trả về qua API (có test kiểm tra).
- Mật khẩu băm `scrypt`; phiên là token ngẫu nhiên, lưu dạng băm SHA-256, thu hồi được; tài khoản bị khoá mất quyền ngay.
- Giới hạn tần suất (đăng nhập, chat, tạo ẩn danh), kiểm tra quyền sở hữu hội thoại, truy vấn tham số hoá, header bảo mật.
- Xoá hội thoại cũ: `python -m server.cli purge --days 90`. Tắt lưu trích đoạn: `SAFETY_STORE_EXCERPT=0`.

## Triển khai

- **Cả giao diện + API trên một máy chủ** (đơn giản nhất): Docker/Render/Railway/Fly (`Dockerfile` có sẵn, `gunicorn gemini_ai_server:app`). Đặt biến môi trường trong bảng điều khiển của dịch vụ. Gắn ổ đĩa bền cho `DB_PATH`.
- **Giao diện trên GitHub Pages, backend nơi khác**: đặt `window.DUAL_API_BASE` trong `assets/config.js` và `CORS_ORIGINS=https://<user>.github.io`.
- GitHub Pages chỉ chạy tĩnh, **không** chạy được backend Flask. GitHub Actions (`.github/workflows/ci.yml`) chạy test và quét lộ khóa ở mỗi lần push.

## Việc cần làm trước khi dùng thật với học sinh

- Chuyên gia tâm lý học đường **rà soát** `knowledge_seed/`, từ khoá trong `safety.py` và kịch bản phản hồi.
- **Xác minh số hotline & giờ hoạt động** trong `config/hotlines.json` (hoặc ở `/admin`), đặc biệt đường dây "Ngày mai".
- Quy trình nhà trường khi có cảnh báo mức 2–3 (ai liên hệ, trong bao lâu).
- Sự đồng ý của phụ huynh/nhà trường và chính sách dữ liệu cá nhân của trẻ em; chạy thử với nhóm nhỏ có giám sát.
- Mục tiêu tiếp theo: lưu bài đăng/nhắn riêng "thì thầm" vào database (hiện vẫn là dữ liệu mẫu phía trình duyệt), embedding vector cho RAG, Redis cho giới hạn tần suất khi chạy nhiều tiến trình.

---
### Ghi chú gốc của repo

if (she_is_interested_in_me == True){
    printf("TOI YEU NGUYEN MINH TRANG LOP 12A VAN LANG HA LONG NGOI GAN TOI >-<")
}
