# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Hải Yến  Mã học viên: 2A202601152

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên cloud, nếu em quên đặt `API_TOKEN` mà ứng dụng vẫn dùng
> `"changeme"`, service vẫn có thể nhận traffic với một token dễ đoán và người
> lạ có thể gọi `/chat` bằng chi phí của em. Việc `api_token` không có mặc định
> làm cấu hình báo lỗi ngay khi khởi động, trước khi service được đưa vào phục
> vụ, nên em phát hiện và bổ sung secret ngay trong lúc deploy thay vì chỉ biết
> khi API đã bị dùng hoặc hóa đơn tăng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log em thu được là:
>
> `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T13:07:37.167394+00:00", "client_id": "sv01", "prompt_tokens": 18, "completion_tokens": 42, "usd_cost": 2.7e-05}`
>
> Từ log có cấu trúc này, em có thể lọc và cộng `usd_cost` theo `client_id` để
> tìm client tiêu nhiều nhất; đồng thời có thể đếm sự kiện theo `severity` và
> thời gian để tạo cảnh báo khi tỷ lệ lỗi tăng. Dòng `print("đã trả lời xong")`
> không chứa client, thời điểm, token hay chi phí nên không làm được hai việc đó.

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
| 1 stage (bản đầu) | khoảng 1800 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch chủ yếu đến từ image `python:3.11` đầy đủ của bản một stage:
> nó mang theo nhiều gói hệ điều hành và công cụ không cần lúc chạy. Bản đầu còn
> `COPY . .` nên đưa cả source không cần thiết vào context/image và `pip install`
> không tắt cache tải xuống. Bản multi-stage dùng `python:3.11-slim`, chỉ chép
> thư viện đã cài từ `/install` sang runtime và chỉ chép `app`, `utils`, nên image
> chạy thật không mang theo phần dư của môi trường build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer builder gồm base image, `WORKDIR`,
> `COPY requirements.txt` và `RUN pip install` vẫn lấy từ cache vì
> `requirements.txt` không đổi. Ở runtime, các layer đến `COPY --from=builder`
> cũng được dùng lại; `COPY app ./app` phải chạy lại và các layer đứng sau nó
> được tạo lại, nhưng không phải cài lại dependency. Nếu đặt `COPY . .` trước
> `RUN pip install`, một thay đổi nhỏ trong source sẽ làm layer copy đổi và vô
> hiệu cache của `pip install`, khiến toàn bộ thư viện bị cài lại ở mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng thực thi lệnh từ xa, kẻ tấn công sẽ chạy lệnh với
> đúng user của process trong container. Khi process là root, họ có toàn quyền
> trong container; kết hợp với volume/socket nhạy cảm, capability thừa hoặc một
> lỗ hổng thoát container, quyền đó có thể dẫn tới quyền cao trên host. Lệnh
> `USER appuser` cắt chuỗi ở bước thực thi lệnh: code bị chiếm quyền chỉ chạy với
> UID 10001 và không có quyền root để sửa file hệ thống hay thực hiện các thao
> tác đặc quyền, nhờ đó giảm phạm vi thiệt hại.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Header `WWW-Authenticate: Bearer` cho client biết endpoint yêu cầu cơ chế xác
> thực Bearer theo chuẩn HTTP, để client biết cách gửi lại request hợp lệ. Em dùng
> cùng thông báo `invalid or missing bearer token` cho thiếu header, sai scheme
> và sai token vì nếu trả lời chi tiết, người dò token sẽ biết phần nào đã đúng
> và thu hẹp dần khả năng. Client hợp lệ vẫn có đủ thông tin từ status 401 và
> header chuẩn để sửa request mà không cần service tiết lộ chi tiết xác thực.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Sau 10 phút, client vẫn chỉ gửi liên tiếp được 10 request rồi request thứ 11
> nhận 429, vì `min(capacity, ...)` chặn số token tối đa ở 10. Nếu xô đã cạn và
> bỏ `min`, tốc độ 10 token/phút trong 10 phút sẽ cộng thành 100 token, nên client
> có thể bắn 100 request liên tiếp. Như vậy thời gian im lặng bị biến thành lượng
> token tích lũy không giới hạn và làm mất ý nghĩa của giới hạn burst.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức 30 USD/tháng, sự cố lúc 2 giờ sáng có thể tiêu hết 30 USD ngay
> trong ngày đó; client bị chặn đến kỳ tháng mới, nên service không tự hồi phục
> vào ngày hôm sau. Với hạn mức 1 USD/ngày, thiệt hại tối đa trong ngày xảy ra sự
> cố là 1 USD và key theo ngày tự đổi lúc sang ngày mới theo UTC, nên service tự
> cho phép lại mà không cần can thiệp. Nếu sự cố vẫn tiếp diễn, nó chỉ có thể tiêu
> thêm tối đa 1 USD cho mỗi ngày tiếp theo thay vì đốt cả ngân sách tháng một lần.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai endpoint, Redis mất kết nối làm cả ba container cùng trả 503 cho
> health check. Orchestrator hiểu nhầm cả ba process đã chết và restart chúng
> cùng lúc. Khi Redis vẫn chưa hồi phục, các container mới lại health check lỗi
> và tiếp tục bị restart, khiến cụm không còn instance nào phục vụ dù code ứng
> dụng vẫn sống. Tách `/healthz` giúp process vẫn báo sống, còn `/readyz` trả 503
> để load balancer tạm ngừng gửi traffic cho đến khi Redis kết nối lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy Railway, build và tạo container thành công nhưng healthcheck thất
> bại. Trong Deploy Logs em thấy lỗi:
> `Invalid value for '--port': '$PORT' is not a valid integer.` Em kiểm tra
> `railway.toml` và nhận ra `startCommand` truyền nguyên chuỗi `$PORT` cho
> Uvicorn, không qua shell để thay biến môi trường. Em xóa `startCommand` khỏi
> `railway.toml` để Railway dùng `CMD` trong Dockerfile; lệnh đó chạy qua
> `sh -c` và đọc `${PORT:-8000}` đúng cách. Sau khi kết nối lại GitHub source và
> deploy commit mới, `/healthz` và `/readyz` đều trả 200.
