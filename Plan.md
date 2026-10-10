# 📋 Douyin Translator — Kế Hoạch Phát Triển Hoàn Chỉnh

> **Mục tiêu:** Xây dựng Web App tải video Douyin, trích xuất phụ đề tiếng Trung,
> dịch sang tiếng Việt bằng Gemini AI, rồi gắn cứng (burn/hardsub) phụ đề vào video MP4.

---

## 1. TỔNG QUAN DỰ ÁN

### 1.1 Bài toán giải quyết
Người dùng muốn xem video Douyin (TikTok Trung Quốc) nhưng không hiểu tiếng Trung.
Hệ thống sẽ tự động: **Tải video → Trích phụ đề gốc → Dịch AI → Burn sub vào video → Trả file MP4**.

### 1.2 Đối tượng sử dụng
- Cá nhân muốn xem video Douyin có phụ đề tiếng Việt
- Content creator muốn repost/tham khảo nội dung Douyin

### 1.3 Yêu cầu phi chức năng
- Xử lý bất đồng bộ (không block UI khi render video dài)
- Tự dọn file rác sau 2 giờ
- Giới hạn 5 job đồng thời, timeout 15 phút/job
- Chạy được trên máy local hoặc VPS bằng Docker Compose

---

## 2. CÔNG NGHỆ SỬ DỤNG (Tech Stack)

| Layer | Công nghệ | Vai trò |
|---|---|---|
| **Frontend** | HTML + Vanilla JS + CSS | Giao diện nhập URL, hiển thị tiến trình, tải file |
| **Backend API** | Python 3.11 + FastAPI | REST API xử lý yêu cầu và quản lý Job |
| **Tải video** | yt-dlp | Tải video + subtitle gốc từ Douyin |
| **Fallback tải** | Selenium (Browser Bridge) | Khi yt-dlp bị Douyin chặn, dùng trình duyệt thật |
| **Trích sub** | ffprobe + pysrt | Đọc/parse file .srt, .ass, .vtt, .ttml |
| **Dịch thuật** | Google Gemini API (gemini-2.5-flash) | Dịch phụ đề Trung → Việt theo ngữ cảnh |
| **Transcribe** | Gemini Video API | Khi video không có sub gốc, upload video lên Gemini để tạo sub |
| **Render video** | FFmpeg (libx264, CPU) | Burn subtitle tiếng Việt vào video, xuất MP4 |
| **Task Queue** | Celery + Redis | Xử lý bất đồng bộ, theo dõi tiến trình từng stage |
| **Lưu trữ** | Local disk (thư mục `/tmp/jobs`) | File tạm: video, subtitle, output |
| **Đóng gói** | Docker + Docker Compose | 4 container: api, worker, bridge, redis |

### 2.1 Dependencies Python (`requirements.txt`)
```
fastapi==0.111.0          # Web framework
uvicorn[standard]==0.29.0 # ASGI server
celery==5.3.6             # Task queue
redis==5.0.4              # Redis client
yt-dlp                    # Tải video Douyin
pysrt==1.1.2              # Parse/tạo file .srt
google-generativeai==0.5.4 # Gemini SDK
python-multipart==0.0.9   # Upload file
aiofiles==23.2.1          # Async file I/O
pydantic-settings==2.2.1  # Config từ .env
httpx==0.27.0             # HTTP client (gọi bridge)
selenium==4.21.0          # Browser automation (bridge)
```

### 2.2 Phần mềm cần cài trên hệ thống
- **Python 3.11+**
- **FFmpeg** (để render video + convert subtitle)
- **Redis** (message broker cho Celery)
- **Docker + Docker Compose** (để đóng gói)
- **Google Chrome/Chromium** (cho Browser Bridge fallback)

### 2.3 Chi phí
| Hạng mục | Chi phí | Ghi chú |
|---|---|---|
| Gemini API | **Miễn phí** (Free Tier) | Giới hạn ~15 request/phút, ~1500 request/ngày. Vượt quá sẽ tính phí |
| yt-dlp, FFmpeg, Redis, Docker | **Miễn phí 100%** | Mã nguồn mở |
| VPS (nếu deploy) | **~$5-10/tháng** | Tuỳ chọn, không bắt buộc |
| GPU | **Không cần** | FFmpeg dùng CPU, Gemini chạy trên cloud Google |

---

## 3. KIẾN TRÚC HỆ THỐNG

### 3.1 Sơ đồ tổng quan

```
┌─────────────────────────────────────────────────────────────────┐
│                        NGƯỜI DÙNG                               │
│              (Trình duyệt web — localhost:8000)                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTP (fetch API)
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    FASTAPI SERVER (:8000)                        │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │ health.py│  │   jobs.py    │  │    main.py               │   │
│  │ GET /    │  │ POST /api/   │  │ Mount static frontend    │   │
│  │ health   │  │ jobs         │  │ CORS middleware           │   │
│  └──────────┘  │ GET  /api/   │  └──────────────────────────┘   │
│                │ jobs/{id}    │                                   │
│                │ GET  /download│                                  │
│                │ GET  /srt    │                                   │
│                │ DELETE /{id} │                                   │
│                └──────┬───────┘                                   │
└───────────────────────┼──────────────────────────────────────────┘
                        │ Celery .delay()
                        ▼
┌───────────────────────────────────────────────────────────────────┐
│                         REDIS (:6379)                             │
│                    (Message Broker + Result Backend)              │
└───────────────────────┬───────────────────────────────────────────┘
                        │ Worker nhận task
                        ▼
┌───────────────────────────────────────────────────────────────────┐
│                    CELERY WORKER (pipeline.py)                    │
│                                                                   │
│  Stage 1: DOWNLOADING ──→ downloader.py ──→ yt-dlp               │
│           │                   │                                   │
│           │        (nếu yt-dlp fail)                              │
│           │                   ▼                                   │
│           │          browser_bridge.py ──→ Bridge Server (:8765)  │
│           ▼                                                       │
│  Stage 2: EXTRACTING ──→ subtitle_extractor.py                   │
│           │               (ffmpeg convert .vtt/.ass → .srt)       │
│           │               (hoặc extract embedded sub)             │
│           ▼                                                       │
│  Stage 3: TRANSLATING ──→ translator.py ──→ Gemini API           │
│           │     (nếu không có sub gốc)                            │
│           │               ▼                                       │
│           │     TRANSCRIBING ──→ gemini_transcriber.py            │
│           │               (Upload video lên Gemini → tạo sub)     │
│           ▼                                                       │
│  Stage 4: RENDERING ──→ renderer.py ──→ FFmpeg (burn sub vào MP4)│
│           ▼                                                       │
│         DONE ──→ Cập nhật job.json, copy output                  │
└───────────────────────────────────────────────────────────────────┘
```

### 3.2 Luồng xử lý chi tiết (Pipeline)

```
Người dùng dán link Douyin
        │
        ▼
[1] POST /api/jobs {url, options}
        │── Validate URL (chỉ chấp nhận douyin.com)
        │── Preflight check (Gemini key? Cookies? FFmpeg? Bridge?)
        │── Kiểm tra max concurrent jobs (≤ 5)
        │── Tạo job_id (UUID hex 12 ký tự)
        │── Lưu job.json vào /tmp/jobs/{job_id}/
        │── Celery .delay(job_id, url, options)
        ▼
[2] Frontend polling: GET /api/jobs/{job_id} mỗi 1-3 giây
        │
        ▼
[3] Worker nhận task từ Redis
        │
        ├── Stage: DOWNLOADING (10%)
        │   ├── Normalize URL (xử lý modal_id → /video/{id})
        │   ├── Ưu tiên: Gọi Browser Bridge (Selenium + cookies)
        │   ├── Fallback: yt-dlp (kèm cookies.txt, User-Agent giả lập)
        │   ├── Kiểm tra file size ≤ 500MB
        │   └── Thu thập subtitle candidates (.srt/.ass/.vtt/.ttml)
        │
        ├── Stage: EXTRACTING (30%)
        │   ├── Nếu có file sub → convert sang .srt bằng FFmpeg
        │   ├── Nếu không có file → thử extract embedded sub từ video
        │   ├── Clean HTML tags, normalize encoding (UTF-8/GBK/GB2312)
        │   └── Nếu vẫn không có → chuyển sang TRANSCRIBING
        │
        ├── Stage: TRANSLATING hoặc TRANSCRIBING (55%)
        │   ├── [Có sub gốc] → translator.py:
        │   │   ├── Chia sub thành batch (50 dòng/batch)
        │   │   ├── Gửi Gemini với prompt dịch chuyên nghiệp
        │   │   ├── Parse response theo format "index|||nội dung|||"
        │   │   ├── Retry 3 lần (backoff: 1s, 2s, 4s)
        │   │   └── Gemini detect vị trí sub gốc trong video
        │   │
        │   ├── [Không có sub gốc] → gemini_transcriber.py:
        │   │   ├── Upload video lên Gemini File API
        │   │   ├── Chờ Gemini xử lý video (polling state)
        │   │   ├── Prompt: tạo sub tiếng Việt + detect vị trí
        │   │   └── Parse JSON response → pysrt SubRipFile
        │   │
        │   └── Xuất file subtitle.vi.srt
        │
        ├── Stage: RENDERING (80%)
        │   ├── FFmpeg burn subtitle vào video
        │   ├── Tuỳ chỉnh: font, size, color, position (top/bottom)
        │   ├── MarginV tự động tính theo % chiều cao video
        │   ├── Codec: libx264, preset fast, CRF 18
        │   └── Xuất output.mp4
        │
        └── Stage: DONE (100%)
            ├── Cập nhật job.json (status=done, download_url, srt_url)
            └── Copy kết quả vào output_dir (tên file = video title)
```

---

## 4. CẤU TRÚC THƯ MỤC DỰ ÁN

```
douyin_downloading/
├── .env.example              # Template biến môi trường
├── .env                      # Biến môi trường thực (không commit)
├── .gitignore
├── .dockerignore
├── docker-compose.yml        # Điều phối 4 container
├── requirements.txt          # Python dependencies
├── Plan.md                   # (File này)
├── README.md                 # Hướng dẫn sử dụng
│
├── backend/                  # ===== TOÀN BỘ BACKEND =====
│   ├── __init__.py
│   ├── main.py               # FastAPI app, mount router + static
│   ├── config.py             # Pydantic Settings đọc .env
│   │
│   ├── models/               # --- Data Models (Pydantic) ---
│   │   ├── __init__.py
│   │   └── job.py            # JobStatus, JobStage, SubtitleOptions,
│   │                         # JobCreate, JobResponse, STAGE_LABELS
│   │
│   ├── routers/              # --- API Endpoints ---
│   │   ├── __init__.py
│   │   ├── jobs.py           # CRUD: POST, GET, DELETE /api/jobs
│   │   └── health.py         # GET /health, GET /health/runtime
│   │
│   ├── services/             # --- Business Logic ---
│   │   ├── __init__.py
│   │   ├── downloader.py     # yt-dlp wrapper + bridge fallback
│   │   ├── browser_bridge.py # Gọi Bridge Server qua HTTP
│   │   ├── subtitle_extractor.py  # Extract + convert sub → .srt
│   │   ├── translator.py     # Gemini batch dịch Trung → Việt
│   │   ├── gemini_transcriber.py  # Upload video → Gemini tạo sub
│   │   └── renderer.py       # FFmpeg burn subtitle vào video
│   │
│   ├── tasks/                # --- Celery Background Tasks ---
│   │   ├── __init__.py
│   │   └── pipeline.py       # Celery app + task process_job()
│   │
│   └── utils/                # --- Tiện ích ---
│       ├── __init__.py
│       ├── job_store.py      # CRUD job.json (create/get/update/delete)
│       ├── file_manager.py   # Tạo/xoá thư mục job, dọn file cũ
│       ├── srt_parser.py     # Load/save/clean/normalize file .srt
│       └── preflight.py      # Kiểm tra runtime: cookies, ffmpeg, bridge
│
├── frontend/                 # ===== GIAO DIỆN WEB =====
│   ├── index.html            # HTML chính (form nhập URL, progress bar)
│   ├── app.js                # Logic: submit, polling, update UI
│   └── style.css             # CSS styling
│
├── docker/                   # ===== DOCKER FILES =====
│   ├── Dockerfile.backend    # Image cho FastAPI API server
│   ├── Dockerfile.worker     # Image cho Celery worker
│   ├── Dockerfile.bridge     # Image cho Browser Bridge (Selenium)
│   └── bridge-requirements.txt
│
├── scripts/                  # ===== SCRIPTS HỖ TRỢ =====
│   ├── check_runtime.py      # Kiểm tra điều kiện chạy trước khi dùng
│   └── host_bridge_server.py # Selenium bridge server (Chromium headless)
│
└── runtime/                  # ===== THƯ MỤC RUNTIME (không commit) =====
    ├── cookies.txt           # Cookies Douyin export từ browser
    └── output/               # Thư mục output chính thức
```

---

## 5. CHI TIẾT API ENDPOINTS

### 5.1 Bảng tổng hợp API

| Method | Endpoint | Mô tả | Request | Response |
|---|---|---|---|---|
| `POST` | `/api/jobs` | Tạo job mới | `{url, options}` | `{job_id, status, created_at}` |
| `GET` | `/api/jobs/{id}` | Lấy trạng thái job | — | `{job_id, status, progress, stage, ...}` |
| `GET` | `/api/jobs/{id}/download` | Tải MP4 đã xử lý | — | File stream MP4 |
| `GET` | `/api/jobs/{id}/srt` | Tải file SRT tiếng Việt | — | File stream SRT |
| `DELETE` | `/api/jobs/{id}` | Xoá job + file liên quan | — | `{success: true}` |
| `GET` | `/health` | Trạng thái server | — | `{ok, version}` |
| `GET` | `/health/runtime` | Kiểm tra runtime chi tiết | — | `{gemini_key, cookies, ffmpeg, bridge}` |

### 5.2 Chi tiết Request/Response

#### `POST /api/jobs` — Tạo job mới
```json
// Request Body
{
  "url": "https://www.douyin.com/video/1234567890",
  "options": {
    "font_size": 24,        // 16-60, mặc định 24
    "font_color": "white",  // white/yellow/cyan/green/red
    "position": "bottom"    // bottom hoặc top
  }
}

// Response 202
{
  "job_id": "a1b2c3d4e5f6",
  "status": "queued",
  "created_at": "2026-10-11T00:00:00Z"
}
```

#### `GET /api/jobs/{id}` — Trạng thái job
```json
// Response 200
{
  "job_id": "a1b2c3d4e5f6",
  "status": "processing",       // queued | processing | done | error
  "progress": 55,               // 0-100
  "stage": "translating",       // queued|downloading|extracting|transcribing|translating|rendering|done|error
  "stage_label": "Gemini dang dich phu de sang tieng Viet...",
  "created_at": "2026-10-11T00:00:00Z",
  "updated_at": "2026-10-11T00:01:30Z",
  "error": null,
  "download_url": null,         // Có khi status=done
  "srt_url": null,              // Có khi status=done
  "video_title": "Tieu de video",
  "duration_seconds": 45.0,
  "subtitle_count": 12
}
```

### 5.3 Stage Labels hiển thị trên giao diện

| Stage | Progress | Hiển thị cho người dùng |
|---|---|---|
| `queued` | 0% | ⏳ Đang chờ trong hàng đợi... |
| `downloading` | 5-25% | ⬇️ Đang tải video từ Douyin... |
| `extracting` | 25-40% | 🔍 Đang trích xuất phụ đề tiếng Trung... |
| `transcribing` | 40-55% | 🎙️ Gemini đang tạo subtitle từ video... |
| `translating` | 55-75% | 🤖 Gemini đang dịch phụ đề sang tiếng Việt... |
| `rendering` | 75-95% | 🎬 Đang ghép phụ đề vào video... |
| `done` | 100% | ✅ Hoàn thành! Video sẵn sàng tải về. |
| `error` | — | ❌ Có lỗi xảy ra: {error_message} |

---

## 6. CHI TIẾT TỪNG MODULE (Source Code Map)

### 6.1 `backend/config.py` — Cấu hình tập trung
- Dùng `pydantic-settings` đọc file `.env` tự động
- Tất cả biến môi trường được khai báo với giá trị mặc định
- Hàm `get_settings()` dùng `@lru_cache` để chỉ khởi tạo 1 lần
- Hàm `has_real_gemini_key()` kiểm tra key có phải placeholder không

### 6.2 `backend/services/downloader.py` — Tải video
- `normalize_douyin_url()`: Chuyển URL dạng `?modal_id=xxx` → `/video/xxx`
- Ưu tiên tải qua **Browser Bridge** (nếu có cookies + bridge đang chạy)
- Fallback sang **yt-dlp** với cấu hình: cookies, proxy, User-Agent giả lập Chrome, geo_bypass
- Thu thập tất cả file subtitle (.srt, .ass, .vtt, .ttml) trong thư mục job
- Kiểm tra file size ≤ `MAX_VIDEO_SIZE_MB` (mặc định 500MB)

### 6.3 `backend/services/subtitle_extractor.py` — Trích xuất phụ đề
- Ưu tiên file subtitle đã tải kèm video
- Nếu không có → thử extract embedded subtitle bằng `ffmpeg -map 0:s:0`
- Convert mọi format (.ass, .vtt, .ttml) sang `.srt` bằng FFmpeg
- Normalize: clean HTML tags, xử lý encoding (UTF-8/GBK/GB2312), wrap dòng dài

### 6.4 `backend/services/translator.py` — Dịch thuật Gemini
- Chia subtitle thành batch (mặc định 50 dòng/batch)
- Prompt chuyên nghiệp: dịch tự nhiên, giữ format `index|||nội dung|||`
- Retry 3 lần với exponential backoff (1s → 2s → 4s)
- Validate: số dòng trả về phải khớp, không cho phép dòng rỗng
- `detect_position()`: Upload video lên Gemini để xác định vị trí sub gốc

### 6.5 `backend/services/gemini_transcriber.py` — Tạo sub từ video
- Dùng khi video **không có subtitle gốc** (fallback)
- Upload video lên Gemini File API → chờ xử lý → prompt tạo sub
- Trả về JSON: `{subtitles: [{start, end, text}], margin_v_pct: 25}`
- Tự động detect vị trí subtitle trong video

### 6.6 `backend/services/renderer.py` — Render video
- Dùng FFmpeg burn subtitle `.srt` vào video gốc
- Tự động tính `MarginV` theo % chiều cao video (probe bằng `ffprobe`)
- Cấu hình: `libx264`, preset `fast`, CRF 18, copy audio
- Hỗ trợ tuỳ chỉnh: font size, color, position (top/bottom)

### 6.7 `backend/services/browser_bridge.py` — Bridge tải video
- Gọi HTTP POST đến Bridge Server (Selenium headless Chromium)
- Fallback IPv4 nếu DNS resolve thất bại
- Bridge Server (`scripts/host_bridge_server.py`): dùng Selenium mở Douyin, inject cookies, tải video

### 6.8 `backend/tasks/pipeline.py` — Celery Pipeline
- Celery app kết nối Redis, cấu hình timeout
- Task `process_job()` chạy tuần tự 4 stage: download → extract → translate → render
- Cập nhật `job.json` ở mỗi stage (progress + label)
- Copy kết quả sang `output_dir` khi hoàn thành
- Bắt `SoftTimeLimitExceeded` để xử lý timeout gracefully

### 6.9 `backend/utils/` — Tiện ích
- **`job_store.py`**: CRUD cho `job.json` (lưu/đọc bằng JSON file, không dùng database)
- **`file_manager.py`**: Tạo thư mục job, xoá thư mục, dọn file cũ hơn `MAX_FILE_AGE_HOURS`
- **`srt_parser.py`**: Load SRT (hỗ trợ nhiều encoding), clean HTML, normalize, save
- **`preflight.py`**: Kiểm tra runtime: Gemini key, cookies, FFmpeg, Bridge status

### 6.10 `frontend/app.js` — Logic giao diện
- Submit form → POST `/api/jobs` → nhận `job_id`
- Polling loop: GET `/api/jobs/{id}` mỗi 1-3 giây (tăng dần)
- Cập nhật progress bar, stage label theo thời gian thực
- Khi `done` → hiển thị nút Download MP4 + Download SRT
- Khi `error` → hiển thị thông báo lỗi

---

## 7. CẤU HÌNH MÔI TRƯỜNG (`.env`)

```env
# === AI ===
GEMINI_API_KEY=your_gemini_api_key       # Bắt buộc — lấy tại https://aistudio.google.com
GEMINI_MODEL=models/gemini-2.5-flash     # Model Gemini (free tier)
GEMINI_BATCH_SIZE=50                     # Số dòng sub mỗi lần gửi dịch

# === Storage ===
TEMP_DIR=/tmp/jobs                       # Thư mục chứa file tạm của job
OUTPUT_DIR=/app/runtime/output           # Thư mục output chính thức
MAX_FILE_AGE_HOURS=2                     # Tự xoá file sau 2 giờ
MAX_VIDEO_SIZE_MB=500                    # Giới hạn kích thước video

# === Queue ===
REDIS_URL=redis://redis:6379/0          # Kết nối Redis (docker service name)
CELERY_CONCURRENCY=2                    # Số worker Celery chạy song song

# === Download ===
YTDLP_PROXY=                            # Proxy (để trống nếu không dùng)
YTDLP_COOKIES_FILE=/app/runtime/cookies.txt  # Path tới cookies Douyin

# === FFmpeg ===
FFMPEG_THREADS=4                        # Số thread FFmpeg render
VIDEO_CRF=18                            # Chất lượng: 18=cao, 23=cân bằng, 28=nhỏ file

# === App ===
MAX_CONCURRENT_JOBS=5                   # Tối đa 5 job cùng lúc
JOB_TIMEOUT_MINUTES=15                  # Timeout mỗi job
APP_VERSION=1.0.0
API_BASE_URL=http://localhost:8000
HOST_DOWNLOADER_URL=http://bridge:8765  # URL Bridge Server trong Docker
```

---

## 8. DOCKER COMPOSE — KIẾN TRÚC 4 CONTAINER

```yaml
services:
  api:          # FastAPI server — port 8000
    build: docker/Dockerfile.backend
    depends_on: [redis, bridge]

  worker:       # Celery worker — xử lý ngầm
    build: docker/Dockerfile.worker
    command: celery -A backend.tasks.pipeline.celery_app worker
    depends_on: [redis, bridge]

  bridge:       # Selenium Chromium headless — fallback tải video
    build: docker/Dockerfile.bridge
    port: 8765

  redis:        # Message broker + result backend
    image: redis:7-alpine
    port: 6379
```

**Kết nối mạng nội bộ Docker:**
- `api` → `redis` (qua `redis://redis:6379/0`)
- `api` → `bridge` (qua `http://bridge:8765`)
- `worker` → `redis` (nhận task)
- `worker` → `bridge` (tải video)

---

## 9. XỬ LÝ LỖI & EDGE CASES

### 9.1 Bảng lỗi và cách xử lý

| Tình huống | Xử lý | Message cho user |
|---|---|---|
| URL không phải `douyin.com` | Validate ở model (`field_validator`) | URL Douyin không đúng định dạng |
| Video private / bị xoá | Bắt `DownloadError` từ yt-dlp | Video không thể tải (private hoặc đã xoá) |
| Douyin chặn tải (403) | Fallback sang Browser Bridge | Douyin yêu cầu cookies mới |
| Cookies hết hạn | Thông báo user export lại | Hay export cookies.txt mới |
| Không có subtitle gốc | Fallback sang `gemini_transcriber` | (Tự động, trong suốt với user) |
| Gemini rate limit | Retry 3 lần, backoff 1→2→4 giây | (Tự động retry) |
| Gemini trả sai format | Raise `TranslationError` | Dịch Gemini thất bại sau nhiều lần thử |
| FFmpeg encode lỗi | Raise `RenderError` | Lỗi khi ghép video, vui lòng thử lại |
| Video quá lớn (>500MB) | Block sau khi download | Video quá lớn, giới hạn 500MB |
| Timeout 15 phút | Celery `SoftTimeLimitExceeded` | Xử lý quá lâu, thử video ngắn hơn |
| Quá 5 job đồng thời | HTTP 429 | Hệ thống đang bận, thử lại sau |
| Bridge không khởi động | Preflight check thất bại | Host bridge chưa sẵn sàng |

### 9.2 Douyin-specific Gotchas
- Douyin thường block nếu không có cookies → phải export `cookies.txt` từ browser
- URL dạng `?modal_id=xxx` cần normalize → đã xử lý trong `normalize_douyin_url()`
- Subtitle Douyin đôi khi là `.ttml` hoặc embed → có fallback extract + convert
- Một số video có dual subtitle (Trung + Anh) → ưu tiên `zh-Hans` > `zh` > `zh-TW`
- Video livestream không hỗ trợ → validate và từ chối sớm

---

## 10. KẾ HOẠCH PHÁT TRIỂN (TIMELINE) — CHO NGƯỜI CLONE

> **Giả định:** 1 lập trình viên Full-stack có kinh nghiệm cơ bản, làm 6-8 tiếng/ngày.
> **Tổng thời gian MVP:** 7-10 ngày làm việc (~50-70 giờ).

### Phase 1 — MVP (Tuần 1: Ngày 1-5)

#### Ngày 1 (4-6 giờ): Phân tích + Setup môi trường
| Việc cần làm | Thời gian | Output |
|---|---|---|
| Đọc hiểu toàn bộ Plan.md và source code | 2h | Nắm kiến trúc |
| Tạo repo Git, setup `.env` từ `.env.example` | 30m | Repo sạch |
| Cài Python venv, `pip install -r requirements.txt` | 30m | Môi trường sẵn sàng |
| Cài FFmpeg, kiểm tra `ffmpeg -version` | 15m | FFmpeg hoạt động |
| Cài Docker, chạy `docker compose up -d redis` | 15m | Redis sẵn sàng |
| Lấy Gemini API Key tại https://aistudio.google.com | 15m | Key trong `.env` |
| Export cookies Douyin (dùng extension trình duyệt) | 30m | `runtime/cookies.txt` |
| Chạy `python scripts/check_runtime.py` | 15m | Tất cả ✅ |

#### Ngày 2 (6-8 giờ): Backend Core — Models + Config + Downloader
| Việc cần làm | Thời gian | File |
|---|---|---|
| Viết `config.py` (pydantic-settings đọc `.env`) | 1h | `backend/config.py` |
| Viết `models/job.py` (Enum, Pydantic models) | 1.5h | `backend/models/job.py` |
| Viết `utils/job_store.py` (CRUD job.json) | 1h | `backend/utils/job_store.py` |
| Viết `utils/file_manager.py` | 30m | `backend/utils/file_manager.py` |
| Viết `services/downloader.py` (yt-dlp wrapper) | 2h | `backend/services/downloader.py` |
| Test tải 1 video Douyin thật bằng script đơn giản | 1h | Kiểm chứng tải được |

#### Ngày 3 (6-8 giờ): Subtitle Extraction + Translation
| Việc cần làm | Thời gian | File |
|---|---|---|
| Viết `utils/srt_parser.py` (load/save/clean SRT) | 1h | `backend/utils/srt_parser.py` |
| Viết `services/subtitle_extractor.py` | 1.5h | `backend/services/subtitle_extractor.py` |
| Test extract sub từ video vừa tải ngày 2 | 30m | Kiểm chứng extract được |
| Viết `services/translator.py` (Gemini batch) | 2h | `backend/services/translator.py` |
| Test dịch 10 dòng sub mẫu qua Gemini | 1h | Kiểm chứng dịch được |
| Viết `services/gemini_transcriber.py` (fallback) | 1.5h | `backend/services/gemini_transcriber.py` |

#### Ngày 4 (6-8 giờ): Renderer + Celery Pipeline ⚠️ Khó nhất
| Việc cần làm | Thời gian | File |
|---|---|---|
| Viết `services/renderer.py` (FFmpeg burn sub) | 2h | `backend/services/renderer.py` |
| Tinh chỉnh font, size, màu, vị trí sub + test | 1.5h | Test visual trên video |
| Viết `tasks/pipeline.py` (Celery task kết hợp 4 services) | 2h | `backend/tasks/pipeline.py` |
| Kết nối Celery ↔ Redis, test chạy task | 1h | Kiểm chứng pipeline chạy |
| Test toàn bộ pipeline end-to-end (script gọi trực tiếp) | 1h | Video output có sub tiếng Việt |

#### Ngày 5 (6-8 giờ): API + Tích hợp
| Việc cần làm | Thời gian | File |
|---|---|---|
| Viết `routers/jobs.py` (POST + GET + DELETE) | 2h | `backend/routers/jobs.py` |
| Viết `routers/health.py` | 30m | `backend/routers/health.py` |
| Viết `utils/preflight.py` (runtime check) | 1h | `backend/utils/preflight.py` |
| Viết `main.py` (FastAPI app, mount router) | 30m | `backend/main.py` |
| Test toàn bộ API qua Swagger UI / curl | 1.5h | API hoạt động |
| Viết `services/browser_bridge.py` | 1h | `backend/services/browser_bridge.py` |
| Test bridge fallback | 1h | Bridge hoạt động |

> **Kết quả cuối tuần 1:** Backend hoàn chỉnh — có thể gửi URL qua API, tải video,
> dịch sub, render ra MP4 có phụ đề tiếng Việt.

---

### Phase 2 — Frontend + Docker (Tuần 2: Ngày 6-9)

#### Ngày 6-7 (12-16 giờ): Frontend
| Việc cần làm | Thời gian | File |
|---|---|---|
| Viết `index.html` (form URL, options, progress bar, kết quả) | 3h | `frontend/index.html` |
| Viết `style.css` | 1h | `frontend/style.css` |
| Viết `app.js` — Submit form + gọi POST API | 2h | `frontend/app.js` |
| Viết `app.js` — Polling loop + cập nhật progress bar | 3h | Phần tốn thời gian nhất |
| Viết `app.js` — Hiển thị kết quả (nút Download) + lỗi | 1.5h | |
| Test end-to-end trong trình duyệt | 2h | Chạy mượt từ A→Z |
| Polish UI: responsive, loading state, error messages | 1.5h | |

#### Ngày 8 (6-8 giờ): Docker + Bridge Server
| Việc cần làm | Thời gian | File |
|---|---|---|
| Viết `docker/Dockerfile.backend` | 1h | Cài FFmpeg, fonts, yt-dlp |
| Viết `docker/Dockerfile.worker` | 30m | Tương tự backend |
| Viết `docker/Dockerfile.bridge` | 1h | Cài Chromium + Selenium |
| Viết `scripts/host_bridge_server.py` | 2h | Selenium bridge |
| Viết `docker-compose.yml` | 1h | 4 services |
| Build + test: `docker compose up --build` | 2h | Debug đường dẫn, network |

#### Ngày 9 (4-6 giờ): Dọn dẹp + Hoàn thiện
| Việc cần làm | Thời gian | File |
|---|---|---|
| Logic xoá file rác (`prune_old_jobs`) | 30m | `utils/file_manager.py` |
| Viết `scripts/check_runtime.py` | 30m | |
| Viết `README.md` hoàn chỉnh | 1h | |
| Test toàn bộ trên Docker (VPS hoặc máy khác) | 2h | |
| Fix bug, edge cases Section 9 | 1.5h | |

> **Kết quả cuối tuần 2:** MVP hoàn chỉnh — có thể chạy bằng `docker compose up --build`.

---

### Phase 3 — Production Ready (Tuỳ chọn, tuần 3-4)
- [ ] Xử lý lỗi toàn diện + retry logic nâng cao
- [ ] Rate limiting: max 3 jobs/IP/giờ
- [ ] Auto cleanup file cũ (cron job)
- [ ] Logging + monitoring cơ bản
- [ ] Deploy lên VPS (Ubuntu + Nginx reverse proxy)
- [ ] HTTPS (Let's Encrypt)

### Phase 4 — Nâng cao (Tuỳ chọn, tháng 2+)
- [ ] Hỗ trợ thêm: TikTok, YouTube Shorts, Instagram Reels
- [ ] Tuỳ chỉnh style sub nâng cao: chọn font Google Fonts
- [ ] Preview sub trước khi render
- [ ] Lịch sử job (localStorage)
- [ ] Whisper fallback khi Gemini không available

---

## 11. CHECKLIST BẮT ĐẦU — "CLONE VỀ LÀ CHẠY"

Copy checklist này và làm từng bước theo thứ tự:

### ✅ Bước 1: Setup
- [ ] Clone repo: `git clone <repo_url>`
- [ ] Tạo `.env` từ `.env.example`
- [ ] Điền `GEMINI_API_KEY` (lấy tại https://aistudio.google.com)
- [ ] Export cookies Douyin → `runtime/cookies.txt` (dùng browser extension)
- [ ] Chạy: `docker compose up --build`
- [ ] Mở: `http://localhost:8000`

### ✅ Bước 2: Kiểm tra
- [ ] `curl http://localhost:8000/health` → `{"ok": true}`
- [ ] `curl http://localhost:8000/health/runtime` → tất cả `true`
- [ ] `curl http://localhost:8765/health` → Bridge OK

### ✅ Bước 3: Test thử
- [ ] Dán 1 link video Douyin ngắn (~30 giây) có phụ đề
- [ ] Chờ progress bar chạy từ 0% → 100%
- [ ] Tải MP4 và kiểm tra phụ đề tiếng Việt hiển thị đúng
- [ ] Tải file SRT riêng

### ✅ Bước 4: Nếu gặp lỗi
- [ ] Douyin chặn? → Export lại cookies mới
- [ ] Gemini 429? → Chờ 1 phút rồi thử lại (rate limit)
- [ ] FFmpeg lỗi font? → Kiểm tra fonts-noto đã cài trong Docker
- [ ] Bridge lỗi? → `docker logs <bridge_container>` để debug

---

## 12. KỸ NĂNG CẦN CÓ VÀ SẼ HỌC ĐƯỢC

### Kỹ năng tiền đề (Cần biết trước)
- Python cơ bản → trung bình (functions, pip, OOP cơ bản)
- HTML + CSS + JavaScript cơ bản (DOM, fetch API)
- Biết dùng Terminal / dòng lệnh
- Khái niệm REST API (GET, POST, DELETE)

### Kỹ năng sẽ master sau dự án
| Kỹ năng | Học từ đâu trong dự án |
|---|---|
| **FastAPI** framework | `main.py`, `routers/` |
| **Pydantic** validation + settings | `models/job.py`, `config.py` |
| **Celery + Redis** (async task queue) | `tasks/pipeline.py` |
| **yt-dlp** (cào dữ liệu video) | `services/downloader.py` |
| **FFmpeg** (xử lý media) | `services/renderer.py` |
| **Gemini API** (tích hợp AI) | `services/translator.py`, `gemini_transcriber.py` |
| **Prompt Engineering** | Prompt trong `translator.py` |
| **Selenium** (browser automation) | `browser_bridge.py`, `host_bridge_server.py` |
| **Docker + Docker Compose** | `docker/`, `docker-compose.yml` |
| **Polling pattern** (realtime UI) | `frontend/app.js` |
| **Error handling** nâng cao | Toàn bộ `services/` |
| **File-based storage** | `utils/job_store.py` |

---

> **Lưu ý cuối:** Nếu bạn đã có kinh nghiệm với yt-dlp và FFmpeg,
> thời gian có thể rút ngắn xuống chỉ còn **4-5 ngày**.
> Phần khó nhất là debug việc Douyin chặn tải video (Stage 1)
> và tinh chỉnh prompt Gemini trả về đúng format (Stage 3).

— Hết kế hoạch — Chúc code vui! 🚀