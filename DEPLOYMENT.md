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

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-f566.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán, không ghi đè |
| `API_TOKEN` | ✅ | set qua `railway variables --set`, không nằm trong repo |
| `REDIS_URL` | ✅ | tham chiếu tới Redis add-on trong cùng project |
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

```
$ curl -i https://agent-production-f566.up.railway.app/healthz
HTTP/2 200
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

$ curl -i https://agent-production-f566.up.railway.app/readyz
HTTP/2 200
{"status":"ready","redis":true}

$ curl -i -X POST https://agent-production-f566.up.railway.app/chat \
  -H "Content-Type: application/json" -d '{"message":"Hello"}'
HTTP/2 401
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

$ curl -i -X POST https://agent-production-f566.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy la gi?"}'
HTTP/2 200
{"reply":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến
môi trường, health check để orchestrator biết trạng thái, và giới hạn tài
nguyên.","client_id":"sv-test","turns_before":0,"usd_cost":2.265e-05,
"usage":{"prompt":3,"completion":37}}

$ for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code} " -X POST \
  https://agent-production-f566.up.railway.app/chat \
  -H "Content-Type: application/json" -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-ratetest" -d '{"message":"test"}'; done
200 200 200 200 200 200 200 200 200 200 429 429 429 429 200
```

(Lần gọi thứ 15 trả 200 thay vì 429: đúng hành vi token bucket — một phần
token đã kịp nạp lại trong lúc chờ các request trước đó qua mạng, khác với
sliding window cứng nhắc luôn chặn đúng request thứ N+1.)

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---

## Ghi Chú Về Phương Án Dự Phòng

Trong lúc làm bài, `railway login` và `railway.app/activate` tạm thời trả về
lỗi rate limit (HTTP 429 / Cloudflare error 1015) không liên quan tới app hay
mạng cục bộ. Đã dùng `LOCAL_FALLBACK=true` + `docker compose` để có một bản
chạy được trong lúc chờ. Rate limit hết sau đó trong buổi làm bài; đã đăng
nhập lại và `railway up` thành công lên Public URL ở trên — `LOCAL_FALLBACK`
đã trả về `false` trong `.env`, kết quả kiểm tra CP5 phía trên là từ bản
deploy thật.
