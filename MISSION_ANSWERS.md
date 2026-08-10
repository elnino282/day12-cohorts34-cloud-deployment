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
