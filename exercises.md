# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Như Ý  Mã học viên: L3B02517

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên production trên nền tảng Cloud (như Render hoặc Railway), lập trình viên vô tình quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
- Nếu để giá trị mặc định `"changeme"`: Ứng dụng vẫn khởi động thành công và báo healthy. Kẻ tấn công hoặc bất kỳ ai scan cổng/endpoint đều có thể gửi request với header `X-API-Key: changeme` để gọi API `/ask`, lợi dụng dịch vụ và đốt sạch ngân sách/hạn ngạch LLM của bạn. Bạn chỉ phát hiện ra khi đã nhận hóa đơn tiền triệu hoặc hệ thống bị cạn kiệt tài nguyên.
- Khi không có giá trị mặc định (Fail Fast): Ứng dụng sẽ ném `ValidationError` và crash ngay lúc khởi động, nền tảng cloud sẽ giữ phiên bản cũ và báo lỗi deploy ngay lập tức. Điều này buộc lập trình viên phải bổ sung secret hợp lệ trước khi traffic công khai chạm được vào hệ thống.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thực tế thu được từ stdout khi gọi `/ask`:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:36:53.205408+00:00", "user_id": "sv-test", "tokens_in": 1, "tokens_out": 33, "cost_usd": 1.995e-05}
```

Hai việc làm được với log JSON có cấu trúc (Structured Logging) mà `print("đã trả lời xong")` không làm được:
1. **Lọc và truy vấn nâng cao theo trường (Querying & Filtering)**: Hệ thống giám sát log tập trung (như Datadog, CloudWatch, Loki, Elasticsearch) có thể bóc tách JSON tự động để thực hiện lọc các request của một `user_id` cụ thể, hoặc lọc ra các request có `cost_usd > 0.05` hay phát sinh trong một khoảng `timestamp` xác định. Với `print()` chuỗi văn bản thuần túy, log parser không thể bóc tách chính xác các trường số liệu này nếu không dùng regex phức tạp và dễ vỡ.
2. **Tổng hợp số liệu & Thiết lập cảnh báo tự động (Metrics Aggregation & Alerting)**: Máy có thể chạy các truy vấn tính toán như `SUM(cost_usd)` theo từng user hoặc tính tổng `tokens_out` theo giờ để vẽ biểu đồ chi phí thời gian thực, đồng thời tự động kích hoạt cảnh báo (Alert) gửi về Slack/PagerDuty khi `SUM(cost_usd)` của một người dùng vượt quá ngưỡng an toàn.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~749 MB) gồm có:
1. **Hệ điều hành nền và công cụ hệ thống không cần thiết**: Base image `python:3.11` đầy đủ dựa trên Debian bản chuẩn chứa toàn bộ compiler (gcc, g++), make, header files (`build-essential`), thư viện đồ họa, tài liệu man pages, và hàng trăm gói tiện ích hệ thống. Trong khi đó, `python:3.11-slim` đã được lược bỏ tối đa các gói này, chỉ giữ lại runtime tối thiểu cần để chạy Python.
2. **Rác phát sinh trong quá trình build**: Ở bản 1-stage, quá trình `pip install` để lại cache wheels, build artifacts, bytecode tạm thời và các công cụ đóng gói. Ở bản Multi-stage, toàn bộ dependency được cài đặt vào thư mục đích (`/install`) trong stage `builder`, stage `runtime` chỉ copy phần kết quả thư viện cần dùng sang (`COPY --from=builder /install /usr/local`), loại bỏ hoàn toàn các layer trung gian, cache của pip và các file tạm.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
  - Các layer trước đó bao gồm: `FROM python:3.11-slim`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install ...`, và `RUN useradd ...` đều được dùng lại hoàn toàn từ Docker cache (CACHED) vì file `requirements.txt` không hề thay đổi.
  - Chỉ có layer `COPY . .` và các lệnh phía sau nó mới phải chạy lại. Nhờ đó, việc build lại chỉ mất chưa đầy 1 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Mỗi khi sửa dù chỉ một ký tự trong `app/main.py`, checksum của layer `COPY . .` sẽ bị thay đổi.
  - Theo cơ chế invalidate cache của Docker, layer `COPY . .` và tất cả các layer bên dưới nó sẽ bị mất cache.
  - Hệ quả là Docker sẽ buộc phải chạy lại toàn bộ lệnh `RUN pip install -r requirements.txt`, tải và cài đặt lại tất cả thư viện từ đầu ở mỗi lần build, khiến quá trình build và CI/CD bị kéo dài hàng phút một cách lãng phí.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện tấn công khi chạy bằng root:
  1. Ứng dụng Python có một lỗ hổng thực thi mã từ xa (RCE) hoặc lỗi ghi đè file tùy ý (arbitrary file write / path traversal).
  2. Kẻ tấn công khai thác lỗ hổng để thực thi lệnh shell bên trong container. Do container chạy không khai báo `USER`, tiến trình Python chạy với quyền `UID 0 (root)`.
  3. Từ quyền root trong container, kẻ tấn công có toàn quyền truy cập các socket được mount (như `/var/run/docker.sock`), các volume mount từ host, hoặc khai thác lỗ hổng kernel của hệ điều hành host (container chia sẻ chung kernel với host) để thực hiện kỹ thuật Container Escape (thoát khỏi container).
  4. Sau khi thoát ra ngoài host, vì tiến trình ban đầu là `UID 0`, kẻ tấn công nghiễm nhiên sở hữu quyền `root` tối cao trên máy host, có thể đọc trộm toàn bộ dữ liệu, cài mã độc, hoặc chiếm quyền kiểm soát máy chủ vật lý.
- Lệnh `USER` cắt đứt chuỗi đó:
  - Lệnh `USER appuser` (với UID 10001 không có quyền root) cắt đứt ngay tại bước 2 và 3. Khi kẻ tấn công khai thác được ứng dụng, họ chỉ có quyền của một user thông thường (`appuser`). Tiến trình bị hạn chế tối đa: không thể ghi vào các thư mục hệ thống trong container, không thể can thiệp vào các device/socket đặc quyền, và nếu có cố gắng thoát container thì trên host họ cũng chỉ có quyền của một user không đặc quyền (unprivileged user), ngăn chặn triệt để nguy cơ chiếm quyền điều khiển máy chủ.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Người dùng có thể gửi tối đa **20 request trong vòng 2 giây liên tiếp** (gấp đôi hạn mức quy định).
- Giải thích:
  - Với cơ chế đếm theo phút đồng hồ (Fixed Window Counter reset ở giây `:00` của mỗi phút):
    - Ở giây `10:00:59` (giây cuối cùng của phút thứ nhất), người dùng gửi dồn dập toàn bộ hạn ngạch cho phép là 10 request. Hệ thống đếm đủ 10 và hợp lệ.
    - Ngay 1 giây sau, đồng hồ điểm `10:01:00` (bắt đầu phút mới), bộ đếm bị reset về 0.
    - Tại giây `10:01:00`, người dùng lập tức gửi tiếp 10 request nữa. Bộ đếm ghi nhận 10 request cho phút mới và vẫn cho qua hợp lệ.
  - Kết quả: Trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ `10:00:59` đến `10:01:00`), hệ thống đã phải gánh tới `10 + 10 = 20 request`, tạo ra một đợt bùng nổ lưu lượng (traffic spike) có thể làm nghẽn hoặc sập server downstream/LLM, dù trên danh nghĩa người dùng "không vi phạm" luật 10 req/phút của từng khung giờ. Thuật toán Sliding Window (cửa sổ trượt) giải quyết triệt để lỗi này bằng cách xét liên tục 60 giây lùi về trước tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Sự khác nhau:
  - **Rate Limiter**: Giới hạn tần suất / số lượng request trong một đơn vị thời gian ngắn (ví dụ: tối đa 10 request/phút) nhằm chống DDoS, chống spam và tránh làm quá tải tài nguyên tính toán/băng thông của server.
  - **Cost Guard**: Giới hạn tổng chi phí tài chính (USD) trong một khoảng thời gian dài (ví dụ: tối đa 10.0 USD/tháng) dựa trên lượng token LLM thực tế tiêu thụ, nhằm bảo vệ ngân sách và tránh việc tài khoản bị cạn kiệt tiền ngoài ý muốn.
- Tình huống Rate Limit cho qua nhưng Cost Guard chặn:
  - Một user chỉ gửi đúng 1 request trong 10 phút (tần suất cực thấp, hoàn toàn nằm trong hạn mức 10 req/phút của Rate Limiter). Tuy nhiên, trong tháng đó user này đã sử dụng hết 9.98 USD ngân sách, và câu hỏi mới kèm prompt quá dài dự kiến tiêu tốn 0.05 USD khiến tổng vượt quá 10.0 USD. Khi đó Rate Limiter cho qua nhưng Cost Guard sẽ chặn với mã lỗi `402 Payment Required`.
- Tình huống Cost Guard cho qua nhưng Rate Limit chặn:
  - Một user mới toanh trong tháng chưa tiêu đồng nào (ngân sách còn nguyên 10.0 USD). Tuy nhiên user này dùng script gửi liên tục 15 request chỉ trong vòng 5 giây với các câu hỏi rất ngắn (mỗi câu chỉ tốn $0.0001). Khi đó Cost Guard thấy ngân sách còn rất nhiều nên đủ điều kiện, nhưng Rate Limiter sẽ lập tức chặn từ request thứ 11 với mã lỗi `429 Too Many Requests` vì vi phạm tần suất gọi quá nhanh.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự các sự kiện xảy ra:
1. **Redis gặp sự cố**: Redis bị nghẽn mạng hoặc khởi động lại, mất kết nối trong 30 giây.
2. **Probe bị fail đồng loạt**: Bộ giám sát (Docker daemon, Kubernetes kubelet hoặc Cloud orchestrator) định kỳ gọi endpoint health check của cả 3 container agent. Do endpoint này kiểm tra Redis và Redis đang chết, cả 3 container đều đồng loạt trả về lỗi 500/503 hoặc timeout.
3. **Orchestrator tiêu diệt cả cụm (Restart storm)**: Vì liveness probe thất bại, orchestrator kết luận rằng chính bản thân ứng dụng Python trong các container đã bị treo/hỏng. Nó lập tức gửi `SIGTERM`/`SIGKILL` để khởi động lại (restart) toàn bộ 3 container.
4. **Hệ thống rơi vào vòng xoáy sập hoàn toàn (Cascading failure)**: Các container mới khởi động lên lại tiếp tục check Redis, Redis vẫn chưa phục hồi trong khoảng 30s đó, nên các container mới lại bị coi là hỏng và tiếp tục bị restart liên tục.
5. **Request của người dùng bị gián đoạn toàn bộ**: Khi Redis phục hồi xong, các container vẫn đang bận trong chu kỳ restart và khởi tạo lại, tạo thêm tải dồn dập vào Redis vừa sống dậy, khiến toàn bộ hệ thống bị downtime kéo dài thay vì chỉ tạm dừng nhận request mới và phục vụ bình thường khi Redis sẵn sàng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless - hệ thống hiện tại):
  - Giá trị `history_length` tăng đều đặn và liên tục theo số lượt hội thoại: `0 -> 2 -> 4 -> 6 -> ...` bất kể request được bộ cân bằng tải phân bổ tới container nào trong số 3 container, vì cả 3 container đều đọc và ghi dữ liệu lịch sử chung trên Redis.
- Khi lưu trong biến dict Python trong bộ nhớ RAM của process (Stateful):
  - Khi scale lên 3 instance (Container 1, 2, 3), mỗi container sở hữu một vùng nhớ RAM hoàn toàn cô lập.
  - Khi load balancer điều hướng request theo thuật toán Round-Robin hoặc Least Connections:
    - Request 1 vào Container 1: `history_length` trả về `0`.
    - Request 2 vào Container 2: Do Container 2 chưa từng gặp user này, `history_length` vẫn trả về `0` thay vì `2` (agent bị mất ngữ cảnh).
    - Request 3 vào Container 3: `history_length` lại tiếp tục là `0`.
    - Request 4 quay lại Container 1: `history_length` nhảy lên `2` (chỉ nhớ câu hỏi từ request 1, bỏ qua request 2 và 3).
  - Kết quả là `history_length` sẽ thay đổi hỗn loạn, trồi sụt thất thường tùy thuộc vào request rơi ngẫu nhiên vào container nào, khiến ngữ cảnh hội thoại của người dùng bị phân mảnh và sai lệch hoàn toàn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**:
  `Application failed to respond on port 8000. Health check timed out after 60s.` (hoặc trên Render/Railway: `Container failed to start: Port binding failed`).
- **Cách tìm ra nguyên nhân**:
  - Mở tab **Runtime Logs / Deploy Logs** trên dashboard của platform.
  - Thấy dòng log thông báo Uvicorn đang khởi động và lắng nghe cố định tại: `Uvicorn running on http://0.0.0.0:8000`.
  - Tuy nhiên, biến môi trường của platform cloud tự động gán một cổng ngẫu nhiên cho container thông qua biến `$PORT` (ví dụ: `PORT=10000` trên Render hoặc một cổng ngẫu nhiên trên Railway).
  - Do orchestrator của platform thực hiện health check bằng cách gửi request vào `$PORT` do nó cấp phát, trong khi container chỉ mở cổng 8000 nên probe không nhận được phản hồi và timeout.
- **Cách sửa**:
  - Cập nhật lệnh khởi chạy trong Dockerfile để đọc giá trị từ biến môi trường `$PORT` thay vì gán cứng 8000:
    `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  - Đồng thời trong `app/config.py`, trường `port: int = 8000` đã được Pydantic tự động đọc ghi đè từ biến môi trường `PORT` nếu có. Sau khi sửa, container lắng nghe đúng cổng của platform và health check chuyển sang màu xanh ngay lập tức.
