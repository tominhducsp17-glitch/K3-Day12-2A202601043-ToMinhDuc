# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới từng câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tô Minh Đức  Mã học viên: 2A202601043

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên môi trường production mà quên không cấu hình biến môi trường `AGENT_API_KEY` trên Dashboard (Railway/Render). Nếu để mặc định `"changeme"`, ứng dụng vẫn khởi động và mở công khai ra Internet. Kẻ tấn công hoặc bot tự động quét sẽ dễ dàng đoán/gọi API với key `"changeme"`, làm rò rỉ dịch vụ và tiêu tốn tiền tài khoản LLM của bạn mà bạn không hay biết. Ngược lại, việc "fail fast" khiến container sập ngay lúc khởi động (Health Check fail, deployment bị cancel), giúp bạn ngay lập tức phát hiện thiếu cấu hình secret trước khi bất kỳ request nào từ bên ngoài truy cập được vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T09:30:00.123456+00:00", "user_id": "sv-test", "tokens_in": 5, "tokens_out": 12, "cost_usd": 0.0001}`

Hai việc làm được:
1. **Truy vấn, lọc và thống kê tự động bằng hệ thống thu gom log (Datadog/ELK/CloudWatch)**: Chạy câu truy vấn để đếm tổng số request, tính tổng chi phí `cost_usd` hoặc tổng token tiêu tốn theo từng `user_id` trong khoảng thời gian cụ thể (ví dụ: "user nào tiêu nhiều tiền nhất hôm nay?").
2. **Cảnh báo tự động (Alerting)**: Cấu hình quy tắc phát hiện bất thường dựa trên các trường cấu trúc, ví dụ khi số lượng event `ask_completed` có `cost_usd > 0.01` tăng đột biến trong 5 phút thì gửi cảnh báo Slack/PagerDuty cho nhóm vận hành. Dòng log thuần text `print("đã trả lời xong")` không chứa metadata dạng JSON nên máy không thể phân tích hay nhóm (group by) theo thuộc tính được.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~210 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch ~800MB bao gồm các công cụ biên dịch (GCC, Make), Python C-extensions build tools, apt cache, header files (`python3-dev`, `libpq-dev`), các file tạm phát sinh trong quá trình build dependency ở stage `builder`. Nhờ Multi-stage build, stage `runtime` chỉ sao chép kết quả đã cài đặt trong `/usr/local` sang một base image sạch (`python:3.11-slim`), loại bỏ toàn bộ bộ công cụ biên dịch nặng nề không cần thiết khi ứng dụng vận hành.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile tối ưu hiện tại: Layer cài đặt dependencies `COPY requirements.txt .` và `RUN pip install ...` được dùng lại hoàn toàn từ Docker cache vì file `requirements.txt` không thay đổi. Chỉ các layer từ `COPY app ./app` trở về sau mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Khi sửa 1 ký tự trong `app/main.py`, layer `COPY . .` bị vô hiệu hóa cache (cache miss), kéo theo tất cả các layer phía sau nó (bao gồm cả `RUN pip install`) đều phải chạy lại từ đầu. Việc này khiến quá trình build lại image mất nhiều phút thay vì chỉ vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Code Python có lỗ hổng (ví dụ RCE thông qua `eval()`, `pickle.loads()`, hoặc Command Injection).
2. Kẻ tấn công lợi dụng lỗ hổng để thực thi lệnh shell bên trong container.
3. Do container chạy bằng `root` (UID 0), lệnh của kẻ tấn công có toàn quyền root trong container filesystem.
4. Kẻ tấn công lợi dụng lỗ hổng vây hãm container (Container Escape / Linux Kernel Exploit hoặc mounted docker.sock) để thoát ra máy host. Do UID bên trong container trùng với root (UID 0) của máy host, kẻ tấn công chiếm luôn quyền root của hệ thống host.

Lệnh `USER appuser` chuyển tiến trình sang UID 10001 (không phải root). Lệnh này cắt đứt chuỗi sự kiện ngay ở bước 3: kẻ tấn công chỉ có quyền của user thường, không thể đọc/ghi các file hệ thống quan trọng trong container và không có đặc quyền root để thực hiện hành vi leo thang đặc quyền (privilege escalation) lên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa: **20 request** trong 2 giây liên tiếp.
- Giải thích cách đạt được: Người dùng gửi 10 request ở giây `10:00:59` (thuộc phút 10:00), sau đó gửi tiếp 10 request ở giây `10:01:01` (thuộc phút 10:01). Cả 2 đợt đều nằm trong hạn mức 10/phút của từng phút đồng hồ tương ứng nên hệ thống cho qua. Tuy nhiên, tính trong khoảng thời gian thực tế 2 giây liên tiếp (từ 10:00:59 đến 10:01:01), hệ thống đã phải gánh tới 20 request, gấp đôi hạn mức mong muốn. Thuật toán Sliding Window loại bỏ kẽ hở này bằng cách luôn tính chính xác trong 60 giây gần nhất.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Sự khác biệt: **Rate Limit** kiểm tra *tần suất/số lượng request* trong khoảng thời gian ngắn (ví dụ: tối đa 10 request/phút). **Cost Guard** kiểm tra *tổng chi phí/tiền tệ phát sinh* trong khoảng thời gian dài hơn (ví dụ: ngân sách 10.0 USD/tháng).
- Tình huống Rate Limit cho qua nhưng Cost Guard chặn: Người dùng chỉ gửi 1 request trong phút đó (đạt Rate Limit), nhưng prompt đầu vào vô cùng lớn dài 100.000 tokens kèm yêu cầu sinh response dài. Request này đốt sạch ngân sách tháng còn lại của user -> Cost Guard phát hiện vượt ngân sách tháng và chặn (trả 402 Payment Required).
- Tình huống Cost Guard cho qua nhưng Rate Limit chặn: Người dùng mới dùng 0.01 USD trong ngân sách 10.0 USD tháng, nhưng gửi 15 request liên tục trong vòng 5 giây. Cost Guard thấy tiền còn rất nhiều nên cho qua, nhưng Rate Limit phát hiện vượt quá 10 request/phút -> Rate Limit chặn ngay (trả 429 Too Many Requests) để bảo vệ server khỏi bị quá tải.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố chập mạng hoặc restart, mất kết nối trong 30 giây.
2. Endpoint gộp gõ kiểm tra Redis -> Redis trả về lỗi -> Endpoint phản hồi 503 Unhealthy.
3. Orchestrator (K8s/Docker) nhận phản hồi 503 từ Liveness probe của cả 3 container agent -> Đánh giá cả 3 container đều "đã chết" và lập tức **kill & restart** cả 3 container cùng lúc.
4. Trong 30 giây Redis chưa hồi phục, cả 3 container mới khởi động lại tiếp tục gọi kiểm tra Redis -> tiếp tục báo 503 -> Orchestrator lại tiếp tục kill & restart vòng lặp.
5. Khi Redis kết nối trở lại sau 30s, toàn bộ 3 container agent đều đang ở trạng thái bị restart dở dang, dẫn đến toàn bộ dịch vụ bị gián đoạn hoàn toàn (Downtime).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python (in-memory của process), con số `history_length` sẽ thay đổi nhảy vọt ngẫu nhiên không ổn định (ví dụ: call 1 = 0, call 2 = 0, call 3 = 1, call 4 = 0...). Lý do là Load Balancer phân phối các request luân phiên tròn (Round-Robin) tới 3 instance khác nhau (A, B, C). Mỗi instance giữ một dict riêng trong RAM. Request thứ 1 vào instance A (A lưu 1 item), request thứ 2 lại vào instance B (B chưa có item nào nên `history_length` lại quay về 0). Agent rơi vào trạng thái "mất trí nhớ" ngẫu nhiên. Khi lưu trong Redis tập trung, cả 3 instance cùng đọc/ghi vào một nguồn chung nên `history_length` luôn tăng dần đều liên tục (0 -> 1 -> 2 -> 3...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Thông báo lỗi: `Error response from daemon: Bind for 0.0.0.0:8000 failed: port is already allocated` / `Healthcheck timeout`.
Cách tìm nguyên nhân: Đọc log hệ thống (`docker compose logs` / docker daemon output), thấy cổng `8000` trên máy host đã bị chiếm dụng bởi một container khác (`math-exam-agent`) đang chạy trước đó.
Cách sửa: Chạy lệnh `docker ps` tìm container đang chiếm cổng 8000, sử dụng `docker stop math-exam-agent` để giải phóng port 8000, sau đó chạy lại `docker compose up -d` để container của ứng dụng bind cổng 8000 thành công và pass healthcheck.
