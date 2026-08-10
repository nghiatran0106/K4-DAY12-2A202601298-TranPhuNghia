# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Tran Phu Nghia |
| Mã học viên | 2A202601298 |
| Repo | https://github.com/nghiatran0106/K4-DAY12-2A202601298-TranPhuNghia |

## Service

> Đang dùng **phương án dự phòng** (`LOCAL_FALLBACK=true`) — xem lý do ở cuối
> file. Stack chạy bằng `docker compose` ở máy, chưa có Public URL thật.

| Mục | Nội dung |
|-----|----------|
| Public URL | không có — phương án dự phòng, chạy `http://localhost:8000` |
| Platform | Railway (dự định), hiện tại: Docker Compose local |
| Ngày deploy | 2026-08-10 (local fallback) |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị (áp dụng cho `.env` local
vì chưa deploy được lên Railway):

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | mặc định 8000 |
| `API_TOKEN` | ✅ | trong `.env` local, không nằm trong repo |
| `REDIS_URL` | ✅ | trỏ tới service `redis` trong docker-compose.yml |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây (chạy qua `http://localhost:8000`, phương
án dự phòng):

```
$ docker compose ps
NAME                                             IMAGE                                         SERVICE   STATUS
k3-day12-cloud-services-and-deployment-chat-1    ...-chat                                      chat      Up (healthy)
k3-day12-cloud-services-and-deployment-nginx-1   nginx:alpine                                  nginx     Up
k3-day12-cloud-services-and-deployment-redis-1   redis:7-alpine                                redis     Up (healthy)

$ curl -i http://localhost:8000/healthz
HTTP/1.1 200 OK
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

$ curl -i http://localhost:8000/readyz
HTTP/1.1 200 OK
{"status":"ready","redis":true}

$ curl -i -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" -d '{"message":"Hello"}'
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

$ curl -i -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy la gi?"}'
HTTP/1.1 200 OK
{"reply":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến
môi trường, health check để orchestrator biết trạng thái, và giới hạn tài
nguyên. (Mình đang nhớ 2 lượt trao đổi trước đó.)","client_id":"sv-test",
"turns_before":2,"usd_cost":3.54e-05,"usage":{"prompt":48,"completion":47}}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Không deploy được lên Railway đúng hạn: CLI login (railway login --browserless)
và trang railway.app/activate liên tục trả về lỗi rate limit (HTTP 429 /
Cloudflare error 1015) từ phía Railway/Cloudflare khi thử xác thực, không phải
lỗi phía app hay mạng cục bộ. Đã thử: cài Railway CLI qua npm và qua install
script, đăng nhập browser thường và --browserless, đợi giữa các lần thử — vẫn
bị chặn trong thời gian làm bài. Dùng phương án dự phòng (docker compose) để
nộp đúng hạn; sẽ redeploy lên Railway thật sau khi rate limit hết hạn.
```
