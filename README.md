<div align="center">

# Todo AI

**Ứng dụng quản lý công việc full-stack với trợ lý AI, thông báo realtime và nền tảng bảo mật nhiều lớp.**

[![Python](https://img.shields.io/badge/Python-3.12+-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-async-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-7.4-DC382D?logo=redis&logoColor=white)](https://redis.io/)
[![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.8-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)

</div>

---

## Tổng quan

Todo AI là ứng dụng web quản lý công việc được xây dựng từ đầu, gồm backend FastAPI bất đồng bộ, frontend React 19 và toàn bộ hạ tầng đóng gói bằng Docker Compose.

Dự án tập trung vào những phần thường bị bỏ qua trong một ứng dụng CRUD thông thường: vòng đời xác thực an toàn, đẩy dữ liệu realtime, tác vụ nền có chống trùng lặp, và tích hợp LLM có cơ chế dự phòng khi nhà cung cấp không khả dụng.

> **Trạng thái:** hoàn chỉnh cho môi trường development/demo bằng Docker. Các hạng mục cần làm trước khi deploy production được liệt kê ở [mục 8](#8-giới-hạn-hiện-tại).

---

## 1. Tính năng

### Xác thực & phân quyền
- Đăng ký, đăng nhập, đăng xuất, cập nhật hồ sơ
- Mật khẩu băm bằng **Argon2**
- **JWT access token** đặt trong cookie `HttpOnly`
- **Refresh token rotation** — token lưu dưới dạng hash, không lưu plaintext
- Chống **CSRF** bằng cơ chế double-submit cookie kết hợp kiểm tra Origin
- Phân quyền `user` / `admin`; admin không thể tự vô hiệu hóa admin hoạt động cuối cùng

### Quản lý công việc
- CRUD đầy đủ, tìm kiếm, lọc theo trạng thái / độ ưu tiên / deadline
- Danh mục (category) và tag
- Phân trang, sắp xếp, thống kê theo múi giờ người dùng
- Reminder theo deadline

### Realtime
- **WebSocket** đẩy thông báo tức thì, xác thực bằng **ticket dùng một lần** thay vì truyền token qua URL
- **Redis Pub/Sub** làm kênh truyền giữa worker và các kết nối WebSocket
- Polling 30 giây làm fallback khi WebSocket mất kết nối

### Tác vụ nền
- **Reminder worker** chạy như service độc lập, quét công việc sắp đến hạn
- **Redis distributed lock** đảm bảo mỗi reminder chỉ được xử lý một lần dù chạy nhiều instance
- Ghi database trước khi publish lên Pub/Sub để tránh mất thông báo

### Trợ lý AI
- Chat hội thoại có lưu lịch sử, giữ ngữ cảnh 10 lượt gần nhất
- Phân tích công việc và gợi ý chia nhỏ thành subtask, đề xuất độ ưu tiên
- Dùng **Anthropic API**, có **fallback rule-based** để ứng dụng vẫn hoạt động khi không có API key
- **Rate limit** 30 request/giờ mỗi user, triển khai bằng **Lua script atomic** trên Redis

---

## 2. Kiến trúc

```text
Browser (React 19 + Tailwind)
  ├─ HTTPS / REST ────> FastAPI ──SQLAlchemy 2 Async──> PostgreSQL 17
  └─ WSS ─────────────> FastAPI WebSocket <──Pub/Sub─── Redis 7.4
                                               ▲
Reminder worker ──── query / insert ───────────┘
        └─ Redis distributed lock (chống xử lý trùng)

FastAPI AI service ──> Anthropic API (khi có key)
                   └─> rule-based fallback (khi không có key)
```

**Nguyên tắc thiết kế**

- PostgreSQL là nguồn dữ liệu bền vững duy nhất: user, role, token hash, todo, notification, lịch sử AI, audit log.
- Redis **không** dùng làm kho lưu trữ chính — chỉ phục vụ rate limit, distributed lock và truyền realtime.
- Toàn bộ tầng I/O là async: `asyncpg` cho database, `redis.asyncio` cho cache, `AsyncAnthropic` cho AI.
- Worker tách khỏi API để lỗi ở tác vụ nền không ảnh hưởng request người dùng.

---

## 3. Công nghệ

| Tầng | Công nghệ |
|---|---|
| API | Python 3.12+, FastAPI, Pydantic v2, Uvicorn |
| Dữ liệu | PostgreSQL 17, SQLAlchemy 2 Async, asyncpg, Alembic |
| Cache / Realtime | Redis 7.4 — Pub/Sub, distributed lock, Lua script |
| Xác thực | Argon2, JWT, opaque refresh token dạng hash |
| AI | Anthropic Python SDK (async) + fallback rule-based |
| Web | React 19, TypeScript, Vite 7, Tailwind CSS 4, TanStack Query, React Hook Form, Zod |
| Kiểm thử | pytest, pytest-asyncio, Ruff, Vitest, Testing Library, ESLint |
| Vận hành | Docker Compose, Nginx |

**Quy mô:** 10 bảng dữ liệu · 27 endpoint · 7 service trong Compose · ~2.100 dòng Python backend.

---

## 4. Chạy thử nhanh

Yêu cầu: **Docker Desktop** (Compose v2), khoảng 4 GB RAM trống.

```bash
git clone https://github.com/devilc312/todo-ai-fullstack.git
cd todo-ai-fullstack

cp .env.example .env
# Mở .env và thay tối thiểu 3 giá trị:
#   SECRET_KEY      — chuỗi ngẫu nhiên tối thiểu 32 ký tự
#   ADMIN_EMAIL     — email admin khởi tạo
#   ADMIN_PASSWORD  — mật khẩu admin mạnh

docker compose up -d --build
```

Tạo `SECRET_KEY`:

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

Khi các container đã chạy:

| Địa chỉ | Nội dung |
|---|---|
| http://localhost:8080 | Giao diện web |
| http://localhost:8000/docs | Swagger / OpenAPI |
| http://localhost:8000/health | Liveness probe |
| http://localhost:8000/ready | Readiness probe (kiểm tra cả PostgreSQL và Redis) |

Đăng nhập lần đầu bằng `ADMIN_EMAIL` / `ADMIN_PASSWORD` đã đặt trong `.env`.

`ANTHROPIC_API_KEY` là tùy chọn — để trống thì tính năng AI tự chuyển sang chế độ fallback.

<details>
<summary><b>Chạy ở chế độ development (hạ tầng Docker, app chạy native)</b></summary>

Cách này thuận tiện khi cần debug Python/React với hot reload.

**Bước 1 — chỉ chạy database và cache**

```bash
docker compose up -d postgres redis
```

**Bước 2 — backend**

```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # Windows: .\.venv\Scripts\Activate.ps1

pip install -e ".[dev]"

export DATABASE_URL="postgresql+asyncpg://todo:<mật-khẩu-trong-env>@localhost:5432/todo"
export REDIS_URL="redis://localhost:6379/0"

alembic upgrade head
python -m app.seed
uvicorn app.main:app --reload --port 8000
```

Chạy reminder worker ở terminal thứ hai, cùng virtual environment:

```bash
python -m app.workers.reminders
```

**Bước 3 — frontend**

```bash
cd frontend
cat > .env.local <<'EOF'
VITE_API_URL=http://localhost:8000/api/v1
VITE_WS_URL=ws://localhost:8000/api/v1/ws/notifications
EOF

npm ci
npm run dev
```

Mở http://localhost:5173

> Khi API chạy trong Docker, `DATABASE_URL` / `REDIS_URL` phải dùng hostname `postgres` / `redis`. Khi chạy native thì dùng `localhost`.

</details>

<details>
<summary><b>Các lệnh Docker thường dùng</b></summary>

```bash
docker compose ps                      # trạng thái service
docker compose logs -f api worker      # theo dõi log
docker compose restart                 # khởi động lại
docker compose up -d --build           # build lại sau khi đổi dependency
docker compose down                    # dừng, giữ nguyên dữ liệu
```

Reset toàn bộ dữ liệu local:

```bash
docker compose down -v && docker compose up -d --build
```

> **Cảnh báo:** `down -v` xóa volume PostgreSQL và Redis. Không dùng với dữ liệu cần giữ.

</details>

---

## 5. API

Prefix mặc định: `/api/v1`

| Method | Route | Chức năng |
|---|---|---|
| `POST` | `/auth/register` | Tạo tài khoản và cấp token |
| `POST` | `/auth/login` | Đăng nhập |
| `POST` | `/auth/refresh` | Xoay vòng refresh token |
| `POST` | `/auth/logout` | Thu hồi refresh token |
| `GET` `PATCH` | `/auth/me` | Xem / cập nhật hồ sơ |
| `GET` `POST` | `/todos` | Danh sách (lọc, phân trang) / tạo mới |
| `GET` `PATCH` `DELETE` | `/todos/{id}` | Thao tác trên một công việc |
| `GET` | `/todos/stats` | Thống kê theo múi giờ người dùng |
| `GET` `POST` | `/categories` | Danh mục của người dùng |
| `PATCH` `DELETE` | `/categories/{id}` | Sửa / xóa danh mục |
| `GET` | `/notifications` | Danh sách kèm số lượng chưa đọc |
| `POST` | `/notifications/ws-ticket` | Cấp ticket một lần cho WebSocket |
| `PATCH` | `/notifications/{id}/read` | Đánh dấu đã đọc |
| `POST` | `/notifications/read-all` | Đánh dấu đã đọc tất cả |
| `WS` | `/ws/notifications?ticket=...` | Kênh thông báo realtime |
| `POST` | `/ai/suggest` | Phân tích công việc, gợi ý subtask |
| `POST` | `/ai/chat` | Chat theo conversation |
| `GET` `DELETE` | `/ai/conversations/{id}` | Xem / xóa hội thoại |
| `GET` | `/admin/users` | Admin: danh sách người dùng |
| `PUT` | `/admin/users/{id}/roles` | Admin: gán quyền |
| `PATCH` | `/admin/users/{id}/active` | Admin: khóa / mở tài khoản |

Contract đầy đủ xem tại `/docs` khi ứng dụng đang chạy.

---

## 6. Kiểm thử

```bash
# Backend
cd backend && pytest tests -q && ruff check .

# Frontend
npm --prefix frontend test
npm --prefix frontend run lint
npm --prefix frontend run build

# Migration (cần database đang chạy)
cd backend && alembic check
```

Kết quả lần xác minh gần nhất:

| Hạng mục | Kết quả |
|---|---|
| Backend pytest | 9 passed — cookie/CSRF/refresh, Todo/category/stats có DB thật |
| Backend Ruff | Passed |
| Frontend Vitest | 3 passed |
| Frontend ESLint | Passed, 0 warning |
| Frontend build | Passed |
| `alembic check` | Passed, không còn model/migration drift |
| Docker health | API, web, PostgreSQL, Redis healthy; worker running |
| Secret scan | Gitleaks passed |

Phạm vi test hiện tại chống regression cho luồng xác thực, CRUD công việc và WebSocket ticket.

---

## 7. Cấu trúc dự án

```text
.
├── compose.yaml              # Stack chính: PostgreSQL, Redis, API, worker, web
├── .env.example              # Template biến môi trường
├── OPERATIONS_GUIDE.md       # Vận hành, backup/restore, triển khai
├── backend/
│   ├── app/
│   │   ├── api/              # Router: auth, todos, notifications, ai, admin
│   │   ├── core/             # Config, database, redis, security
│   │   ├── models/           # SQLAlchemy models (10 bảng)
│   │   ├── schemas/          # Pydantic schemas
│   │   ├── services/         # AI service, auth service, notifications
│   │   └── workers/          # Reminder worker
│   ├── alembic/              # Migration
│   ├── tests/
│   └── Dockerfile
└── frontend/
    ├── src/
    │   ├── api/              # API client (axios)
    │   ├── pages/            # Dashboard, AI, Admin, Auth, Profile
    │   ├── hooks/            # useAuth, useApi, useNotificationSocket
    │   └── contexts/
    ├── nginx.conf
    └── Dockerfile
```

---

## 8. Giới hạn hiện tại

Dự án đã hoàn chỉnh cho development và demo local. Trước khi đưa lên môi trường production cần bổ sung:

- Bật HTTPS/WSS, `COOKIE_SECURE=true`, cấu hình CORS theo domain thật
- Chuyển secret sang secret manager của nền tảng triển khai
- Mở rộng test cho RBAC nhiều người dùng, lỗi worker và timeout từ AI provider
- Thêm structured logging, request ID, metrics và alerting
- Thiết lập CI, tự động backup PostgreSQL và chính sách retention cho audit log

Một số API backend (lọc nâng cao, trạng thái `archived`) đã có nhưng chưa được đưa đầy đủ lên giao diện.

Chi tiết quy trình vận hành, backup và restore: [OPERATIONS_GUIDE.md](OPERATIONS_GUIDE.md)

---

## 9. Khắc phục sự cố

<details>
<summary><b>Các lỗi thường gặp</b></summary>

**`no configuration file provided: not found`** — đang đứng sai thư mục, hãy `cd` về thư mục gốc chứa `compose.yaml`.

**`ConnectionRefusedError`** — API không kết nối được PostgreSQL hoặc Redis:

```bash
docker compose ps
docker compose logs --tail 100 postgres redis api worker
```

Hostname phải đúng ngữ cảnh: trong Docker dùng `postgres` / `redis`, chạy native dùng `localhost`.

**Port bị chiếm** — các cổng cần trống: `5432` (PostgreSQL), `6379` (Redis), `8000` (API), `8080` (web), `5173` (Vite dev).

**`/ready` lỗi nhưng `/health` vẫn OK** — `/health` chỉ kiểm tra tiến trình API; `/ready` kiểm tra cả PostgreSQL và Redis. Xem log hai dependency này.

**Không nhận được thông báo realtime** — kiểm tra worker đang `Up`, `/ready` trả OK, và `VITE_WS_URL` đúng scheme (`ws://` cho local, `wss://` cho production). Reverse proxy phải hỗ trợ header `Upgrade` / `Connection`.

**AI luôn trả lời theo fallback** — `ANTHROPIC_API_KEY` trống hoặc sai, model không được tài khoản cho phép, hoặc `AI_ENABLED=false`. Khởi động lại API sau khi đổi biến môi trường.

</details>

---

## Giấy phép

MIT

<div align="center">
<sub>Xây dựng bởi <a href="https://github.com/devilc312">@devilc312</a></sub>
</div>
