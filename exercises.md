# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Hoàng Vũ  Mã học viên: 2A202602000

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên môi trường Cloud mà quên khai báo biến `AGENT_API_KEY`:
- Nếu để mặc định `"changeme"`: App vẫn khởi động bình thường. Kẻ tấn công hoặc bot tự động dò quét các key phổ biến (`"changeme"`, `"123456"`) có thể truy cập thành công vào endpoint `/ask` và tiêu tốn toàn bộ hạn mức API LLM của bạn mà bạn không phát hiện ra.
- Nếu "chết sớm" (fail fast): App sẽ văng ra `ValidationError` và ngưng khởi động ngay lập tức. Deploy pipeline báo lỗi Healthcheck, ngăn chặn việc phát hành một ứng dụng hổng bảo mật lên sản phẩm.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T12:00:00.123456+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00015}
```

Hai việc làm được:
1. **Lọc và truy vấn cấu trúc tự động (Structured Filtering)**: Các hệ thống tập trung log (như Datadog, Loki, CloudWatch) có thể tự parse JSON để truy vấn chính xác các log có `cost_usd > 0.01` hoặc lọc log theo từng `user_id`.
2. **Thống kê và phát cảnh báo thời gian thực (Metrics & Alerting)**: Có thể tự động tính tổng chi phí `cost_usd` tiêu tốn theo giờ/người dùng và bật cảnh báo khi chi phí vượt mức mà không cần viết regex để đọc chuỗi văn bản tự do.

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
| 1 stage (bản đầu) | 1010 MB |
| Multi-stage | 170 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Dung lượng chênh lệch (~840 MB) là các công cụ phát triển, build tools (GCC, g++, header files, debian package manager đầy đủ, documentation...) có trong base image `python:3.11` tiêu chuẩn và các file tạm/cache trong quá trình `pip install` ở stage `builder`. Stage runtime cuối cùng chỉ copy lại các file thực thi tối thiểu trên nền `python:3.11-slim`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer `FROM`, `COPY requirements.txt .` và `RUN pip install ...` đều được dùng lại từ cache. Chỉ có layer `COPY app/ app/` và các layer sau đó mới phải chạy lại, giúp rebuild cực kỳ nhanh.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mọi sự thay đổi dù chỉ 1 ký tự trong `app/main.py` sẽ làm vô hiệu hóa cache của layer `COPY . .` và kéo theo layer `RUN pip install` phải chạy lại từ đầu (tải và cài đặt lại toàn bộ thư viện từ Internet), làm lãng phí rất nhiều thời gian build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện:
  1. Code Python có lỗ hổng (ví dụ Command Injection hoặc RCE).
  2. Kẻ tấn công gửi payload độc hại để mở shell code bên trong container.
  3. Do container chạy dưới quyền root (UID 0), kẻ tấn công chiếm toàn bộ quyền root bên trong container.
  4. Kẻ tấn công khai thác tiếp lỗ hổng cgroups/kernel để escape khỏi container và chiếm quyền root trực tiếp trên máy Host OS.
- Lệnh `USER appuser` cắt đứt chuỗi sự kiện ở bước 3: Tiến trình ứng dụng bị ép chạy dưới quyền user thường (non-root UID 1000+). Dù kẻ tấn công khai thác được RCE trong Python, chúng cũng chỉ có quyền user hạn chế trong container, không thể đọc file nhạy cảm hay thực hiện hành vi escape container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được: Người dùng gửi 10 request vào giây 59 của phút thứ 1 (ví dụ 10:00:59). Đến giây 00 của phút thứ 2 (10:01:00), bộ đếm theo phút đồng hồ reset về 0, người dùng lập tức gửi tiếp 10 request nữa ở giây 01 (10:01:01). Trong khoảng thời gian 2 giây từ 10:00:59 đến 10:01:01, user đã gửi tổng cộng 20 request mà không vượt hạn mức của thuật toán đếm theo phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác biệt: **Rate limit** kiểm soát *tần suất/số lượng request* trong cửa sổ thời gian ngắn (ví dụ max 10 req/phút). **Cost guard** kiểm soát *tổng chi phí/tiền bạc* phát sinh trong chu kỳ dài hơn (ví dụ max 10 USD/tháng).
- Rate limit cho qua nhưng Cost guard chặn: User mới gửi 2 request/phút (duy trì tần suất thấp), nhưng mỗi request có prompt cực lớn tiêu tốn hàng trăm ngàn token làm chi phí cộng dồn trong tháng vượt mốc 10 USD. Rate limit thấy 2 req/phút < 10 nên cho qua, nhưng Cost guard kiểm tra vượt ngân sách nên chặn (HTTP 402).
- Cost guard cho qua nhưng Rate limit chặn: User gửi 15 request ngắn liên tiếp trong vòng 3 giây. Chi phí 15 request rất nhỏ (chỉ 0.001 USD, ngân sách tháng vẫn còn dư nên Cost guard cho qua), nhưng tần suất 15 req/phút đã vượt quá 10 req/phút nên Rate limit sẽ chặn lại (HTTP 429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối trong 30 giây.
2. Liveness probe của Orchestrator gọi vào `/health` của 3 container `agent`.
3. Do `/health` kiểm tra kết nối Redis bị lỗi, endpoint trả về lỗi (503/timeout).
4. Orchestrator coi cả 3 container `agent` đều đã chết và ra lệnh **kill/restart** liên tục toàn bộ cụm container (CrashLoopBackOff).
5. Khi Redis phục hồi sau 30s, hệ thống vẫn mất thêm thời gian khởi động lại ứng dụng từ đầu thay vì chỉ đơn thuần tạm ngừng nhận traffic và khôi phục phục vụ ngay lập tức.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Nếu lưu trong Redis (như hiện tại): `history_length` tăng đều liên tục (1, 2, 3, 4...) dù request được Load Balancer gửi tới container nào.
- Nếu lưu trong dict Python (RAM): Con số `history_length` sẽ nhảy thất thường (ví dụ: request 1 vào agent-1 -> len=1; request 2 vào agent-2 -> len=1; request 3 vào agent-1 -> len=2...). User sẽ thấy agent bị "mất trí nhớ" bất chợt do dữ liệu lịch sử bị phân mảnh ở bộ nhớ của từng instance.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: `Network > Healthcheck: Healthcheck failure (Deployment failed during network process)` trên Railway/Render.
- Nguyên nhân: Xem log khởi động (`View logs`) phát hiện lỗi `pydantic.ValidationError` khiến ứng dụng crash ngay khi vừa bật. Lý do là `Settings` yêu cầu `agent_api_key: str` không có mặc định (fail-fast), nhưng chưa thêm biến môi trường `AGENT_API_KEY` vào Dashboard.
- Cách sửa: Vào tab **Variables** trên Dashboard của Cloud Platform, thêm biến `AGENT_API_KEY` với API key hợp lệ rồi bấm Redeploy. Ứng dụng đã khởi động thành công và Healthcheck qua `/health` trả về status 200 OK.
