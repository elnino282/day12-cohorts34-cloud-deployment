# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

Sau khi đọc `01-localhost-vs-production/develop/app.py`, tôi xác định các anti-pattern sau:

1. **Hardcode API key trong source code:** secret có thể bị lộ qua Git, lịch sử commit hoặc repository public; việc thu hồi và thay key cũng khó kiểm soát.
2. **Hardcode thông tin kết nối database:** username, password, host và database name bị gắn vào code, vừa không an toàn vừa không dùng được cho nhiều environment.
3. **Không quản lý config tập trung:** `DEBUG` và `MAX_TOKENS` là hằng số trong code; muốn đổi cấu hình phải sửa và deploy lại ứng dụng.
4. **Ghi secret ra standard output:** câu lệnh `print()` làm API key xuất hiện trong terminal hoặc log của cloud platform.
5. **Dùng `print()` thay cho structured logging:** log không có level, timestamp hoặc trường dữ liệu ổn định nên khó tìm kiếm, cảnh báo và phân tích tập trung.
6. **Không có health/readiness endpoint:** nền tảng không biết tiến trình còn sống hay đã sẵn sàng nhận traffic để restart hoặc loại instance khỏi load balancer.
7. **Chỉ bind vào `localhost`:** service chỉ nhận kết nối loopback và không thể được truy cập đúng cách từ bên ngoài container.
8. **Cố định port `8000`:** Railway, Render và các nền tảng khác thường cấp port qua biến môi trường `PORT`.
9. **Luôn bật `reload=True`:** file watcher tạo thêm tiến trình, tiêu tốn tài nguyên và có thể gây hành vi shutdown không mong muốn trên production.
10. **Không có lifecycle hoặc graceful shutdown:** ứng dụng không khai báo bước khởi tạo/dọn dẹp tài nguyên và có nguy cơ cắt request đang xử lý khi nhận tín hiệu dừng.

### Exercise 1.2: Run the basic version

Môi trường kiểm thử sử dụng Python 3.11.9. Từ repository root:

```bash
cd 01-localhost-vs-production/develop
python -m pip install -r requirements.txt
python app.py
```

Gửi request ở terminal thứ hai:

```bash
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

Kết quả kiểm thử thực tế:

```text
HTTP status: 200
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

Kết luận: phiên bản basic chạy được về mặt chức năng, nhưng chưa production-ready vì secret, config, network binding, logging, health check và shutdown đều chưa đáp ứng môi trường cloud.

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Why Important? |
|---|---|---|---|
| Configuration | Hằng số nằm trong `app.py` | `Settings` đọc environment variables | Một artifact có thể chạy ở dev, staging và production mà không cần sửa code. |
| Secrets | API key và database credential nằm trong source | Secret lấy từ environment; production fail fast nếu thiếu `AGENT_API_KEY` | Tránh lộ credential trong Git và cho phép rotate secret độc lập với deployment. |
| Host binding | Cố định `localhost` | `HOST`, mặc định `0.0.0.0` | Container phải lắng nghe trên mọi interface để nhận traffic từ proxy hoặc platform. |
| Port | Cố định `8000` | Đọc từ `PORT` | Cloud platform có thể cấp port động; hardcode có thể làm service không khởi động được. |
| Debug/reload | Luôn bật reload | Điều khiển bằng `DEBUG`, mặc định tắt | Tránh thêm file watcher/process, giảm tài nguyên và hành vi không ổn định trên production. |
| Logging | `print()` và còn ghi API key | Structured JSON với level, event và metadata không nhạy cảm | Log có cấu trúc dễ tìm kiếm, tổng hợp, cảnh báo và không làm rò rỉ secret. |
| Health/readiness | Không có | Có `/health` và `/ready` | Orchestrator biết khi nào restart instance và khi nào được phép route traffic. |
| Shutdown | Dừng đột ngột | FastAPI lifespan và xử lý `SIGTERM` | Cho phép ngừng nhận traffic, hoàn tất request và dọn kết nối trước khi tiến trình thoát. |
| CORS | Không cấu hình | Origins lấy từ `ALLOWED_ORIGINS` | Chỉ frontend được cho phép mới có thể gọi API từ trình duyệt. |
| Runtime metadata | Chỉ trả thông báo localhost | Trả app name, version, environment, uptime và timestamp | Hỗ trợ kiểm tra đúng phiên bản, quan sát deployment và chẩn đoán sự cố. |

Kiểm thử production với cấu hình `.env` tạm thời (`PORT=8011`, `DEBUG=false`, `LLM_MODEL=mock-llm`) cho kết quả:

```text
GET  /health  -> 200 {"status":"ok", ...}
GET  /ready   -> 200 {"ready":true}
POST /ask     -> 200 {"question":"Hello production", ... ,"model":"mock-llm"}
```

Điều này minh họa các nguyên tắc 12-Factor chính: config nằm trong environment, service tự port-bind, log là event stream, và process có lifecycle rõ ràng để dễ thay thế khi deploy.

## Part 2: Docker

### Exercise 2.1: Basic Dockerfile questions

1. **Base image là gì?** `python:3.11`. Đây là image Python đầy đủ dựa trên Linux, cung cấp OS userspace, Python interpreter, `pip` và các thư viện runtime. Bản production dùng `python:3.11-slim` để giảm thành phần không cần thiết.
2. **Working directory là gì?** `WORKDIR /app` tạo/chọn `/app` làm thư mục mặc định cho các lệnh `COPY`, `RUN` và `CMD` theo sau, đồng thời là current directory khi container khởi động.
3. **Tại sao copy `requirements.txt` trước source code?** Docker cache theo từng layer. Dependencies chỉ được cài lại khi `requirements.txt` đổi; sửa `app.py` không làm mất cache của layer `pip install`, nên rebuild nhanh hơn.
4. **`CMD` và `ENTRYPOINT` khác nhau thế nào?** `ENTRYPOINT` xác định executable chính và khó bị thay thế vô tình; `CMD` cung cấp command hoặc arguments mặc định và có thể override trực tiếp ở cuối `docker run`. Khi dùng cả hai ở exec form, `CMD` thường là arguments mặc định truyền cho `ENTRYPOINT`.

Dockerfile còn dùng `EXPOSE 8000` để mô tả port ứng dụng lắng nghe. `EXPOSE` không tự publish port; cần `docker run -p HOST_PORT:8000`.

### Exercise 2.2: Build and run the basic container

Build từ repository root vì các lệnh `COPY` dùng đường dẫn bắt đầu bằng `02-docker/` và `utils/`:

```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
docker run --rm -p 8000:8000 my-agent:develop
```

Test:

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is Docker?"}'
curl http://localhost:8000/health
```

Kết quả thực tế (máy kiểm thử dùng `18000:8000` do host port 8000 đang bận):

```text
Image size: 424,604,361 bytes = 424.6 MB = 404.9 MiB
POST /ask   -> 200 {"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
GET /health -> 200 {"status":"ok","uptime_seconds":0.2,"container":true}
```

### Exercise 2.3: Multi-stage build and image comparison

**Stage 1 — `builder`:** bắt đầu từ `python:3.11-slim`, cài `gcc` và `libpq-dev` phục vụ compile, rồi cài Python dependencies vào `/root/.local`. Stage này chỉ tạo build artifacts, không phải image được deploy.

**Stage 2 — `runtime`:** bắt đầu lại từ một image slim sạch, tạo user `appuser`, chỉ copy dependencies đã cài cùng source cần chạy, thiết lập healthcheck và chạy hai Uvicorn workers. Compiler, APT metadata và các build packages không được đưa vào image cuối.

Kết quả build và kiểm thử thực tế:

| Image | Bytes | MiB | Runtime user |
|---|---:|---:|---|
| `my-agent:develop` | 424,604,361 | 404.9 | `root` mặc định |
| `my-agent:advanced` | 56,819,592 | 54.2 | `appuser` (UID 999) |

```text
Reduction = (424,604,361 - 56,819,592) / 424,604,361 × 100 = 86.6%
Docker health: healthy
GET /health: 200
POST /ask: 200
```

Image nhỏ hơn giúp pull/start nhanh, giảm storage và attack surface. Multi-stage còn tách build tools khỏi runtime, trong khi non-root user giảm tác động nếu ứng dụng bị khai thác.

### Exercise 2.4: Docker Compose architecture and stack test

```text
                              internal bridge network
Client ──HTTP :80──> Nginx ──agent:8000──> FastAPI Agent
                                                │
                      ┌─────────────────────────┴──────────────────────┐
                      │                                                │
              redis:6379                                      qdrant:6333
          session/rate-limit cache                         vector database
          volume: redis_data                              volume: qdrant_data
```

Compose khởi động bốn service:

- **Nginx** là service duy nhất publish host port 80; nó reverse proxy đến `agent:8000` và áp dụng rate limiting.
- **Agent** không publish trực tiếp ra host. Nó nhận `REDIS_URL=redis://redis:6379/0` và `QDRANT_URL=http://qdrant:6333`.
- **Redis** cung cấp cache nội bộ và lưu dữ liệu trong named volume `redis_data`.
- **Qdrant** cung cấp vector database nội bộ và lưu dữ liệu trong `qdrant_data`.

Tên `agent`, `redis` và `qdrant` được Docker DNS phân giải trên private network `internal`. `depends_on` đợi Redis và Qdrant healthy trước khi start agent; Nginx chỉ chuyển traffic đến agent trong mạng này.

Kiểm thử stack với project riêng `day12-part2-test`:

```text
agent:  healthy                 redis: healthy, PING -> PONG
qdrant: healthy                 nginx: running
agent resolved redis and qdrant through Docker DNS
GET  http://localhost/health -> 200 {"status":"ok", ...}
POST http://localhost/ask    -> 200 {"answer":"Agent đang hoạt động tốt! ..."}
```

Sau kiểm thử, stack được dừng bằng `docker compose down --volumes`; containers, private network và hai test volumes đã được xóa.
