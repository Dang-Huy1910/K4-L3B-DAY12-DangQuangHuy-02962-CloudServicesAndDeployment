# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder mẫu bằng câu trả lời tương ứng.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đặng Quang Huy  Mã học viên: 02962

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Một tình huống cụ thể là khi deploy ứng dụng lên môi trường production (hoặc staging) mới trên Cloud (Railway, Render, K8s), người cấu hình quên thêm biến môi trường `AGENT_API_KEY` vào dashboard.
Nếu đặt giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động trơn tru. Khi đó, bot quét lỗ hổng trên Internet có thể dùng khóa mặc định `"changeme"` để gửi hàng ngàn request vào endpoint `/ask`, gây cạn kiệt hạn mức chi phí LLM hoặc làm rò rỉ dữ liệu. Đồng thời, client hợp lệ gửi khóa thật sẽ bị từ chối với mã 401, khiến lập trình viên mất rất nhiều thời gian dò tìm lỗi vì server vẫn báo trạng thái Healthy.
Ngược lại, với cơ chế Fail Fast (không đặt mặc định), `pydantic-settings` sẽ ném ngoại lệ `ValidationError` ngay lúc process khởi động. Platform lập tức đánh dấu bản deploy thất bại và giữ nguyên phiên bản cũ đang chạy ổn định. Sự cố được phát hiện và ngăn chặn ngay lập tức trên log deploy trước khi bất kỳ request độc hại nào lọt vào hệ thống.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:02:16.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 28, "cost_usd": 0.00008}`

Hai việc làm được với dòng log có cấu trúc này mà `print("đã trả lời xong")` không làm được:
1. **Truy vấn, lọc và tổng hợp số liệu tự động (Structured Querying & Metrics Aggregation):** Các hệ thống thu thập log tập trung (Datadog, ElasticSearch, CloudWatch, Loki) có thể tự động parse JSON để thực hiện các truy vấn phân tích như: tính tổng chi phí `cost_usd` của từng `user_id` trong ngày, đo lường lượng token tiêu thụ trung bình, hoặc nhóm tần suất gọi API theo thời gian thực mà không cần viết regex bóc tách chuỗi phức tạp.
2. **Cấu hình cảnh báo tự động theo ngưỡng (Automated Alerting & Monitoring):** Có thể dễ dàng thiết lập các quy tắc cảnh báo trên hệ thống giám sát dựa trên từng trường dữ liệu, ví dụ: kích hoạt cảnh báo tới Slack/PagerDuty khi `level == "error"` vượt quá 5% trong 5 phút, hoặc khi `cost_usd > 0.5` trong một request đơn lẻ (phát hiện dấu hiệu prompt injection hoặc lạm dụng), điều hoàn toàn bất khả thi đối với chuỗi print tự do.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 175 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~845 MB) chủ yếu gồm:
1. Base image: Bản 1-stage ban đầu dùng `python:3.11` đầy đủ dựa trên Debian tiêu chuẩn, chứa rất nhiều công cụ, tiện ích dòng lệnh, package hệ thống và tài liệu trợ giúp không cần thiết cho runtime. Trong khi bản multi-stage dùng `python:3.11-slim` chỉ giữ lại các thành phần tối thiểu để chạy Python.
2. Công cụ biên dịch và thư viện build (Build Tools & Dependencies): Khi cài đặt các thư viện Python (đặc biệt là các thư viện có C extensions), hệ thống cần `gcc`, `g++`, `make`, header files (`python3-dev`, `linux-headers`) và cache tạm của pip. Ở multi-stage build, toàn bộ trình biên dịch và file build tạm thời này nằm ở stage `builder` và bị hủy hoàn toàn. Stage `runtime` chỉ copy đúng thư mục kết quả `/usr/local` và source code ứng dụng, loại bỏ hoàn toàn các rác thải build giúp image giảm từ hơn 1GB xuống dưới 200MB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa 1 ký tự trong `app/main.py` và build lại:
- Các layer được dùng lại từ cache (CACHED): Toàn bộ stage `builder` (bao gồm `FROM python:3.11-slim AS builder`, `COPY requirements.txt .`, `RUN pip install ...`) và các layer đầu của stage `runtime` (`FROM python:3.11-slim AS runtime`, `RUN useradd ...`, `COPY --from=builder /install /usr/local`).
- Layer phải chạy lại: Bắt đầu từ layer `COPY . .` (do checksum của thư mục context thay đổi) và các lệnh đứng sau nó (`RUN chown ...`).

Nếu đặt `COPY . .` lên trước `RUN pip install`:
Do Docker cache theo từng layer tuần tự từ trên xuống dưới, việc thay đổi một ký tự trong code sẽ làm invalid cache của layer `COPY . .`. Khi đó, toàn bộ các layer phía sau nó bao gồm cả `RUN pip install` bắt buộc phải chạy lại từ đầu. Kết quả là mỗi lần chỉnh sửa dù nhỏ nhất, Docker đều phải tải và cài lại toàn bộ thư viện qua mạng, khiến thời gian build tăng từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện leo thang quyền hạn:
1. Kẻ tấn công phát hiện một lỗ hổng trong code Python (ví dụ: command injection, unsafe deserialization pickle, hoặc RCE qua thư viện ngoài).
2. Kẻ tấn công gửi payload kích hoạt thành công shell thực thi lệnh bên trong container.
3. Vì container không khai báo `USER`, tiến trình Python chạy với tư cách `root` (UID 0). Theo kiến trúc Linux container, root trong container mặc định chia sẻ cùng kernel và tương đương với UID 0 trên host OS.
4. Với quyền root trong container, kẻ tấn công có thể khai thác các lỗ hổng kernel (như Dirty COW/Dirty Pipe), tận dụng Linux capabilities được cấp (CAP_SYS_ADMIN), hoặc truy cập các volume/socket nhạy cảm được mount từ host (như `/var/run/docker.sock`) để thực hiện container breakout (thoát khỏi container).
5. Sau khi thoát khỏi container, tiến trình của kẻ tấn công trên host vẫn mang UID 0 (root), giúp chúng chiếm toàn quyền kiểm soát máy host và toàn bộ các container khác trên hệ thống.

Lệnh `USER appuser` cắt đứt chuỗi này tại bước 3:
Khi chỉ định `USER appuser`, tiến trình Python chạy dưới quyền user thông thường không có đặc quyền (UID 10001). Nếu kẻ tấn công chiếm được shell ở bước 1-2, chúng chỉ có quyền hạn chế của `appuser`, không có quyền can thiệp vào file hệ thống, không có capabilities đặc biệt để thao tác kernel, và không thể thực hiện các kỹ thuật breakout vốn bắt buộc phải có quyền root. Cuộc tấn công bị chặn đứng hoàn toàn bên trong sandbox của container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Một người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.

Giải thích:
Cơ chế đếm theo phút đồng hồ (Fixed Window) reset bộ đếm về 0 tại mỗi mốc giây `:00` của phút mới.
Người dùng có thể tấn công vào ranh giới chuyển giao giữa 2 cửa sổ:
- Lúc `10:00:59` (giây cuối cùng của phút thứ nhất): Người dùng gửi dồn dập 10 request. Bộ đếm phút 10:00 ghi nhận 10/10 request (hợp lệ).
- Lúc `10:01:00` (giây đầu tiên của phút thứ hai): Bộ đếm phút 10:01 được reset về 0. Người dùng lập tức gửi tiếp 10 request nữa. Bộ đếm ghi nhận 10/10 request (vẫn hợp lệ).
Như vậy, chỉ trong vòng 2 giây (từ 10:00:59 đến 10:01:01), hệ thống đã phải xử lý tới 20 request (gấp 2 lần hạn mức thiết kế).
Thuật toán sliding window bằng Redis Sorted Set giải quyết triệt để vấn đề này vì nó luôn tính tổng request trong đúng 60 giây gần nhất tính từ thời điểm hiện tại (`now - 60`).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Điểm khác nhau cốt lõi:
- **Rate Limit:** Kiểm soát **vận tốc/tần suất** request trong khung thời gian ngắn (ví dụ: tối đa 10 req/phút). Mục đích bảo vệ hạ tầng máy chủ khỏi nghẽn tải, từ chối dịch vụ (DDoS) và đảm bảo tính sẵn sàng.
- **Cost Guard:** Kiểm soát **tổng chi phí tài chính tích lũy** trong chu kỳ dài hạn (ngân sách USD mỗi tháng). Mục đích kiểm soát rủi ro tài chính phát sinh từ việc tiêu thụ token LLM hoặc API trả phí theo lượng dùng.

Tình huống Rate limit cho qua nhưng Cost guard phải chặn:
Một người dùng gửi chỉ 1 request duy nhất trong 15 phút (tần suất cực thấp, hoàn toàn hợp lệ với rate limit 10 req/phút). Tuy nhiên request này chứa tài liệu rất dài với 60,000 token khiến chi phí ước tính là $0.60. Trong khi đó, người này đã tiêu $9.70 trong tháng trên tổng ngân sách $10.00 (chỉ còn $0.30). Rate limit cho qua vì không spam request, nhưng Cost guard phát hiện vượt hạn mức tháng và lập tức chặn với mã lỗi `402 Payment Required`.

Tình huống Cost guard cho qua nhưng Rate limit phải chặn:
Một người dùng mới bắt đầu tháng, ngân sách còn nguyên $10.00 ($0 spent). Người này chạy script gửi liên tục 15 request "hello" (mỗi request chỉ 5 token, chi phí chỉ $0.00001) trong vòng 2 giây. Tổng chi phí gần như bằng 0 nên Cost guard cho qua, nhưng Rate limiter lập tức can thiệp chặn từ request thứ 11 với mã lỗi `429 Too Many Requests` để bảo vệ server khỏi bị spam burst traffic.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố mạng hoặc restart tạm thời trong 30 giây.
2. Cả 3 container nhận probe kiểm tra sức khỏe từ orchestrator (Docker/K8s/Railway). Vì endpoint kiểm tra phụ thuộc vào Redis, cả 3 container đều đồng loạt báo lỗi (HTTP 503).
3. Do đây là liveness probe (kiểm tra sự sống của process), orchestrator suy đoán rằng tiến trình của cả 3 container đã bị treo hoặc chết và tiến hành gửi tín hiệu restart đồng loạt cả 3 container.
4. Cả cụm 3 container cùng bị tắt và rơi vào trạng thái khởi động lại. Hệ thống rơi vào trạng thái sập hoàn toàn (complete outage), không còn bất kỳ container nào phục vụ request của người dùng (kể cả các endpoint tĩnh không cần Redis).
5. Khi các container mới khởi động lại, nếu Redis vẫn chưa phục hồi trong khoảng thời gian 30 giây đó, các container mới lại tiếp tục báo lỗi và orchestrator lại tiếp tục restart chúng liên tục (vòng lặp CrashLoopBackOff).
Sự cố mất kết nối phụ thuộc ngắn 30 giây đã bị phóng đại thành sự cố sập toàn bộ hệ thống (cascading failure).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless): Giá trị `history_length` tăng tuần tự và nhất quán sau mỗi câu hỏi: 0 → 2 → 4 → 6 → 8... dù mỗi request được load balancer chuyển đến các container khác nhau (container A, B hay C), vì tất cả cùng chia sẻ bộ nhớ tập trung trong Redis.
- Nếu lưu trong dict Python (Stateful trong RAM): Giá trị `history_length` sẽ thay đổi hỗn loạn và nhảy cóc không thể đoán trước. Ví dụ với load balancer xoay vòng giữa 3 container A, B, C:
  - Lượt 1 vào A: `history_length = 0` (A lưu lượt 1 vào RAM của A)
  - Lượt 2 vào B: `history_length = 0` (B chưa từng thấy user này, RAM của B rỗng)
  - Lượt 3 vào C: `history_length = 0` (C cũng chưa từng thấy user)
  - Lượt 4 quay lại A: `history_length = 2` (A chỉ nhớ lượt 1, không hề biết gì về lượt 2 và 3)
  - Lượt 5 vào B: `history_length = 2` (B chỉ nhớ lượt 2)
  Người dùng sẽ thấy agent bị "mất trí nhớ ngẫu nhiên", câu trả lời của AI mất hoàn toàn ngữ cảnh liền mạch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi:
  Trên dashboard runtime log của cloud platform xuất hiện lỗi: `Application failed to respond on port 8000. Health check timed out after 30s. Container killed.` Bên ngoài khi gọi vào domain công khai gặp lỗi `502 Bad Gateway` hoặc `curl: (52) Empty reply from server`.
- Cách tìm ra nguyên nhân:
  Xem runtime log chi tiết trên dashboard của platform, thấy uvicorn log dòng: `Uvicorn running on http://127.0.0.1:8000`. Điều này cho thấy 2 vấn đề:
  1. Uvicorn đang bind vào `127.0.0.1` (loopback nội bộ của container) thay vì `0.0.0.0`, khiến proxy/load balancer của platform từ bên ngoài không thể kết nối tới container.
  2. Platform cấp phát cổng động qua biến môi trường `$PORT` (ví dụ `PORT=49152`), nhưng ứng dụng lại đang cố định ở cổng 8000, khiến probe của platform kiểm tra vào `$PORT` bị từ chối kết nối.
- Cách sửa:
  Sửa lệnh khởi chạy trong `Dockerfile` và `railway.toml`/`render.yaml` thành:
  `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  để uvicorn bind vào `0.0.0.0` và tự động lắng nghe trên cổng do platform chỉ định qua `$PORT`. Sau khi cấu hình lại, container vượt qua health check thành công và service hoạt động bình thường.
