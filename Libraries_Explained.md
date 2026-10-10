# 📚 Giải Thích Các Thư Viện Sử Dụng (Dependencies)

Dự án này kết hợp nhiều thư viện mạnh mẽ để tạo nên một hệ thống tải, xử lý video và AI dịch thuật hoàn chỉnh. Dưới đây là giải thích chi tiết về vai trò của từng thư viện Python được sử dụng trong dự án (dựa trên file `requirements.txt`).

---

## 1. Web Framework & Server (API)

### **FastAPI** (`fastapi==0.111.0`)
- **Là gì?** Một framework Python hiện đại, hiệu năng rất cao dùng để xây dựng web API.
- **Tại sao dùng?** FastAPI sinh ra để làm API. Nó hỗ trợ lập trình bất đồng bộ (async/await), tự động sinh tài liệu API (Swagger UI), và kiểm tra dữ liệu đầu vào (data validation) rất chặt chẽ nhờ Pydantic.
- **Trong dự án này:** Dùng để tạo các endpoint API như `POST /api/jobs` (để nhận URL Douyin) hay `GET /api/jobs/{id}` (để cập nhật tiến trình trên giao diện web).

### **Uvicorn** (`uvicorn[standard]==0.29.0`)
- **Là gì?** Một máy chủ ASGI (Asynchronous Server Gateway Interface) siêu tốc độ dành cho Python.
- **Tại sao dùng?** FastAPI chỉ là framework viết code, nó cần một server (máy chủ) để chạy code đó và lắng nghe các request từ mạng internet. Uvicorn đóng vai trò này.
- **Trong dự án này:** Nó là lệnh `uvicorn main:app` được dùng để khởi động toàn bộ hệ thống API Backend.

### **python-multipart** (`python-multipart==0.0.9`)
- **Là gì?** Thư viện hỗ trợ phân tích dữ liệu dạng multipart form data.
- **Tại sao dùng?** Đây là thư viện đi kèm thường bắt buộc phải có khi dùng FastAPI nếu dự án của bạn có xử lý Form gửi từ HTML hoặc tính năng Upload File trực tiếp (mặc dù hiện tại dự án gửi/nhận JSON, cài sẵn để mở rộng).

### **aiofiles** (`aiofiles==23.2.1`)
- **Là gì?** Thư viện hỗ trợ đọc/ghi file bất đồng bộ (async I/O) cho Python.
- **Tại sao dùng?** Khi bạn đọc/ghi file video vài trăm MB bằng hàm Python thông thường (`open()`), server sẽ bị nghẽn và không phục vụ được người khác. `aiofiles` giúp việc lưu file và trả file (FileResponse) mượt mà, không block các request khác của API.

---

## 2. Hàng Đợi & Xử Lý Nền (Background Tasks)

### **Celery** (`celery==5.3.6`)
- **Là gì?** Hệ thống hàng đợi tác vụ (Task Queue) mạnh mẽ bậc nhất của Python.
- **Tại sao dùng?** Việc tải, dịch và render video tốn rất nhiều thời gian (vài phút đến chục phút). Không thể để API chờ ngần ấy thời gian rồi mới trả kết quả cho web được (sẽ bị lỗi Timeout). Celery cho phép bạn ném tác vụ xử lý video vào "Hàng đợi", để cho một "Công nhân" (Worker) chạy ngầm phía sau từ từ xử lý.
- **Trong dự án này:** Module `backend/tasks/pipeline.py` chính là worker của Celery thực hiện việc download -> dịch -> render video.

### **Redis** (`redis==5.0.4`)
- **Là gì?** Một hệ thống lưu trữ dữ liệu trên RAM cực nhanh.
- **Tại sao dùng?** Celery cần một nơi để làm "Trạm trung chuyển" (Message Broker). Khi FastAPI có job mới, nó sẽ ghi yêu cầu vào Redis. Celery Worker sẽ liên tục đứng nhìn Redis, thấy có job mới là kéo về xử lý.
- **Trong dự án này:** Đóng vai trò là cầu nối giao tiếp duy nhất giữa FastAPI Server và Celery Worker.

---

## 3. Cào Dữ Liệu & Automation

### **yt-dlp** (`yt-dlp`)
- **Là gì?** Công cụ dòng lệnh (và thư viện Python) mạnh nhất thế giới hiện nay chuyên dùng để tải video và phụ đề từ hơn 1000 trang web khác nhau (YouTube, Douyin, TikTok, Facebook...).
- **Tại sao dùng?** Douyin mã hóa link video rất phức tạp. Dùng `yt-dlp`, bạn không cần viết code bóc tách HTML, nó sẽ tự động tìm link MP4 và link file phụ đề gốc tiếng Trung để tải về máy cho bạn.

### **Selenium** (`selenium==4.21.0`)
- **Là gì?** Thư viện tự động hóa trình duyệt web (điều khiển Chrome, Firefox bằng code).
- **Tại sao dùng?** Douyin thi thoảng sẽ coi `yt-dlp` là "bot" và chặn không cho tải (báo lỗi 403, bắt nhập captcha).
- **Trong dự án này:** Selenium được dùng làm "Vũ khí dự phòng" (Browser Bridge). Nếu yt-dlp chết, Selenium sẽ âm thầm mở một trình duyệt Chrome (Headless không hiện UI), gắn cookies vào, lừa Douyin đây là người thật đang truy cập để lấy link video MP4 gốc.

### **HTTPX** (`httpx==0.27.0`)
- **Là gì?** Thư viện gửi HTTP Request (giống `requests` nhưng hiện đại hơn và hỗ trợ async).
- **Trong dự án này:** FastAPI backend dùng `httpx` để bắn request sang cho Browser Bridge (chạy Selenium) mỗi khi cần hỗ trợ lấy link video bị khóa.

---

## 4. Xử Lý Phụ Đề & AI

### **pysrt** (`pysrt==1.1.2`)
- **Là gì?** Thư viện Python chuyên dùng để thao tác với file phụ đề `.srt` (SubRip).
- **Tại sao dùng?** Chỉnh sửa phụ đề không đơn giản là chỉnh sửa văn bản. Phụ đề có format đặc biệt gồm: *Số thứ tự*, *Thời gian bắt đầu/kết thúc* (timestamp), và *Nội dung*.
- **Trong dự án này:** Dùng để đọc file sub gốc của Douyin, trích nội dung ra gửi cho AI dịch, sau đó lấy bản dịch ghép lại vào file `.srt` với timestamp giữ nguyên.

### **google-generativeai** (`google-generativeai==0.5.4`)
- **Là gì?** Bộ thư viện (SDK) chính thức của Google để gọi API cho các mô hình AI Gemini.
- **Tại sao dùng?** Thay vì dịch bằng Google Translate (dịch từng từ thô cứng), hệ thống dùng Gemini (LLM - Mô hình ngôn ngữ lớn) để hiểu ngữ cảnh, dịch câu mượt mà, tự nhiên và sát nghĩa video hơn (đặc biệt là tiếng lóng, idiom của tiếng Trung).

---

## 5. Quản Lý Cấu Hình

### **pydantic-settings** (`pydantic-settings==2.2.1`)
- **Là gì?** Tiện ích mở rộng của Pydantic để quản lý biến môi trường (`.env`).
- **Tại sao dùng?** Thay vì dùng hàm `os.getenv()` lộn xộn khắp nơi, thư viện này gom toàn bộ cấu hình từ file `.env` vào một Object chung (file `config.py`). Nó tự động kiểm tra xem bạn đã nhập đủ cấu hình chưa (ví dụ: port phải là số nguyên, URL phải hợp lệ).

---

## 6. Công Cụ Độc Lập Bên Ngoài (Không nằm trong requirements.txt)

*Dù không được cài qua `pip`, dự án này bắt buộc phải phụ thuộc vào:*

### **FFmpeg**
- **Là gì?** "Phù thủy" xử lý video dòng lệnh phổ biến nhất thế giới. 
- **Trong dự án này:** Nó là thành phần cốt lõi để làm bước cuối cùng: Burn (gắn chết) chữ phụ đề từ file `.srt` vào trực tiếp các khung hình của file `.mp4`, giúp video tải về bật ở bất kỳ đâu cũng có phụ đề. Lệnh FFmpeg được Python tạo ra và chạy thông qua `subprocess`.
