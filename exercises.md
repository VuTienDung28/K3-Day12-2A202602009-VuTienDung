# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu trả lời mẫu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Tiến Dũng  Mã học viên: 2A202602009

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên Cloud (như Render/Railway), nếu kỹ sư quên thiết lập biến môi trường `AGENT_API_KEY`:
- Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường. Bạn tưởng rằng hệ thống đã hoạt động an toàn, nhưng thực tế bất kỳ ai trên Internet cũng có thể sử dụng API Key mặc định `"changeme"` để gửi request gọi LLM, gây tiêu tốn toàn bộ ngân sách LLM của bạn mà bạn không hề hay biết cho tới khi nhận hóa đơn.
- Ngược lại, khi không có giá trị mặc định (Fail-fast), ứng dụng sẽ ném ngay lỗi `ValidationError` và crash ngay lúc khởi động. Log crash trên Dashboard giúp bạn phát hiện việc thiếu cấu hình bí mật ngay lập tức và khắc phục trước khi công khai API.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T10:30:00.123456+00:00", "user_id": "sv01", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00015}`

Hai việc làm được với log JSON:
1. **Tự động hóa và tạo Dashboard (Machine-Readable)**: Các hệ thống tập trung log như Datadog, Grafana Loki hay AWS CloudWatch có thể tự động parse các thuộc tính trong JSON (như `cost_usd`, `user_id`, `tokens_out`) để tạo biểu đồ theo dõi chi phí và lượng token theo thời gian thực mà không cần viết các chuỗi Regex phức tạp.
2. **Truy vấn và lọc dữ liệu chính xác (Searchability & Alerting)**: Có thể dễ dàng truy vấn, lọc log theo các trường cụ thể (ví dụ: tìm tất cả log của `user_id = "sv01"` hoặc cảnh báo khi có log `level = "error"`), trong khi chuỗi `print` thô chỉ là văn bản không có cấu trúc.

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
| 1 stage (bản đầu) | ~850 MB |
| Multi-stage | ~150 MB |

Giải thích: Phần dung lượng chênh lệch (~700 MB) bao gồm bộ biên dịch C/C++ (GCC, g++), các file header phục vụ biên dịch, công cụ build, bộ nhớ cache cài đặt của pip (`~/.cache/pip`), và các gói hệ thống không cần thiết trong môi trường runtime ở Stage 2.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile chuẩn: Layer ở stage `builder` và layer `RUN pip install` được tận dụng 100% từ cache vì file `requirements.txt` không thay đổi. Docker chỉ phải chạy lại layer `COPY app/ app/` và các layer tiếp theo, giúp quá trình rebuild hoàn thành chỉ trong vài giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa 1 ký tự trong mã nguồn, hash của layer `COPY . .` thay đổi khiến toàn bộ cache của các lệnh phía sau bị vô hiệu hóa. Docker sẽ buộc phải chạy lại lệnh `RUN pip install` từ đầu, làm tốn thời gian rebuild vô ích.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Code Python tồn tại lỗ hổng (ví dụ: RCE - Remote Code Execution hoặc Command Injection qua `os.system`).
2. Kẻ tấn công gửi request chứa payload độc hại $\rightarrow$ thực thi mã lệnh từ xa $\rightarrow$ vì container chạy bằng root nên kẻ tấn công có quyền root trong container.
3. Kẻ tấn công khai thác kẽ hở kernel Linux (Container Escape) từ quyền root trong container để chiếm luôn quyền điều khiển root trên máy chủ vật lý (Host OS).

Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay ở bước 2: Kẻ tấn công chỉ có quyền của một user thường bị cô lập trong container, không thể cài phần mềm độc hại, không thể sửa file hệ thống và không thể thoát rào ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được: 
- Người dùng gửi 10 request vào giây `10:00:59` (cuối phút thứ nhất).
- Đến giây `10:01:00`, đồng hồ bước sang phút mới và bộ đếm reset về 0.
- Người dùng tiếp tục gửi 10 request vào giây `10:01:01` (đầu phút thứ hai).
Tổng cộng trong khoảng thời gian 2 giây (từ `10:00:59` đến `10:01:01`), người dùng đã gửi 20 request thành công vì bộ đếm theo phút cố định tính 2 khoảng thời gian này độc lập.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt:
- Rate Limiter giới hạn về **tần suất / số lượng request** trong khoảng thời gian ngắn (ví dụ: tối đa 10 req/phút).
- Cost Guard giới hạn về **tổng chi phí tài chính / số tiền** trong khoảng thời gian dài (ví dụ: tối đa $10.0/tháng).

Tình huống Rate Limit cho qua nhưng Cost Guard chặn:
- User gửi 1 request duy nhất trong phút (rất thong thả, không vi phạm rate limit), nhưng prompt chứa 500,000 token khiến chi phí vượt quá ngân sách tháng $10.0 $\rightarrow$ Cost Guard chặn (HTTP 402 Payment Required).

Tình huống Cost Guard cho qua nhưng Rate Limit chặn:
- User mới tiêu $0.01 (rất xa hạn mức $10.0), nhưng gửi 15 request liên tục trong 5 giây $\rightarrow$ Rate Limiter chặn (HTTP 429 Too Many Requests).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối 30 giây $\rightarrow$ Endpoint gộp trả về mã lỗi 503.
2. Orchestrator (Docker/Kubernetes) gọi endpoint gộp (Liveness Probe), nhận kết quả 503 $\rightarrow$ lầm tưởng process app bị ngã/treo.
3. Orchestrator lập tức KILL và RESTART toàn bộ 3 container app.
4. Các container khởi động lại nhưng kết nối Redis vẫn chưa hồi phục $\rightarrow$ tiếp tục trả về 503 $\rightarrow$ lại bị restart liên tục (CrashLoopBackOff).
5. Mọi request của user đang xử lý bị ngắt đột ngột và hệ thống rơi vào vòng xoáy restart không cần thiết, làm gián đoạn dịch vụ nghiêm trọng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict RAM của từng process:
Do Load Balancer phân phối ngẫu nhiên các request tới 3 instance (A, B, C):
- Request 1 vào instance A: `history_length` = 0 (A lưu câu 1 vào RAM của A).
- Request 2 vào instance B: `history_length` = 0 (vì RAM của B chưa có câu nào!).
- Request 3 vào instance A: `history_length` = 2 (vì A đã lưu câu 1 trước đó).
- Request 4 vào instance C: `history_length` = 0.
Con số `history_length` sẽ nhảy thất thường (lúc 0, lúc 2, lúc 4...) tùy thuộc vào request rơi vào instance nào, làm agent bị "mất trí nhớ" luân phiên giữa các lượt hỏi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: App không kết nối được tới Redis trên môi trường Docker Compose / Cloud.
Thông báo lỗi: `redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379. Connection refused.`
Nguyên nhân: Trong file `.env` hoặc cấu hình mặc định `REDIS_URL` để là `redis://localhost:6379/0`. Trong môi trường container, `localhost` trỏ về chính bên trong container của app chứ không phải container Redis.
Cách tìm ra & sửa: Kiểm tra log của container app bằng `docker compose logs agent`, phát hiện kết nối tới `localhost:6379` bị từ chối. Sửa `REDIS_URL` trong `docker-compose.yml` thành `redis://redis:6379/0` (dùng tên service `redis` làm hostname trong mạng nội bộ Docker).
