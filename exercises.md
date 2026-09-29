# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder (in nghiêng, bắt đầu bằng "Câu trả lời
> của bạn") ở mỗi câu bằng nội dung trả lời thật.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tiến Lương  Mã học viên: 2A202602378

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống thật của tôi: khi deploy CP5 lên Railway, tôi tạo service `agent`
> bằng CLI trước rồi mới `railway variables --set` sau. Nếu `Settings` có
> `agent_api_key: str = "changeme"` thì trong khoảng thời gian đó (giữa lúc
> container khởi động và lúc tôi kịp set biến), app vẫn chạy bình thường, `/ask`
> vẫn trả 200 cho bất kỳ ai gõ đúng chuỗi `"changeme"` — một public URL chấp
> nhận khóa mặc định mà ai đọc source code trên GitHub cũng biết. Vì tôi khai
> báo `agent_api_key: str` không mặc định, thử thật:
> ```
> >>> Settings(_env_file=None)
> ValidationError: 1 validation error for Settings
> agent_api_key
>   Field required [type=missing, input_value={}, input_type=dict]
> ```
> App crash ngay lúc khởi động, container không bao giờ vào trạng thái "Running"
> để nhận traffic. Tôi biết ngay khi xem `railway logs`, thay vì biết qua hóa
> đơn cuối tháng hoặc qua việc ai đó đã gọi `/ask` miễn phí bằng khóa đoán được.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy được khi gọi `/ask` với câu hỏi "Docker la gi?":
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T16:07:01.277384+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
> ```
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc và tổng hợp theo trường.** Vì log là JSON có key rõ ràng, tôi có thể
>    chạy `jq 'select(.event=="ask_completed") | .cost_usd' logs.json | paste -sd+ | bc`
>    để cộng dồn `cost_usd` của tất cả các lượt gọi trong ngày và biết user nào
>    tốn tiền nhất bằng cách group theo `user_id`. `print` chỉ ra một chuỗi tự do,
>    muốn lấy con số ra phải viết regex đoán mò và dễ vỡ khi câu chữ đổi.
> 2. **Đặt cảnh báo (alerting) theo điều kiện số.** Vì `level` là một field tách
>    biệt, tôi có thể cấu hình cảnh báo "báo động khi có >5 dòng `level:error`
>    trong 5 phút" trên Datadog/Grafana. `print("đã trả lời xong")` không có khái
>    niệm mức độ nghiêm trọng, hệ thống giám sát không phân biệt được đây là log
>    bình thường hay một lỗi cần đánh thức người trực.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu, `python:3.11`) | 1730 MB |
| Multi-stage (`python:3.11-slim`) | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Build thật cả hai bằng `docker build -f Dockerfile.single -t day12-agent:single .`
> và `docker build -t day12-agent:multi .`, kết quả `docker images`: bản 1 stage
> **1.73GB**, bản multi-stage **271MB** — giảm khoảng 6.4 lần.
>
> Phần chênh lệch (~1.46GB) đến từ hai nguồn:
> 1. **Base image đầy đủ vs slim.** `python:3.11` mang theo toàn bộ Debian với
>    build tools, compiler, header files, docs, locale... để có thể biên dịch
>    bất kỳ package C-extension nào ngay trong image cuối. `python:3.11-slim`
>    bỏ hết phần đó, chỉ giữ runtime Python tối thiểu.
> 2. **Build tool không bị mang theo.** Bản 1 stage `RUN pip install` ngay trên
>    image cuối, nên pip cache, wheel tạm, và mọi công cụ build (nếu package nào
>    cần compile) ở lại vĩnh viễn trong layer đó. Bản multi-stage cài dependency
>    ở stage `builder` (dùng xong bị vứt), rồi chỉ `COPY --from=builder /install
>    /usr/local` — mang đúng các file `.py`/`.so` đã cài xong sang, không mang
>    theo pip cache hay compiler.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Thêm một dòng comment vào cuối `app/main.py` rồi `docker build` lại, output
> thật:
> ```
> [builder 3/4] COPY requirements.txt .                 CACHED
> [builder 4/4] RUN pip install --no-cache-dir ...       CACHED
> [runtime 3/6] COPY --from=builder /install /usr/local  CACHED
> [runtime 4/6] COPY app ./app                           (chạy lại — app/ đổi)
> [runtime 5/6] COPY utils ./utils                       (chạy lại)
> [runtime 6/6] RUN useradd --create-home ...            (chạy lại)
> ```
> Hai layer đầu (`COPY requirements.txt`, `RUN pip install`) và layer copy kết
> quả build từ stage `builder` sang đều dùng lại cache vì nội dung của chúng
> (requirements.txt, thư mục `/install`) không đổi. Ba layer cuối phải chạy lại:
> không chỉ `COPY app ./app` (vì `app/main.py` đổi thật) mà cả `COPY utils ./utils`
> và `RUN useradd` cũng bị hủy cache theo — Docker cache theo **chuỗi tuyến
> tính**, một layer đổi thì mọi layer đứng sau nó trong cùng stage đều phải
> build lại, dù bản thân `utils/` không hề thay đổi.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: mọi lần sửa dù chỉ một dấu
> phẩy trong code, layer `COPY . .` sẽ đổi (checksum khác) → cache bị hủy từ đó
> trở đi → `RUN pip install` buộc phải chạy lại toàn bộ, tải và cài lại ~30
> package mất khoảng 13 giây (đo thật lúc build multi-stage ban đầu) thay vì 0
> giây nhờ cache. Với vòng lặp sửa code → build → test nhiều lần trong ngày,
> đây là khác biệt giữa build "gần như tức thì" và "luôn mất hàng chục giây".

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện nếu container chạy root: (1) code Python có lỗ hổng, ví dụ một
> thư viện xử lý JSON/pickle không an toàn hoặc một dependency có CVE cho phép
> remote code execution; (2) kẻ tấn công khai thác lỗ hổng đó, process Python
> (đang chạy với UID 0 vì không có `USER`) thực thi lệnh tùy ý của kẻ tấn công
> — và vì process là root bên trong container, lệnh đó cũng chạy với quyền root;
> (3) root bên trong container map trực tiếp tới UID 0 của host trong cấu hình
> Docker mặc định (không dùng user namespace remapping); (4) nếu container có
> bất kỳ điểm hở nào ra host — volume mount, một lỗ hổng escape trong container
> runtime, hay chỉ đơn giản là docker socket bị mount nhầm — kẻ tấn công giờ
> thao túng được với quyền root thật trên máy chủ, không chỉ trong một "hộp cô
> lập".
>
> Lệnh `USER appuser` (tôi dùng `useradd --create-home --uid 10001 appuser` rồi
> `USER appuser`) cắt đứt chuỗi ở bước (2): process Python giờ chạy với UID
> 10001 không có quyền gì đặc biệt. Kẻ tấn công khai thác được lỗ hổng vẫn thực
> thi lệnh được, nhưng lệnh đó bị giới hạn trong quyền của user thường — không
> ghi được vào filesystem hệ thống, không tự ý cài package, và quan trọng nhất:
> nếu có escape ra host thì cũng chỉ mang theo quyền UID 10001, không phải root.
> `USER` không sửa được lỗ hổng ở bước (1), nhưng giảm hẳn mức thiệt hại nếu
> bước (1) xảy ra — đúng nguyên tắc "defense in depth".

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: gửi 10 request lúc
> 10:00:59 (vẫn còn nằm trong "phút 10:00", bộ đếm của phút đó cho phép đủ 10)
> rồi gửi tiếp 10 request lúc 10:01:01 (bộ đếm vừa reset về 0 khi đồng hồ nhảy
> sang phút 10:01, nên lại được phép đủ 10 request mới). Tổng cộng 20 request
> lọt qua trong khoảng thời gian thực tế chỉ 2 giây (từ 10:00:59 đến 10:01:01),
> dù "về mặt sổ sách" mỗi phút đều đúng luật ≤10. Sliding window mà tôi cài
> (`app/rate_limiter.py`, dùng ZSET với `zremrangebyscore` để loại bỏ mọi entry
> cũ hơn 60 giây tính từ thời điểm gọi, không phải từ mốc phút cố định) không
> có kẽ hở này vì cửa sổ luôn trượt theo thời điểm hiện tại — không có ranh
> giới phút cố định để "canh me" gửi request sát hai bên.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm **số lượng request** trong một cửa sổ thời gian (bao nhiêu lần
> gọi), cost guard đếm **số tiền** đã tiêu trong một chu kỳ (tháng). Một request
> luôn tính là 1 với rate limit dù hỏi câu ngắn hay dài, nhưng cost guard thì
> không — hai request có thể tốn số tiền rất khác nhau tùy độ dài câu hỏi và
> lịch sử hội thoại đi kèm (prompt càng dài, `tokens_in` càng lớn).
>
> - **Rate limit cho qua, cost guard phải chặn:** user gửi đúng 1 request/phút
>   (dưới xa hạn mức `RATE_LIMIT_PER_MINUTE=10`), nhưng câu hỏi rất dài kèm lịch
>   sử hội thoại dài (gần `HISTORY_MAX_MESSAGES=20` message) khiến mỗi request
>   tốn 0.5 USD. Sau 20 request rải đều cả tháng (không hề vi phạm rate limit
>   lần nào), tổng chi phí đã chạm `MONTHLY_BUDGET_USD=10.0` — request thứ 21
>   dù cách request trước 1 phút vẫn bị `guard.check()` chặn 402.
> - **Cost guard cho qua, rate limit phải chặn:** user hỏi các câu cực ngắn,
>   không có lịch sử ("hi", "ok", "?"), mỗi request chỉ tốn ~0.00002 USD — với
>   ngân sách 10 USD/tháng thì phải mất hàng trăm nghìn request mới chạm ngân
>   sách. Nhưng nếu user (hoặc một script lỗi) gọi liên tục 15 request trong
>   cùng một phút, `limiter.check()` chặn 429 ngay từ request thứ 11, dù ngân
>   sách tháng gần như chưa hề bị đụng tới.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu gộp `/health` và `/ready` làm một, cả hai đều gọi
> `store.ping()`:
> 1. Redis mất kết nối. Cả 3 container `agent` cùng gọi Redis (vì cùng trỏ tới
>    một `REDIS_URL`), nên cả 3 đều thấy `ping()` thất bại cùng lúc.
> 2. Endpoint gộp trả 503 cho cả 3 container — vì giờ nó vừa đóng vai liveness
>    vừa readiness.
> 3. Orchestrator (Docker/Railway/K8s) đọc 503 từ endpoint mà nó dùng làm
>    **liveness probe**, hiểu nhầm là "process bị treo, cần khởi động lại" —
>    thay vì hiểu đúng là "process vẫn sống, chỉ tạm thời không có dependency".
>    Nó **restart cả 3 container cùng lúc**.
> 4. Trong lúc cả 3 đang restart, không còn container nào đứng phục vụ request
>    — kể cả các request không cần Redis (ví dụ chỉ hỏi mà không cần lưu lịch
>    sử) cũng bị từ chối, vì toàn bộ cụm đang khởi động lại.
> 5. 30 giây sau, Redis khôi phục. Nhưng nếu các container vẫn đang trong chu kỳ
>    khởi động lại (container khởi động, health check, sẵn sàng nhận traffic
>    mất vài giây mỗi lần) thì có một khoảng trống ngắn ngay cả sau khi Redis
>    đã sống lại, chưa container nào kịp quay về phục vụ.
>
> Nếu tách riêng (`/health` không đụng Redis, chỉ `/ready` đụng): ở bước 2-3,
> `/health` vẫn trả 200 (process vẫn sống, đúng sự thật) nên orchestrator không
> restart gì cả. Chỉ `/ready` báo 503, load balancer rút 3 container này khỏi
> vòng xoay nhận traffic mới **mà không giết chúng** — Redis vừa sống lại,
> `/ready` lập tức trả 200 trở lại và LB đẩy traffic vào ngay, không phải chờ
> chu kỳ khởi động lại nào cả. Một sự cố Redis 30 giây chỉ gây gián đoạn 30
> giây, không kéo dài thêm vì restart không cần thiết.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Quan sát thật (chạy 1 instance qua Docker, gọi 3 lần liên tiếp với cùng
> `X-User-Id: sv01`): `history_length` tăng dần đều đặn **0 → 2 → 4** (mỗi lượt
> `/ask` append cả câu hỏi lẫn câu trả lời, tức +2). Với `--scale agent=3`, ba
> container đứng sau load balancer round-robin request, nhưng vì `store.get_history`
> và `store.append` đều đọc/ghi cùng một Redis List `history:sv01` — dùng chung
> giữa cả 3 container — nên dù request lượt 1 rơi vào container A, lượt 2 rơi
> vào container B, `history_length` vẫn tăng liền mạch 0, 2, 4, 6... không hề
> phụ thuộc container nào xử lý.
>
> Nếu lịch sử nằm trong một `dict` Python trong RAM thay vì Redis: mỗi container
> có một dict riêng, không container nào nhìn thấy dict của container khác.
> `history_length` sẽ **không tăng đều** mà nhảy lộn xộn theo container nào vừa
> xử lý request đó — ví dụ 0 (rơi vào A lần đầu), 0 (rơi vào B, B chưa từng thấy
> user này), 2 (rơi lại A, A nhớ lượt trước), 0 (rơi vào C)... Agent trông như
> bị "mất trí nhớ" ngẫu nhiên tùy request đi vào container nào — đúng vấn đề mà
> CP4 giải quyết bằng cách chuyển state ra khỏi process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi service đã deploy thành công lên Railway (`curl` từ terminal gọi
> `/health` trả 200 bình thường), chạy `pytest tests/test_cp5.py` lại báo lỗi
> ở đúng những test gọi ra URL thật:
> ```
> ConnectError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed:
> unable to get local issuer certificate (_ssl.c:1006)
> ```
> Lạ ở chỗ `curl -i <URL>/health` chạy được nhưng Python (`httpx`) lại không —
> cùng một URL, hai công cụ khác kết quả. Tôi tìm nguyên nhân bằng cách kiểm tra
> chuỗi chứng chỉ TLS thật mà server trả về:
> `openssl s_client -connect <domain>:443 -servername <domain>`, và thấy
> issuer không phải Let's Encrypt hay CA công khai nào, mà là
> `CN=Norton Web/Mail Shield Root, O=Norton Web/Mail Shield`. Norton Antivirus
> trên máy tôi đang chặn HTTPS scanning (man-in-the-middle hợp pháp cho mục đích
> quét virus): mọi kết nối TLS ra ngoài bị Norton chặn lại và ký lại bằng chứng
> chỉ gốc riêng của nó. `curl` trên Windows tin được vì nó dùng kho chứng chỉ hệ
> điều hành (nơi Norton đã tự cài root cert của mình lúc cài đặt), còn Python
> dùng bundle `certifi` đóng gói riêng — không biết gì về Norton.
>
> Cách sửa: export chứng chỉ gốc của Norton từ Windows Certificate Store
> (`Get-ChildItem Cert:\LocalMachine\Root | Where Subject -like "*Norton*"`),
> chuyển sang PEM bằng `openssl x509`, rồi append vào file
> `.venv/Lib/site-packages/certifi/cacert.pem`. Sau đó `httpx`/`pytest` tin được
> chứng chỉ do Norton ký lại, và toàn bộ `tests/test_cp5.py` pass (9/9). Đây là
> lỗi của môi trường máy cá nhân, không phải lỗi trong code hay cấu hình deploy
> — nhưng nó dạy tôi một bài học thực tế: "curl chạy được" và "app của tôi chạy
> được" không phải lúc nào cũng là một, vì mỗi công cụ HTTP có thể tin một tập
> chứng chỉ khác nhau.
