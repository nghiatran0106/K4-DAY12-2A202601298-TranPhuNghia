# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng gợi ý (in nghiêng, ngay dưới mỗi câu hỏi) bằng
> câu trả lời của bạn. `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tran Phu Nghia  Mã học viên: 2A202601298

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway lần đầu, tôi tạo service trước rồi mới nhớ ra phải
> set `API_TOKEN` — nếu quên set hẳn, container không khởi động được:
> `pydantic_settings` raise `ValidationError` ngay từ `Settings()`, log báo rõ
> field nào thiếu, và Railway đánh dấu deploy fail. Tôi biết ngay và vào set
> biến môi trường trước khi có traffic nào lọt vào.
> Nếu `api_token` có mặc định `"changeme"`, app vẫn "khởi động thành công",
> health check xanh, deploy trông như ổn — nhưng `/chat` gọi bằng
> `Authorization: Bearer changeme` sẽ luôn qua được, vì đây là chuỗi ai cũng
> đoán được (nó nằm sẵn trong code mẫu). Tôi sẽ không phát hiện ra cho tới khi
> nhận hóa đơn LLM tăng bất thường và phải lục log mới biết có client lạ gọi
> bằng đúng token mặc định — "chết sớm" biến lỗi cấu hình thành một lần deploy
> fail rõ ràng, thay vì một lỗ hổng bảo mật âm thầm chảy tiền.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Chạy `uvicorn app.main:app` cục bộ (Redis giả `fake://`) rồi gọi `/chat` hai
> lần, thu được dòng log thật:
>
> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:46:10.716046+00:00", "client_id": "demo", "prompt_tokens": 2, "completion_tokens": 34, "usd_cost": 2.07e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc/truy vấn theo trường** — vì đây là JSON có key cố định, tôi có thể
>    dùng `jq 'select(.usd_cost > 0.0001)'` hoặc query trên Cloud Logging/Datadog
>    để tìm tất cả request của một `client_id`, hoặc tổng `usd_cost` trong ngày.
>    `print` chỉ là chuỗi tự do, muốn lấy số liệu phải viết regex đoán mò và dễ
>    vỡ mỗi khi câu chữ thay đổi.
> 2. **Dựng cảnh báo/biểu đồ tự động** — vì `severity` được viết hoa
>    (`"INFO"`) đúng key mà Google Cloud Logging/Datadog hiểu để tô màu, lọc
>    theo mức độ, và `ts` là ISO-8601 chuẩn nên hệ thống log ghép được đúng
>    thời gian request thật (khác giờ log của chính process ghi ra). Từ đó có
>    thể dựng alert "usd_cost trung bình tăng đột biến" hay dashboard theo
>    `client_id`. `print("đã trả lời xong")` không có timestamp máy đọc được,
>    không có severity, không tách được field nào để tổng hợp.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1190 MB |
| Multi-stage | 183 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Build thật hai bản (`docker images` sau khi build): bản 1 stage dùng
> `python:3.11` đầy đủ (~1.19GB), bản multi-stage dùng `python:3.11-slim` ở cả
> hai giai đoạn và chỉ copy `/install` sang stage runtime (~183MB) — chênh
> lệch khoảng **1GB**.
> Phần chênh lệch đó là: (1) `python:3.11` đầy đủ mang theo toàn bộ toolchain
> build C (gcc, make, các dev header...) mà bản slim không có — cần để biên
> dịch một số gói Python có phần mở rộng C, nhưng runtime không dùng tới sau
> khi `pip install` xong; (2) bản 1 stage giữ luôn cache `pip`, file
> `.dockerignore`-loại (vì `COPY . .` copy nguyên thư mục repo bao gồm cả
> `.git`, `.venv` nếu có) và toàn bộ apt lists của base image; (3) multi-stage
> chỉ `COPY --from=builder /install /usr/local` — tức chỉ mang theo *kết quả*
> cài đặt (site-packages đã build xong), không mang theo compiler hay cache đã
> dùng để tạo ra kết quả đó.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sửa `SERVICE_VERSION = "1.0.0"` thành `"1.0.1"` rồi build lại (đã xoá sạch
> layer cache trước để đo cho sạch). Kết quả thật:
> - **Dùng lại cache**: `FROM ... AS builder`, `WORKDIR`, `COPY requirements.txt .`,
>   `RUN pip install ...` (stage builder), và `FROM ... AS runtime`,
>   `WORKDIR`, `COPY --from=builder /install /usr/local` — tức toàn bộ phần
>   cài dependency, vì `requirements.txt` không đổi.
> - **Chạy lại**: từ `COPY app ./app` trở đi — `COPY utils ./utils`,
>   `RUN useradd`, `USER`, `EXPOSE`, `HEALTHCHECK`, `CMD` — tất cả các layer
>   *sau* layer đổi nội dung đều rebuild, kể cả `COPY utils ./utils` dù thư
>   mục `utils` không hề đổi. Lý do: cache Docker theo chuỗi — mỗi layer cache
>   theo cặp (layer cha, lệnh, nội dung); layer `COPY app ./app` đổi làm layer
>   cha của mọi bước sau đó đổi theo, nên toàn bộ đuôi chuỗi phải build lại dù
>   input của riêng chúng không đổi.
> - Nếu đặt `COPY . .` lên **trước** `RUN pip install` (giống bản 1-stage gốc):
>   sửa một ký tự bất kỳ trong code cũng làm `COPY . .` cache-miss, và vì nó
>   đứng trước `RUN pip install`, `pip install` — layer cha của nó đã đổi —
>   cũng phải chạy lại toàn bộ, tức cài lại **tất cả** dependency (FastAPI,
>   uvicorn, pydantic...) mỗi lần sửa 1 dòng code, dù `requirements.txt`
>   không hề thay đổi. Build đó chậm hẳn (phải tải + cài lại cả chục gói) thay
>   vì chỉ mất vài giây copy lại `app/`.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) một lỗ hổng trong code hoặc trong một dependency Python
> (deserialize dữ liệu không tin cậy, RCE qua một gói bị dính CVE...) cho
> attacker chạy lệnh tuỳ ý bên trong container; (2) nếu process đang chạy
> bằng root (UID 0, hành vi mặc định của image không có `USER`), lệnh đó chạy
> với toàn quyền *bên trong* container — ghi đè bất kỳ file nào, cài
> backdoor, đọc mọi secret nạp vào container; (3) nếu container đó còn được
> chạy với quyền mở rộng (privileged, mount `/var/run/docker.sock`, hoặc kernel
> host có lỗ hổng escape container) thì root-trong-container là bước đệm để
> leo thang thành root-trên-host, vì ranh giới cô lập container dựa trên
> namespace/cgroup của Linux, không phải một bức tường tuyệt đối như VM.
> Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi ngay ở bước (2): dù lỗ hổng ở
> bước (1) vẫn tồn tại và attacker vẫn chạy được lệnh, lệnh đó chạy với quyền
> một user thường — không ghi được vào hầu hết filesystem của image (đã
> `COPY` với owner root), không tự ý cài package hệ thống, và mất đi đúng thứ
> nhiều kỹ thuật escape container dựa vào (UID 0 bên trong khớp với UID 0 của
> host). Không loại bỏ hoàn toàn rủi ro, nhưng thu hẹp đáng kể "quyền" mà kẻ
> tấn công có được ngay sau khi khai thác thành công.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate` là bắt buộc theo chuẩn HTTP (RFC 7235) cho mọi response
> 401 — nó là phần của "hợp đồng" giao thức, nói cho client (và các thư viện
> HTTP tổng quát, không riêng gì client của service này) biết phải xác thực
> bằng scheme nào để thử lại đúng cách, thay vì client phải đoán hoặc đọc tài
> liệu riêng của từng API.
> Trả cùng một thông báo cho cả ba trường hợp (thiếu header, sai scheme, sai
> token) là để không biến response lỗi thành một "oracle" cho kẻ đang dò token.
> Nếu 3 trường hợp trả 3 thông báo khác nhau, kẻ tấn công thử token ngẫu nhiên
> sẽ biết được khi nào mình đã "đúng định dạng nhưng sai giá trị" (tức đang đi
> đúng hướng) so với khi "sai hoàn toàn cách gửi" — thông tin đó thu hẹp không
> gian dò tìm. Một thông báo duy nhất khiến brute-force token khó hơn vì không
> có tín hiệu phản hồi nào để tối ưu chiến lược dò, giống lý do
> `secrets.compare_digest` được dùng thay vì `==` — cả hai đều nhằm không rò
> rỉ thông tin qua kênh phụ (response content, thời gian xử lý).

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Test thật bằng cách bắn 12 request liên tiếp vào `/chat` (bucket đầy sẵn vì
> client mới): kết quả là `200 200 200 200 200 200 200 200 200 200 429 429` —
> đúng **10 request** trước khi bị 429, khớp `capacity=10`.
> Nếu bỏ `min(self.capacity, tokens)` trong `available()`: với client đã tiêu
> hết bucket (về 0) rồi im lặng 10 phút, số token tính được sẽ là
> `0 + 10 phút × (10 token/phút) = 100`, tức gửi được tới **100 request**
> trước khi hết token — gấp 10 lần capacity. Lý do: `refill_per_second` vẫn
> cộng dồn tuyến tính theo thời gian trôi qua kể từ lần cập nhật cuối
> (`(now - last) * refill_per_second`), không có gì chặn phần cộng dồn đó lại
> ở mức `capacity` nếu thiếu `min(...)`. Cái xô về bản chất không còn "sức
> chứa" nữa — im lặng càng lâu thì càng có thể xả một burst càng lớn, phá vỡ
> đúng mục đích của rate limiting (giới hạn tốc độ tối đa client có thể tiêu
> thụ tài nguyên, bất kể đã im lặng bao lâu).

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hai hạn mức có cùng tổng danh nghĩa (~$30/tháng ≈ 30 × $1/ngày) nhưng khác
> nhau ở "bán kính nổ" của một sự cố đơn lẻ. Với hạn mức **$30/tháng**: nếu sự
> cố bắt đầu lúc 2h sáng ngày đầu tháng và client còn nguyên $30 chưa tiêu,
> thiệt hại tối đa của riêng đêm đó có thể lên tới **$30** — toàn bộ ngân sách
> cả tháng cháy trong một đêm nếu không ai canh. Và vì `CostGuard` chỉ khoá
> theo `budget` cố định không gắn với "ngày", một khi hết ngân sách, client bị
> chặn (402) cho tới hết tháng — không có cơ chế tự làm mới giữa chừng, phải
> có người can thiệp hoặc đợi sang tháng mới.
> Với hạn mức **$1/ngày** (đúng cách lab implement, key là
> `spend:{client_id}:{YYYY-MM-DD}` theo UTC): thiệt hại tối đa của cùng sự cố
> đó chỉ là **$1** — đúng 1/30 so với phương án tháng. Vì `CostGuard.today()`
> tính theo ngày UTC, sang 00:00 UTC hôm sau là một key Redis mới hoàn toàn
> (`spent()` trả 0 cho ngày mới), nên service **tự hồi phục** mà không cần ai
> reset thủ công — client lại gọi được bình thường từ sáng hôm sau, chỉ mất
> đúng $1 của đêm sự cố.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/healthz` (dùng làm liveness probe, quyết định "có restart container
> không") cũng gọi `store.ping()` như `/readyz`, thứ tự sự kiện khi Redis rớt
> 30 giây là:
> 1. **t=0s** — Redis ngừng trả lời. Cả 3 container cùng dùng chung một Redis
>    nên cả 3 cùng bị ảnh hưởng đồng thời, không phải lần lượt.
> 2. **t=0s → t≈90s** — orchestrator gọi `/healthz` theo chu kỳ
>    (`HEALTHCHECK --interval=30s --retries=3` trong Dockerfile); mỗi lần gọi
>    `store.ping()` timeout/lỗi → endpoint trả 503 "not ready". Sau 3 lần liên
>    tiếp thất bại (~90s), Docker đánh dấu cả 3 container "unhealthy" gần như
>    cùng lúc.
> 3. **t≈90s** — orchestrator restart cả 3 container cùng lúc vì cùng bị đánh
>    dấu unhealthy cùng thời điểm — toàn bộ cụm mất khả năng phục vụ tạm thời,
>    dù bản thân process Python không hề crash, chỉ là dependency chập chờn.
> 4. **t=30s** — (song song với bước 2) Redis đã hồi phục thật, nhưng vòng
>    lặp restart ở bước 3 đã hoặc đang được kích hoạt do độ trễ tích luỹ từ
>    các lần retry trước đó, nên container vẫn bị khởi động lại dù root cause
>    đã hết.
> 5. Container khởi động lại, kết nối Redis (giờ đã sống) thành công, health
>    check xanh trở lại — nhưng khách hàng đã chịu một đợt gián đoạn hoàn toàn
>    (toàn bộ cụm restart) chỉ vì một lần Redis chập chờn 30 giây.
> Nếu giữ tách biệt như hiện tại: `/readyz` fail trong 30 giây đó khiến load
> balancer ngừng đẩy traffic mới vào (503), nhưng `/healthz` vẫn 200 vì không
> đụng Redis → không container nào bị restart, request đang xử lý dở vẫn chạy
> tiếp, và ngay khi Redis sống lại `/readyz` tự xanh lại — không có outage,
> không có restart storm.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi chạy `railway login` / `railway.app/activate` để deploy lần đầu, lệnh
> liên tục trả về lỗi rate limit — HTTP 429 kèm trang lỗi Cloudflare 1015 —
> chứ không phải lỗi từ app hay Dockerfile của tôi.
> Tìm nguyên nhân: kiểm tra lại Dockerfile/app chạy tốt cục bộ
> (`docker compose up` thành công, `/healthz` `/readyz` trả 200 bình thường),
> thử mạng khác và các trang khác vẫn vào được bình thường, và thông báo lỗi
> nêu rõ tên "Cloudflare error 1015" — đây là rate limit ở tầng CDN của
> railway.app, không liên quan tới code hay cấu hình của tôi, chỉ là tạm thời
> bị chặn do gọi CLI quá nhiều trong thời gian ngắn lúc thử nghiệm.
> Cách xử lý: để không bị kẹt lại, tôi dùng phương án dự phòng tạm thời
> (`LOCAL_FALLBACK=true` + `docker compose`) để vẫn kiểm chứng được CP1–CP4
> chạy đúng trong lúc chờ. Sau khi rate limit hết hiệu lực (đợi một khoảng
> thời gian), đăng nhập lại `railway login` thành công, chạy `railway up` và
> nối Redis add-on trong cùng project — deploy thật thành công tại
> `https://agent-production-f566.up.railway.app`. Đã cập nhật
> `LOCAL_FALLBACK=false` trong `.env` và kết quả `curl` trong `DEPLOYMENT.md`
> hiện tại là từ bản deploy thật trên Railway, không còn là fallback local.
