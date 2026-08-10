# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tạ Minh Đức  Mã học viên: 2A202601497

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, nếu tôi quên đặt `AGENT_API_KEY` thì ứng dụng dừng ngay lúc khởi động và log báo thiếu cấu hình. Nhờ vậy tôi biết lỗi nằm ở biến môi trường trước khi service nhận request. Nếu dùng mặc định `"changeme"`, service vẫn có thể báo healthy nhưng bất kỳ ai đoán được khóa này đều gọi được `/ask`, làm phát sinh request và chi phí ngoài ý muốn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được là: `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T04:47:18.038173+00:00","user_id":"exercise-student","tokens_in":6,"tokens_out":44,"cost_usd":2.73e-05}`. Với log JSON này, tôi có thể lọc hoặc nhóm theo `user_id`, `event`, `level` trong hệ thống thu thập log; đồng thời có thể tính tổng token và chi phí theo người dùng hoặc theo khoảng thời gian. Dòng `print("đã trả lời xong")` không có các trường cố định nên máy khó truy vấn và tổng hợp tự động.

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
| 1 stage (bản đầu) | khoảng 1.12 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản multi-stage tôi đo bằng `docker image ls` có dung lượng 271 MB, nhỏ hơn nhiều so với bản một stage dùng image `python:3.11` đầy đủ. Phần chênh lệch chủ yếu đến từ base image đầy đủ, công cụ build, cache và các tệp chỉ cần trong lúc cài thư viện. Stage runtime chỉ dùng `python:3.11-slim` và copy thư viện đã cài từ builder, nên không mang theo môi trường build không cần thiết.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer lấy base image, đặt `WORKDIR`, copy `requirements.txt` và chạy `pip install` vẫn được dùng lại từ cache. Layer `COPY . .` và các layer phía sau nó phải chạy lại vì nội dung source đã đổi. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ một thay đổi nhỏ trong source cũng làm mất cache của layer copy, khiến pip phải cài lại toàn bộ dependency và thời gian build lâu hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng trong code Python có thể cho kẻ tấn công thực thi lệnh trong container. Nếu tiến trình chạy bằng root và container còn bị cấu hình nguy hiểm như mount thư mục nhạy cảm hoặc cấp quyền cao, kẻ tấn công có thể sửa file được mount hoặc khai thác lỗ hổng thoát container để tác động tới host với quyền lớn. Lệnh `USER appuser` cắt chuỗi ở bước thực thi trong container: mã bị chiếm quyền chỉ có UID 10001 và bị giới hạn bởi quyền của user này. Nó không loại bỏ mọi khả năng thoát container, nhưng giảm đáng kể hậu quả.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với fixed window theo phút, người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp: gửi 10 request ở giây cuối của phút trước, rồi ngay sau thời điểm giây 00 gửi thêm 10 request thuộc phút mới. Mỗi cửa sổ riêng vẫn chỉ có 10 request nhưng thực tế có một burst 20 request sát ranh giới. Sliding window 60 giây sẽ nhìn cả hai nhóm trong cùng khoảng 60 giây nên chặn nhóm vượt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limiter giới hạn tần suất ngắn hạn, ví dụ 10 request trong 60 giây, còn cost guard giới hạn tổng tiền đã dùng trong cả tháng. Một user gọi một request đắt sau thời gian nghỉ vẫn được rate limiter cho qua nhưng cost guard phải chặn nếu ngân sách tháng đã hết. Ngược lại, một user chưa tốn đáng kể ngân sách nhưng gửi 11 request rất rẻ liên tiếp sẽ được cost guard cho qua về tiền, còn rate limiter chặn request thứ 11 để chống burst hoặc spam.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/health` với `/ready` và liveness cũng kiểm tra Redis, khi Redis mất kết nối thì cả ba container agent lần lượt trả lỗi health check. Hệ thống điều phối sẽ cho rằng tiến trình bị hỏng và restart cả ba container. Sau khi khởi động, chúng vẫn không kết nối được Redis nên lại fail và tiếp tục bị restart, gây vòng lặp và làm mất khả năng phục vụ kể cả các chức năng không cần Redis. Khi tách riêng, `/health` vẫn báo tiến trình còn sống, còn `/ready` trả 503 để load balancer tạm ngừng chuyển traffic; khi Redis hồi phục, container tự ready lại mà không cần restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi dùng Redis chung, dù request được chuyển tới container nào thì `history_length` vẫn tăng nhất quán theo lịch sử đã lưu, ví dụ 0, 2, 4, 6 vì mỗi lượt thêm một message user và một message assistant. Nếu dùng dict Python, mỗi container có một bản lịch sử riêng. Khi load balancer phân phối request, tôi có thể thấy các số lặp lại hoặc nhảy không đều như 0, 0, 2, 0, 2 thay vì một chuỗi tăng liên tục; restart container còn làm mất toàn bộ lịch sử của container đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy lên Render, tôi mở `https://day12-agent-8fx4.onrender.com/` và nhận HTTP 404 với `{"detail":"Not Found"}`, nên ban đầu nghĩ service deploy lỗi. Tôi kiểm tra log thấy Uvicorn vẫn chạy, sau đó thử đúng probe `/health` và `/ready`; cả hai đều trả HTTP 200 và `/ready` báo Redis đã kết nối. Nguyên nhân là FastAPI không khai báo route `/`, không phải deploy thất bại. Tôi sửa cách kiểm tra và nội dung `DEPLOYMENT.md` để dùng các endpoint đã định nghĩa thay vì URL gốc.
