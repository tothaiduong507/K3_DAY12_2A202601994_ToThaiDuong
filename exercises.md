# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tô Thái Dương  Mã học viên: 2A202601994

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ khi deploy lên Render, nếu quên đặt `AGENT_API_KEY` thì `Settings` báo
> lỗi ngay lúc khởi tạo thay vì cho service chạy với một khóa ai cũng đoán được.
> Nhờ fail fast, tôi nhìn thấy lỗi cấu hình trong log triển khai và sửa secret
> trước khi mở API cho người dùng. Nếu mặc định là `"changeme"`, health check có
> thể vẫn xanh nhưng người ngoài có thể dùng khóa đó để gọi API và làm phát sinh
> chi phí; lỗi chỉ được phát hiện sau khi hệ thống đã bị sử dụng sai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thực tế khi tôi gọi `/ask` là:
>
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T05:43:21.443512+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`
>
> Với log này, hệ thống thu thập log có thể (1) lọc hoặc truy vết các lần gọi
> theo `event`, `user_id`, `level` và khoảng thời gian; (2) cộng `tokens_in`,
> `tokens_out`, `cost_usd` để tạo dashboard hoặc cảnh báo vượt chi phí. Một câu
> `print("đã trả lời xong")` không có các trường có cấu trúc để truy vấn hay tổng hợp.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản                 | Dung lượng |
| -------------------- | ------------ |
| 1 stage (bản đầu) | 1.172,4 MB   |
| Multi-stage          | 204,8 MB     |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi đo bằng `docker image inspect`: `agent:single` có 1.172.409.297 byte và
> `agent:multi` có 204.848.913 byte, chênh khoảng 967,6 MB. Phần chênh lệch chủ
> yếu là base image `python:3.11` đầy đủ của bản một stage, gồm compiler, header,
> công cụ build/VCS và nhiều gói hệ điều hành không cần lúc chạy. Bản production
> dùng `python:3.11-slim`; stage runtime chỉ nhận virtualenv đã cài từ builder,
> cùng `app` và `utils`, nên không mang toàn bộ môi trường build sang image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa một ký tự trong `app/main.py`, các layer từ base image, tạo
> virtualenv, `COPY requirements.txt` và `pip install` vẫn được lấy từ cache vì
> `requirements.txt` không đổi. Ở runtime, base image, tạo user và
> `COPY --from=builder /opt/venv` cũng được dùng lại; `COPY app ./app` phải chạy
> lại vì nội dung thư mục `app` đổi (các bước sau đó được BuildKit kiểm tra/lắp
> lại, còn `utils` không đổi nên có thể lấy kết quả tương ứng từ cache). Nếu đặt
> `COPY . .` trước `RUN pip install`, mọi thay đổi source đều làm checksum của
> layer `COPY` đổi và buộc bước cài toàn bộ dependency chạy lại, khiến build chậm
> dù danh sách thư viện không thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng Python có thể cho kẻ tấn công thực thi lệnh từ xa trong process.
> Nếu container chạy root, lệnh đó lập tức có quyền root trong container; từ đó
> kẻ tấn công có thể sửa file hệ thống, đọc secret/mount nhạy cảm và lợi dụng cấu
> hình nguy hiểm như `--privileged`, Docker socket được mount, hoặc một lỗ hổng
> kernel/container runtime để thoát ra và chiếm quyền cao trên host. `USER app`
> cắt chuỗi ngay sau bước thực thi mã: mã độc chỉ chạy với UID ít quyền, không
> mặc nhiên sửa được hệ thống hay dùng các thao tác cần root. Đây là giảm thiểu
> tác động, không thay thế việc tránh mount Docker socket và vá runtime/kernel.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa **20 request** trong hai giây: gửi 10 request ngay trước khi
> phút cũ kết thúc, chẳng hạn từ `10:00:59.000`, rồi gửi 10 request ngay sau lúc
> bộ đếm reset ở `10:01:00.000`. Mỗi phút đồng hồ vẫn chỉ ghi nhận 10 request,
> nhưng cả 20 request nằm trong một khoảng hai giây. Cửa sổ trượt 60 giây sẽ nhìn
> thấy 10 request cũ khi xét loạt mới và chặn loạt vượt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tần suất/số request trong 60 giây**, còn cost guard giới
> hạn **tổng tiền của từng user trong tháng**. Một user gọi chậm, mỗi phút chỉ một
> request nhưng đã tiêu gần hoặc hết 10 USD thì rate limit vẫn cho qua còn cost
> guard trả 402. Ngược lại, một user mới gửi dồn request thứ 11 rất rẻ trong chưa
> đầy 60 giây thì tổng tiền vẫn dưới ngân sách nên cost guard cho qua, nhưng rate
> limiter trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp probe và cho liveness kiểm tra Redis, thứ tự sẽ là: (1) Redis mất kết
> nối; (2) cả ba container cùng trả 503 ở probe; (3) orchestrator đánh dấu cả ba
> là unhealthy; (4) nó restart các container gần như cùng lúc; (5) trong lúc
> Redis vẫn lỗi, các container mới lại không qua health check và có thể tiếp tục
> bị restart, làm cụm mất toàn bộ khả năng phục vụ; (6) khi Redis trở lại, cả ba
> cùng khởi động lại, gây cold start/thundering herd rồi mới nhận traffic. Nếu
> tách đúng, `/health` vẫn 200 nên process không bị restart, còn `/ready` trả 503
> để load balancer tạm ngừng gửi request; Redis phục hồi thì `/ready` tự về 200.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi request lưu hai message (`user` và `assistant`), nên
> `history_length` quan sát được tăng đều như `0, 2, 4, 6, ...` bất kể request
> tới container nào. Nếu thay bằng dict Python và nginx phân phối vòng qua ba
> container, mỗi container chỉ biết lịch sử riêng của nó; kết quả có thể thành
> `0, 0, 0, 2, 2, 2, 4, 4, 4, ...` khi round-robin đều, hoặc tăng/giảm thất
> thường khi cân bằng tải không đều. Vì vậy cùng một `X-User-Id` vẫn có cảm giác
> lúc nhớ, lúc quên khi request đổi replica.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thật tôi gặp trên Render là sau lần deploy đầu, `GET /ready` và
> `POST /ask` trả HTTP 500 (trong khi `/health` vẫn 200). Tôi đối chiếu log Render
> và cấu hình `Settings`, rồi xác định Blueprint đã được tạo mà chưa nhập secret
> bắt buộc `AGENT_API_KEY`; các endpoint cần Settings bị `ValidationError`, còn
> liveness không đọc Settings nên vẫn xanh. Tôi thêm `AGENT_API_KEY` trong phần
> Environment của `day12-agent` và redeploy. Sau đó `/ready` trả 200,
> `/ask` không key trả 401, `/ask` với key hợp lệ trả 200, và bộ CP5 đã từng đạt
> 9 test cloud pass.
