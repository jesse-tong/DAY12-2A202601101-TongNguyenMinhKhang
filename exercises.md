# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tống Nguyễn Minh Khang  Mã học viên: 2A202601101

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> *Tình huống: Deploy dịch vụ Agent AI lên môi trường Production. Nếu quên cấu hình biến môi trường `AGENT_API_KEY` mà ứng dụng để mặc định `"changeme"`, app vẫn khởi động bình thường nhưng mở ra lỗ hổng bảo mật nghiêm trọng cho phép kẻ xấu sử dụng API key mặc định `"changeme"` để gọi endpoint `/ask` và tiêu tốn toàn bộ ngân sách LLM của hệ thống. Việc app "chết sớm" (Fail Fast) ngay lúc khởi động giúp hệ thống CI/CD hoặc container orchestrator phát hiện và ngăn chặn ngay bản deploy lỗi trước khi ứng dụng tiếp nhận traffic thật.*

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> *Dòng log JSON thu được:*
> `{"timestamp": "2026-08-10T10:30:42.123456+00:00", "event": "ask_completed", "service": "day12-agent", "user_id": "sv-demo", "tokens_in": 5, "tokens_out": 25, "cost_usd": 0.00001575}`
>
> *Hai việc làm được với log JSON mà print không làm được:*
> *1. **Truy vấn & Giám sát tự động (Structured Monitoring):** Đẩy log vào hệ thống tập trung (Elasticsearch/Datadog/Loki) để lọc theo `user_id`, tính tổng chi phí `cost_usd` hay số token theo thời gian thực và tự động phát cảnh báo khi có bất thường.*
> *2. **Đối soát & Định danh rõ ràng (Contextual Auditability):** Mỗi dòng log chứa mốc thời gian chuẩn ISO, định danh người dùng (`user_id`), tên service và các thông số định lượng cụ thể, phục vụ việc đối soát tài chính và truy vết lỗi thay vì một câu thông báo chung chung.*

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f Dockerfile-one-stage -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 67.9 MB |
| Multi-stage | 65.8 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> *Phần dung lượng chênh lệch đó bao gồm các build tools, compiler, artifact và file trung gian trong quá trình build, khi dùng multi-stage thì chỉ những file và artifact cần trong quá trình chạy thì mới được copy nên dung lượng giảm so với 1 stage (khoảng 2.1 MB).*

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *Với Docker file này, nếu sửa một ký tự trong `app/main.py` thì layer `COPY . .` phải chạy lại do thư mục workspace `app/` có thay đổi (thay đổi ở file `app/main.py`), các layer `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install --no-cache-dir -r requirements.txt `, `COPY --from=builder /usr/local/lib/python3.11/site-package`, và `RUN useradd -m appuser` được cache lại. Bước `FROM python:3.11-slim` không cache lại.*
> *Nếu đặt `COPY . .` lên trước `RUN pip install` thì chỉ có các bước `FROM python:3.11-slim`, `WORKDIR /app`, và `COPY requirements.txt . ` thì được cache lại, các bước còn lại phải chạy lại.*

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> *1. Chuỗi sự kiện dẫn tới chiếm quyền máy host:*
> *- Bước 1 (Lỗ hổng ứng dụng): Kẻ tấn công khai thác lỗ hổng RCE / Command Injection trong code Python.*
> *- Bước 2 (Chiếm quyền trong Container): Vì container chạy bằng root (UID 0), kẻ tấn công thu được quyền root bên trong container.*
> *- Bước 3 (Container Breakout): Kẻ tấn công lợi dụng lỗ hổng Kernel Linux/Docker runtime hoặc socket mount (như docker.sock) để vượt khỏi ranh giới container.*
> *- Bước 4 (Chiếm toàn bộ máy host): Vì process sở hữu UID 0, khi rò rỉ ra ngoài host nó vẫn khớp với user root trên máy host, cho phép kẻ tấn công chiếm toàn quyền kiểm soát máy chủ.*
>
> *2. Lệnh USER cắt đứt chuỗi ở chỗ nào:*
> *Lệnh `USER appuser` chuyển process sang user thường (non-root). Lệnh này cắt đứt ngay tại **Bước 2**: kẻ tấn công chỉ thu được quyền user thường trong container, không thể sửa file hệ thống và khi rò rỉ ra host ở Bước 3 cũng chỉ mang UID thường (không có quyền root).*

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> *Một người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.*
>
> *Cách đạt được con số đó:*
> *- Giả sử mốc reset đếm là giây 00 của mỗi phút.*
> *- Tại giây `10:00:59` (1 giây cuối phút trước), người dùng gửi liên tiếp 10 request (dùng hết quota phút 10:00).*
> *- Tại giây `10:01:00` (đầu phút mới), counter được reset về 0.*
> *- Tại giây `10:01:01` (1 giây đầu phút mới), người dùng gửi tiếp 10 request nữa (dùng quota phút 10:01).*
> *Tổng cộng trong 2 giây liên tiếp (`10:00:59` đến `10:01:01`), hệ thống chịu đợt bùng nổ 20 request (gấp đôi hạn mức 10 req/phút). Thuật toán cửa sổ trượt (Sliding Window) giải quyết vấn đề này bằng cách luôn đếm tổng request trong 60 giây gần nhất.*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *1. Sự khác biệt:*
> *- Rate Limiter: Kiểm soát số lượng request (tần suất) trong khoảng thời gian ngắn để chống DDoS, Spam và quá tải server.*
> *- Cost Guard: Kiểm soát tổng chi phí tài chính (USD/token) tích lũy trong tháng để bảo vệ ngân sách API LLM.*
>
> *2. Tình huống minh họa:*
> *- **Rate limit cho qua nhưng Cost guard phải chặn:** Người dùng chỉ gửi 1 request trong phút (thỏa mãn < 10 req/phút), nhưng câu hỏi chứa prompt cực lớn khiến chi phí vượt quá ngân sách tháng $10.00 còn lại -> CostGuard chặn và trả về lỗi HTTP 402 (Payment Required).*
> *- **Cost guard cho qua nhưng Rate limit phải chặn:** Người dùng mới tiêu $0.10/$10.00 ngân sách tháng (tiền còn rất nhiều), nhưng lại gửi 20 request ngắn liên tiếp trong 5 giây -> RateLimiter chặn và trả về lỗi HTTP 429 (Too Many Requests).*

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Trong trường hợp /ready khác /health: Redis sập (/ready trả về HTTP 503 Service Unavailable) mà app vẫn sống (/health trả về 200 OK), Orchestrator không restart container và Load Balancer nhận biết tạm ngừng request tới container và restart Redis service lại, phục vụ traffic bình thường. Nếu gộp cả /ready và /health, /health check sẽ báo lỗi khi Redis sập, Docker cho rằng cả 3 container đã treo/chết, nên nó sẽ restart 3 container liên tục trong lúc Redis kết nối lại. Khi Redis kết nối lại, thì có thể không còn container agent nào ở trạng thái sẵn sàng để phục vụ (do đang trong quá trình restart lại từ đầu), dẫn tới toàn bộ hệ thống bị sập hoàn toàn thay vì chỉ tạm ngưng nhận traffic.*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Kết quả chạy /ask nhiều lần, nhận thấy history_length trong response tăng dần theo 2 mỗi request (0, 2, 4, 6, 8).*
> *Nếu lịch sử được lưu trong dict Python thay vì Redis: mỗi container sẽ có một dict Python riêng trong RAM. Do Nginx phân phối các request luân phiên qua 3 container khác nhau, con số `history_length` sẽ thay đổi thất thường và tăng giảm không đều (ví dụ: request 1 vào container A trả 0, request 2 vào container B trả 0, request 3 vào container A mới trả 2). Agent sẽ bị "mất trí nhớ" nếu request tiếp theo bị đẩy sang container khác.*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *- Thông báo lỗi: `Error: Invalid value for '--port': '$PORT' is not a valid integer` làm container crash liên tục.*
> *- Nguyên nhân: Kiểm tra log bằng `docker compose logs agent`, phát hiện CMD trong Dockerfile sử dụng dạng exec array `["uvicorn", "app.main:app", "--port", "$PORT"]` khiến Docker không thông qua shell để eval biến môi trường `$PORT` mà truyền thẳng chuỗi thô `"$PORT"`.*
> *- Cách khắc phục: Sửa CMD trong Dockerfile sang dạng shell wrapper: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` giúp đọc và mở rộng biến `$PORT` thành số cổng chính xác.*
