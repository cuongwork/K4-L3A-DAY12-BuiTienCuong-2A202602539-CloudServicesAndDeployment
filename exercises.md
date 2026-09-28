# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết câu trả lời vào từng mục bên dưới.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bùi Tiến Cường  Mã học viên: 2A202602539

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu quên cấu hình `AGENT_API_KEY` trên Railway, ứng dụng dừng ngay lúc khởi
động, khiến em phải sửa cấu hình trước khi service nhận lưu lượng. Nếu dùng
giá trị mặc định `"changeme"`, service vẫn có thể vượt health check nhưng
người biết khóa mặc định sẽ gọi `/ask` và tiêu thụ ngân sách của em. Lỗi
cấu hình được phát hiện sớm vì vậy an toàn hơn một service có vẻ hoạt động
nhưng dùng thông tin xác thực yếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Khi gọi `/ask` cục bộ thành công, em thu được dòng log JSON thực tế:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T14:08:08.290780+00:00", "user_id": "exercise-log-20260928", "tokens_in": 1, "tokens_out": 35, "cost_usd": 2.115e-05}
```

Có thể lọc các bản ghi theo `event` và `user_id` để truy vết yêu cầu;
đồng thời tổng hợp `tokens_in`, `tokens_out`, `cost_usd` để theo dõi
sử dụng và đặt cảnh báo chi phí. Câu `print("đã trả lời xong")` không
mang các trường dữ liệu có cấu trúc cho hai thao tác này.

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
| 1 stage (bản đo bằng `Dockerfile.single-stage.measure`) | 290 MB |
| Multi-stage (bản trong `Dockerfile`) | 312 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Em đo bằng `docker build -f Dockerfile.single-stage.measure -t
agent:single-measure .`, `docker build -t agent:multi-measure .` và
`docker image ls`. Kết quả của repo này là bản multi-stage lớn hơn 22 MB.
Hai bản cùng dùng `python:3.11-slim` và cùng cài `requirements.txt`;
bản multi-stage sao chép toàn bộ virtual environment sang một runtime
vẫn có sẵn Python. Vì builder không chứa công cụ biên dịch đáng kể để
loại bỏ, lợi ích giảm dung lượng không xuất hiện; virtual environment
còn tạo thêm tệp và metadata. Do đó cần đo thực tế thay vì mặc định
kết luận multi-stage luôn nhỏ hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa `app/main.py`, base image, `WORKDIR`, bước tạo virtual
environment, `COPY requirements.txt` và `RUN pip install` không đổi
nên có thể dùng cache. `COPY app ./app` và các layer phía sau phải chạy
lại. Trong lần build thực tế khi mã chưa đổi, Docker đánh dấu các bước
cài dependency là `CACHED`. Nếu đưa `COPY . .` lên trước
`RUN pip install`, sửa một tệp mã nguồn cũng làm mất cache của bước
cài dependency, khiến build phải cài lại các gói không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Lỗ hổng thực thi lệnh trong ứng dụng trước hết trao cho kẻ tấn công
quyền của tiến trình trong container. Nếu tiến trình chạy bằng root,
họ có thể sửa các tệp hệ thống trong container; nếu thêm mount nhạy
cảm như Docker socket hoặc tồn tại lỗ hổng thoát container, tác động
có thể lan tới host. Chạy root trong container không tự động đồng nghĩa
với root trên host. Lệnh `USER agent` cắt chuỗi ở bước đầu bằng cách
hạ quyền của mã ứng dụng, qua đó giảm phạm vi thao tác sau khi bị khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Có thể gửi tối đa **20 request** trong hai giây quanh ranh giới phút:
10 request ở cuối phút trước và 10 request ở đầu phút sau. Mỗi bộ đếm
theo phút riêng lẻ đều thấy đúng hạn mức 10. Cửa sổ trượt 60 giây
nhìn thấy cả 20 request trong cùng khoảng gần nhất và chặn từ request
thứ 11. Trên Railway, em quan sát 10 phản hồi 200 rồi 5 phản hồi 429
khi gửi liên tiếp với cùng một user ID.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tần suất request trong 60 giây, còn cost guard
giới hạn chi phí tích lũy theo user trong tháng. Một user có chi phí
tháng đã vượt ngân sách nhưng mới gửi một request trong phút sẽ qua
rate limit và bị cost guard trả 402. Ngược lại, user còn nhiều ngân
sách nhưng gửi request thứ 11 trong cùng cửa sổ sẽ bị rate limit trả
429. Mã hiện tại kiểm tra chi phí đã ghi trước khi gọi mô hình, nhưng
truyền chi phí ước tính mặc định bằng 0; vì thế một request lớn có thể
làm vượt ngân sách và chỉ request tiếp theo mới bị chặn. Đây là giới
hạn cần lưu ý khi diễn giải cơ chế bảo vệ chi phí.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Khi Redis mất kết nối, endpoint gộp sẽ báo lỗi trên cả ba container
dù tiến trình HTTP vẫn sống. Bộ cân bằng tải loại ba instance khỏi
danh sách sẵn sàng, nên request mới không có đích phục vụ. Nếu endpoint
này cũng được dùng làm liveness probe trong Kubernetes, sau ngưỡng
thất bại orchestrator có thể khởi động lại cả ba container; restart
không sửa được Redis và còn tăng gián đoạn. Khi Redis phục hồi, probe
lại thành công và lưu lượng trở lại. Tách `/health` khỏi `/ready`
tránh restart vô ích. Với Docker Compose, healthcheck chỉ đánh dấu
`unhealthy`, không tự restart container vì trạng thái đó.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Trong phép thử cục bộ dùng Redis và cùng `X-User-Id`, ba lời gọi
`/ask` trả `history_length` lần lượt **0, 2, 4**: mỗi lượt thêm
một message của user và một của assistant. Em chưa kiểm chứng trực
tiếp ba container vì Compose hiện ánh xạ cố định cổng `8000:8000`,
gây xung đột khi scale. Nếu dùng dict Python riêng trong mỗi process,
request chuyển sang instance khác sẽ đọc lịch sử riêng, khiến số
`history_length` có thể quay về 0 hoặc tăng không đều theo tuyến
request thay vì tiếp tục tuần tự.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Trong bước kiểm tra sau deploy trên Railway, `/health` trả 200 và
`/ready` trả 200 với `"redis": true`, nhưng `POST /ask` dùng
`DEPLOY_API_KEY` ban đầu trả **401** với thông báo
`{"detail":"invalid or missing API key"}`. Vì các probe đều đạt,
em tập trung kiểm tra cấu hình xác thực. Em đối chiếu khóa kiểm thử
cục bộ với `AGENT_API_KEY` trên Railway và cập nhật
`DEPLOY_API_KEY` trong `.env` cho khớp, không ghi khóa vào repo.
Lần thử lại trả 200; 15 request tiếp theo cho 10 lần 200 và 5
lần 429, xác nhận xác thực và rate limit của bản deploy.
