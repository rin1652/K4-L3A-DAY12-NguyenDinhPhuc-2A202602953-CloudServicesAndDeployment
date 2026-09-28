# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần trả lời mẫu dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đình Phúc — Mã học viên: 2A202602953

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy thiếu `AGENT_API_KEY`, service dừng ngay khi khởi động và dashboard báo lỗi cấu hình. Điều này tốt hơn việc chạy với `changeme`, vì request sẽ không vô tình được xác thực bằng một khóa mà ai cũng đoán được.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ: `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T07:48:08Z","user_id":"anonymous","tokens_in":3,"tokens_out":37,"cost_usd":2.265e-05}`. Từ đó có thể lọc riêng các request đã hoàn tất và tính tổng chi phí/token theo user; log dạng `print` không có cấu trúc ổn định để máy làm hai việc này.

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
| 1 stage (bản đầu) | lớn hơn vì giữ toàn bộ lớp build và cache pip |
| Multi-stage | nhỏ hơn vì runtime chỉ nhận dependency đã cài và source cần chạy |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Với Dockerfile hiện tại, `COPY requirements.txt` và lớp cài dependency được cache lại khi chỉ sửa `app/main.py`; các lớp copy source và lệnh chạy image được tạo lại. Nếu đặt `COPY . .` trước `pip install`, mọi thay đổi source sẽ làm mất cache của lớp cài dependency và khiến build chậm hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Lỗ hổng trong app có thể cho phép tiến trình Python bị điều khiển. Nếu tiến trình chạy root, kẻ tấn công có quyền cao trong container và có thể tận dụng thêm lỗi cấu hình để ảnh hưởng host. `USER appuser` giới hạn quyền của tiến trình ngay từ lúc container chạy.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Với giới hạn 10 request/phút, cách đếm theo phút đồng hồ cho phép tối đa 20 request trong khoảng 2 giây: gửi 10 request ở giây 59 của phút trước rồi 10 request ở giây 00 hoặc 01 của phút sau. Sliding window nhìn lại đúng 60 giây nên chặn trường hợp này.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Rate limit giới hạn số lần gọi trong một khoảng thời gian, còn cost guard giới hạn số tiền đã tiêu trong tháng. Một prompt rất dài có thể vẫn dưới rate limit nhưng bị cost guard chặn khi ngân sách gần hết; nhiều request nhỏ thì ngược lại có thể bị rate limit trước khi chạm ngân sách.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Nếu `/health` cũng gọi Redis và Redis mất kết nối, cả ba container có thể bị coi là không khỏe và bị restart/loại khỏi traffic. Tách hai endpoint giúp liveness vẫn 200, còn readiness 503 để load balancer ngừng gửi request khi Redis lỗi.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Khi history nằm trong Redis, cả ba instance cùng đọc một danh sách nên `history_length` tăng nhất quán. Nếu dùng dict trong RAM, mỗi instance có dữ liệu riêng; khi request chuyển instance, `history_length` có thể quay về 0 hoặc tăng theo chuỗi riêng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trong môi trường này chưa tạo service cloud public, nên không ghi nhận lỗi deploy cloud giả. Lỗi thực tế gặp khi chạy local là Python hệ thống 3.9 trong khi project yêu cầu 3.11; mình kiểm tra bằng `python --version`, chuyển `.venv` sang Python 3.11, cài lại dependency và test chạy được.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Trong môi trường này chưa tạo service cloud public, nên không ghi nhận lỗi deploy cloud giả. Lỗi thực tế gặp khi chạy local là Python hệ thống 3.9 trong khi project yêu cầu 3.11; mình kiểm tra bằng `python --version`, chuyển `.venv` sang Python 3.11, cài lại dependency và test chạy được.
