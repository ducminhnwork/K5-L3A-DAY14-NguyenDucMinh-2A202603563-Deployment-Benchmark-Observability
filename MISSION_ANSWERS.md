# MISSION_ANSWERS — Day 14

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-pattern trong `01-localhost-vs-production/develop/app.py`

1. **Hardcode secret:** `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`. Push lên GitHub là lộ ngay.
2. **Config viết cứng trong code:** `DEBUG = True`, `MAX_TOKENS = 500`. Muốn đổi giữa các môi trường phải sửa code.
3. **Log ra secret:** `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`. Chạy thật thì key nằm trong log.
4. **Dùng `print()` thay vì logging có cấu trúc:** khó lọc, khó gom log.
5. **Không có health check:** không có `/health`, nên platform không biết khi nào agent chết để restart.
6. **Port cố định:** `port=8000`, không đọc biến môi trường `PORT` mà Railway/Render cấp.
7. **`host="localhost"`:** chỉ nhận kết nối từ chính máy, container hoặc máy khác không vào được.
8. **`reload=True`:** chế độ reload của debug còn bật khi chạy thật.
9. **Không xử lý SIGTERM:** khi platform tắt container, request đang chạy bị cắt đột ngột.

### Exercise 1.2: Chạy bản develop

Chạy trong WSL (Debian), `python app.py` trong `01-localhost-vs-production/develop`. Bản này nhận `question` qua query string.

```
$ curl -X POST "http://localhost:8000/ask?question=Hello"
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

Log của server (đo được từ lần chạy này):

```
Starting agent on localhost:8000...
[DEBUG] Got question: Hello
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this
[DEBUG] Response: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.
INFO:     127.0.0.1:55032 - "POST /ask?question=Hello HTTP/1.1" 200 OK
```

Nhận xét: bản này chạy và trả lời được, nhưng key bị in ra log (anti-pattern 3), chỉ lắng nghe trên `localhost` (anti-pattern 7). "Chạy được" chưa có nghĩa là sẵn sàng cho production.

### Exercise 1.3: So sánh với bản production

Chạy `production/app.py` (đã `cp .env.example .env`). Bản này nhận JSON. Kết quả:

```
$ curl -s localhost:8000/health
{"status":"ok","uptime_seconds":3.8,"version":"1.0.0","environment":"development","timestamp":"2026-11-25T09:07:54.982344+00:00"}
$ curl -s localhost:8000/ready
{"ready":true}
$ curl -s -X POST localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","model":"gpt-4o-mini"}
```

Gửi SIGTERM (`pkill -TERM -f "python app.py"`), log ghi:

```
INFO:     Shutting down
INFO:     Waiting for application shutdown.
INFO:     Application shutdown complete.
INFO:     Finished server process [122]
```

Server lắng nghe trên `http://0.0.0.0:8000` (bản develop là `localhost:8000`).

Ghi chú trung thực: trong lần chạy này, các dòng log JSON do code của app ghi (`logger.info(json.dumps(...))`) không xuất hiện trong `/tmp/prod.log`, chỉ thấy log của uvicorn. Mình chưa kiểm tra nguyên nhân, nên phần "logging JSON" trong bảng dưới dựa trên đọc code (`logging.basicConfig(... format='{"time":...}')`), không phải trên log đã quan sát.

| Feature | Basic (develop) | Advanced (production) | Tại sao quan trọng? |
| --- | --- | --- | --- |
| Config | Hardcode trong code (`DEBUG = True`, `MAX_TOKENS = 500`) | Đọc từ biến môi trường qua `config.py` (`settings`) | Đổi cấu hình giữa dev/staging/prod không cần sửa code; không commit secret |
| Secret | `OPENAI_API_KEY = "sk-..."`, mật khẩu DB trong code | Biến môi trường, chỉ commit `.env.example` | Secret trong repo là bị lộ vĩnh viễn trong lịch sử git |
| Host / Port | `localhost`, `8000` cố định | `settings.host` (0.0.0.0), `settings.port` từ `PORT` | Container cần nghe `0.0.0.0`; Railway/Render cấp `PORT` động |
| Health check | Không có | `GET /health` (liveness), `GET /ready` (readiness) | Platform biết khi nào restart, khi nào được route traffic |
| Logging | `print()`, có in cả secret | `logging` với định dạng JSON, không log secret | Log có cấu trúc dễ tìm kiếm; log không được chứa secret |
| Shutdown | Tắt đột ngột | Có `lifespan`, bắt SIGTERM, chờ request đang chạy | Không mất request khi deploy lại hoặc scale |
| Reload | `reload=True` luôn bật | Chỉ bật khi `DEBUG=true` | Reload tốn tài nguyên và không an toàn khi chạy thật |
| Khác | Không có | CORS giới hạn origin, `GET /metrics` | Giảm bề mặt tấn công; có số liệu để giám sát |

**Tự kiểm tra (CP1):** anti-pattern `host="localhost"` và `port=8000` trong `develop/app.py` đã được `production/app.py` sửa thành `host=settings.host` và `port=settings.port`; `reload=True` sửa thành `reload=settings.debug`.

## Part 2: Docker

### Exercise 2.1: Đọc `02-docker/develop/Dockerfile`

1. **Base image:** `python:3.11` (bản đầy đủ, khoảng 1 GB).
2. **Working directory:** `/app`.
3. **Vì sao `COPY requirements.txt` trước `COPY app.py`:** Docker cache theo từng layer. Requirements ít đổi, nên layer `pip install` được dùng lại; chỉ khi sửa code thì mới copy lại phần code, không phải cài lại thư viện.
4. **CMD và ENTRYPOINT:** `CMD ["python", "app.py"]` là lệnh mặc định, có thể bị ghi đè khi `docker run <image> <lệnh khác>`. ENTRYPOINT là lệnh cố định của container, tham số truyền vào được nối thêm sau nó.

### Exercise 2.2-2.3: Build và so sánh image (đo thật bằng `docker images`)

```
$ docker images | grep my-agent
my-agent:advanced      0249a31b522b        240MB         57.7MB
my-agent:develop       46922c8581f8       1.67GB          427MB
```

| Image | Kích thước | Ghi chú |
| --- | --- | --- |
| `my-agent:develop` (single-stage, `python:3.11`) | 1.67 GB | cột thứ nhất của Docker Desktop |
| `my-agent:advanced` (multi-stage, `python:3.11-slim`) | 240 MB | chạy bằng user `appuser`, không phải root |

Chênh lệch: (1.67 GB ≈ 1710 MB) giảm xuống 240 MB, tức giảm khoảng 86%. Multi-stage giảm dung lượng vì stage runtime chỉ copy `site-packages` đã cài, không mang theo build tools; thêm vào đó base `-slim` nhỏ hơn nhiều so với bản đầy đủ.

Gọi container production (`docker run -d -p 8000:8000 my-agent:advanced`), bản này nhận JSON:

```
$ curl -s localhost:8000/health
{"status":"ok","uptime_seconds":12.4,"version":"2.0.0","timestamp":"2026-11-25T09:45:35.682583"}
$ curl -s -X POST localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

Lần đầu mình gọi `/ask?question=Hello` (query string) thì server trả 500 vì code đọc JSON body (`request.json()`); đổi sang gửi JSON thì chạy. Mình chưa chạy container của image develop, nên không có output gọi bản develop.

**Chỉnh sửa vì môi trường:** mạng của mình chặn `apt-get` trong lúc build (lỗi `401 VG-AuthenticationRequired`). `requirements.txt` của bản production chỉ có `fastapi` và `uvicorn` (không cần biên dịch), nên mình xóa bước `RUN apt-get ... gcc libpq-dev` khỏi `02-docker/production/Dockerfile`, rồi build lại thành công.

### Exercise 2.4: Stack Docker Compose

Sơ đồ:

```
Client --> Nginx (cổng 80/443) --> agent (FastAPI, cổng 8000 nội bộ)
                                      |--> redis  (cache, rate limit)
                                      |--> qdrant (vector DB)
```

Cả 4 service chung network `internal`; chỉ Nginx mở cổng ra ngoài. Agent chờ redis và qdrant `healthy` mới khởi động (`depends_on` + `condition: service_healthy`).

Kết quả thật khi gọi qua Nginx:

```
$ curl -s localhost/health
{"status":"ok","uptime_seconds":92.6,"version":"2.0.0","timestamp":"2026-11-25T09:52:31.676074"}
$ curl -s -X POST localhost/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
$ docker compose ps
production-agent-1    Up (healthy)   8000/tcp
production-nginx-1    Up             0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
production-qdrant-1   Up (healthy)   6333-6334/tcp
production-redis-1    Up (healthy)   6379/tcp
```

**Chỉnh sửa vì môi trường:** Docker Desktop không mount được `nginx/nginx.conf` từ Debian-Lab (lỗi `not a directory`, cả khi chép sang `/mnt/d`). Mình chạy stack trong bản sao `/mnt/d/lab14`, thêm `Dockerfile.nginx` (`FROM nginx:alpine` + `COPY nginx/nginx.conf`) và đổi service nginx sang `build`, bỏ dòng mount. Chỉ sửa trong bản sao, repo gốc không đổi phần này.

## Part 3: Cloud Deployment

**Không làm phần này.** Mình không deploy `06-lab-complete` lên Railway hay Render nên không có URL công khai, ảnh `dashboard.png`/`running.png`/`test.png` và `DEPLOYMENT.md`. Bản `06-lab-complete` vẫn có sẵn `railway.toml` và `render.yaml`; nó đã chạy đủ (PASS 12/12 ở `check_production_ready.py`) bằng Docker Compose trên máy local, xem Part 6.

## Part 4: API Security

### Exercise 4.1: API key (`04-api-gateway/develop/app.py`)

- **Key được kiểm tra ở đâu:** hàm `verify_api_key` (dòng 39), dùng làm dependency của `/ask` (`Depends(verify_api_key)`). Key đúng đọc từ biến môi trường `AGENT_API_KEY` (dòng 35).
- **Sai key:** không có header trả 401, có header nhưng sai trả 403.
- **Rotate key:** đổi giá trị `AGENT_API_KEY` trên platform rồi redeploy; key cũ mất hiệu lực ngay vì không còn nằm trong code.

Kết quả thật (chạy `AGENT_API_KEY=my-secret-key python app.py`, bản này nhận JSON body):

```
$ curl -si -X POST "localhost:8000/ask?question=hello" | head -1
HTTP/1.1 401 Unauthorized
$ curl -si -H "X-API-Key: sai" -X POST "localhost:8000/ask?question=hello" | head -1
HTTP/1.1 403 Forbidden
$ curl -si -H "X-API-Key: my-secret-key" -X POST localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"hello"}' | sed -n '1p;$p'
HTTP/1.1 200 OK
{"question":"hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

Ghi chú: lần gọi đầu có key đúng nhưng dùng query string cho 500, vì code đọc JSON body; gửi JSON thì 200.

### Exercise 4.2: JWT (`04-api-gateway/production`)

Lấy token bằng `POST /auth/token` với user `student`/`demo123` (token đã cắt ngắn, che phần sau):

```
$ curl -s -X POST localhost:8000/auth/token -H "Content-Type: application/json" -d '{"username":"student","password":"demo123"}' | cut -c1-120
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3OTU2MDA3MDgsIm...
```

Luồng: đăng nhập, nhận token (payload có `sub` = username và `role`), gửi lại trong header `Authorization: Bearer <token>`; `verify_token` giải mã và cấp quyền theo role. Token hết hạn sau 60 phút. Không có token, `/ask` trả 401 (xem bên dưới).

### Exercise 4.3: Rate limiting (`production/rate_limiter.py`)

- **Thuật toán:** sliding window (mỗi user có một `deque` các mốc thời gian, loại bỏ mốc cũ hơn cửa sổ).
- **Giới hạn:** 10 request mỗi 60 giây cho user thường (`max_requests=10, window_seconds=60`).
- **Admin:** `app.py` chọn `rate_limiter_admin` thay vì `rate_limiter_user` khi `role == "admin"`, nên admin có hạn mức riêng cao hơn.

Output thật (che token), gọi liên tiếp 13 lần:

```
$ curl -si localhost:8000/ask -X POST -H "Content-Type: application/json" -d '{"question":"hi"}' | head -1
HTTP/1.1 401 Unauthorized
Req  1: 200
...
Req 10: 200
Req 11: 429
Req 12: 429
Req 13: 429
$ curl -si -X POST localhost:8000/ask -H "Authorization: Bearer ***" ... | sed -n '1p;$p'
HTTP/1.1 429 Too Many Requests
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```

### Exercise 4.4: Cost guard (`production/cost_guard.py`)

Cách làm: mỗi user có ngân sách `$1/ngày` (`daily_budget_usd=1.0`), toàn hệ thống `$10/ngày` (`global_daily_budget_usd`). Trước khi gọi LLM, `check_budget(user_id)` kiểm tra: nếu tổng chi của hệ thống hoặc của user đã chạm trần thì trả 402. Sau mỗi lần gọi, `record_usage` cộng chi phí tính theo token (giá GPT-4o-mini: `$0.15/1M` input, `$0.60/1M` output), và cảnh báo khi dùng 80% (`warn_at_pct`). Trạng thái lưu trong bộ nhớ của process, nên với nhiều instance cần chuyển sang Redis (như gợi ý trong đề).

Vì sao cần cả hai: rate limit bảo vệ tốc độ (chống spam), cost guard bảo vệ tiền (một user gọi chậm nhưng prompt rất dài vẫn có thể đốt ngân sách). Mình chưa chạy thử trường hợp 402.

## Part 5: Scaling & Reliability

### Exercise 5.1-5.3: Đọc `05-scaling-reliability/production/app.py`

- **Health check (5.1):** `GET /health` là liveness: process còn sống thì trả 200, kèm `instance_id`, `uptime_seconds` và trạng thái Redis (`ok` hoặc `degraded`). `GET /ready` là readiness: ping Redis, nếu Redis lỗi thì trả 503, tức instance chưa nên nhận traffic. Khác nhau: `/health` trả lời "process còn chạy không" (platform dùng để restart); `/ready` trả lời "nhận request được chưa" (load balancer dùng để route).
- **Graceful shutdown (5.2):** app dùng `lifespan` của FastAPI; khi uvicorn nhận SIGTERM thì chạy phần sau `yield` để ghi log `Instance ... shutting down`, request đang chạy được uvicorn xử lý xong trước khi dừng. (Mình đã thấy log shutdown tương tự ở Part 1 khi gửi SIGTERM; chưa kiểm riêng trường hợp này.)
- **Stateless (5.3):** state không nằm trong RAM của process. Session và lịch sử hội thoại lưu trong Redis với khóa `session:<id>`, TTL 1 giờ, giữ tối đa 20 message. Bản `develop` lưu trong bộ nhớ nên mỗi instance một bản riêng, scale ra nhiều instance sẽ mất ngữ cảnh.

### Exercise 5.4: Stack nhiều instance

Chạy: `docker compose -p k5-day14-scaling up -d --build --scale agent=3` trong `05-scaling-reliability/production/`. Nginx nghe cổng 8080 và chia tải (round-robin) tới 3 instance agent; cả 3 dùng chung một Redis.

**Chỉnh sửa vì môi trường (giống Part 2):**
- Mạng chặn `apt-get`, nên mình xóa 2 dòng `RUN apt-get install gcc libpq-dev` trong `Dockerfile` (requirements chỉ có `fastapi`, `uvicorn`, `redis`, không cần biên dịch).
- Docker Desktop không mount được `nginx.conf` từ Debian-Lab, nên thêm `Dockerfile.nginx` (`FROM nginx:alpine` + `COPY nginx.conf ...`) và đổi service nginx sang `build`, bỏ dòng mount.
- Lần build đầu `pip` bị timeout do mạng, build lại thì qua.

### Exercise 5.5: Test stateless (`python3 test_stateless.py`, output thật)

```
Session ID: 3d2c6780-759b-45a8-8e19-2dee978288a1
Request 1: [instance-bfb655]  Q: What is Docker?
Request 2: [instance-cb55d2]  Q: Why do we need containers?
Request 3: [instance-e7e175]  Q: What is Kubernetes?
Request 4: [instance-bfb655]  Q: How does load balancing work?
Request 5: [instance-cb55d2]  Q: What is Redis used for?
Total requests: 5
Instances used: {'instance-e7e175', 'instance-cb55d2', 'instance-bfb655'}
All requests served despite different instances!
--- Conversation History --- Total messages: 10
Session history preserved across all instances via Redis!
```

Giải thích: 5 request cùng một `session_id` được Nginx chia cho 3 instance khác nhau (bfb655, cb55d2, e7e175), nhưng lịch sử cuối cùng có đủ 10 message (5 câu hỏi và 5 câu trả lời). Vì state nằm trong Redis chứ không nằm trong RAM, instance nào cũng đọc và ghi cùng một session. Nếu lưu trong RAM thì request 2 sang instance khác sẽ không thấy request 1. Mình chưa tắt thử một instance giữa chừng.

## Output `check_production_ready.py`

Chạy đầy đủ (có Docker, trong WSL) tại `06-lab-complete/`: `python3 check_production_ready.py`. Phần "Checklist Deliverable" của output:

```
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 41/41 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
```

Một số dòng kiểm tra chạy thật: rate limit trả 429 sau 5 request thành công (`RATE_LIMIT_PER_MINUTE=5`) và vẫn 429 sau khi restart agent (state nằm trong Redis); cost guard trả 402 ở request đầu khi budget gần 0; `docker stop` thoát với exit code 0 trong 1.3s; 12 dòng log đều là JSON hợp lệ.

Cách làm chính trong `06-lab-complete/`:
- `config.py` đọc cấu hình từ biến môi trường; `.env.example` liệt kê đủ biến, không có secret.
- `auth.py`: so sánh `X-API-Key` bằng `hmac.compare_digest`, thiếu hoặc sai trả 401.
- `rate_limiter.py`: sliding window bằng Redis sorted set (mỗi API key), vượt trả 429 kèm `Retry-After`.
- `cost_guard.py`: cộng dồn chi phí theo ngày trong Redis (`cost:YYYY-MM-DD`), vượt `DAILY_BUDGET_USD` trả 402.
- `/health` luôn 200; `/ready` trả 503 khi Redis không kết nối được.
- `Dockerfile` multi-stage (`python:3.11-slim`), user non-root, `HEALTHCHECK`, `exec uvicorn` để là PID 1 nhận SIGTERM, nghe theo `PORT`.

## AI log

- **Part 1 (Exercise 1.1):** hỏi Claude đọc `develop/app.py` và liệt kê anti-pattern. Nhận danh sách 9 mục. Mình tự đối chiếu từng mục với đoạn code tương ứng trong file.
- **Môi trường:** nhờ Claude hướng dẫn dựng WSL2 (Debian từ image Docker Python 3.11) và bật Docker trong WSL. Mình tự chạy lệnh và kiểm tra bằng `python3 --version`, `docker version`, `docker compose version`, `docker run --rm hello-world`.
- **Exercise 1.2, 1.3:** lệnh chạy do Claude gợi ý; mình tự chạy, kết quả ở trên là output thật.
- **Part 4, 5:** Claude gợi ý lệnh kiểm tra (401/403/200, vòng lặp 429, test stateless); mình tự chạy và dán output thật. Một số Dockerfile gọi `apt-get` bị mạng chặn nên mình bỏ các dòng đó vì dependency toàn là Python thuần.
- **Part 6:** Claude viết mã cho `06-lab-complete` (config, auth, rate limit, cost guard, health/ready, Dockerfile nhiều tầng). Mình tự build, chạy `check_production_ready.py` và dán kết quả PASS 12/12.
- **Part 7:** Claude hướng dẫn chạy load test, eval, dựng stack Prometheus/Grafana/Jaeger/Locust và các thí nghiệm alert; mình tự chạy và chụp màn hình. Mọi số trong `BENCHMARK_REPORT.md` lấy từ `reports/*.json` và `benchmark-results/*.csv`. Lần đo scale đầu bị sai (chỉ chạy 2 agent) nên mình chạy lại. Lỗi `ModuleNotFoundError: uvicorn` do thư mục home của user trong container được sửa bằng `PYTHONPATH`.
